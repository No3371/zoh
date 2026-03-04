# Walkthrough: Runtime API Surface Revision — Spec Update

> **Execution Date:** 2026-03-04
> **Completed By:** Agent
> **Source Plan:** 20260304-runtime-api-surface-spec-plan.md
> **Duration:** ~20 minutes
> **Result:** Success

---

## Summary

Revised `impl/09_runtime.md` to internalize `Context` and expose `ContextHandle`, `ExecutionResult`, and `VariableAccessor` as the public API surface. All five planned sections were updated, plus two additional fixes discovered during verification (replacing `now()` with `runtime.elapsedMs` throughout `blockOnRequest`/`resolveWait` and updating the `Runtime.tick()` signature in the scheduler pseudo-code). The user also independently updated one reference in the Blocking Operations table.

---

## Objectives Completion

| Objective | Status | Notes |
|-----------|--------|-------|
| Runtime Interface uses `ContextHandle` instead of `Context` | Complete | All public operations updated |
| `createContext` replaced by `startContext → ContextHandle` | Complete | |
| `run(context)` and `runToCompletion(context)` removed from public surface | Complete | |
| `tick(deltaTimeMs: double)` is the sole execution driver | Complete | Includes scheduler pseudo-code update |
| `resume(handle, value)` replaces direct `context.resume()` for host interaction | Complete | |
| `ExecutionResult` type defined with lazy `VariableAccessor` | Complete | |
| `RuntimeConfig` includes `maxStatementsPerTick` | Complete | |
| `elapsedMs` documented as internal runtime state | Complete | All `now()` calls replaced with `runtime.elapsedMs` |
| `Context` section marked as internal implementation | Complete | Header updated, `runtime: Runtime` field replaced with note |
| Host Resume Path examples use `runtime.resume(handle, value)` | Complete | |
| Execution Model Compatibility updated | Complete | Both tick and async models |

---

## Execution Detail

### Step 1: Revise Runtime Interface (L53–91)

**Planned:** Replace `createContext`/`run`/`runToCompletion` with `startContext`/`tick(deltaTimeMs)`/`resume(handle, value)`. Mark Signals as internal. Add `maxStatementsPerTick`. Add `elapsedMs` to state.

**Actual:** Applied exactly as planned. Four targeted edits to the Runtime code block.

**Deviation:** None.

**Files Changed (ACTUAL):**
| File | Change Type | Planned? | Details |
|------|-------------|----------|---------|
| `impl/09_runtime.md` | Modified | Yes | Lines 67–91: state, operations, signals, config blocks |

---

### Step 2: Add New Types Section

**Planned:** Insert `ContextHandle`, `ExecutionResult`, `VariableAccessor` definitions between the Runtime Interface block and Handler Types.

**Actual:** Applied as planned. Inserted after the closing backtick fence of the runtime block.

**Deviation:** None.

**Files Changed (ACTUAL):**
| File | Change Type | Planned? | Details |
|------|-------------|----------|---------|
| `impl/09_runtime.md` | Modified | Yes | New Public Types subsection added (~22 lines) |

---

### Step 3: Mark Context Structure as Internal

**Planned:** Rename heading to `## Context Structure (Internal)`, add implementation detail note, replace `runtime: Runtime` field with a comment.

**Actual:** Applied as planned.

**Deviation:** None.

**Files Changed (ACTUAL):**
| File | Change Type | Planned? | Details |
|------|-------------|----------|---------|
| `impl/09_runtime.md` | Modified | Yes | Lines 371–376: heading + context block |

---

### Step 4: Update Host Resume Path

**Planned:** Update prose and pseudocode examples to use `ContextHandle` and `runtime.resume(handle, value)`.

**Actual:** Applied as planned. Both `onChoose` and `onConverse` examples updated.

**Deviation:** None.

**Files Changed (ACTUAL):**
| File | Change Type | Planned? | Details |
|------|-------------|----------|---------|
| `impl/09_runtime.md` | Modified | Yes | Lines 662–680: prose + code block |

---

### Step 5: Update Execution Model Compatibility

**Planned:** Update Cooperative Tick Model prose and loop example to use `tick(deltaTimeMs)` and handles. Add internal note to Async Task Model.

**Actual:** Applied as planned.

**Deviation:** None.

