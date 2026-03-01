# Walkthrough: Two-Phase Continuation Model — Spec Update

> **Execution Date:** 2026-03-01
> **Completed By:** agent
> **Source Plan:** `20260301-two-phase-continuation-spec-update-plan.md`
> **Result:** Success

---

## Summary

Updated `impl/09_runtime.md` and `impl/08_concurrency.md` to replace the one-phase `ExecutionResult`/`Continuation` model with the two-phase `DriverResult`/`WaitRequest`/`WaitOutcome` model from the accepted proposal. All 9 plan steps completed with no blockers. One unplanned extension: four additional non-blocking drivers (OpenDriver, CloseDriver, SignalDriver, FlagDriver) were updated to the new return convention, as the type system change mandated it even though the plan did not enumerate them.

---

## Objectives Completion

| Objective | Status | Notes |
|-----------|--------|-------|
| Replace `ExecutionResult`/flat `Continuation` union with new type system | Complete | `DriverResult`, `Continuation` (with `onFulfilled`), `WaitRequest`, `WaitOutcome` all in place |
| `Context` gains `pendingContinuation`, `resumeToken`, `resume()`, `applyResult()`, `blockOnRequest()` | Complete | All added with correct semantics |
| `ContextState` gains `WAITING_HOST` | Complete | Added between `WAITING_CONTEXT` and `SLEEPING` |
| Execution loop uses `applyResult()` for both blocking and non-blocking paths | Complete | Jump guard applied per plan's pre-existing spec bug fix note |
| `resume()` has token-guarded double-call protection | Complete | Token check + `pendingContinuation == null` guard |
| Tick loop is a pure condition resolver — no verb-specific logic | Complete | `resolveWait()` returns `WaitOutcome?`, calls `context.resume()` |
| All driver pseudocode in `08_concurrency.md` returns `DriverResult` with `onFulfilled` closures | Complete | Includes unplanned drivers |
| Async model section updated symmetrically | Complete | `fulfillAsync(WaitRequest) → WaitOutcome` pattern |

---

## Execution Detail

### Steps 1–2: Type Definitions and Context Structure (`impl/09_runtime.md`)

**Planned:** Replace `VerbDriver.execute` return type; replace `ExecutionResult`/`Continuation` with `DriverResult`/`Continuation`/`WaitRequest`/`WaitOutcome`; add `pendingContinuation`, `resumeToken` to Context; add `WAITING_HOST` to `ContextState`.

**Actual:** Replaced Verb Driver section (old L178–204). Introduced `DriverResult` as a discriminated union (`Complete`/`Suspend`), `Continuation` as a struct with `request` and `onFulfilled`, `WaitRequest` with 6 variants (`Sleep`, `ChannelPull`, `ChannelPush`, `Signal`, `JoinContext`, `Host`), `WaitOutcome` with 3 variants (`Completed`, `TimedOut`, `Cancelled`). Added `pendingContinuation`, `resumeToken` to Context waiting-state block. Added `WAITING_HOST` to `ContextState` enum.

**Deviation:** None.

**Files Changed:**
| File | Change Type | Planned? | Details |
|------|-------------|----------|---------|
| `impl/09_runtime.md` | Modified | Yes | L185–226: replaced 20 lines with 42 lines of new type definitions; L374–387: added 2 fields + 1 enum variant |

**Verification:** `grep` confirmed `DriverResult`, `WaitRequest`, `WaitOutcome`, `onFulfilled`, `WAITING_HOST`, `pendingContinuation`, `resumeToken` all present; `ExecutionResult` absent.

---

### Steps 3–4: Execution Loop and Tick Loop (`impl/09_runtime.md`)

**Planned:** Replace `Context.run()` with jump-guard version using `applyResult()`; add `applyResult()`, `resume()`, `blockOnRequest()`; replace `Runtime.tick()` with `resolveWait()` pure condition resolver; add Host resume path; update async model section.

