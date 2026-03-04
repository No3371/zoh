# Plan: Document Story Transition Lifecycle in Runtime Spec

> **Status:** Ready
> **Created:** 2026-03-05
> **Author:** Agent
> **Source:** 20260304-runtime-spec-gaps-memo.md — Issue #1 (Critical)
> **Related Projex:** 20260304-runtime-spec-gaps-memo.md, 20260304-context-runtime-coupling-eval.md

---

## Summary

`impl/09_runtime.md` defines `terminate()` but has no `leaveStory()` (or equivalent) concept. The spec mandates that crossing a story boundary fires story-scoped defers (LIFO), drops story variables, and clears per-statement driver state (`statementState`) — but none of this is described in the impl doc. The C# implementation already handles this correctly via `Context.ExitStory()`, so this is purely a spec gap.

**Scope:** `impl/09_runtime.md` only. No C# source changes.
**Estimated Changes:** 1 file, ~40 lines added in three locations.

---

## Objective

### Problem / Gap / Need

The impl spec documents `Context.terminate()` (which fires both story and context defers), but provides no documented path for a within-session story transition. As a result:

- `JumpDriver` implementers have no specified API to call when switching stories — they must reverse-engineer behavior from the C# source.
- `statementState` clearing on story exit is described only in a comment inside the `Context` structure block; no method documents it as a responsibility.
- The sequencing guarantee (defers fire *before* story variables are dropped) is unspecified.

Spec authority (in priority order):
- `spec/2_verbs.md` L772: story-scoped variables dropped on cross-story jump.
- `spec/2_verbs.md` L1383: defers with `scope: story` fire on leaving the story, LIFO.

### Success Criteria

- [ ] `impl/09_runtime.md` defines a `Context.leaveStory()` method with documented behavior: story defers (LIFO) → clear `statementState` → drop story variables.
- [ ] The `Context Structure` block notes that `statementState` is also cleared by `leaveStory()` (supplements the existing complete/terminate mentions).
- [ ] The `Context.terminate()` pseudocode is annotated to clarify it delegates story-defer firing to `leaveStory()` rather than duplicating it.
- [ ] `JumpDriver` pseudocode (or a navigation verb section) references `leaveStory()` as the required call before switching `currentStory`.
- [ ] All existing C# tests pass unchanged (`cd csharp && dotnet test`).

### Out of Scope

