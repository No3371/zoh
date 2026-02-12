# Impl Spec Channel Inbox/Outbox Rewrite Plan

> **Status:** Complete
> **Completed:** 2026-02-12
> **Walkthrough:** [20260212-channel-inbox-outbox-impl-spec-walkthrough.md](impl/projex/closed/20260212-channel-inbox-outbox-impl-spec-walkthrough.md)
> **Created:** 2026-02-11
> **Author:** Agent
> **Source:** `20260211-channel-inbox-outbox-proposal.md`, `20260211-channel-inbox-outbox-redteam.md`
> **Related Projex:** `20260211-channel-inbox-outbox-spec-plan.md` (spec-scope sibling)

---

## Summary

Rewrite the channel sections of `impl/08_concurrency.md` and `impl/09_runtime.md` to implement the inbox/outbox hub architecture: replace the global `ChannelManager` with a `ChannelHubRegistry`, add outbox/inbox fields to Context, rewrite all channel verb driver pseudocode (Open, Push, Pull, Close), add context termination cleanup, and update the scheduler for `WAITING_CHANNEL_PUSH`.

**Scope:** `impl/08_concurrency.md` and `impl/09_runtime.md` — channel-related sections only
**Estimated Changes:** 2 files, ~10 sections

---

## Objective

### Problem / Gap / Need

The current impl spec uses a global `ChannelManager` holding `Map<string, Channel>` with a single `ConcurrentQueue` per channel. The accepted proposal restructures this to per-context outboxes/inboxes coordinated by lightweight hubs. The impl spec must be rewritten to reflect this architecture.

### Success Criteria
- [ ] `ChannelManager` + `Channel` replaced with `ChannelHubRegistry` + `ChannelHub`
- [ ] `ChannelHub` includes `participants` dictionary, `sequenceCounter`, `waitingPullers`, `waitingPushers`
- [ ] Context structure has `outboxes` and `inboxes` fields
- [ ] `ContextState` includes `WAITING_CHANNEL_PUSH`
- [ ] Open driver registers context with hub participants (as side effect of channel creation)
- [ ] Push driver: auto-register, outbox enqueue, waited push logic, direct delivery fast path, `timeout` support
- [ ] Pull driver: auto-register, participating-outbox scan with atomicity note, inbox check, waited-pusher wake
- [ ] Close driver: wakes both pullers AND pushers
- [ ] `Context.terminate()` includes `cleanupChannels()` flow
- [ ] Scheduler tick handles `WAITING_CHANNEL_PUSH` state
- [ ] Concurrency-safety requirement stated explicitly
- [ ] Blocking operations table updated with `/push` entry

### Out of Scope
- `spec.md` changes — covered by sibling plan
- C# runtime changes — separate scope
- Wait/Signal verbs — unchanged
- Fork/Call/Jump verbs — unchanged (except clone note: outboxes not cloned)

---

## Context

### Current State

`impl/08_concurrency.md` lines 292-560 define the channel system: `ChannelManager` (lines 308-328), `Channel` (lines 329-369), and verb drivers for Open (lines 386-424), Push (lines 428-473), Pull (lines 477-524), Close (lines 528-560).

`impl/09_runtime.md` defines the Runtime interface (line 70: `channels: ChannelManager`), Context structure (lines 316-351, no outbox/inbox fields), ContextState enum (lines 344-350, no `WAITING_CHANNEL_PUSH`), terminate flow (lines 401-408, no channel cleanup), blocking operations table (lines 416-421, no push entry), and scheduler tick (lines 438-489, no `WAITING_CHANNEL_PUSH` handler).

### Key Files

| File | Purpose | Changes Needed |
|------|---------|----------------|
| `impl/08_concurrency.md` | Concurrency impl spec | Replace ChannelManager/Channel, rewrite all verb drivers |
| `impl/09_runtime.md` | Runtime architecture spec | Add Context fields, ContextState, scheduler handler, cleanup, blocking table |

### Dependencies
- **Requires:** `20260211-channel-inbox-outbox-spec-plan.md` (spec amendments should land first or simultaneously)
- **Blocks:** C# runtime implementation

