# Patch: Handle-Backed Join for `/call [inline]`

> **Date:** 2026-03-07
> **Author:** Antigravity Agent
> **Directive:** patch-projex @[s:\Repos\zoh\impl\projex\20260306-inline-call-handle-join-spec-plan.md]
> **Source Plan:** [20260306-inline-call-handle-join-spec-plan.md](20260306-inline-call-handle-join-spec-plan.md)
> **Result:** Success

---

## Summary

Updated runtime and concurrency implementation docs so `/call [inline]` join semantics use `ContextHandle` explicitly. This cleanly models post-termination child variable access without requiring continuous context lookup during scheduler evaluation.

---

## Changes

### Runtime Spec Updates

**File:** `impl/09_runtime.md`
**Change Type:** Modified
**What Changed:**
- Replaced `JoinContext { contextId: string }` with `JoinContext { handle: ContextHandle }`.
- Updated `ContextWaitCondition` to store `targetHandle`.
- Changed `resolveWait` WAITING_CONTEXT condition to evaluate termination and value reading via `targetHandle.state` and `targetHandle.result.value`.
- Enriched `ContextHandle` struct to expose a lazy `ExecutionResult` visible post-termination.

**Why:**
This removes the implicit list lookups within conditions (`contexts.find()`), ensuring cleaner synchronization when waiting on a context that has already terminated or been removed from active execution pools.

---

### Concurrency /call Continuation

**File:** `impl/08_concurrency.md`
**Change Type:** Modified
**What Changed:**
- `CallDriver.execute` pseudo-code now reads the created child context handle via `childHandle = context.runtime.addContext(...)`.
- `[inline]` completion callback queries `childHandle.result.variables.get(ref.name)` directly.

**Why:**
Reflects the architectural decision to read from a handled scope state instead of doing runtime-level context resolution during child termination callback logic.

---

## Verification

**Method:** Executed the designated RegEx assertions in the specification implementation plan to ensure all orphaned context-IDs were removed from join requests and wait resolutions.

**Result:**
```text
> rg -n "targetContextId|contextId: string|findContext\(childId\)|contexts\.find\(c => c\.id" impl/09_runtime.md impl/08_concurrency.md
impl/09_runtime.md:83:  subscribe(name: string, contextId: string): void
impl/09_runtime.md:84:  unsubscribe(name: string, contextId: string): void
```
*(Existing contextual message signal bindings correctly skipped.)*

**Status:** PASS

---

## Impact on Related Projex

| Document | Relationship | Update Made |
|----------|-------------|-------------|
| [20260306-inline-call-handle-join-spec-plan.md](20260306-inline-call-handle-join-spec-plan.md) | Source plan | Marked fully completed with `[PATCHED]` and closed correctly. |

---

## Notes

No issues encountered.