- Changes to C# source (`Context.cs`, `JumpDriver.cs`, or any `.cs` file).
- Addressing any other memo issues (Issues #2–#9).
- Adding unit tests for `leaveStory` behavior (C# already implements and tests this implicitly via `RunToCompletion` integration; isolated unit tests can be a follow-up).

---

## Context

### Current State

`impl/09_runtime.md` contains:

1. **`Context Structure` block** (around L401–L414): `storyDefers` and `contextDefers` stacks exist. `statementState` comment says it is cleared on complete, terminate, and "story exit (jump to different story)" — but no corresponding method exists in the spec.

2. **`Context.terminate()` pseudocode** (around L511–L523):
    ```
    Context.terminate():
        executeStoryDefers()
        executeContextDefers()
        statementState = null
        cleanupChannels()
        state = TERMINATED
        runtime.removeContext(this)
    ```
    Fires `executeStoryDefers()` as a subroutine — but this subroutine is unnamed/undefined in the spec. There is no `leaveStory()` that `terminate()` could reuse.

3. **No navigation verbs section**: No pseudocode exists for what `JumpDriver` must call when transitioning stories.

The C# implementation (`Context.ExitStory()`, line 237–242) already matches the intended behavior:
```csharp
public void ExitStory()
{
    ExecuteDefers(_storyDefers);
    statementState = null;
    Variables.ClearStory();
}
```
`JumpDriver.cs` calls `ctx.ExitStory()` then sets `ctx.CurrentStory = story` before invoking `ValidateContract`.

### Key Files

| File | Purpose | Changes Needed |
|------|---------|----------------|
| `impl/09_runtime.md` | Runtime architecture spec | Add `leaveStory()` method; annotate `terminate()` to delegate; add cross-story jump pseudocode |

### Dependencies

- **Requires:** Nothing — purely additive spec documentation.
- **Blocks:** Any future plan targeting C# navigation verb alignment against impl spec.

### Constraints

- Do not contradict `spec/2_verbs.md` L772 or L1383.
- Preserve all existing prose and pseudocode; only append or annotate.

---

## Implementation

### Overview

Three edits to `impl/09_runtime.md`:

1. **Add `leaveStory()` to the Context Structure block** — document the method signature alongside `terminate()`.
2. **Refactor `terminate()` pseudocode** — have it call `leaveStory()` instead of inlining `executeStoryDefers()`, and update the `statementState` comment accordingly.
3. **Add cross-story jump navigation note** — in the Execution Loop or a new Navigation section, document that any driver switching `currentStory` must call `leaveStory()` first.

---

### Step 1: Add `leaveStory()` to the Context Structure block

**Objective:** Establish `leaveStory()` as a first-class runtime method.

**File:** `impl/09_runtime.md`

**Change:** In the `Context Structure (Internal)` section, after the `Context:` pseudo-struct block (which ends with the `waitCondition` / `resumeToken` fields), locate the existing pseudocode comment about `statementState` clearing. Update the comment to include `leaveStory()`:

```diff
   # Per-statement driver state
   statementState: Map<string, object>?  # Driver-private scratch space.
                                         # Persists across suspend/resume for the SAME statement.
-                                        # Cleared by applyResult on Complete, by terminate,
-                                        # and on story exit (jump to different story).
+                                        # Cleared by applyResult on Complete, by leaveStory(),
+                                        # and by terminate.
                                         # The runtime never reads or interprets this — only clears it.
```

Then, after the `Context.blockOnRequest()` and `Context.cleanupChannels()` methods, add:

```
# Called whenever the context crosses a story boundary (cross-story /jump, and internally by terminate()).
# Fires all story-scoped deferred verbs in LIFO order, clears per-statement driver state,
# then drops all story-scoped variables.
# MUST be called before currentStory is reassigned.
Context.leaveStory():
    # Execute story-scoped defers (LIFO)
    while storyDefers.count > 0:
        verb = storyDefers.pop()
        executeVerb(verb)

    # Clear per-statement driver state
    statementState = null

    # Drop story-scoped variables
    storyVars.clear()
```

**Rationale:** Spec L1383 says story defers fire on leaving the story. Spec L772 says story variables are dropped on cross-story jump. Establishing `leaveStory()` as a named method makes both obligations explicit and gives `terminate()` a reuse point.

**Verification:** Read the edited section; confirm `leaveStory()` appears immediately after `cleanupChannels()`.

---

### Step 2: Refactor `terminate()` to delegate to `leaveStory()`

**Objective:** Eliminate the implicit `executeStoryDefers()` duplication in `terminate()` and make the delegation explicit.

**File:** `impl/09_runtime.md`

**Change:** Update the `Context.terminate()` pseudocode:

```diff
 Context.terminate():
-    # Execute defers
-    executeStoryDefers()
-    executeContextDefers()
-
-    statementState = null
+    leaveStory()   # fires story defers, clears statementState, drops story vars

+    # Execute context-scoped defers (LIFO)
+    while contextDefers.count > 0:
+        verb = contextDefers.pop()
+        executeVerb(verb)

     # Channel cleanup
     cleanupChannels()
```

**Rationale:** `terminate()` should not re-specify story-defer firing. Delegating to `leaveStory()` ensures the two code paths stay in sync.

**Verification:** Read the updated `terminate()` block; confirm it calls `leaveStory()` rather than `executeStoryDefers()`.

---

### Step 3: Document cross-story jump driver obligation

**Objective:** State explicitly that any driver performing a cross-story transition must call `leaveStory()` before switching `currentStory`.

**File:** `impl/09_runtime.md`

**Change:** In the `Checkpoint Contract Validation` section (or directly below the `Execution Loop` section — whichever is closest), add a new subsection:

```markdown
## Story Transition Protocol

When a verb driver executes a cross-story `/jump` (or similar navigation), it must follow this sequence before returning `Complete`:

```
# Cross-story transition — required call sequence in driver:
context.leaveStory()                  # story defers, statementState, story vars
context.currentStory = targetStory    # switch story
context.instructionPointer = targetIp # set checkpoint IP (after contract validation)
```

The contract validation (`validateContract`) must be called *after* `leaveStory()` but note that only transferred `var` parameters are available as story-scoped variables at the time of validation — the old story vars have been cleared.

`leaveStory()` is the only documented way to perform a clean story exit. Drivers that directly reassign `currentStory` without calling it violate the deferred execution and variable lifetime guarantees in the spec.
```

**Rationale:** Without this note, driver implementers have no spec-level guidance on the required call order. The current `JumpDriver.cs` already follows this sequence by calling `ctx.ExitStory()`.

**Verification:** Confirm the new subsection appears in the document and references `leaveStory()`.

---

## Verification Plan

### Automated Checks

- [ ] `cd csharp && dotnet build 2>&1 | Select-String -NotMatch "warning"` — must succeed (no source changes, but confirms doc edits didn't corrupt any embedded code snippets).
- [ ] `cd csharp && dotnet test 2>&1 | Select-String -Pattern "passed|failed|skipped"` — all existing tests must pass.

### Manual Verification

- [ ] Open `impl/09_runtime.md` and confirm:
  1. `Context.leaveStory()` pseudocode exists after `cleanupChannels()`.
  2. `Context.terminate()` calls `leaveStory()` instead of `executeStoryDefers()`.
  3. A "Story Transition Protocol" (or equivalent) subsection exists with the required call sequence.
  4. `statementState` comment references `leaveStory()`.

### Acceptance Criteria Validation

| Criterion | How to Verify | Expected Result |
|-----------|---------------|-----------------|
| `leaveStory()` documented | Read `impl/09_runtime.md` Context section | Method present with 3-step sequence |
| `terminate()` delegates | Read `terminate()` pseudocode | `leaveStory()` called, no inline `executeStoryDefers()` |
| Driver obligation stated | Read Story Transition Protocol section | Call order documented with pseudocode |
| `statementState` comment updated | Read Context struct `statementState` field | Comment references `leaveStory()` |
| Tests green | `dotnet test` | 0 failures |

---

## Rollback Plan

1. `git revert HEAD` — plan is a single file addition.
2. If the plan was committed and edits to `impl/09_runtime.md` were also committed, `git revert` the impl commit separately.

---

## Notes

### Assumptions

- The C# `Context.ExitStory()` is the ground truth for correct behavior; this plan only documents it.
- `var` transfer parameters in `/jump` (spec L780) are `/set` into story vars *after* `leaveStory()` clears them — consistent with C# `JumpDriver`'s current behavior (it calls `ExitStory()` before `ValidateContract`, not before copying vars, but the C# impl has no var-transfer support yet; this plan does not address that gap).
- `[clone]` behavior on `/fork` is not a story-transition concern and is out of scope.

### Risks

- **Stale context-defer wording:** If someone reads the old `terminate()` and the new one side-by-side, the rename from `executeContextDefers()` subroutine (implicit) to an inline `while` loop may look like a behavior change. It is not — clarify in the commit message.

### Open Questions

*(None — all resolved by spec and C# source inspection.)*
