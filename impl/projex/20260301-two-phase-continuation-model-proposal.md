# Two-Phase Continuation Model

> **Created:** 2026-03-01
> **Author:** agent
> **Status:** Draft
> **Impact:** `impl/09_runtime.md`, `impl/08_concurrency.md`, `csharp/src/Zoh.Runtime/`
> **Related Projex:** `20260301-continuation-resume-ip-gap-eval.md`

---

## Problem

The current continuation model is **one-phase**: a driver returns a `Continuation` describing what it's waiting for, then the **runtime scheduler** handles everything that happens after the wait is fulfilled — setting `lastReturnValue`, copying inline variables, producing diagnostics, etc. This creates three problems:

1. **IP advancement gap** — The instruction pointer is never advanced after yield, causing infinite re-execution of blocking verbs (see related eval).

2. **Post-resume logic is in the wrong place** — The tick loop contains verb-specific logic (inline var copying for `/call`, timeout diagnostics for `/pull`, etc.) that belongs in the verb driver. The scheduler knows too much about verb semantics.

3. **Extensions can't block** — The `Continuation` type is a closed union and `ContextState` is a closed enum. An extension verb driver cannot define new blocking conditions without modifying the runtime core.

---

## Proposal

Replace the one-phase `ExecutionResult`/`Continuation` contract with a **two-phase model** where the driver owns both its pre-yield *and* post-resume logic. The runtime becomes a pure condition fulfiller — it knows how to wait for things, but not what verb semantics mean.

---

## Design

### Core Types

```
VerbDriver:
  namespace: string?
  name: string
  priority: int

  execute(call: CompiledVerbCall, context: Context): DriverResult

# === DRIVER RESULT ===
# Discriminated union: a verb either completes or suspends.
# This replaces the old ExecutionResult.

DriverResult:
  | Complete {
      value: Value                    # Return value (becomes lastReturnValue)
      diagnostics: List<Diagnostic>
    }
  | Suspend {
      continuation: Continuation      # What to wait for + what to do after
      diagnostics: List<Diagnostic>   # Pre-yield diagnostics (optional)
    }

# === CONTINUATION ===
# Pairs a wait request with a resume handler.
# The runtime fulfills the request; the driver handles the outcome.
# This replaces the old flat Continuation union.

Continuation:
  request: WaitRequest                # WHAT the driver is waiting for
  onFulfilled: (WaitOutcome) -> DriverResult   # WHAT to do when fulfilled

# === WAIT REQUEST ===
# Describes the external condition. The runtime maps these to scheduling.
# Drivers MUST NOT set context.state or waitCondition directly.

WaitRequest:
  | Sleep       { durationMs: double }
  | ChannelPull { channelName: string, generation: int, timeoutMs: double? }
  | ChannelPush { channelName: string, seqNum: int, generation: int, timeoutMs: double? }
  | Signal      { messageName: string, timeoutMs: double? }
  | JoinContext { contextId: string }
  | Host        { timeoutMs: double? }

# === WAIT OUTCOME ===
# What actually happened. Produced by the runtime, consumed by onFulfilled.

WaitOutcome:
  | Completed { value: Value }        # Condition met, here's the result
  | TimedOut                          # Deadline expired
  | Cancelled { code: string, message: string }  # Channel closed, context gone, etc.
```

### Design Rationale

**Why `DriverResult` is a discriminated union (not a struct with optional fields):**

The old `ExecutionResult { value, diagnostics, continuation? }` was ambiguous: is `value` meaningful when `continuation != null`? With `Complete` vs `Suspend`, the contract is explicit — `Complete` carries the final value, `Suspend` carries the continuation. There is no ambiguous middle state.

**Why `onFulfilled` returns `DriverResult` (not just `Value`):**

This enables **chaining**. A resume handler can return another `Suspend` to wait for something else without re-entering the execution loop. Example: a hypothetical verb that pulls from a channel, then sleeps before returning the result. The execution loop handles chaining transparently — it keeps calling `onFulfilled` until it gets a `Complete`.

**Why `WaitOutcome` is a union (not just a value):**

Drivers need to distinguish success from timeout from cancellation to produce correct return values and diagnostics. The old model pushed this distinction into the tick loop, which then had to know what each verb expected. With `WaitOutcome`, the runtime says "here's what happened" and the driver decides what it means.

### Execution Loop

