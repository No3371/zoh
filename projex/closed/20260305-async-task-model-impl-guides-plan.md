# Plan: Async Host Pump Guidance for impl/09_runtime.md

> **Status:** Complete
> **Created:** 2026-03-05
> **Author:** Agent
> **Source:** 20260305-async-task-model-impl-guides-proposal.md
> **Completed:** 2026-03-05
> **Patch:** 20260305-async-host-pump-guidance-patch.md
> **Related Projex:** 20260305-async-task-model-stale-memo.md, 20260305-impl-guidelines-not-prescriptive-memo.md, 20260304-runtime-api-surface-spec-plan.md, 20260304-runtime-api-surface-spec-plan-walkthrough.md, 20260305-async-host-pump-guidance-patch.md

---

## Summary

Rewrite the stale "Async Task Model" subsection in `impl/09_runtime.md` to describe an async host pump (adapter) that still uses the public host API (`tick(deltaTimeMs)` + `resume(handle, value)`) and preserves push-only host interactions. Fix two nearby inconsistencies: `elapsedMs` advancement in the `Runtime.tick(deltaTimeMs)` pseudocode and the stale `WAITING_HOST` comment that still implies `context.resume(...)` is host-facing.

**Scope:** `impl/09_runtime.md` only (plus optional `impl/scenarios/` example).
**Estimated Changes:** 1 file required, +1 file optional.

---

## Objective

### Problem / Gap / Need

- `impl/09_runtime.md` "Async Task Model" still references `host.awaitInteractionAsync(timeout)` and describes a "no tick loop required" async mapping that contradicts the current push-only host integration (`runtime.resume(handle, value)`).
- Sleep/timeout semantics are defined in terms of `elapsedMs`, but the `Runtime.tick(deltaTimeMs)` pseudocode does not increment `elapsedMs`, contradicting the prose and the wait condition checks.
- The `resolveWait()` `WAITING_HOST` comment says the host calls `context.resume(outcome, token)` directly, contradicting the handle-based `runtime.resume(handle, value)` host resume path described immediately below.

### Success Criteria

- [x] No `awaitInteractionAsync` remains in `impl/09_runtime.md`.
- [x] `Runtime.tick(deltaTimeMs: double)` pseudocode increments `elapsedMs` before resolving waits.
- [x] `resolveWait()` `WAITING_HOST` comments align with `runtime.resume(handle, value)` (host push-only), not direct `context.resume`.
- [x] The async execution guidance is explicitly a host-side adapter around `tick()` + `resume()` (no invented host APIs).
- [x] The doc states the concurrency contract: `tick()` and `resume()` must not run concurrently unless the runtime is explicitly thread-safe.
- [x] (Optional) A short `impl/scenarios/` example demonstrates a background pump + serialized resume queue.

### Out of Scope

- Runtime/verb behavior changes (documentation only).
- C# runtime implementation changes (separate projex scope).
- Adding new required public APIs (e.g., deadline peeking helpers) to the Runtime Interface.

---

## Context

### Key Files

| File | Role | Change Summary |
|------|------|----------------|
| `impl/09_runtime.md` | Runtime implementation guidelines | Rewrite async section; fix `elapsedMs` tick advancement and `WAITING_HOST` comment consistency |
| `impl/scenarios/async_host_pump.md` (optional) | Host harness example | Minimal pump + resume-queue patterns for server/bot/test environments |

### Constraints

- Keep the public host surface stable: `tick(deltaTimeMs)` + `resume(handle, value)`.
- Preserve push-only host interactions (host resumes via `runtime.resume`).
- Keep the guidance non-prescriptive (impl docs are adaptable by environment); frame as a reference adapter pattern with allowed variations.

### Assumptions

- The runtime core is not thread-safe by default; hosts should serialize `tick()` + `resume()` per runtime instance.
- `elapsedMs` remains the timebase for `/sleep` and timeout evaluation as described elsewhere in `impl/09_runtime.md`.