**Actual:** Rewrote `Context.run()` — captures `entryIp`/`entryStory` before driver call, passes to `applyResult()`. Added `Context.applyResult(result, entryIp, entryStory)` — `Complete` branch sets value/diagnostics and conditionally advances IP (jump guard: only if `instructionPointer == entryIp and currentStory == entryStory`); `Suspend` branch increments `resumeToken`, stores `pendingContinuation`, calls `blockOnRequest()`. Added `Context.resume(outcome, token)` — token guard + null guard on `pendingContinuation`, clears both fields, calls `onFulfilled`, sets `state = RUNNING`, calls `applyResult()`. Renamed `Context.block()` to `Context.blockOnRequest()` with identical structure plus new `Host` match arm (sets `WAITING_HOST` + `HostWaitCondition`). Replaced old tick loop (which contained inline var copying, diagnostic construction) with `resolveWait()` returning `WaitOutcome?` for each `ContextState`. Added Host resume path section with host-side pseudocode. Updated async model to `runContextAsync()`/`fulfillAsync()` pattern using `WaitRequest`.

**Deviation:** Minor — the plan noted a pre-existing spec bug at old L416 (unconditional `instructionPointer++` would break jump verbs). The jump guard fix was already incorporated in the plan's pseudocode. Applied as planned.

**Files Changed:**
| File | Change Type | Planned? | Details |
|------|-------------|----------|---------|
| `impl/09_runtime.md` | Modified | Yes | L399–481: replaced `run()` + `block()` with `run()` + `applyResult()` + `resume()` + `terminate()` + `blockOnRequest()`; L563–702: replaced one-phase tick loop + Execution Model Compatibility with `resolveWait()` + Host resume path + updated async model |

**Verification:** `grep` confirmed: `resolveWait` present, `context.resume(outcome, token)` called from scheduler, zero inline var copying or diagnostic construction in tick loop, `WAITING_HOST` handled, double-call guard in `resume()`.

---

### Step 5: Checkpoint Contract Validation (`impl/09_runtime.md`)

**Planned:** Verify section does not reference `ExecutionResult` or old `Continuation`.

**Actual:** Read-only verification. Section was clean — no stale references. Also noted that `run()` now calls `validateContract(statement)` when encountering a checkpoint (minor improvement over the old `# Checkpoints are markers, skip` comment, matching the proposal).

**Deviation:** None. No changes made.

---

### Step 6: CallDriver (`impl/08_concurrency.md`)

**Planned:** Replace `return ok(continuation: Context { contextId, inlineVars })` with `Suspend { JoinContext, onFulfilled }` closure moving inline var copying out of tick loop. Update `fatal()` → `Complete { Nothing, [Diagnostic(FATAL, ...)] }`.