### Constraints
- Must use pseudocode consistent with existing impl doc style
- Concurrency-safety requirements must be stated, not assumed
- All resolved proposal questions are final decisions

---

## Implementation

### Overview

Two phases: (1) rewrite `impl/08_concurrency.md` channel section, (2) update `impl/09_runtime.md` supporting sections. Within 08, the changes flow: data structures → Open → Push → Pull → Close → cleanup.

---

### Step 1: Replace Channel Data Structures (impl/08_concurrency.md lines 292-383)

**Objective:** Replace `ChannelManager`/`Channel` with `ChannelHubRegistry`/`ChannelHub`.

**Files:**
- `impl/08_concurrency.md`

**Changes:**

Replace lines 292-383 (Channel Properties, Runtime Channel Storage, notes) with:

```markdown
## Channels

Channels are named FIFO pipes for inter-context communication, routed through hubs.

### Channel Properties

| Property | Description |
|----------|-------------|
| Name | Unique identifier (case-insensitive) |
| Type | Unbounded FIFO with per-context outboxes |
| Scope | Global across all contexts (via hub) |
| Safety | Thread-safe, concurrent access. All hub and outbox/inbox operations must use concurrency-safe data structures. |

### Channel Hub Registry

` ` `
ChannelHubRegistry:
  hubs: Map<string, ChannelHub>
  
  get(name: string): ChannelHub?
    return hubs.get(name.toLowerCase())
  
  getOrCreate(name: string): ChannelHub
    name = name.toLowerCase()
    if name not in hubs:
      hubs[name] = ChannelHub.new(name)
    return hubs[name]
  
  recreate(name: string): ChannelHub
    name = name.toLowerCase()
    old = hubs.get(name)
    newHub = ChannelHub.new(name)
    if old != null:
      newHub.generation = old.generation + 1
    hubs[name] = newHub
    return newHub
  
  remove(name: string)
    hubs.remove(name.toLowerCase())
` ` `

### Channel Hub

` ` `
ChannelHub:
  name: string
  generation: int
  closed: bool
  sequenceCounter: int                                     # Monotonic, incremented on each push
  participants: Map<contextId, ParticipantState>            # Auto-registered on first push/pull (or /open)
  waitingPullers: Queue<Context>                            # Contexts blocked on /pull (FIFO)
  waitingPushers: Queue<(Context, int)>                     # Contexts blocked on waited push (context, seqNum)

ParticipantState:
  context: Context
  outbox: Queue<(Value, int)>                              # (value, sequenceNumber) pairs
  inbox: Queue<Value>                                      # Values dispatched to this context
``` 

### Helper: ensureParticipant

` ` `
ensureParticipant(hub, context, channelName):
    if context.id not in hub.participants:
        if channelName not in context.outboxes:
            context.outboxes[channelName] = Queue.new()
        if channelName not in context.inboxes:
            context.inboxes[channelName] = Queue.new()
        hub.participants[context.id] = ParticipantState {
            context: context,
            outbox: context.outboxes[channelName],
            inbox: context.inboxes[channelName]
        }
` ` `

> [!IMPORTANT]
> **Concurrency safety**: All hub fields and outbox/inbox queues must be implemented with concurrency-safe data structures (e.g., concurrent collections, lock-guarded access). The scan-and-dequeue in the pull flow must be atomic — concurrent pullers must not race on the same outbox entry.

> [!IMPORTANT]
> **Close Notification**: Closing a channel MUST instantly wake all blocked pullers AND waited pushers.
> Pullers receive an error, not a nothing, to distinguish from timeout.
> Waited pushers receive an error to unblock.

> [!NOTE]
> **Generation IDs**: Each hub has a generation number. Pullers and waited pushers should check
> generation before/after blocking to ensure they don't accidentally operate on
> a NEW channel with the same name created after the old one closed.
>
> When `/open` re-creates a closed channel, the new hub has `generation = oldGeneration + 1`.

> [!NOTE]
> **Clone behavior**: When forking with `[clone]`, outboxes and inboxes are NOT cloned.
> A forked context starts with empty channel state. It will be auto-registered with hubs on first push/pull.
```