**Files Changed (ACTUAL):**
| File | Change Type | Planned? | Details |
|------|-------------|----------|---------|
| `impl/09_runtime.md` | Modified | Yes | Lines 690–713: tick model prose, loop example, async model comment |

---

### Fix Deviations: `now()` and tick signature (unplanned)

**Planned:** Not explicitly planned — the plan noted `blockOnRequest` and `resolveWait` would keep internal `Context` access, but `now()` calls were an inconsistency with the new time model.

**Actual:** Replaced all `now()` calls in `blockOnRequest` and `resolveWait` with `runtime.elapsedMs`. Updated `Runtime.tick()` signature to `Runtime.tick(deltaTimeMs: double)` in the scheduler pseudo-code to match the public API.

**Deviation:** Unplanned but necessary — `now()` would imply system-clock dependency, contradicting the spec's stated design. Consistent with plan intent.

**Files Changed (ACTUAL):**
| File | Change Type | Planned? | Details |
|------|-------------|----------|---------|
| `impl/09_runtime.md` | Modified | No (unplanned fix) | Lines 519–555: `blockOnRequest`; Lines 596–613: `resolveWait`+tick signature |

---

### User Intervention: Blocking Operations Table

**Timing:** Post-execution, before close.

**User change:** Updated "Unblock Condition" for `/converse`/`/choose`/`/prompt` in the Blocking Operations table from `Host calls context.resume() directly` → `Host calls runtime.resume(handle, value)`.

**Impact:** Consistent with the spec revision intent. No action required.

---

## Complete Change Log

> **Derived from:** `git diff --stat main..06cc77f` (plan execution commits only)

### Files Created
| File | Purpose | In Plan? |
|------|---------|----------|
| `projex/20260304-runtime-api-surface-spec-plan-log.md` | Execution log | Yes (by workflow) |

### Files Modified
| File | Changes | In Plan? |
|------|---------|----------|
| `impl/09_runtime.md` | Runtime Interface, Public Types, Context Structure, Host Resume Path, Tick compatibility, `now()` → `elapsedMs`, tick signature | Yes (+ unplanned fixes) |
| `projex/20260304-runtime-api-surface-spec-plan.md` | Status updated to `Complete` | Yes |

---

## Success Criteria Verification

### Acceptance Criteria Summary

| Criterion | Method | Result |
|-----------|--------|--------|
| `ContextHandle` replaces `Context` in public ops | Read Runtime Interface section | **Pass** |
| `startContext` replaces `createContext` | Read Runtime Interface | **Pass** |
| `run(context)` and `runToCompletion` removed | Read Runtime Interface | **Pass** |
| `tick(deltaTimeMs: double)` is the execution driver | Read Runtime Interface + scheduler | **Pass** |
| `resume(handle, value)` exists | Read Runtime Interface | **Pass** |
| `ExecutionResult` defined | Read Public Types section | **Pass** |
| `maxStatementsPerTick` in config | Read RuntimeConfig | **Pass** |
| `elapsedMs` in runtime state | Read Runtime state block | **Pass** |
| No `now()` in resolveWait/blockOnRequest | `git grep "now()"` showed zero remaining hits | **Pass** |
| Context section marked internal | Read section header | **Pass** (`## Context Structure (Internal)`) |
| Host examples use handles | Read Host Resume Path | **Pass** |

**Overall: 11/11 criteria passed**

---

## Deviations from Plan

### Deviation 1: `now()` surviving in internal pseudo-code
- **Planned:** Not explicitly addressed — plan noted `blockOnRequest` and `resolveWait` stay as internal
- **Actual:** `now()` calls were replaced with `runtime.elapsedMs` during verification
- **Reason:** `now()` implies system clock reads, violating the stated design principle (time is host-supplied)
- **Impact:** Positive — tightens consistency with the new model
- **Recommendation:** Plan could have explicitly listed this; acceptable unplanned fix

---

## Recommendations

### Immediate Follow-ups
- [ ] Create C# implementation plan to align runtime with revised spec (`ContextHandle`, `ExecutionResult`, new `tick(deltaTimeMs)` API)
- [ ] Consider updating the Blocking Operations table to also update the scheduler description in the Tick-Loop Scheduler comments (still says `context.resume()`)

### New Projex Suggested
| Type | Description |
|------|-------------|
| Plan | C# runtime alignment — implement `ContextHandle`, `tick(deltaTimeMs)`, `resume(handle, value)` in `ZohRuntime.cs` |
