# Option A Spec Plan: Handle-Backed Join for `/call [inline]`

> **Status:** Complete [PATCHED]
> **Created:** 2026-03-06
> **Author:** Agent
> **Source:** 20260305-inline-call-variable-copy-proposal.md (Option A selected)
> **Related Projex:** 20260305-inline-call-variable-copy-proposal.md, 20260304-runtime-spec-gaps-memo.md, 20260306-inline-call-handle-join-plan.md
> **Patch:** [20260307-inline-call-handle-join-patch.md](20260307-inline-call-handle-join-patch.md)

---

## Summary

Update runtime and concurrency implementation docs so `/call [inline]` join semantics are handle-based rather than ID-list-based. This makes post-termination child result/variable access explicit and removes the `contexts.find(id)` dependency that creates the race described in Issue #2.

**Scope:** `impl/09_runtime.md` and `impl/08_concurrency.md` only.
**Estimated Changes:** 2 files, ~50-80 lines edited.

---

## Objective

### Problem / Gap / Need

`impl/09_runtime.md` currently models `JoinContext` with `contextId: string` and resolves `WAITING_CONTEXT` via active `contexts` list lookup. `impl/08_concurrency.md` `CallDriver` pseudocode also retrieves child context by ID during continuation fulfillment. This conflicts with Option A, where a stateful `ContextHandle` should carry termination state/result independent of scheduler-list membership.

### Success Criteria

- [ ] `impl/09_runtime.md` `WaitRequest.JoinContext` is documented as `handle: ContextHandle` (not `contextId`).
- [ ] `impl/09_runtime.md` `Context.blockOnRequest()` stores `ContextWaitCondition { targetHandle }`.
- [ ] `impl/09_runtime.md` `resolveWait()` `WAITING_CONTEXT` branch checks `targetHandle.state == TERMINATED` and returns `Completed { targetHandle.result.value }`.
- [ ] `impl/09_runtime.md` `ContextHandle` public type docs include eventual terminated result access semantics.
- [ ] `impl/08_concurrency.md` `CallDriver.execute` pseudocode captures child handle and uses it for `[inline]` copyback.
- [ ] No `contexts.find(...targetContextId...)` or `runtime.findContext(childId)` remains in join/call pseudocode paths.

### Out of Scope

- C# runtime code changes (`csharp/src/...`).
- Expanding `WaitOutcome.Completed` payload shape (Option B path).
- Scheduler architecture changes beyond handle-based join condition resolution.

---

## Context

### Current State

1. `impl/09_runtime.md`:
   - `WaitRequest.JoinContext { contextId: string }`
   - `Context.blockOnRequest`: `ContextWaitCondition { targetContextId: contextId }`
   - `resolveWait` `WAITING_CONTEXT`: looks up `target = contexts.find(...)` then returns value
2. `impl/08_concurrency.md`:
   - `CallDriver` captures `childId`
   - `[inline]` copyback in `onFulfilled` uses `runtime.findContext(childId)` before `context.set(...)`

### Key Files

| File | Role | Change Summary |
|------|------|----------------|
| `impl/09_runtime.md` | Runtime architecture/types/scheduler pseudocode | Replace ID-based join with handle-based join; document handle result lifecycle |
| `impl/08_concurrency.md` | `/call` verb pseudocode | Capture `childHandle`; copy inline vars from handle result variables |

### Dependencies

- **Requires:** 20260305-inline-call-variable-copy-proposal.md accepted with Option A.
- **Blocks:** 20260306-inline-call-handle-join-plan.md implementation execution.

### Constraints

- Preserve two-phase continuation model (`DriverResult`, `WaitRequest`, `WaitOutcome`) introduced by 20260301-two-phase-continuation-spec-update-plan.md.
- Keep scheduler as pure condition resolver; no direct callback-style completion wiring.

### Assumptions

- `ContextHandle` remains an opaque external type; result exposure can be read-only/lazy.
- Inline copyback remains a driver concern (`onFulfilled`), not scheduler logic.

---

## Implementation

### Overview

Apply two synchronized doc edits:
1. Runtime type/condition model in `impl/09_runtime.md`.
2. `/call` pseudocode in `impl/08_concurrency.md`.

---

### Step 1: Update Runtime Join Contracts in `impl/09_runtime.md`

**Objective:** Replace join-by-ID model with join-by-handle model in the runtime architecture spec.
**Confidence:** High
**Depends on:** None

**Files:**
- `impl/09_runtime.md`

**Changes:**