**Rationale:** Replaces the single-queue `Channel` with `ChannelHub` + `ParticipantState`. Explicit concurrency-safety callout per red team finding.

**Verification:** Data structures match proposal's resolved design. Concurrency-safety requirement stated.

---

### Step 2: Rewrite Open Driver (impl/08_concurrency.md lines 386-424)

**Objective:** Open creates hub (or re-creates closed hub). Also registers the calling context as a side effect.

**Files:**
- `impl/08_concurrency.md`

**Changes:**

Replace the Open section (lines 386-424) with:

```markdown
## Open

**Purpose**: Create a new channel or re-create a closed channel. Also registers the calling context with the hub as a side effect.

### Signature
` ` `
/open channel;
` ` `

### Behavior

- Creates a new hub if it doesn't exist
- If hub exists and is closed, creates a new hub with incremented generation
- If hub exists and is open, no-op for the hub
- Initializes context-local outbox and inbox for this channel (idempotent)
- Registers context with hub participation dictionary (idempotent)

### Implementation

` ` `
OpenDriver.execute(call, context):
    channelRef = resolve(call.params[0], context)
    
    if channelRef is not ChannelValue:
        return fatal("invalid_type", "Expected channel, got: " + channelRef.getType())
    
    channelName = channelRef.name
    
    hub = context.runtime.channelHubs.getOrCreate(channelName)
    if hub.closed:
        hub = context.runtime.channelHubs.recreate(channelName)
    
    # Initialize context's outbox and inbox for this channel (idempotent)
    if channelName not in context.outboxes:
        context.outboxes[channelName] = Queue.new()
    if channelName not in context.inboxes:
        context.inboxes[channelName] = Queue.new()
    
    # Register context with hub participation dictionary (idempotent)
    if context.id not in hub.participants:
        hub.participants[context.id] = ParticipantState {
            context: context,
            outbox: context.outboxes[channelName],
            inbox: context.inboxes[channelName]
        }
    
    return ok()
` ` `
```

**Verification:** Registration step present. Idempotent.

---

### Step 3: Rewrite Push Driver (impl/08_concurrency.md lines 428-473)

**Objective:** Push enqueues to context outbox with waited push logic, direct delivery fast path, and timeout support.

**Files:**
- `impl/08_concurrency.md`

**Changes:**

Replace the Push section (lines 428-473) with:

```markdown
## Push

**Purpose**: Add a value to a channel. Optionally block until consumed.

### Signature
` ` `
/push channel, value, wait:bool?, timeout:seconds?;
` ` `

### Behavior

- Auto-registers context with hub if not already a participant (lazy outbox/inbox creation)
- Pushes value to the calling context's outbox for the channel
- If a puller is waiting, delivers directly to puller's inbox (fast path — no blocking regardless of `wait`)
- If `wait: true` (default), blocks until the value is consumed by a puller
- If `wait: false`, returns immediately after enqueuing
- `timeout` controls max wait time when `wait: true`; ignored when `wait: false`

### Diagnostics

- Fatal: `invalid_type` — Parameter is not a channel
- Error: `not_found` — Channel does not exist
- Error: `closed` — Channel is closed, or closed while waiting
- Info: `timeout` — Push timed out before value was consumed (waited push only)

### Implementation

` ` `
PushDriver.execute(call, context):
    channelRef = resolve(call.params[0], context)
    value = resolve(call.params[1], context)
    wait = getNamedParam(call, "wait") ?? true         # default: blocking
    timeout = getNamedParam(call, "timeout")            # optional, double or null
    
    if channelRef is not ChannelValue:
        return fatal("invalid_type", "Expected channel, got: " + channelRef.getType())
    
    channelName = channelRef.name
    
    hub = context.runtime.channelHubs.get(channelName)
    if hub == null or hub.closed:
        return error("not_found" / "closed")
    
    # Auto-register context if not yet a participant
    ensureParticipant(hub, context, channelName)
    
    seq = hub.sequenceCounter++
    
    # Fast path: if a puller is waiting, deliver directly (skip outbox)
    if hub.waitingPullers.hasNext():
        target = hub.waitingPullers.dequeue()
        targetState = hub.participants[target.id]
        targetState.inbox.enqueue(value)
        target.state = RUNNING
        target.waitCondition = null
        return ok()    # Value consumed immediately — no blocking needed
    
    # No puller waiting — park value in pusher's outbox
    participantState = hub.participants[context.id]
    participantState.outbox.enqueue((value, seq))
    
    if wait:
        # Block until this specific value is consumed
        hub.waitingPushers.enqueue((context, seq))
        context.state = WAITING_CHANNEL_PUSH
        context.waitCondition = ChannelPushWaitCondition {
            channelName, seq, generation: hub.generation,
            timeout: timeout != null ? resolve(timeout, context).toDouble() : null,
            startTime: now()
        }
    
    return ok()