```
Context:
  instructionPointer: int
  state: ContextState
  pendingContinuation: Continuation?    # Stored while blocked
  waitCondition: WaitCondition?         # For scheduler to check
  resumeToken: int                      # Incremented on each Suspend; callers must match

  run():
    while state == RUNNING:
      if instructionPointer >= currentStory.statements.length:
        terminate()
        return

      statement = currentStory.statements[instructionPointer]

      if statement is CompiledCheckpoint:
        validateContract(statement)
        instructionPointer++
        continue

      call = statement as CompiledVerbCall
      driver = runtime.verbDrivers.resolveBySuffix(call.qualifiedName)

      if driver == null:
        runtime.log(WARN, "Unknown verb: " + call.qualifiedName)
        instructionPointer++
        continue

      # Capture state before execution to detect jumps
      entryIp = instructionPointer
      entryStory = currentStory

      try:
        result = driver.execute(call, this)
      catch FatalError as e:
        lastDiagnostics = [Diagnostic(FATAL, e.message)]
        terminate()
        return

      applyResult(result, entryIp, entryStory)

  # Unified result handler — used by both run() and resume().
  # entryIp/entryStory: IP and story before driver execution.
  # IP only advances if they match current values (jump guard).
  applyResult(result: DriverResult, entryIp: int, entryStory: CompiledStory):
    match result:
      Complete { value, diagnostics }:
        lastReturnValue = value
        lastDiagnostics = diagnostics
        if hasFatal(diagnostics):
          terminate()
        elif instructionPointer == entryIp and currentStory == entryStory:
          instructionPointer++
          # run() loop continues to next statement

      Suspend { continuation, diagnostics }:
        lastDiagnostics = diagnostics
        resumeToken++
        pendingContinuation = continuation
        blockOnRequest(continuation.request)
        # state is now SLEEPING/WAITING_*
        # run() loop exits (state != RUNNING)

  # Called by the scheduler OR the host when the wait condition is met.
  #
  # token: must match the current resumeToken. Rejects stale resumes
  #   (e.g., host responds after timeout already advanced the context).
  #   The scheduler reads context.resumeToken when it detects fulfillment.
  #   The host receives it via the handler callback (onChoose, onConverse, etc.).
  resume(outcome: WaitOutcome, token: int):
    if token != resumeToken:
      return    # Stale token — context has moved on
    if pendingContinuation == null:
      return    # Already resumed (race loser on same token) — no-op

    handler = pendingContinuation.onFulfilled
    pendingContinuation = null
    waitCondition = null

    result = handler(outcome)
    state = RUNNING
    applyResult(result, instructionPointer, currentStory)
    # If Complete → IP advances, run() can continue
    # If Suspend  → re-blocks, IP stays, new continuation stored

  # Maps a WaitRequest to internal scheduler state.
  # Same role as old block(), but thinner — no verb logic.
  blockOnRequest(request: WaitRequest):
    match request:
      Sleep { durationMs }:
        state = SLEEPING
        waitCondition = SleepCondition { wakeTime: now() + durationMs }

      ChannelPull { channelName, generation, timeoutMs }:
        hub = runtime.channelHubs.get(channelName)
        hub.waitingPullers.enqueue(this)
        state = WAITING_CHANNEL
        waitCondition = ChannelWaitCondition {
          channelName, generation, startTime: now(), timeout: timeoutMs
        }

      ChannelPush { channelName, seqNum, generation, timeoutMs }:
        hub = runtime.channelHubs.get(channelName)
        hub.waitingPushers.enqueue((this, seqNum))
        state = WAITING_CHANNEL_PUSH
        waitCondition = ChannelPushWaitCondition {
          channelName, seq: seqNum, generation, startTime: now(), timeout: timeoutMs
        }

      Signal { messageName, timeoutMs }:
        runtime.signals.subscribe(messageName, this)
        state = WAITING_MESSAGE
        waitCondition = MessageWaitCondition {
          messageName, startTime: now(), timeout: timeoutMs
        }

      JoinContext { contextId }:
        state = WAITING_CONTEXT
        waitCondition = ContextWaitCondition { targetContextId: contextId }

      Host { timeoutMs }:
        state = WAITING_HOST
        waitCondition = HostWaitCondition {
          startTime: now(), timeout: timeoutMs
        }
```

### Tick-Loop Scheduler

The scheduler becomes a pure **condition resolver**. It checks each blocked context, determines if the condition is met, and if so produces a `WaitOutcome`. It no longer contains any verb-specific logic.

