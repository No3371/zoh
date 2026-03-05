# Patch: Async Host Pump Guidance in Runtime Implementation Docs

> **Date:** 2026-03-05
> **Author:** Agent
> **Directive:** `patch-projex` (applied to the active async-task-model documentation objective)
> **Source Plan:** 20260305-async-task-model-impl-guides-plan.md
> **Result:** Success

---

## Summary

Updated `impl/09_runtime.md` to remove stale async-task guidance and replace it with an async host pump adapter that stays aligned with the public host API (`tick(deltaTimeMs)` + `resume(handle, value)`). Also fixed two consistency gaps near the scheduler pseudocode: `elapsedMs` is now advanced explicitly in `tick()`, and the `WAITING_HOST` comment no longer implies direct host calls to `context.resume(...)`. Added a short optional scenario document under `impl/scenarios/` with a minimal serialized inbox pump pattern.

---

## Changes

### Runtime Guide Updates

**File:** `impl/09_runtime.md`  
**Change Type:** Modified

**What Changed:**
- Added `runtime.elapsedMs += deltaTimeMs` in `Runtime.tick(deltaTimeMs: double)` (`impl/09_runtime.md:625`).
- Updated `WAITING_HOST` scheduler comment to use the handle-based host resume path (`impl/09_runtime.md:684`).
- Replaced `### Async Task Model` with `### Async Host Pump (Adapter Pattern)` and inserted new async host adapter guidance (`impl/09_runtime.md:736`).
- Added an explicit serialization contract (`tick()` and `resume()` must not execute concurrently unless thread-safe) (`impl/09_runtime.md:781`).
- Added a pointer to the optional scenario example (`impl/09_runtime.md:783`).
- Clarified internal comment around resume call path (`impl/09_runtime.md:492`).

**Why:**
The previous async section referenced a non-existent pull host API and implied a no-tick model inconsistent with the rest of the runtime document. The new content keeps scheduling guidance practical while preserving the documented host-facing contract.

---

### Optional Scenario Example

**File:** `impl/scenarios/async_host_pump.md`  
**Change Type:** Created

**What Changed:**
- Added a minimal, copyable host adapter with queue inbox + periodic tick + optional `tick(0)` flush (`impl/scenarios/async_host_pump.md:1`).
- Documented integration constraints (monotonic clock, single mutator, lazy timeout behavior when periodic ticking is disabled) (`impl/scenarios/async_host_pump.md:36`).

**Why:**
Provides a concrete, non-prescriptive example for server/bot/test hosts that need async integration without changing runtime public APIs.

---

## Verification

**Method:** `rg` checks against updated docs plus targeted section review.

**Result:**
```text
NO_MATCH                                      # rg "awaitInteractionAsync" impl/09_runtime.md
625:    runtime.elapsedMs += deltaTimeMs      # elapsedMs advancement exists
684: # Fulfillment occurs ... runtime.resume  # WAITING_HOST comment aligned
736:### Async Host Pump (Adapter Pattern)     # stale heading replaced
```

**Status:** PASS

---

## Impact on Related Projex

| Document | Relationship | Update Made |
|----------|-------------|-------------|
| `20260305-async-task-model-impl-guides-plan.md` | Source plan | Marked as patched complete with patch reference |
| `20260305-async-task-model-impl-guides-proposal.md` | Source proposal | Added patch record under Next Steps / related context |

---

## Notes

Patch scope stayed bounded to documentation and pseudocode guidance, so this remained within patch-projex thresholds (no architecture migration, no runtime code changes, immediate verification possible).
