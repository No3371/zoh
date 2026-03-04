# Add Per-Statement State to Context (Spec)

> **Status:** Complete (via patch)
> **Created:** 2026-03-05
> **Author:** agent
> **Source:** 20260304-statement-cache-staging-eval.md
> **Related Projex:** 20260305-statement-state-impl-plan.md, 20260304-std-verbs-driver-alignment-plan-review.md, [20260305-statement-state-spec-patch.md](closed/20260305-statement-state-spec-patch.md)

---

## Summary

Add `statementState` to the `Context` structure in the runtime spec — a host-native, driver-private scratch space that persists across suspend/resume cycles for the same statement and is cleared when the statement completes. No changes to `Continuation`, `DriverResult`, `VerbDriver`, or the execution loop.

**Scope:** `impl/09_runtime.md` only
**Estimated Changes:** 1 file — 3 insertion sites

---

## Objective

### Problem / Gap / Need

Multi-step verbs (e.g., `/converse` with multiple content items) must persist driver state across suspend/resume cycles. The only current mechanism is closure captures in `onFulfilled`, which are opaque to the host, unserializable, and force recursive patterns. A per-statement scratch space on `Context` lets drivers store stage numbers, cached resolved values, and loop indices as plain data — inspectable, serializable, and transparent. The existing `onFulfilled` closure mechanism is unchanged; staging is a driver-level convention layered on top.

### Success Criteria

- [ ] `statementState` field exists on Context with type `Map<string, object>?`
- [ ] Clearing documented in `applyResult` on `Complete` (all cases — natural advance and jump)
- [ ] Clearing documented in `terminate`
- [ ] NOT cleared on `Suspend`
- [ ] No changes to `Continuation`, `DriverResult`, `VerbDriver`, execution loop, or `resume`

### Out of Scope

- C# implementation (separate plan: 20260305-statement-state-impl-plan.md)
- Rewriting `/converse` driver to use staging (covered by the alignment plan)
- Serialization

---

## Context

### Current State

`Context` structure (`impl/09_runtime.md:376–418`) defines execution state, variable storage, channel storage, defers, and waiting state. No per-statement scratch space exists. `applyResult` (L466–482) handles `Complete` (IP advance + jump guard) and `Suspend` (store continuation, block). `terminate` (L503–512) runs defers and cleans up channels.

### Key Files

| File | Purpose | Changes Needed |
|------|---------|----------------|
| `impl/09_runtime.md` | Runtime spec | Add field to Context, clearing to applyResult and terminate |

### Dependencies

- **Requires:** None — additive spec change
- **Blocks:** 20260305-statement-state-impl-plan.md (C# implementation)

---

## Implementation

### Step 1: Add `statementState` to Context Structure

**Objective:** Declare the field and document its lifetime semantics.

**Files:** `impl/09_runtime.md`

**Changes:** Insert after the `# Deferred verbs` group (after L402), before `# Waiting state` (L404):

```
# Before (L400–407):
  # Deferred verbs
  storyDefers: Stack<CompiledVerbCall>
  contextDefers: Stack<CompiledVerbCall>

  # Waiting state
  pendingContinuation: Continuation?   # Stored while blocked; cleared on resume

# After:
  # Deferred verbs
  storyDefers: Stack<CompiledVerbCall>
  contextDefers: Stack<CompiledVerbCall>

  # Per-statement driver state
  statementState: Map<string, object>?  # Driver-private scratch space.
                                        # Persists across suspend/resume for the SAME statement.
                                        # Cleared by applyResult on Complete, by terminate,
                                        # and on story exit (jump to different story).
                                        # The runtime never reads or interprets this — only clears it.

  # Waiting state
  pendingContinuation: Continuation?   # Stored while blocked; cleared on resume
```

**Rationale:** Logically adjacent to defers (both are statement-lifetime state) and before waiting state (which is suspend-lifetime).

**Verification:** Field appears in Context with comment describing clearing semantics.

### Step 2: Add Clearing to `applyResult`

**Objective:** Clear `statementState` when a statement completes.

**Files:** `impl/09_runtime.md`

**Changes:** Insert `statementState = null` in the `Complete` branch of `applyResult` (L466–475):

```
# Before:
Context.applyResult(result: DriverResult, entryIp: int, entryStory: CompiledStory):
    match result:
        Complete { value, diagnostics }:
            lastReturnValue = value
            lastDiagnostics = diagnostics
            if hasFatal(diagnostics):
                terminate()
            elif instructionPointer == entryIp and currentStory == entryStory:
                instructionPointer++

# After:
Context.applyResult(result: DriverResult, entryIp: int, entryStory: CompiledStory):
    match result:
        Complete { value, diagnostics }:
            lastReturnValue = value
            lastDiagnostics = diagnostics
            statementState = null
            if hasFatal(diagnostics):
                terminate()
            elif instructionPointer == entryIp and currentStory == entryStory:
                instructionPointer++
```

Clearing happens on ALL `Complete` results — natural IP advance and jump alike. If a jump moved IP/story during `execute()`, the current statement is still finished. NOT cleared on `Suspend` — persistence across suspend/resume is the mechanism's purpose.

**Verification:** `statementState = null` in Complete branch. Absent from Suspend branch.

### Step 3: Add Clearing to `terminate`

**Objective:** Clean up `statementState` on context termination.

**Files:** `impl/09_runtime.md`

**Changes:** Insert before `cleanupChannels()` in `terminate` (L503–512):

```
# Before:
Context.terminate():
    executeStoryDefers()
    executeContextDefers()
    cleanupChannels()
    state = TERMINATED
    runtime.removeContext(this)

# After:
Context.terminate():
    executeStoryDefers()
    executeContextDefers()
    statementState = null
    cleanupChannels()
    state = TERMINATED
    runtime.removeContext(this)
```

**Verification:** `statementState = null` appears in terminate.

---

## Verification Plan

### Manual Verification

- [ ] Context structure includes `statementState: Map<string, object>?` with clearing comment
- [ ] `applyResult` Complete branch clears `statementState` before the fatal/advance check
- [ ] `applyResult` Suspend branch does NOT clear `statementState`
- [ ] `terminate` clears `statementState`
- [ ] No changes to Continuation, DriverResult, VerbDriver, execution loop, resume, or blockOnRequest

---

## Rollback Plan

Remove the three insertions. No other spec text depends on `statementState`.

---

## Notes

### Assumptions

- Within-story jumps don't need a separate clearing path — the jump driver returns `Complete`, so `applyResult` clears `statementState` via the Complete branch.
- Cross-story jumps call `ExitStory` (which clears story vars/defers) before returning `Complete`. The spec doesn't currently show `ExitStory` as a named method — clearing in `applyResult/Complete` covers it. The C# implementation adds explicit clearing in `ExitStory` as belt-and-suspenders.

### Follow-Up

- C# implementation: 20260305-statement-state-impl-plan.md
- Alignment plan Step 1 can be updated to use staging once this spec change lands