```
Runtime.tick():
  for context in contexts:
    if context.state != RUNNING and context.state != TERMINATED:
      token = context.resumeToken
      outcome = resolveWait(context)
      if outcome != null:
        context.resume(outcome, token)

    if context.state == RUNNING:
      context.run()

resolveWait(context: Context): WaitOutcome?
  match context.state:
    SLEEPING:
      if now() >= context.waitCondition.wakeTime:
        return Completed { Nothing }
      return null

    WAITING_CHANNEL:
      hub = runtime.channelHubs.get(context.waitCondition.channelName)
      if hub == null:
        return Cancelled { "not_found", "Channel not found" }
      if hub.closed:
        return Cancelled { "closed", "Channel closed" }
      if context.waitCondition.isTimedOut():
        hub.waitingPullers.remove(context)
        return TimedOut
      # Value delivery handled by PushDriver fast path (see below)
      return null

    WAITING_CHANNEL_PUSH:
      hub = runtime.channelHubs.get(context.waitCondition.channelName)
      if hub == null:
        return Cancelled { "not_found", "Channel not found" }
      if hub.closed:
        return Cancelled { "closed", "Channel closed" }
      if context.waitCondition.isTimedOut():
        hub.waitingPushers.remove(context, context.waitCondition.seq)
        return TimedOut
      return null

    WAITING_CONTEXT:
      target = contexts.find(c => c.id == context.waitCondition.targetContextId)
      if target == null or target.state == TERMINATED:
        return Completed { target?.lastReturnValue ?? Nothing }
      return null

    WAITING_MESSAGE:
      if context.waitCondition.isFulfilled():
        runtime.signals.unsubscribe(context.waitCondition.messageName, context)
        return Completed { context.waitCondition.payload }
      if context.waitCondition.isTimedOut():
        runtime.signals.unsubscribe(context.waitCondition.messageName, context)
        return TimedOut
      return null

    WAITING_HOST:
      # Host-driven: the scheduler does NOT resolve this.
      # The host calls context.resume(outcome) directly.
      # Scheduler only handles timeout if configured.
      if context.waitCondition.isTimedOut():
        return TimedOut
      return null

  return null
```

**Host resume path:** For `WAITING_HOST`, the runtime scheduler does not resolve the condition — the host application does, by calling `context.resume(outcome)` directly. This is the primary integration point for presentation verbs (`/converse`, `/choose`, `/prompt`, etc.) where the host UI collects input and feeds it back.

```
# Host-side (game engine, UI framework, etc.):
# The host handler receives the resumeToken alongside the request.
# It must pass it back when resuming — stale tokens are rejected.

onChoose(context, request):
  token = context.resumeToken
  displayChoiceUI(request, onSelected: (value) ->
    context.resume(Completed { value }, token)
  )

onConverse(context, request):
  token = context.resumeToken
  displayDialog(request, onDismissed: () ->
    context.resume(Completed { Nothing }, token)
  )
```

### Async Task Scheduler

The async model maps naturally — `resolveWait` becomes an awaitable:

```
async runContextAsync(context: Context):
  while context.state != TERMINATED:
    context.run()

    if context.pendingContinuation != null:
      outcome = await fulfillAsync(context.pendingContinuation.request)
      context.resume(outcome)

async fulfillAsync(request: WaitRequest): WaitOutcome
  match request:
    Sleep { durationMs }       -> await asyncSleep(durationMs); return Completed { Nothing }
    ChannelPull { ... }        -> return await channel.pullAsync(timeout)  # returns Completed/TimedOut/Cancelled
    ChannelPush { ... }        -> return await channel.awaitConsumedAsync(seq, timeout)
    Signal { messageName }     -> return await signals.waitAsync(messageName, timeout)
    JoinContext { contextId }  -> return Completed { await contexts.awaitTerminationAsync(contextId) }
    Host { ... }               -> return await host.awaitInteractionAsync(timeout)
```

---

## Rewritten Drivers

### Sleep — Trivial (no post-resume logic)

```
SleepDriver.execute(call, context):
  seconds = resolve(call.params[0], context).toDouble()
  return Suspend {
    continuation: Continuation {
      request: Sleep { durationMs: seconds * 1000 },
      onFulfilled: (_) -> Complete { Nothing, [] }
    }
  }
```

### Pull — Post-resume transforms outcome to value