` ` `
```

**Verification:** Has fast path, outbox enqueue, waited push with timeout, auto-registration.

---

### Step 4: Rewrite Pull Driver (impl/08_concurrency.md lines 477-524)

**Objective:** Pull checks inbox first, then scans participating outboxes atomically for lowest sequence number.

**Files:**
- `impl/08_concurrency.md`

**Changes:**

Replace the Pull section (lines 477-524) with:

```markdown
## Pull

**Purpose**: Remove and return first value from channel.

### Signature
` ` `
/pull channel, timeout:seconds?;
` ` `

### Behavior

- Auto-registers context with hub if not already a participant (lazy outbox/inbox creation)
- Checks own inbox first (previously dispatched values)
- If inbox empty, scans all participating outboxes for the value with lowest sequence number
- Wakes blocked waited pusher if the consumed value was from a waited push
- Blocks if no values available anywhere

### Implementation

` ` `
PullDriver.execute(call, context):
    channelRef = resolve(call.params[0], context)
    timeout = getNamedParam(call, "timeout")
    
    if channelRef is not ChannelValue:
        return fatal("invalid_type", "Expected channel, got: " + channelRef.getType())
    
    channelName = channelRef.name
    
    hub = context.runtime.channelHubs.get(channelName)
    if hub == null or hub.closed:
        return error("not_found" / "closed")
    
    # Auto-register context if not yet a participant
    ensureParticipant(hub, context, channelName)
    
    # Check own inbox first (previously dispatched values)
    myState = hub.participants[context.id]
    if myState.inbox.hasValue():
        return ok(myState.inbox.dequeue())
    
    # Scan participating outboxes — find lowest sequence number
    # NOTE: This scan-and-dequeue MUST be atomic (hub-level lock)
    bestSource = null
    bestSeq = MAX_INT
    for (ctxId, state) in hub.participants:
        if state.outbox.hasValue():
            (_, seq) = state.outbox.peek()
            if seq < bestSeq:
                bestSeq = seq
                bestSource = state
    
    if bestSource != null:
        (value, consumedSeq) = bestSource.outbox.dequeue()
        
        # Wake the pusher if they're blocked on a waited push for this value
        if hub.waitingPushers.contains(bestSource.context, consumedSeq):
            hub.waitingPushers.remove(bestSource.context, consumedSeq)
            bestSource.context.state = RUNNING
            bestSource.context.waitCondition = null
        
        return ok(value)
    
    # Nothing available — block
    hub.waitingPullers.enqueue(context)
    context.state = WAITING_CHANNEL
    context.waitCondition = ChannelWaitCondition {
        channelName: channelName,
        timeout: timeout != null ? resolve(timeout, context).toDouble() : null,
        generation: hub.generation,
        startTime: now()
    }
    return ok()
` ` `
```

**Verification:** Auto-registration, inbox-first check, atomic scan note, participants-only iteration, waited-pusher wake.

---

### Step 5: Rewrite Close Driver (impl/08_concurrency.md lines 528-560)

