# Proposal: Practical Async Execution Model Guidance for impl/09_runtime.md

> **Status:** Draft
> **Created:** 2026-03-05
> **Author:** Agent
> **Related Projex:** 20260305-async-task-model-stale-memo.md, 20260305-impl-guidelines-not-prescriptive-memo.md, 20260304-runtime-api-surface-spec-plan.md, 20260305-async-task-model-impl-guides-plan.md, 20260305-async-host-pump-guidance-patch.md, 20260305-async-task-model-impl-guides-interview.md

---

## Summary

Replace the stale "Async Task Model" subsection in `impl/09_runtime.md` with guidance that is consistent with the current public host API (`tick(deltaTimeMs)` + `resume(handle, value)`) and the push-only host interaction model. The new section should be practical for server/bot/test environments by documenting an async *host pump* (adapter) that advances time and serializes `tick()`/`resume()` calls without inventing non-existent APIs like `host.awaitInteractionAsync()`.

---

## Problem Statement

### Current State

`impl/09_runtime.md` includes an "Execution Model Compatibility" section with two models:

- **Cooperative Tick Model** (frame loop calling `runtime.tick(dt)`).
- **Async Task Model** (each context runs as an async task, `WaitRequest` mapped to awaitables).

The Async section currently contains pseudocode (`runContextAsync` + `fulfillAsync`) that predates the now-formalized host API and contradicts other parts of the same document:

- It implies a pull-based host interface (`await host.awaitInteractionAsync(timeout)`) even though host interactions are defined as push-only via `runtime.resume(handle, value)`.
- It uses timeouts/sleeps without a `tick()`-driven `elapsedMs` advancement model, leaving the section unclear on how time progresses and how timeouts are enforced.

This makes the section misleading for implementers who target a non-frame-loop environment (server, bots, CLI, async tests), because it suggests there is a coherent "pure async runtime" when the rest of the guide assumes a `tick()`-advanced timebase plus explicit `resume()`.

### Gap / Need / Opportunity

Implementers still need an async-friendly execution model, but it should:

- Keep the **public host surface** stable (`tick(dt)`, `resume(handle, value)`).
- Preserve the core design goal that **`WaitRequest` decouples blocking intent from scheduling strategy**.
- Provide concrete, copyable patterns for async environments (task-based runtimes, event loops, timers) without implying impossible host APIs.

### Why Now?

The runtime host API was recently locked down in the implementation guides (tick signature + resume handle path). The async section is now the most prominent remaining piece that conflicts with that shape, and it risks becoming "cargo cult" reference code for downstream implementations.

---

## Proposed Change

### Overview

Update `impl/09_runtime.md` "Execution Model Compatibility" to:

1. Keep **Cooperative Tick Model** as the reference model.
2. Replace **Async Task Model** with an async model that is explicitly an *adapter* around the same `tick()` + `resume()` host API.
3. (Optional) Mention an internal pure-async scheduler strategy as an advanced alternative, but avoid presenting it as the default or as host-facing.

The key framing shift:

- "Async" is about **how the host drives the runtime** (timer/event loop and serialized calls), not about changing the runtime public surface or inventing awaitable host interaction APIs.

### Approach Options

#### Option A: Remove the Async Task Model Section Entirely

- **Description:** Delete the Async subsection. Keep only the Cooperative Tick Model and a one-paragraph note that other scheduling strategies exist.
- **Pros:**
  - Removes misleading content quickly.
  - Avoids needing to standardize async patterns.
- **Cons:**
  - Leaves server/bot/test implementers without guidance.
  - Undermines the stated point that `WaitRequest` is model-agnostic.
- **Effort:** Low.

#### Option B: Rewrite as "Async Host Pump (Adapter Pattern)" (Recommended)

- **Description:** Document a canonical async adapter that:
  - Advances time by calling `runtime.tick(dt)` from an async loop (timer or event-driven).
  - Accepts `resume(handle, value)` calls from arbitrary async callbacks by posting them into a single-threaded runtime loop/queue (so runtime mutations remain serialized).
  - Keeps host interactions push-only: the host callback ultimately calls `runtime.resume(handle, value)`; there is no `awaitInteractionAsync`.
- **Pros:**
  - Fully consistent with the current public API and push-based host interactions.
  - Practical and directly applicable to servers, bots, and automated tests.
  - Preserves deterministic testing: host can control the clock and feed dt explicitly.
- **Cons:**
  - Requires saying "you still need a time pump" (tick is still how timeouts/sleep progress).
  - Needs careful wording to avoid being overly prescriptive.
- **Effort:** Medium (doc rewrite + good pseudocode + a couple of small consistency fixes).

#### Option C: Document a Pure Async Scheduler Strategy (Internal) as a First-Class Alternative