```text
# Before
WaitRequest:
  | JoinContext { contextId: string }

... blockOnRequest ...
JoinContext { contextId }:
    state = WAITING_CONTEXT
    waitCondition = ContextWaitCondition { targetContextId: contextId }

... resolveWait ...
WAITING_CONTEXT:
    target = contexts.find(c => c.id == context.waitCondition.targetContextId)
    if target == null or target.state == TERMINATED:
        return Completed { target?.lastReturnValue ?? Nothing }

# After (shape)
WaitRequest:
  | JoinContext { handle: ContextHandle }

... blockOnRequest ...
JoinContext { handle }:
    state = WAITING_CONTEXT
    waitCondition = ContextWaitCondition { targetHandle: handle }

... resolveWait ...
WAITING_CONTEXT:
    target = context.waitCondition.targetHandle
    if target.state == TERMINATED:
        return Completed { target.result.value }
    return null
```

Also extend `ContextHandle` docs under **Public Types** to include terminated-result semantics (read-only view over final value/diagnostics/variables).

**Rationale:** This is the core Option A model and removes join fulfillment dependence on active context-list membership.

**Verification:** `impl/09_runtime.md` has no `targetContextId` references in join resolver paths.

**If this fails:** Revert `impl/09_runtime.md` entirely and re-apply as one cohesive edit.

---

### Step 2: Update `/call` Pseudocode in `impl/08_concurrency.md`

**Objective:** Align `CallDriver` pseudocode with handle-captured continuation fulfillment and `[inline]` copyback.
**Confidence:** High
**Depends on:** Step 1

**Files:**
- `impl/08_concurrency.md`

**Changes:**

```text
# Before
childId = newContext.id
...
request: JoinContext { contextId: childId }
...
childCtx = context.runtime.findContext(childId)
val = childCtx?.get(ref.name) ?? Nothing
context.set(ref.name, val)

# After (shape)
childHandle = context.runtime.addContext(newContext)  # or equivalent returned handle
...
request: JoinContext { handle: childHandle }
...
for ref in inlineVars:
    val = childHandle.result.variables.get(ref.name)
    context.set(ref.name, val)
```

Keep scheduler purity note: inline copying remains inside `onFulfilled`, not in `resolveWait`.

**Rationale:** `CallDriver` must consume final child state through the captured handle to avoid race-prone list lookups.

**Verification:** `impl/08_concurrency.md` call pseudocode no longer references `childId` lookups in runtime context list.

**If this fails:** Revert `impl/08_concurrency.md` and re-apply with a smaller diff scoped to the `Call` section.

---

## Verification Plan

### Automated Checks

- [ ] `rg -n "targetContextId|contextId: string|findContext\\(childId\\)|contexts.find\\(c => c.id" impl/09_runtime.md impl/08_concurrency.md` should return no hits in the updated join/call sections.

### Manual Verification

- [ ] Read `impl/09_runtime.md` `Public Types`, `WaitRequest`, `Context.blockOnRequest`, and `resolveWait` `WAITING_CONTEXT`.
- [ ] Read `impl/08_concurrency.md` `Call` implementation block and confirm captured handle usage end-to-end.

### Acceptance Criteria Validation

| Criterion | How to Verify | Expected Result |
|-----------|---------------|-----------------|
| Join request is handle-based | Read `WaitRequest` union in `impl/09_runtime.md` | `JoinContext { handle: ContextHandle }` present |
| Wait condition stores handle | Read `blockOnRequest` join branch | `targetHandle` used |
| Scheduler no longer scans by ID | Read `resolveWait` waiting-context branch | No `contexts.find` call for join |
| `/call` continuation captures handle | Read `impl/08_concurrency.md` `CallDriver` pseudocode | `childHandle` used in request + inline copy |
| Handle result semantics documented | Read `ContextHandle` section | Terminated result access explained |

---

## Rollback Plan

1. Revert `impl/09_runtime.md`.
2. Revert `impl/08_concurrency.md`.
3. Re-run `rg` checks to confirm ID-based model is fully restored.

---

## Notes

### Risks

- **Terminology drift risk:** `ContextHandle.result` naming may diverge from current public type wording in `impl/09_runtime.md`.
  - **Mitigation:** Keep wording aligned with existing `ExecutionResult`/`VariableAccessor` terms.
- **Spec/impl skew risk:** C# plan may proceed before spec update lands.
  - **Mitigation:** Mark this plan as blocking dependency in the C# plan.

### Open Questions

- [ ] None.