**Objective:** Close wakes both blocked pullers AND waited pushers.

**Files:**
- `impl/08_concurrency.md`

**Changes:**

Replace the Close section (lines 528-560) with:

```markdown
## Close

**Purpose**: Close a channel, preventing further operations.

### Signature
` ` `
/close channel;
` ` `

### Implementation

` ` `
CloseDriver.execute(call, context):
    channelRef = resolve(call.params[0], context)
    
    if channelRef is not ChannelValue:
        return fatal("invalid_type", "Expected channel")
    
    channelName = channelRef.name
    
    hub = context.runtime.channelHubs.get(channelName)
    if hub == null:
        return error("not_found", "Channel does not exist: " + channelName)
    if hub.closed:
        return error("closed", "Channel already closed: " + channelName)
    
    hub.closed = true
    hub.generation++
    
    # Wake all blocked pullers with close error
    while hub.waitingPullers.hasNext():
        waiter = hub.waitingPullers.dequeue()
        waiter.state = RUNNING
        waiter.lastDiagnostics = [Diagnostic(ERROR, "closed", "Channel closed")]
        waiter.waitCondition = null
    
    # Wake all blocked waited pushers with close error
    while hub.waitingPushers.hasNext():
        (pusher, seq) = hub.waitingPushers.dequeue()
        pusher.state = RUNNING
        pusher.lastDiagnostics = [Diagnostic(ERROR, "closed", "Channel closed")]
        pusher.waitCondition = null
    
    # Hub is NOT removed — kept with closed=true so subsequent
    # push/pull returns "closed" (not "not_found"), and generation
    # is preserved for /open re-creation.
    return ok()
` ` `
```

**Verification:** Both puller and pusher wake loops present. Hub removed after close.

---

### Step 6: Update Runtime Interface (impl/09_runtime.md line 70)

**Objective:** Replace `channels: ChannelManager` with `channelHubs: ChannelHubRegistry`.

**Files:**
- `impl/09_runtime.md`

**Changes:**

```
// Before (line 70):
  channels: ChannelManager

// After:
  channelHubs: ChannelHubRegistry
```

Also update the architecture diagram (lines 33-36):

```
// Before:
│  │                      CHANNEL MANAGER                                 │   │
│  │  <channel1> → Queue, <channel2> → Queue, ...                         │   │

// After:
│  │                    CHANNEL HUB REGISTRY                              │   │
│  │  <channel1>:gen → Hub, <channel2>:gen → Hub, ...                     │   │
```

**Verification:** Runtime interface reflects new type. Diagram matches.

---

### Step 7: Update Context Structure (impl/09_runtime.md lines 316-351)

**Objective:** Add outbox/inbox fields and `WAITING_CHANNEL_PUSH` state.

**Files:**
- `impl/09_runtime.md`

**Changes:**

```
// Before Context (lines 328-342):
  # Variable storage
  storyVars: Map<string, Variable>
  contextVars: Map<string, Variable>
  flags: Map<string, Value>
  
  # Execution state
  lastReturnValue: Value
  lastDiagnostics: List<Diagnostic>
  
  # Deferred verbs
  storyDefers: Stack<CompiledVerbCall>
  contextDefers: Stack<CompiledVerbCall>
  
  # Waiting state
  waitCondition: WaitCondition?   # For /pull, /wait, /sleep

// After:
  # Variable storage
  storyVars: Map<string, Variable>
  contextVars: Map<string, Variable>
  flags: Map<string, Value>
  
  # Channel storage
  outboxes: Map<string, Queue<(Value, int)>>   # channel → queue of (value, seqNum)
  inboxes: Map<string, Queue<Value>>           # channel → queue of received values
  
  # Execution state
  lastReturnValue: Value
  lastDiagnostics: List<Diagnostic>
  
  # Deferred verbs
  storyDefers: Stack<CompiledVerbCall>
  contextDefers: Stack<CompiledVerbCall>
  
  # Waiting state
  waitCondition: WaitCondition?   # For /pull, /push, /wait, /sleep
```