**Actual:** Replaced both `return fatal(...)` calls and the final `return ok(continuation: Context {...})`. The `onFulfilled` closure captures `inlineVars` and `childId`, matches `Completed`/`Cancelled`, copies vars from child context on `Completed`. The old `Context { contextId, inlineVars }` variant (which embedded inline var logic in the tick loop's `WAITING_CONTEXT` handler) is fully eliminated.

**Deviation:** None.

**Files Changed:**
| File | Change Type | Planned? | Details |
|------|-------------|----------|---------|
| `impl/08_concurrency.md` | Modified | Yes | L250–277: replaced 28 lines with 42 lines including `Suspend`+closure |

**Verification:** Inline var copying confirmed in `onFulfilled`, not in `resolveWait()`.

---

### Step 7: PushDriver and PullDriver (`impl/08_concurrency.md`)

**Planned:** Fast paths → `Complete`; slow (blocking) paths → `Suspend` with `onFulfilled`.

**Actual:**

**PushDriver:** `return fatal(...)` → `Complete { Nothing, [Diagnostic(FATAL,...)] }`. `return error(...)` (two cases) → `Complete { Nothing, [Diagnostic(ERROR,...)] }`. Fast path (puller waiting) updated: instead of directly setting `target.state = RUNNING; target.waitCondition = null`, now calls `target.resume(Completed { value }, target.resumeToken)` — this is two-phase compliant (the puller's `onFulfilled` is invoked rather than bypassed). Fast path return: `return ok()` → `return Complete { Nothing, [] }`. Non-waiting path: `return ok()` → `return Complete { Nothing, [] }`. Waited path: `return ok(continuation: ChannelPush {...})` → `return Suspend { Continuation { ChannelPush, onFulfilled } }` with `Completed/TimedOut/Cancelled` handling in closure.

**PullDriver:** `return fatal(...)` → `Complete { Nothing, [Diagnostic(FATAL,...)] }`. `return error(...)` (two cases) → `Complete { Nothing, [Diagnostic(ERROR,...)] }`. Inbox fast path: `return ok(value)` → `return Complete { value, [] }`. Scan fast path: wake waited pusher via `bestSource.context.resume(Completed { Nothing }, bestSource.context.resumeToken)` instead of direct state mutation; `return ok(value)` → `return Complete { value, [] }`. Blocking path: `return ok(continuation: ChannelPull {...})` → `return Suspend { Continuation { ChannelPull, onFulfilled } }` with `Completed/TimedOut/Cancelled` handling.

**Deviation:** The plan did not explicitly specify that the fast-path wake-up in PushDriver/PullDriver (where a blocked context is directly unblocked) should use `context.resume()`. The old code mutated `target.state`/`target.waitCondition` directly. This was updated to use `context.resume()` for two-phase compliance — the woken context's `onFulfilled` handler is now correctly invoked. This is a necessary correctness fix implied by the model change.

**Files Changed:**
| File | Change Type | Planned? | Details |
|------|-------------|----------|---------|
| `impl/08_concurrency.md` | Modified | Yes (plus deviation) | PushDriver: L467–506 rewritten; PullDriver: L530–581 rewritten |

---

### Step 8: WaitDriver and SleepDriver (`impl/08_concurrency.md`)

**Planned:** WaitDriver → `Suspend { Signal, onFulfilled }`; SleepDriver → `Suspend { Sleep, onFulfilled: (_) -> Complete }`.

**Actual:** Exactly as planned. WaitDriver: `Message` variant renamed to `Signal`; added `onFulfilled` with `Completed/TimedOut/Cancelled` match. SleepDriver: `Sleep` variant unchanged; added `onFulfilled: (_) -> Complete { Nothing, [] }`.

**Deviation:** None.

**Files Changed:**
| File | Change Type | Planned? | Details |
|------|-------------|----------|---------|
| `impl/08_concurrency.md` | Modified | Yes | WaitDriver: L649–657 → 16 lines; SleepDriver: L700–701 → 5 lines |

---

### Step 9: ForkDriver, JumpDriver, and Remaining Drivers (`impl/08_concurrency.md`)

**Planned:** ForkDriver and JumpDriver — update `ok()`/`fatal()` return convention. Plan did not enumerate OpenDriver, CloseDriver, SignalDriver, FlagDriver.

**Actual:** ForkDriver and JumpDriver updated as planned. Additionally: OpenDriver, CloseDriver, SignalDriver, FlagDriver all updated to `Complete`/`Suspend` convention. CloseDriver also updated to wake blocked pullers/pushers via `context.resume(Cancelled {...}, context.resumeToken)` instead of direct `state = RUNNING; waitCondition = null` mutation — required for two-phase compliance (same reasoning as PushDriver/PullDriver fast paths).

**Deviation:** Four additional drivers updated beyond plan scope. Required by type system change — leaving them with `return ok()` syntax would leave the spec inconsistent (the interface now specifies `DriverResult`).

**Files Changed:**
| File | Change Type | Planned? | Details |
|------|-------------|----------|---------|
| `impl/08_concurrency.md` | Modified | Partial | JumpDriver L92–136, ForkDriver L169–208, OpenDriver, CloseDriver, SignalDriver, FlagDriver all updated |

---

## Complete Change Log

> **Derived from:** `git diff --stat main..HEAD`

### Files Created
| File | Purpose | In Plan? |
|------|---------|----------|
| `impl/projex/20260301-two-phase-continuation-spec-update-log.md` | Execution log | No (standard workflow artifact) |

### Files Modified
| File | Changes | In Plan? |
|------|---------|----------|
| `impl/09_runtime.md` | Type definitions, context structure, execution loop, tick loop, async model | Yes |
| `impl/08_concurrency.md` | All 11 verb drivers updated to `DriverResult` convention | Yes (9/11 planned, 2 additions) |
| `impl/projex/20260301-two-phase-continuation-spec-update-plan.md` | Status: Ready → In Progress → Complete | Yes |

### Files Deleted
None.

### Planned But Not Changed
None — all planned files were changed.

---

## Success Criteria Verification

### Criterion: `ExecutionResult` and flat `Continuation` union replaced

**Verification Method:** `grep` for `ExecutionResult` and old `| Message`/`| Context` continuation variants across both spec files.

**Evidence:** No matches in `impl/09_runtime.md` or `impl/08_concurrency.md`. Only references are in historical projex documents (proposal's "before" sections, evals, closed plans) — expected.

**Result:** PASS

---

### Criterion: `Context` gains `pendingContinuation`, `resumeToken`, `resume()`, `applyResult()`, `blockOnRequest()`

**Verification Method:** `grep` for each identifier in `impl/09_runtime.md`.

**Evidence:** All present — `pendingContinuation` (L377), `resumeToken` (L379), `applyResult` (L438, L453, L470, L480), `resume` (L457–481), `blockOnRequest` (L486, L489, L453).

**Result:** PASS

---

### Criterion: `ContextState` gains `WAITING_HOST`

**Verification Method:** Read `ContextState` enum in `impl/09_runtime.md`.

**Evidence:** `WAITING_HOST  # Blocked on host interaction (/converse, /choose, etc.)` present at L387.

**Result:** PASS

---

### Criterion: Execution loop uses `applyResult()` for both blocking and non-blocking paths

**Verification Method:** Read `Context.run()` and `Context.resume()` in `impl/09_runtime.md`.

**Evidence:** `run()` calls `applyResult(result, entryIp, entryStory)` unconditionally after driver execution. `resume()` calls `applyResult(result, instructionPointer, currentStory)` after `onFulfilled`. No direct IP manipulation or state setting in the loop body.

**Result:** PASS

---

### Criterion: `resume()` has token-guarded double-call protection

**Verification Method:** Read `Context.resume()`.

**Evidence:**
```
Context.resume(outcome: WaitOutcome, token: int):
    if token != resumeToken:
        return    # Stale token — context has moved on
    if pendingContinuation == null:
        return    # Already resumed (double-call race) — no-op
```

**Result:** PASS

---

### Criterion: Tick loop is a pure condition resolver — no verb-specific logic

**Verification Method:** Read `resolveWait()` in `impl/09_runtime.md` and confirm no inline var copying, no diagnostic construction.

**Evidence:** `resolveWait()` matches on `ContextState` and returns `WaitOutcome?` or `null`. No `context.set(...)`, no `Diagnostic(...)` construction, no `inlineVars` references. All post-resume logic is in driver `onFulfilled` closures.

**Result:** PASS

---

### Criterion: All driver pseudocode in `08_concurrency.md` returns `DriverResult` with `onFulfilled` closures

**Verification Method:** `grep` for `return ok(`, `return fatal(`, `return error(` in `impl/08_concurrency.md`.

**Evidence:** No matches. All blocking drivers (`CallDriver`, `PushDriver`, `PullDriver`, `WaitDriver`, `SleepDriver`) use `Suspend { continuation: Continuation { ..., onFulfilled: ... } }`. All non-blocking drivers use `Complete { ... }`.

**Result:** PASS

---

### Criterion: Async model section updated symmetrically

**Verification Method:** Read Execution Model Compatibility section in `impl/09_runtime.md`.

**Evidence:** Section uses `fulfillAsync(request: WaitRequest): WaitOutcome` pattern with `runContextAsync()` calling `context.resume(outcome, token)`. Maps all 6 `WaitRequest` variants including `Host`.

**Result:** PASS

---

### Acceptance Criteria Summary

| Criterion | Result |
|-----------|--------|
| `ExecutionResult`/flat `Continuation` replaced | PASS |
| `Context` gains new fields and methods | PASS |
| `ContextState` gains `WAITING_HOST` | PASS |
| `applyResult()` used on both paths | PASS |
| `resume()` token-guarded | PASS |
| Tick loop is pure condition resolver | PASS |
| All drivers return `DriverResult` with `onFulfilled` | PASS |
| Async model updated symmetrically | PASS |

**Overall: 8/8 criteria passed.**

---

## Deviations from Plan

### Deviation 1: Four Additional Drivers Updated
- **Planned:** Steps 6–9 enumerated CallDriver, PushDriver, PullDriver, WaitDriver, SleepDriver, ForkDriver, JumpDriver (7 drivers).
- **Actual:** Also updated OpenDriver, CloseDriver, SignalDriver, FlagDriver (11 total).
- **Reason:** The type system change (`execute` now returns `DriverResult`) mandated it. Leaving `return ok()` in these drivers would leave the spec internally inconsistent.
- **Impact:** None negative. Spec is fully consistent.
- **Recommendation:** Plan could have enumerated all drivers in the scope. Minor oversight.

### Deviation 2: Fast-Path Wake-Ups Use `context.resume()`
- **Planned:** Plan specified `Suspend`/`Complete` for the blocking path but did not address how fast-path wake-ups (PushDriver delivering directly to a waiting puller, PullDriver waking a waited pusher, CloseDriver unblocking all waiters) should work under two-phase.
- **Actual:** Fast-path wake-ups updated to use `context.resume(outcome, token)` instead of direct `state`/`waitCondition` mutation. This ensures the woken context's `onFulfilled` handler is invoked correctly.
- **Reason:** Two-phase compliance — direct state mutation bypasses the driver's post-resume logic.
- **Impact:** Correct behavior. Without this, fast-path wake-ups would skip `onFulfilled` and leave `pendingContinuation` set.
- **Recommendation:** Future plans involving model changes should explicitly address all state mutation sites, not just the returning driver.

---

## Issues Encountered

None. All steps proceeded without blockers.

---

## Key Insights

### Lessons Learned

1. **Enumerate all drivers when changing an interface**
   - Context: The plan listed 7 specific drivers but the interface change affected 11.
   - Insight: When changing a base interface return type, audit all implementors — not just the interesting/complex ones.
   - Application: Future plans should include a "find all implementors" step when changing interfaces.

2. **Fast-path state mutations are hidden two-phase violations**
   - Context: PushDriver/PullDriver/CloseDriver had inline `target.state = RUNNING` blocks that bypassed the driver's `onFulfilled`.
   - Insight: Any place that directly mutates a blocked context's state is a one-phase pattern. Two-phase requires routing through `resume()`.
   - Application: When adopting a callback-based continuation model, search for all direct `context.state = RUNNING` mutations and convert them.

### Pattern Discoveries

1. **`applyResult()` as unified result handler**
   - Observed in: `run()` and `resume()` both delegating to `applyResult()`
   - Description: A single function handles both the post-execute (run path) and post-resume (scheduler path) cases, eliminating duplication and guaranteeing consistent IP advancement.
   - Reuse potential: Any language runtime with a yield/resume execution model.

2. **Jump guard pattern**
   - Observed in: `applyResult()` checking `instructionPointer == entryIp and currentStory == entryStory` before advancing IP.
   - Description: Capture IP/story before driver call; only auto-advance if neither was modified by the driver. Lets jump verbs set IP directly and return `Complete` without being double-incremented.
   - Reuse potential: Any interpreter where verbs can modify control flow directly.

### Technical Insights

- The `Host` `WaitRequest` variant covers all presentation verbs (`/converse`, `/choose`, `/prompt`) without requiring new `ContextState` entries for each. The host handler registration happens before the `Suspend` return, so the scheduler doesn't need to know what interaction is pending.
- `resumeToken` prevents both stale host responses (timeout already advanced context) and double-call races (scheduler and host both try to resume). A simple integer is sufficient — no UUIDs needed.

---

## Recommendations

### Immediate Follow-ups
- [ ] Execute C# implementation plan (`20260301-two-phase-continuation-csharp-impl-plan.md`) — this spec update is the prerequisite.

### Future Considerations
- The `engine-time-awareness-eval` (`20260222-engine-time-awareness-eval.md`) references `Context.block()` — should be updated to `blockOnRequest()` in any follow-up spec work.
- Consider adding `HostWaitCondition` structure definition to `09_runtime.md` alongside the other `WaitCondition` types.

---

## Related Projex

| Document | Status |
|----------|--------|
| `20260301-two-phase-continuation-model-proposal.md` | Active — C# impl plan still pending |
| `20260301-continuation-resume-ip-gap-eval.md` | Resolved by this execution |
| `20260301-two-phase-continuation-csharp-impl-plan.md` | Next step — unblocked by this |

---

## References

- Commits: `9d3cd8d` (09_runtime.md), `007458d` (08_concurrency.md)
- Ephemeral branch: `projex/20260301-two-phase-continuation-spec-update`
- Execution log: `20260301-two-phase-continuation-spec-update-log.md`