- **Description:** Keep the current `runContextAsync`/`fulfillAsync` narrative but make it coherent by:
  - Defining internal awaitables for host waits (e.g., `TaskCompletionSource` completed by `runtime.resume()`).
  - Defining how timeouts/sleep advance (inject a clock/timer source; or compute dt internally).
  - Clarifying that this is a different scheduler strategy that may diverge from strict "host-supplied dt" determinism.
- **Pros:**
  - Matches common async-runtime mental models (awaitables per wait request).
  - Potentially more CPU-efficient than periodic ticking when idle.
- **Cons:**
  - Easy to drift away from the tick/elapsedMs model and introduce hidden time semantics.
  - Risks implying a different public API surface or creating pressure to add host-await APIs.
- **Effort:** Medium/High (requires nailing down time semantics and threading constraints).

### Recommended Approach

Adopt **Option B**: rewrite the section as an **async host pump** pattern that still uses `tick()` + `resume()` and explicitly calls out serialization and time advancement. If we keep any "pure async mapping" content, it should be explicitly labeled as an *internal* implementation strategy and must not mention pull-based host methods.

While editing the section, fix two nearby doc inconsistencies that directly affect async readers:

- In `Runtime.tick(deltaTimeMs)` pseudocode, include `elapsedMs += deltaTimeMs` (the prose already claims this).
- In `resolveWait()` for `WAITING_HOST`, update the comment to reflect the `runtime.resume(handle, value)` path (not `context.resume(...)`).

Async-specific doc guidance that should be explicit (based on interview clarifications):

- **Timeout behavior is implementer-defined:** Hosts can enforce timeouts in real time (background ticking while idle) or lazily (timeouts/sleep only observed when the host next calls `tick()`).
- **Tick cadence is implementer-defined:** Do not prescribe a default interval; explain the latency vs CPU trade-off.
- **Runtime mutation should be serialized:** `tick()` and `resume()` should not run concurrently unless the implementation is explicitly designed to be thread-safe.
- **Optional `tick(0)` flush:** If shown, it must be labeled optional and explained (it runs immediately without advancing time, to avoid waiting for the next scheduled tick after a resume).

---

## Impact Analysis

### Affected Areas

- `impl/09_runtime.md`: "Execution Model Compatibility" section (Async subsection rewrite) and small adjacent comment/pseudocode fixes.
- (Optional) `impl/scenarios/`: add a short "server/bot runner" snippet as an example, or reference `scenarios/mud_server.md` as a motivating environment.

### Dependencies

- None on spec semantics. This is implementation guidance only.
- Assumes the runtime core is not thread-safe and should be driven serially (a property we should state explicitly in the guide).

### Risks

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| The async guidance becomes overly prescriptive | Med | Med | Frame it as a "reference adapter pattern" and list variations (periodic vs deadline-driven tick pump). |
| Confusion about time semantics (real time vs deterministic time) | Med | High | Explicitly state: time advances only when `tick(dt)` is called; servers can compute dt from a monotonic clock; tests can supply dt deterministically. |
| Thread-safety assumptions conflict with a future runtime impl | Low/Med | Med | Document the requirement: `tick()` and `resume()` must not run concurrently; if an implementation is thread-safe, the adapter is still valid. |

### Breaking Changes

- None (documentation change).

---

## Open Questions

- Should `impl/09_runtime.md` explicitly declare a host contract like: "`tick()` and `resume()` must be serialized (single-threaded access)"?
- Do we want to state the serialization contract as "MUST" or "should (unless thread-safe)"?
- Is it worth adding an *optional* internal helper surface (not part of the public Runtime Interface) like `runtime.peekNextDeadlineMs()` to support a deadline-driven (low-CPU) pump?
- When a resume is queued, should the adapter also call `tick(0)` immediately to run resumed contexts without waiting for the next scheduled tick?

---

## Next Steps

If accepted:

1. Create a Plan projex to edit `impl/09_runtime.md`:
   - [x] Replace "Async Task Model" with "Async Host Pump (Adapter Pattern)" and coherent pseudocode.
   - [x] Remove any mention of `awaitInteractionAsync` and other invented host APIs.
   - [x] Fix `elapsedMs` advancement and the `WAITING_HOST` comment.
   - Plan: 20260305-async-task-model-impl-guides-plan.md
2. (Optional) Add a short server/bot example under `impl/scenarios/` that shows a background pump and a resume queue.
   - [x] Added: `impl/scenarios/async_host_pump.md`

Patch record:
- 20260305-async-host-pump-guidance-patch.md

---

## Appendix

### Candidate Replacement Content (for impl/09_runtime.md)

Below is candidate text intended to replace the current "Async Task Model" subsection.

#### Async Host Pump (Adapter Pattern)