```
// Before ContextState (lines 344-350):
ContextState:
  RUNNING           # Actively executing
  WAITING_CHANNEL   # Blocked on /pull
  WAITING_MESSAGE   # Blocked on /wait
  WAITING_CONTEXT   # Blocked on /call
  SLEEPING          # Blocked on /sleep
  TERMINATED        # Finished execution

// After:
ContextState:
  RUNNING                # Actively executing
  WAITING_CHANNEL        # Blocked on /pull
  WAITING_CHANNEL_PUSH   # Blocked on waited /push
  WAITING_MESSAGE        # Blocked on /wait
  WAITING_CONTEXT        # Blocked on /call
  SLEEPING               # Blocked on /sleep
  TERMINATED             # Finished execution
```

**Verification:** New fields and state present. Comment on waitCondition updated.

---

### Step 8: Update Terminate Flow (impl/09_runtime.md lines 401-408)

**Objective:** Add `cleanupChannels()` call in context termination.

**Files:**
- `impl/09_runtime.md`

**Changes:**

```
// Before (lines 401-408):
Context.terminate():
    # Execute defers
    executeStoryDefers()
    executeContextDefers()
    
    state = TERMINATED
    runtime.removeContext(this)

// After:
Context.terminate():
    # Execute defers
    executeStoryDefers()
    executeContextDefers()
    
    # Channel cleanup
    cleanupChannels()
    
    state = TERMINATED
    runtime.removeContext(this)

Context.cleanupChannels():
    # Deregister from all hub participation dictionaries
    for channelName in outboxes.keys():
        hub = runtime.channelHubs.get(channelName)
        if hub != null:
            hub.participants.remove(this.id)
    
    # Discard remaining outbox values (data belongs to the context)
    for channelName in outboxes.keys():
        outboxes[channelName].clear()
    
    # Discard unread inbox values
    for channelName in inboxes.keys():
        inboxes[channelName].clear()
    
    # Prune any waited-pusher entries for this context
    for channelName in outboxes.keys():
        hub = runtime.channelHubs.get(channelName)
        if hub != null:
            hub.waitingPushers.removeByContext(this.id)
```

**Verification:** Cleanup runs before TERMINATED state. Deregistration, discard, and prune all present.

---

### Step 9: Update Blocking Operations Table and Scheduler (impl/09_runtime.md lines 416-489)

**Objective:** Add `/push` to blocking operations table and `WAITING_CHANNEL_PUSH` handler to scheduler tick.

**Files:**
- `impl/09_runtime.md`

**Changes:**

Blocking operations table (line 416-421):

```
// Before:
| Verb | Block Condition | Unblock Condition |
|------|-----------------|-------------------|
| `/sleep` | Always | Timer expires |
| `/pull` | Channel empty | Value available or timeout |
| `/wait` | No message | Message received or timeout |
| `/call` | Forked context | Forked context terminates |

// After:
| Verb | Block Condition | Unblock Condition |
|------|-----------------|-------------------|
| `/sleep` | Always | Timer expires |
| `/push` | `wait: true` and no puller | Value consumed, timeout, or channel closed |
| `/pull` | Channel empty | Value available, timeout, or channel closed |
| `/wait` | No message | Message received or timeout |
| `/call` | Forked context | Forked context terminates |
```

Scheduler tick — add `WAITING_CHANNEL_PUSH` handler after `WAITING_CHANNEL` (after line 455):

```
            WAITING_CHANNEL_PUSH:
                hub = channelHubs.get(context.waitCondition.channelName)
                if hub == null or hub.closed:
                    # Channel gone — wake with error
                    context.state = RUNNING
                    context.lastDiagnostics = [Diagnostic(ERROR, "closed", "Channel closed")]
                    context.waitCondition = null
                elif context.waitCondition.isTimedOut():
                    # Timeout — remove from waiting pushers, value stays in outbox
                    hub.waitingPushers.remove(context, context.waitCondition.seq)
                    context.state = RUNNING
                    context.lastDiagnostics = [Diagnostic(INFO, "timeout", "Push timed out")]
                    context.waitCondition = null
                # Otherwise: still waiting — puller will wake us via PullDriver
```