```
PullDriver.execute(call, context):
  channelRef = resolve(call.params[0], context)
  channelName = channelRef.name
  hub = runtime.channelHubs.get(channelName)

  if hub == null:
    return Complete { Nothing, [Diagnostic(ERROR, "not_found", "Channel not found")] }

  # Fast path: value already available
  if context.inboxes[channelName].isNotEmpty():
    value = context.inboxes[channelName].dequeue()
    return Complete { value, [] }

  # Slow path: suspend
  timeout = resolveNamedParam(call, "timeout")?.toDouble()
  return Suspend {
    continuation: Continuation {
      request: ChannelPull {
        channelName, generation: hub.generation,
        timeoutMs: timeout ? timeout * 1000 : null
      },
      onFulfilled: (outcome) ->
        match outcome:
          Completed { value }:
            Complete { value, [] }
          TimedOut:
            Complete { Nothing, [Diagnostic(INFO, "timeout", "Pull timed out")] }
          Cancelled { code, message }:
            Complete { Nothing, [Diagnostic(ERROR, code, message)] }
    }
  }
```

### Call — Post-resume handles inline variable copying

This is the clearest demonstration of why two-phase matters. The old model had inline var logic in the tick loop. Now it lives in the driver where it belongs.

```
CallDriver.execute(call, context):
  storyPath = resolve(call.params[0], context)
  labelName = resolve(call.params[1], context).toString()
  initVars = call.unnamedParams.slice(2)
  shouldClone = hasAttribute(call, "clone")
  shouldInline = hasAttribute(call, "inline")

  # Resolve target
  targetStory = storyPath.isNothing()
    ? context.currentStory
    : context.runtime.loadStory(storyPath.toString())
  if targetStory == null:
    return Complete { Nothing, [Diagnostic(FATAL, "invalid_story", "...")] }

  labelIndex = targetStory.labels.get(labelName.toLowerCase())
  if labelIndex == null:
    return Complete { Nothing, [Diagnostic(FATAL, "invalid_label", "...")] }

  # Fork child context
  newContext = shouldClone ? context.clone() : Context.new(context.runtime)
  for ref in initVars:
    newContext.set(ref.name, context.get(ref.name))
  newContext.currentStory = targetStory
  newContext.instructionPointer = labelIndex
  context.runtime.addContext(newContext)
  context.runtime.scheduleContext(newContext)

  # Capture references for the closure
  inlineVars = shouldInline ? initVars : []
  childId = newContext.id

  return Suspend {
    continuation: Continuation {
      request: JoinContext { contextId: childId },
      onFulfilled: (outcome) ->
        match outcome:
          Completed { value }:
            # Inline: copy vars back from child to parent
            for ref in inlineVars:
              childCtx = context.runtime.findContext(childId)
              val = childCtx?.get(ref.name) ?? Nothing
              context.set(ref.name, val)
            Complete { value, [] }
          Cancelled { code, message }:
            Complete { Nothing, [Diagnostic(ERROR, code, message)] }
    }
  }
```

### Wait — Post-resume just unwraps

```
WaitDriver.execute(call, context):
  messageName = resolve(call.params[0], context).toString()
  timeout = resolveNamedParam(call, "timeout")?.toDouble()

  return Suspend {
    continuation: Continuation {
      request: Signal { messageName, timeoutMs: timeout ? timeout * 1000 : null },
      onFulfilled: (outcome) ->
        match outcome:
          Completed { value }:
            Complete { value, [] }
          TimedOut:
            Complete { Nothing, [Diagnostic(INFO, "timeout", "Wait timed out")] }
          Cancelled { code, message }:
            Complete { Nothing, [Diagnostic(ERROR, code, message)] }
    }
  }
```

### Push (waited) — Post-resume confirms delivery

```
PushDriver.execute(call, context):
  channelRef = resolve(call.params[0], context)
  value = resolve(call.params[1], context)
  shouldWait = hasAttribute(call, "wait")
  channelName = channelRef.name
  hub = runtime.channelHubs.get(channelName)

  if hub == null:
    return Complete { Nothing, [Diagnostic(ERROR, "not_found", "Channel not found")] }

  # Enqueue in outbox
  seqNum = context.outboxes[channelName].nextSeq()
  context.outboxes[channelName].enqueue((value, seqNum))

  if not shouldWait:
    return Complete { Nothing, [] }

  # Waited push: suspend until consumed
  return Suspend {
    continuation: Continuation {
      request: ChannelPush {
        channelName, seqNum, generation: hub.generation,
        timeoutMs: resolveNamedParam(call, "timeout")?.toDouble()
      },
      onFulfilled: (outcome) ->
        match outcome:
          Completed { _ }:
            Complete { Nothing, [] }
          TimedOut:
            Complete { Nothing, [Diagnostic(INFO, "timeout", "Push timed out")] }
          Cancelled { code, message }:
            Complete { Nothing, [Diagnostic(ERROR, code, message)] }
    }
  }
```

