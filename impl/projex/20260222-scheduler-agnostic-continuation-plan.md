# Impl Spec: Scheduler-Agnostic Driver Blocking via Continuation

> **Status:** Ready
> **Created:** 2026-02-22
> **Author:** Agent
> **Source:** Direct request — follow-up from `20260222-verb-driver-sync-eval.md`
> **Related Projex:** `20260222-verb-driver-sync-eval.md`

---

## Summary

The current impl spec defines blocking verbs (sleep, pull, push, wait, call) by having drivers directly mutate `context.state` and `context.waitCondition` as side effects. This hard-couples every driver to the tick-loop scheduler model. Introducing a `Continuation` discriminated union in `ExecutionResult` lets drivers declare *what* they are waiting for as a return value — the runtime then decides *how* to fulfill it (tick-loop or async). No new language semantics. Spec-only changes.

**Scope:** `impl/` spec files only. No C# source changes.
**Estimated Changes:** 2 files (`impl/09_runtime.md`, `impl/08_concurrency.md`)

---

## Objective

### Problem / Gap / Need
Drivers currently block by calling `context.state = SLEEPING` / `context.waitCondition = ...` directly — a side-effect pattern that:
- Bakes the tick-loop scheduler model into every driver
- Makes drivers impossible to reuse in an async host without the tick-loop
- Violates the principle that a result should fully describe the outcome of a call

### Success Criteria
- [ ] `ExecutionResult` has an optional `continuation` field carrying a `Continuation` value
- [ ] `Continuation` is a discriminated union covering all five blocking cases
- [ ] No blocking driver pseudocode contains `context.state =` or `context.waitCondition =`
- [ ] `Context.block(continuation)` handles all registrations (subscribe, enqueue) and internal state
- [ ] Execution loop checks `result.continuation != null` (not `state != RUNNING`) as the yield signal
- [ ] A new "Execution Model Compatibility" section explains tick-loop and async scheduler patterns
- [ ] All existing tick-loop scheduler logic in `Runtime.tick()` is preserved and still correct

### Out of Scope
- C# implementation changes
- Language spec changes (`spec/`)
- `WaitCondition` type restructuring (kept as runtime-internal detail)
- Cross-context waking cleanup (`target.state = RUNNING` in PushDriver fast-path and CloseDriver)

---

## Context

### Key Files
| File | Purpose | Changes Needed |
|------|---------|----------------|
| `impl/09_runtime.md` | Runtime arch: types, execution loop, blocking pattern | Add `Continuation`, update `ExecutionResult`, update execution loop, add compat section |
| `impl/08_concurrency.md` | Concurrency drivers: Call, Push, Pull, Wait, Sleep | Remove `context.state/waitCondition =` from all five drivers; return Continuation |

### Dependencies
- **Requires:** Nothing — spec-only change
- **Blocks:** C# implementation of async verb driver interface (separate plan, `c#/projects/`)

---

## Implementation

### Overview

Three coordinated changes:
1. **Type layer** (`impl/09_runtime.md`): add `Continuation` union, extend `ExecutionResult`
2. **Runtime layer** (`impl/09_runtime.md`): `Context.block()` helper, updated execution loop
3. **Driver layer** (`impl/08_concurrency.md` + `impl/09_runtime.md`): remove all `context.state=` side effects from drivers, return Continuation instead

### Step 1: Add `Continuation` type and extend `ExecutionResult` — `impl/09_runtime.md`

Replace `### Verb Driver` block. Add `continuation` field to `ExecutionResult` and define `Continuation` discriminated union.

### Step 2: Add `Context.block()` and update execution loop — `impl/09_runtime.md`

- Update execution loop: check `result.continuation != null` instead of `state != RUNNING`
- Add `Context.block(continuation)` helper after `Context.run()`: centralises all registration side-effects and internal state-setting

### Step 3: Update blocking driver pseudocode — `impl/08_concurrency.md`

For CallDriver, PushDriver, PullDriver, WaitDriver, SleepDriver: remove `context.state =` and `context.waitCondition =`, return `ok(continuation: ...)` instead.

### Step 4: Update illustrative SleepDriver — `impl/09_runtime.md`

Update the duplicate SleepDriver in `## Blocking Operations → ### Implementation Pattern`.

### Step 5: Add "Execution Model Compatibility" section — `impl/09_runtime.md`

New section after `## Blocking Operations` explaining tick-loop and async-task patterns.

---

## Verification Plan

### Acceptance Criteria
- [ ] No `context.state =` or `context.waitCondition =` in driver pseudocode in `impl/08_concurrency.md` (except cross-context waking: `target.state = RUNNING`)
- [ ] `Continuation` union has 5 variants matching the 5 blocking `ContextState` values
- [ ] `Context.block()` covers all 5 variants and handles registrations
- [ ] `Runtime.tick()` logic unchanged
- [ ] `## Execution Model Compatibility` section present in `impl/09_runtime.md`

---

## Notes

### Assumptions
- `WaitCondition` types remain as runtime-internal detail; no restructuring needed
- Cross-context state mutation (`target.state = RUNNING` in PushDriver fast-path and CloseDriver) is a scheduler operation — out of scope
- `ok(continuation: X)` syntax means `ExecutionResult(value: nothing, diagnostics: [], continuation: X)`

### Risks
- **Timeout unit:** plan normalises to `timeoutMs` in `Continuation`; `Context.block()` must not double-convert when populating `WaitCondition`
