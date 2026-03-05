# Memo: Async Task Model Section in 09_runtime.md is Stale / Misleading

> **Date:** 2026-03-05
> **Author:** Agent
> **Source Type:** Issue
> **Origin:** Conversation — user asked whether `impl/09_runtime.md` L735–758 is still valid given latest runtime design

---

## Source

The "Async Task Model" subsection of `impl/09_runtime.md` (L735–758) contains a `runContextAsync` + `fulfillAsync` pseudocode example that predates the finalized public API. The section was written to demonstrate that `WaitRequest` decouples blocking intent from scheduling strategy, but the pseudocode is now incoherent with the current design.

Two concrete problems identified in conversation:

1. **Host case contradiction.** `fulfillAsync` maps `Host { ... }` to `await host.awaitInteractionAsync(timeout)` — a pull-based model. But the current API defines host interaction as push-only: the driver calls the host *before* suspending, the host calls `runtime.resume(handle, value)` back. There is no `awaitInteractionAsync` method anywhere in the design. The async model implies the runtime await's a host object, which contradicts the spec.

2. **`elapsedMs` not addressed.** `Sleep` and `WAITING_HOST` timeouts both reference `elapsedMs`, which is only accumulated by `tick(deltaTimeMs)`. The async model has no `tick()` call, so `elapsedMs` never advances. `asyncSleep(durationMs)` as written implies a real OS sleep — but that also means the runtime is blocking a real thread, which contradicts the intended async nature.

---

## Context

The runtime public API is now formally: `tick(deltaTimeMs)` and `resume(handle, value)`. The section was written before that API was locked down (likely during early design where async was an equal-weight alternative).

Valid async scenarios for Zoh were discussed:
- Server / cloud: multiple concurrent player sessions, no frame loop
- Automated testing with real sleep/channels
- Bots and CLI tools
- Long-lived background contexts (NPC simulations, hours/days)

In all of these, the right model is not to bypass `tick()` and `resume()` but to bridge them: an async host would drive `tick()` via a timer loop, and route `runtime.resume(handle, value)` calls through `TaskCompletionSource` or equivalent to satisfy the `WAITING_HOST` awaitable. The public API stays consistent; only the driver layer is async.

The section's *intent* (WaitRequest is model-agnostic) remains valid; the pseudocode is the problem. Fix options:
- Replace `runContextAsync` with a description of how an async host still uses `tick()` + `resume()`, with a note on TaskCompletionSource bridging for host interactions.
- Or simply delete the section — it was illustrative, not normative — and leave async adaptation to implementers.

---

## Related Projex

- 20260304-runtime-api-surface-spec-plan-log.md (API surface formalization that locked in `tick` + `resume`)
- 20260304-runtime-spec-gaps-memo.md (related runtime spec gap inventory)