### Choose — Host interaction with post-resume validation

This demonstrates why `Host` is a first-class `WaitRequest`, not a workaround. The driver builds a choice list, yields to the host for UI presentation, then validates and maps the host's response on resume.

```
ChooseDriver.execute(call, context):
  speaker = resolveAttribute(call, "by")
  prompt = resolveNamedParam(call, "prompt")
  timeout = resolveNamedParam(call, "timeout")?.toDouble()

  # Build visible choices (evaluate visibility conditions)
  choices = []
  for i in range(0, call.unnamedParams.length, 3):
    visible = resolve(call.params[i], context).isTruthy()
    if not visible: continue
    text = resolve(call.params[i + 1], context).toString()
    value = resolve(call.params[i + 2], context)
    choices.append({ text, value, index: i })

  if choices.isEmpty():
    return Complete { Nothing, [] }

  # Notify host handler (display the UI)
  runtime.hostHandler.onChoose(context, ChooseRequest { speaker, prompt, timeout, choices })

  # Yield — host calls context.resume() when player picks
  return Suspend {
    continuation: Continuation {
      request: Host { timeoutMs: timeout ? timeout * 1000 : null },
      onFulfilled: (outcome) ->
        match outcome:
          Completed { value }:
            # Post-resume: validate the host's response
            # The host returns the selected value directly.
            # Driver can transform, validate, or log here.
            Complete { value, [] }
          TimedOut:
            Complete { Nothing, [Diagnostic(INFO, "timeout", "Choose timed out")] }
          Cancelled { code, message }:
            Complete { Nothing, [Diagnostic(ERROR, code, message)] }
    }
  }
```

### Converse — Host interaction with optional wait

```
ConverseDriver.execute(call, context):
  speaker = resolveAttribute(call, "by")
  shouldWait = resolveAttribute(call, "wait") ?? context.get("interactive") ?? true
  contents = resolveAllUnnamed(call, context)

  if contents.isEmpty():
    return Complete { Nothing, [] }

  runtime.hostHandler.onConverse(context, ConverseRequest { speaker, contents, ... })

  if not shouldWait:
    return Complete { Nothing, [] }    # Fire-and-forget, no yield

  # Yield for host acknowledgment (player dismisses dialog)
  return Suspend {
    continuation: Continuation {
      request: Host {},
      onFulfilled: (_) -> Complete { Nothing, [] }
    }
  }
```

---

## IP Lifecycle Summary

The instruction pointer question resolves cleanly under this model:

```
                    driver.execute()
                          │
                    ┌─────┴─────┐
                    │           │
                Complete    Suspend
                    │           │
               IP++, next   store continuation,
               statement    block, run() exits
                                │
                            ┌───┴──── ... scheduler waits ...
                            │
                        resume(outcome)
                            │
                      onFulfilled(outcome)
                            │
                    ┌───────┴───────┐
                    │               │
                Complete        Suspend (chain)
                    │               │
               IP++, next       re-block,
               statement        wait again
```

**IP only advances on `Complete`, and only when the driver hasn't already modified it** (jump guard). A verb that suspends keeps IP at its statement until the entire continuation chain resolves. Jump verbs modify IP directly and return `Complete` — the guard detects this and skips the automatic increment.

---

## What Changes

| Component | Before | After |
|-----------|--------|-------|
| `ExecutionResult` | Flat struct with optional `continuation` | `DriverResult` discriminated union: `Complete` / `Suspend` |
| `Continuation` | Closed union of wait types (data only) | Struct with `request: WaitRequest` + `onFulfilled` callback |
| `Context.block()` | Sets state + waitCondition + verb-specific logic | Renamed `blockOnRequest()`: sets state + waitCondition only |
| `Runtime.tick()` | Resolves waits AND applies verb-specific post-resume logic | Resolves waits only, calls `context.resume(outcome)` |
| `Context` (new) | — | Gains `pendingContinuation`, `resume()`, `applyResult()` |
| IP advancement | Missing for blocking path | Handled uniformly in `applyResult()` on `Complete` |
| `HostContinuation` | Opaque tag, no post-resume driver logic | `Host` WaitRequest + `onFulfilled` for validation/transform |
| Extension verbs | Cannot define new blocking conditions | Can block via any `WaitRequest`, own their resume logic |