Async environments (servers, bots, CLI tools, automated tests) typically do not have a frame loop, but they still need two things:

1. A mechanism to advance time for `/sleep` and timeouts (i.e., call `runtime.tick(dt)`).
2. A safe way to accept host callbacks (UI events, network IO) that call `runtime.resume(handle, value)` without racing the runtime core.

The simplest pattern is a single-threaded async pump that serializes all runtime mutation:

```text
# Host-side adapter: runs the runtime on one logical "runtime thread".
# Any external callbacks (network/UI) enqueue resumes into this loop.

AsyncRuntimePump:
    runtime: Runtime
    tickIntervalMs: double
    inbox: AsyncQueue<fn()>      # actions that call runtime.resume(...)

    async runUntilCancelled():
        last = monotonicNowMs()
        while not cancelled:
            # Wait for either a resume action or the next tick deadline.
            action = await inbox.tryDequeueAsync(timeoutMs: tickIntervalMs)

            now = monotonicNowMs()
            dt = now - last
            last = now
            runtime.tick(dt)

            if action != null:
                action()
                # Optional flush: resume() updates state/IP but does not run the main loop.
                # tick(0) continues execution immediately without advancing time.
                runtime.tick(0)

    # Called from host callbacks (any thread/task):
    postResume(handle, value):
        inbox.enqueue(() -> runtime.resume(handle, value))
```

Notes:

- **Push-only host interactions:** drivers invoke host handlers before suspending; the host later calls `runtime.resume(handle, value)`. There is no host-await API.
- **Time model:** timeouts and sleep are evaluated against `elapsedMs`, which advances only when `tick(dt)` is called. Servers can compute `dt` from a monotonic clock; tests can supply deterministic `dt`.
- **Timeout enforcement is a host choice:** with periodic ticking you get real-time timeouts; with event-only ticking, timeouts are lazy (only observed on the next `tick()`).
- **Serialization:** `tick()` and `resume()` must not execute concurrently unless the runtime is explicitly implemented as thread-safe.

Wait fulfillment under this model remains the same as the tick-based runtime; the difference is only *how the host drives it*:

- `Sleep`: fulfilled by `tick()` when `elapsedMs >= wakeTime`.
- `ChannelPull` / `ChannelPush`: fulfilled by channel hub fast paths (push/pull), with `tick()` enforcing timeout/close cancellation.
- `Signal`: fulfilled when `broadcastSignal` delivers a payload, with `tick()` enforcing timeout.
- `JoinContext`: fulfilled by `tick()` once the target context is terminated (or immediately if already terminated).
- `Host`: fulfilled when the host callback posts `runtime.resume(handle, value)`, with `tick()` enforcing timeout.

An optimized variant uses a deadline-driven timeout (sleep until the next known wake-up), but that requires runtime-internal visibility into the earliest pending deadline. This is an optional optimization and should not change the public host API.

#### Multi-Session Server Variant (One Pump, Many Runtimes)

For servers hosting many independent sessions, a common layout is **one `Runtime` per session** with a shared async pump that ticks all sessions and processes queued resume events. The key rule remains: each runtime must be mutated serially.

```text
ServerPump:
    runtimes: List<Runtime>
    tickIntervalMs: double
    inbox: AsyncQueue<fn()>      # actions that call runtime.resume(...) on a specific runtime

    async runUntilCancelled():
        last = monotonicNowMs()
        while not cancelled:
            action = await inbox.tryDequeueAsync(timeoutMs: tickIntervalMs)

            now = monotonicNowMs()
            dt = now - last
            last = now

            for rt in runtimes:
                rt.tick(dt)

            if action != null:
                action()
```

### Scenario Playbooks (Async Hosts)

These scenarios are intentionally practical: they show how an implementer can keep the **public runtime API** the same (`tick(dt)` + `resume(handle, value)`) while adapting to common async environments.

#### Scenario: Discord Bot (Event-Driven, Many Concurrent Interactions)

**Goal:** Run one or more Zoh contexts per Discord "session" (user DM, channel thread, or guild+channel+user tuple), with host interactions resumed by Discord events.

Key constraints:

- Discord events arrive concurrently.
- The runtime must not be mutated concurrently: **serialize `tick()` + `resume()` per runtime**.
- Timeouts should progress even when no events occur (otherwise `/sleep` and timeouts never fire).

Reference pattern:

```text
# One runtime per session key (recommended), each with an actor-style mailbox.

BotSession:
    key: SessionKey
    runtime: Runtime
    inbox: AsyncQueue<fn()>     # serialized runtime actions
    tickIntervalMs: double      # implementer-defined

    async runLoop():
        last = monotonicNowMs()
        while active:
            # Either an inbound Discord event (resume) or a periodic tick.
            action = await inbox.tryDequeueAsync(timeoutMs: tickIntervalMs)
            now = monotonicNowMs()
            dt = now - last
            last = now
            runtime.tick(dt)
            if action != null:
                action()
                runtime.tick(0)

    onDiscordMessage(msg):
        # Locate the target ContextHandle for this session (implementation detail:
        # likely stored alongside session state when the driver issued Host wait).
        handle = findPendingHandleForSession(key)
        inbox.enqueue(() -> runtime.resume(handle, ZohString(msg.content)))
```

Notes:

- The driver must surface enough information for the bot to route a Discord event to the correct handle (e.g., by storing the handle in a session table when the driver initiates a host wait).
- For Discord UI components (buttons/select menus), the host resumes with a structured `Value` (string/number/map) representing the user's selection.
- Use a monotonic clock for `dt` and clamp large `dt` jumps if the process pauses (sleep/GC) to avoid skipping far forward unexpectedly.
- If you drop the periodic tick (no timeout on `tryDequeueAsync`), then timeouts/sleep become lazy: they will only be observed when the next Discord event causes a `tick()`.

#### Scenario: Stateless HTTP Server (Request/Response, No In-Memory Sessions)

**Goal:** Treat HTTP requests as external stimuli that can resume a waiting context, while keeping the server horizontally scalable.

Reality check: a truly stateless server cannot hold the runtime/context state in memory between requests. If you want long-lived Zoh sessions across HTTP calls, you must persist enough state to reconstruct the runtime (or persist a serialized runtime snapshot). This proposal does not standardize snapshotting, but the async model guidance should not imply in-memory requirements.

Two practical modes:

1. **Run-to-completion per request** (best fit for stateless):
   - Each request loads a story + initial vars, runs until termination (or until host wait), and returns an HTTP response.
   - If the script blocks on `Host`, return `202 Accepted` with a token describing what input is needed, and rely on the client to call back.

2. **Persisted session** (stateful semantics, stateless infra):
   - Persist runtime state keyed by `sessionId`.
   - On each request: load snapshot, reconstruct runtime, post a `resume()`, then tick/run until the next suspension, persist snapshot again.

Reference pattern (persisted session, pump collapses to "tick on activation"):

```text
handleHttpRequest(req):
    snap = db.load(sessionId)
    rt = Runtime.fromSnapshot(snap)

    # Advance time based on wall clock since last activation.
    dt = monotonicNowMs() - snap.lastTickMs
    rt.tick(dt)

    # Resume the waiting host interaction (if applicable).
    if req.hasUserInput:
        rt.resume(snap.pendingHandle, parseInputValue(req))
        rt.tick(0)

    # Run until blocked/terminated is runtime-internal; host drives via tick/resume only.
    snap2 = rt.toSnapshot()
    snap2.lastTickMs = monotonicNowMs()
    db.save(sessionId, snap2)

    return buildHttpResponseFrom(rt)
```

Notes:

- This model intentionally **does not** require a continuous tick loop; time advances when the session is activated. That can be acceptable for servers, but it changes the user experience of timeouts (they may only fire when the next request arrives) unless you also run a background activator.
- If strict timeout behavior is required, add a background worker that periodically activates sessions whose next deadline is approaching (requires some form of "next deadline" index).

#### Scenario: CLI Runtime (Interactive REPL or Batch Runner)

**Goal:** Provide a command-line host that can run scripts headlessly or interactively with `/prompt`/`/choose` style waits.

Two modes that should both work with the same runtime public API:

1. **Batch mode (non-interactive):**
   - Disallow `Host` waits (treat as fatal diagnostic), or provide a fixed scripted input stream.
   - Provide a simple tick loop until termination.

2. **Interactive mode (REPL):**
   - When the context blocks on `Host`, present the prompt to the user, then call `runtime.resume(handle, value)` with the parsed input.
   - Decide how "time" behaves while waiting for user input (important for `/sleep` and timeouts).

Reference pattern (interactive, time paused while waiting for user input):

```text
runCli(rt):
    while not terminated:
        # Drive execution forward.
        rt.tick(0)

        if hasPendingHostWait():
            req = renderHostRequest()
            line = readLineBlocking(req.promptText)

            # Resume and immediately continue without advancing time.
            rt.resume(pendingHandle, parseCliValue(line))
            rt.tick(0)
        else:
            # No host wait; advance time at a steady rate.
            sleepMs(10)
            rt.tick(10)
```

Notes:

- "Time paused during input" is often the least surprising for CLI UX and keeps determinism (timeouts do not expire while a human is typing).
- If you want real-time timeouts even while waiting for input, the host must read input asynchronously and continue ticking in the meantime (same pump pattern as the Discord bot scenario).