Also update `WAITING_CHANNEL` handler to use hub-based lookup (line 447-455):

```
// Before:
            WAITING_CHANNEL:
                channel = channels.get(context.waitCondition.channelName)
                if channel.hasValue() or context.waitCondition.isTimedOut():
                    result = channel.pull(context.waitCondition.timeout, context.waitCondition.generation)
                    context.state = RUNNING
                    context.lastReturnValue = result.value
                    context.lastDiagnostics = result.diagnostics
                    context.waitCondition = null

// After:
            WAITING_CHANNEL:
                hub = channelHubs.get(context.waitCondition.channelName)
                if hub == null or hub.closed:
                    context.state = RUNNING
                    context.lastDiagnostics = [Diagnostic(ERROR, "closed", "Channel closed")]
                    context.waitCondition = null
                elif context.waitCondition.isTimedOut():
                    hub.waitingPullers.remove(context)
                    context.state = RUNNING
                    context.lastReturnValue = Nothing
                    context.lastDiagnostics = [Diagnostic(INFO, "timeout", "Pull timed out")]
                    context.waitCondition = null
                # Otherwise: still waiting — pusher will wake us via PushDriver fast path
```

**Rationale:** In the new model, the scheduler doesn't actively pull values — the push fast path delivers directly. The scheduler only handles timeout and close conditions.

**Verification:** Both new states handled. Timeout produces correct diagnostics. Close wakes with error.

---

## Verification Plan

### Automated Checks
- [ ] No automated tests for impl spec docs — they are markdown documents

### Manual Verification
- [ ] Read `impl/08_concurrency.md` channel section end-to-end — all verb drivers use `hub.participants` instead of `runtime.contexts`
- [ ] Verify Close driver wakes both pullers and pushers
- [ ] Verify `impl/09_runtime.md` Context has outboxes/inboxes fields
- [ ] Verify ContextState has `WAITING_CHANNEL_PUSH`
- [ ] Verify `terminate()` calls `cleanupChannels()`
- [ ] Verify scheduler tick handles both `WAITING_CHANNEL` and `WAITING_CHANNEL_PUSH`
- [ ] Cross-reference with proposal pseudocode — all resolved decisions reflected
- [ ] Cross-reference with red team conditions for approval — all addressed

### Acceptance Criteria Validation

| Criterion | How to Verify | Expected Result |
|-----------|---------------|-----------------|
| Hub registry replaces ChannelManager | Read 08_concurrency.md data structures | `ChannelHubRegistry` + `ChannelHub` present |
| Participants dictionary exists | Read ChannelHub structure | `participants: Map<contextId, ParticipantState>` |
| Open registers with hub (side effect) | Read OpenDriver pseudocode | `hub.participants[context.id] = ...` present |
| Push auto-registers + supports wait+timeout | Read PushDriver pseudocode | `ensureParticipant`, `wait` and `timeout` params, `WAITING_CHANNEL_PUSH` state |
| Pull auto-registers + scans participants | Read PullDriver pseudocode | `ensureParticipant`, `for (ctxId, state) in hub.participants` |
| Close wakes pushers | Read CloseDriver pseudocode | `waitingPushers` loop present |
| Context cleanup specified | Read Context.terminate() | `cleanupChannels()` call present |
| Scheduler handles push wait | Read Runtime.tick() | `WAITING_CHANNEL_PUSH` case present |

---

## Rollback Plan

If impl spec changes cause confusion or are rejected:
1. Revert `impl/08_concurrency.md` and `impl/09_runtime.md` to previous content (git revert)
2. Re-open proposal for further discussion

---

## Notes

### Assumptions
- The proposal's resolved questions are final
- Spec amendments land first or simultaneously
- All concurrency-safety requirements are implementor responsibility — pseudocode specifies WHAT, not HOW

### Risks
- Pseudocode complexity: mitigated by following existing impl doc style
- Missing edge cases in scheduler: mitigated by explicit timeout and close handling