---

## Implementation

### [PATCHED] Step 1: Fix `elapsedMs` Advancement in `Runtime.tick(deltaTimeMs)` Pseudocode

**File:** `impl/09_runtime.md`

**Location:** `## Blocking Operations` → `### Tick-Loop Scheduler` code block.

**Change:** Add time advancement at the start of `Runtime.tick(deltaTimeMs: double):`

- Insert:
  - `runtime.elapsedMs += deltaTimeMs`
- Immediately under:
  - `Runtime.tick(deltaTimeMs: double):`

This makes the pseudo-code consistent with:
- the runtime state field comment (`elapsedMs` is accumulated from tick calls), and
- the prose in "Cooperative Tick Model" that states `elapsedMs += deltaTimeMs`.

### [PATCHED] Step 2: Fix `WAITING_HOST` Comment in `resolveWait()`

**File:** `impl/09_runtime.md`

**Location:** same code block, `resolveWait(context: Context): WaitOutcome?` → `WAITING_HOST:`.

**Change:** Replace the host-facing comment that implies direct `context.resume` with push-only host resume wording.

- Replace:
  - `# The host calls context.resume(outcome, token) directly.`
- With (example wording; keep it consistent with the "Host Resume Path" section immediately below):
  - `# Fulfillment occurs when the host calls runtime.resume(handle, value).`

Optional minor consistency fix in the same pass:
- Update the `Context.resume(...)` doc comment from "scheduler OR the host" to "scheduler OR runtime.resume (host path)".

### [PATCHED] Step 3: Replace "Async Task Model" With "Async Host Pump (Adapter Pattern)"

**File:** `impl/09_runtime.md`

**Location:** `## Execution Model Compatibility` → replace the entire `### Async Task Model` subsection (currently the heading, its explanatory paragraph, code block, and the concluding note).

**Replacement content (insert verbatim):**

````markdown
### Async Host Pump (Adapter Pattern)

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
                # Optional flush: if resume() does not immediately run the execution loop,
                # tick(0) continues execution without advancing time.
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

> **Note:** Multi-context parallelism (`/fork`) still requires a scheduler under both models — either `tick()` advancing all contexts, or a host-driven pump serializing `tick()`/`resume()` for each runtime/session. `WaitRequest` does not change this requirement.
````

### [PATCHED] Step 4 (Optional): Add a Short Async Host Example Under `impl/scenarios/`

**Files:**

- `impl/scenarios/async_host_pump.md` (new)
- (Optional) `impl/09_runtime.md` (add a one-line "See also" pointer under the async section)

**Goal:** Provide a copyable harness-style snippet that shows:

- a background pump that advances time via `tick(dt)`,
- a serialized mailbox/queue for `resume(handle, value)` calls,
- an explicit note about real-time vs lazy timeout behavior.

Keep it short and clearly non-normative (an example, not a required architecture). Optionally reference `impl/scenarios/mud_server.md` as a motivating headless/server environment that can be driven by the same pump pattern.

---

## Verification

- [x] `rg "awaitInteractionAsync" impl/09_runtime.md` returns no matches.
- [x] Read the updated "Execution Model Compatibility" section; confirm it never implies a pull-based host interface.
- [x] Confirm the `Runtime.tick(deltaTimeMs)` pseudocode contains `elapsedMs += deltaTimeMs` (or equivalent).
- [x] Confirm `WAITING_HOST` comments and "Host Resume Path" are consistent about the host calling `runtime.resume(handle, value)`.
- [x] (Optional) Confirm the new `impl/scenarios/async_host_pump.md` example reads as guidance and does not invent new public API.

---

## Patch Execution

- [x] Completed via `20260305-async-host-pump-guidance-patch.md`.
- [x] All plan steps were implemented in this patch, including the optional scenario step.

---

## Rollback Plan

Revert edits to `impl/09_runtime.md` (and remove the optional scenario file, if added).