---

## Considerations

### Closure Lifetime

`onFulfilled` is a closure that lives as long as the context is blocked. In ZOH, contexts are transient runtime objects — they are not serialized for save/load (persistence uses `/write`/`/read` at the variable level). The closure is discarded when the context terminates. No serialization concern.

### Closure Captures

Resume handlers may capture references to the context, the runtime, or local driver state. This is by design — the handler runs in the same process, same thread (or scheduled coroutine). Implementors should be mindful that captured references must remain valid for the duration of the block (no dangling references to disposed objects).

### Chaining Depth

A resume handler that returns `Suspend` creates a chain. In practice, built-in verbs are single-phase (one wait, one resume). Chaining exists for future extensibility (e.g., a verb that retries on timeout). No stack depth concern — each link is a tail call replacing the previous continuation, not nesting.

### ContextState Enum

The `ContextState` enum (`SLEEPING`, `WAITING_CHANNEL`, etc.) is retained for the scheduler's benefit — it needs to know *how* to check the condition. But it is set exclusively by `blockOnRequest()` and read exclusively by `resolveWait()`. Verb drivers never see it. If an extension needs a new wait type, it would require a new `WaitRequest` variant and corresponding `ContextState` + `resolveWait` case. This is a deliberate chokepoint — new wait *mechanisms* are rare; new wait *behaviors* (what to do on resume) are handled entirely by `onFulfilled`.

### Host-Defined Wait Conditions

`Host { timeoutMs }` is a core `WaitRequest` variant. The scheduler only checks timeout; fulfillment is the host's responsibility via `context.resume(outcome)`. The host already knows what interaction is happening because the driver calls the host handler (e.g., `onChoose()`, `onConverse()`) *before* yielding — no tag needed.

This covers all presentation verbs (`/converse`, `/choose`, `/prompt`, `/choosefrom`) and any future host-specific interactions without requiring new `WaitRequest` variants or `ContextState` entries. The host can also use the `/wait` + `/signal` pattern for fire-and-forget events. `Host` is for synchronous request/response interactions where the driver needs the host's answer before it can complete.

---

## Approach: Adoption Path

### 1. Spec Update

Update `impl/09_runtime.md`:
- Replace `ExecutionResult` / `Continuation` type definitions with `DriverResult` / `Continuation` / `WaitRequest` / `WaitOutcome`
- Replace `Context.run()` with the new execution loop using `applyResult()`
- Add `Context.resume()` and `Context.blockOnRequest()`
- Add `Context.pendingContinuation` field
- Replace `Runtime.tick()` with condition-only resolver + `context.resume()`
- Update async model section symmetrically

Update `impl/08_concurrency.md`:
- Rewrite `CallDriver`, `ForkDriver` (fork doesn't block, no change needed), and all channel drivers to return `DriverResult`

### 2. C# Implementation

Update `csharp/src/Zoh.Runtime/`:
- New types: `DriverResult`, `WaitRequest`, `WaitOutcome`, `Continuation` (with `Func<WaitOutcome, DriverResult>`)
- Update `Context.cs`: add `PendingContinuation`, `Resume()`, `ApplyResult()`, rename `Block()` to `BlockOnRequest()`
- Update all verb drivers to return `DriverResult` with `onFulfilled` closures
- Remove verb-specific logic from any scheduler code

---

## Open Questions

- [ ] Should `WaitRequest` be extensible (open union / interface) for host-defined wait types, or stay closed with a `HostAction` escape hatch?
- [ ] Should `onFulfilled` receive the `Context` as a parameter, or should it close over it? (Passing explicitly is more testable; closing over it is more ergonomic.)

---

## Appendix

### Sources
- `impl/09_runtime.md` — Current execution loop, blocking, tick loop
- `impl/08_concurrency.md` — `/call`, `/fork`, channel drivers
- `csharp/src/Zoh.Runtime/Execution/Context.cs` — Current C# implementation
- `20260301-continuation-resume-ip-gap-eval.md` — Gap analysis that motivated this proposal
