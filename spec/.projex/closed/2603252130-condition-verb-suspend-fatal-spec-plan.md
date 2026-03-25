# Spec: Condition Verb Suspend & Fatal Propagation

> **Status:** Complete
> **Created:** 2026-03-25
> **Author:** Agent
> **Source:** `2603252130-phase4-condition-suspend-fatal-propagation-proposal.md`
> **Related Projex:** `2603252101-phase4-flowutils-condition-suspend-fatal-memo.md`, `20260315-channel-semantics-try-suspension-plan.md`
> **Blocks:** `2603252130-phase4-condition-suspend-fatal-impl-plan.md` (csharp)
> **Worktree:** No
> **Completed:** 2026-03-26
> **Walkthrough:** `2603252130-condition-verb-suspend-fatal-spec-plan-walkthrough.md`

---

## Summary

Add explicit spec language requiring suspend and fatal propagation when a verb is used as a condition — in `/if` and `/while` subjects, `/switch` subject/case, and `breakif`/`continueif` parameters. Currently the spec says "the returned value is used" for verb conditions but is silent on non-value outcomes (suspend, fatal). The `/try` "Suspension Transparency" note establishes the principle; this plan extends it to all condition-evaluation sites.

**Scope:** `spec/2_verbs.md` (language spec) and `impl/07_control_flow.md` (implementation pseudo-code).
**Estimated Changes:** 2 files, ~6 sections.

---

## Objective

### Problem / Gap / Need

The spec text for flow verbs says "In case of `/verb`, the returned value is used" for conditions/subjects. This only covers the happy path (verb completes with a value). Two non-value outcomes are unaddressed:

1. **Suspend** — verb yields `Suspend` (e.g. channel read, host interaction). Without spec guidance, implementations can silently collapse suspend to "nothing" → falsy, losing the suspension.
2. **Fatal** — verb produces a fatal diagnostic. Without spec guidance, the fatal can be swallowed (condition treated as falsy) instead of terminating the context.

The `/try` verb (spec §Behavior Notes, item 7) already establishes "Suspension Transparency" as a principle. Fatal propagation is implicit in the runtime contract (fatals terminate the context). Neither is stated at the condition-evaluation sites.

### Success Criteria

- [x] `spec/2_verbs.md` — Each flow verb that accepts a `/verb` as condition/subject (`/if`, `/while`, `/switch`, `breakif`/`continueif` in `/sequence`, `/loop`, `/foreach`) has a behavior note or shared reference covering suspend and fatal propagation.
- [x] `impl/07_control_flow.md` — `shouldBreak` pseudo-code handles suspend and fatal from `executeVerb`. `WhileDriver` pseudo-code handles suspend and fatal from condition verb. `IfDriver` pseudo-code handles suspend and fatal from subject verb.
- [x] No changes to verb parameters, return types, or diagnostic codes — only clarification of behavior on non-value outcomes.

### Out of Scope

- C# runtime changes (separate plan: `2603252130-phase4-condition-suspend-fatal-impl-plan.md`).
- New verb parameters or attributes.
- Changes to `/try`, `/do`, or any verbs not evaluating conditions.
- `continueif` parameter for `/sequence` (not in spec).

---

## Context

### Current State

**`spec/2_verbs.md`:**
- `/if` subject: "In case of `/verb`, the returned value is used." No suspend/fatal note.
- `/while` subject: Same phrasing. No suspend/fatal note.
- `/switch` subject/case: "In case of `/verb`, it takes the return value of the verb." No suspend/fatal note.
- `breakif` (sequence, loop, foreach): "In case of `/verb`, the returned value is used." No suspend/fatal note.
- `/try` §Behavior Notes item 7: "Suspension Transparency: If the inner verb suspends … `/try` must propagate the suspension and resume from the same point."

**`impl/07_control_flow.md`:**
- `IfDriver.execute`: `subject = executeVerb(subject, context)` — assigns directly, no suspend/fatal check.
- `WhileDriver.execute`: `subject = executeVerb(subject, context)` — same.
- `shouldBreak`: `value = executeVerb(condition, context)` — same.

### Key Files

| File | Role | Change Summary |
|------|------|----------------|
| `spec/2_verbs.md` | Language spec — flow verbs | Add behavior note on condition verb suspend/fatal propagation |
| `impl/07_control_flow.md` | Implementation pseudo-code | Update `IfDriver`, `WhileDriver`, `shouldBreak` to handle suspend/fatal |

### Dependencies

- **Requires:** None.
- **Blocks:** `2603252130-phase4-condition-suspend-fatal-impl-plan.md` (csharp implementation).

### Constraints

- Keep spec text concise — a shared behavior note referenced from each verb is preferable to duplicating prose.
- Pseudo-code must remain implementation-agnostic (no C#-specific types).
- Preserve existing diagnostic codes; do not invent new ones.

### Assumptions

- The "Suspension Transparency" principle from `/try` is intended to apply universally, not just to `/try`.
- Fatal propagation from condition verbs is the expected behavior (fatals terminate the context per the runtime contract).

---

## Implementation

### Overview

Add a shared "Condition Verb Evaluation" behavior note to the flow verbs section of `spec/2_verbs.md`, then reference it from each relevant verb. Update the three pseudo-code blocks in `impl/07_control_flow.md`.

---

### Step 1: Add shared behavior note to `spec/2_verbs.md`

**Objective:** Define condition verb evaluation semantics once, covering suspend and fatal.
**Confidence:** High
**Depends on:** None

**File:** `spec/2_verbs.md`

**Changes:**

Insert a behavior note after the `/if` section header (before or after the first flow verb) that applies to all flow verbs accepting `/verb` as condition/subject. The note is referenced by name from each verb.

```markdown
// After: (insert between the heading line for Core.Flow.If and its description, or as a
// standalone subsection before /if — placement TBD based on existing document structure)

// New section — insert before "#### Core.Flow.If" (line 406):

#### Condition Verb Evaluation

When a flow verb accepts a `/verb` as a condition or subject parameter (e.g. `/if`, `/while`, `breakif`), the verb is executed and its result determines the condition value. The following rules apply:

1. **Complete result**: The returned value is used as the condition value (existing behavior).
2. **Suspend result**: If the condition verb suspends, the outer verb propagates the suspension unchanged. On resume, evaluation continues from the suspension point.
3. **Fatal result**: If the condition verb produces a fatal diagnostic, the outer verb propagates the fatal unchanged (context terminates).

These rules are consistent with [Suspension Transparency](#core-error-try) and the general runtime contract that fatal diagnostics terminate the context.
```

**Rationale:** A single shared note avoids duplicating the same prose across 6 verb descriptions. Each verb references it by name.

**Verification:** Read the section and confirm it covers suspend, fatal, and references existing principles.

**If this fails:** Remove the inserted section; no other content is affected.

---

### Step 2: Reference the shared note from each flow verb in `spec/2_verbs.md`

**Objective:** Link each verb's condition/subject parameter to the shared behavior note.
**Confidence:** High
**Depends on:** Step 1

**File:** `spec/2_verbs.md`

**Changes:**

For each verb that accepts `/verb` as condition/subject, append a one-line reference in the parameter description or diagnostics section:

**`/if` — subject parameter (line ~414):**
```markdown
// Before:
- `subject`: Accept `any`. In case of reference, the value is used. In case of `` `expr` ``, it is evaluated. In case of `/verb`, the returned value is used.

// After:
- `subject`: Accept `any`. In case of reference, the value is used. In case of `` `expr` ``, it is evaluated. In case of `/verb`, the returned value is used. See [Condition Verb Evaluation](#condition-verb-evaluation) for suspend and fatal handling.
```

**`/while` — subject parameter (line ~507):**
```markdown
// Before:
- `subject`: Accept `any`. In case of reference, the value is used. In case of `` `expr` ``, it is evaluated. In case of `/verb`, the returned value is used.

// After:
- `subject`: Accept `any`. In case of reference, the value is used. In case of `` `expr` ``, it is evaluated. In case of `/verb`, the returned value is used. See [Condition Verb Evaluation](#condition-verb-evaluation) for suspend and fatal handling.
```

**`/switch` — subject + case parameters (lines ~551–553):**
```markdown
// Before:
- `subject`: the subject to compare. Accept `/verb`, `` `expr` `` or any reference. In case of reference, the value will be used. In case of `/verb`, it takes the return value of the verb. In case of `` `expr` ``, it evaluates the expression.

// After:
- `subject`: the subject to compare. Accept `/verb`, `` `expr` `` or any reference. In case of reference, the value will be used. In case of `/verb`, it takes the return value of the verb. In case of `` `expr` ``, it evaluates the expression. See [Condition Verb Evaluation](#condition-verb-evaluation) for suspend and fatal handling.
```

Same one-line addition to the `case` parameter line.

**`breakif` in `/sequence`, `/loop`, `/foreach` (lines ~441, ~482, ~528):**
```markdown
// Before:
- `breakif`: Optional. The condition to break the loop or execute next verb. Accept `*boolean`, `/verb`/`*verb` or `` `expr` ``/`` *`expr` ``. In case of reference, the , the value is used. In case of `` `expr` ``, it is evaluated. In case of `/verb`, the returned value is used.

// After:
- `breakif`: Optional. The condition to break the loop or execute next verb. Accept `*boolean`, `/verb`/`*verb` or `` `expr` ``/`` *`expr` ``. In case of reference, the value is used. In case of `` `expr` ``, it is evaluated. In case of `/verb`, the returned value is used. See [Condition Verb Evaluation](#condition-verb-evaluation) for suspend and fatal handling.
```

(Also fix the existing typo "the , the value" → "the value" in the breakif lines.)

**Rationale:** Keeps each verb's description self-contained via anchor link while avoiding duplication.

**Verification:** Follow each anchor link and confirm it resolves to the shared section.

**If this fails:** Revert the one-line additions; the shared note from Step 1 remains harmless.

---

### Step 3: Update `IfDriver` pseudo-code in `impl/07_control_flow.md`

**Objective:** Add suspend/fatal handling to the `IfDriver` pseudo-code after `executeVerb`.
**Confidence:** High
**Depends on:** None (independent of spec changes)

**File:** `impl/07_control_flow.md`

**Changes:**

```
// Before (lines 108–110):
    # Evaluate subject if needed
    if subject is VerbValue:
        subject = executeVerb(subject, context)

// After:
    # Evaluate subject if needed
    if subject is VerbValue:
        result = executeVerb(subject, context)
        if result is Suspend:
            return result
        if result.isFatal:
            return result
        subject = result.value
```

**Rationale:** Matches what the C# `IfDriver` already implements. Pseudo-code should reflect actual semantics.

**Verification:** Compare with C# `IfDriver` (lines 26–31 of `IfDriver.cs`).

**If this fails:** Revert to original two-line form.

---

### Step 4: Update `WhileDriver` pseudo-code in `impl/07_control_flow.md`

**Objective:** Add suspend/fatal handling to the `WhileDriver` condition evaluation.
**Confidence:** High
**Depends on:** None

**File:** `impl/07_control_flow.md`

**Changes:**

```
// Before (lines 217–218):
        if subject is VerbValue:
            subject = executeVerb(subject, context)

// After:
        if subject is VerbValue:
            result = executeVerb(subject, context)
            if result is Suspend:
                return result
            if result.isFatal:
                return result
            subject = result.value
```

**Rationale:** Aligns with the spec change (Step 1) and the proposed C# implementation.

**Verification:** Confirm pseudo-code propagates suspend and fatal before proceeding to comparison.

**If this fails:** Revert to original two-line form.

---

### Step 5: Update `shouldBreak` pseudo-code in `impl/07_control_flow.md`

**Objective:** Add suspend/fatal handling to the `shouldBreak` helper after `executeVerb`.
**Confidence:** High
**Depends on:** None

**File:** `impl/07_control_flow.md`

**Changes:**

```
// Before (lines 52–66):
shouldBreak(condition, context): bool | fatal
    if condition is ReferenceValue:
        value = resolve(condition, context)
    elif condition is ExpressionValue:
        value = evaluate(condition, context)
    elif condition is VerbValue:
        value = executeVerb(condition, context)
    else:
        value = condition

    # Validate condition is boolean or nothing
    if value is not BoolValue and not value.isNothing():
        return fatal("invalid_type", "Break condition must be boolean or nothing, got: " + value.getType())

    return value.toBool()

// After:
shouldBreak(condition, context): bool | Suspend | fatal
    if condition is ReferenceValue:
        value = resolve(condition, context)
    elif condition is ExpressionValue:
        value = evaluate(condition, context)
    elif condition is VerbValue:
        result = executeVerb(condition, context)
        if result is Suspend:
            return result
        if result.isFatal:
            return result
        value = result.value
    else:
        value = condition

    # Validate condition is boolean or nothing
    if value is not BoolValue and not value.isNothing():
        return fatal("invalid_type", "Break condition must be boolean or nothing, got: " + value.getType())

    return value.toBool()
```

Update the callers (`SequenceDriver`, `LoopDriver`, `ForeachDriver` pseudo-code) to propagate the non-bool returns:

```
// Before:
        if shouldBreak(breakCondition, context):
            break

// After:
        breakResult = shouldBreak(breakCondition, context)
        if breakResult is Suspend:
            return breakResult
        if breakResult is fatal:
            return breakResult
        if breakResult == true:
            break
```

**Rationale:** The pseudo-code must reflect the suspend/fatal propagation contract specified in Step 1.

**Verification:** Trace each caller to confirm suspend and fatal are surfaced, not swallowed.

**If this fails:** Revert `shouldBreak` and its callers to original form.

---

## Verification Plan

### Automated Checks
- [x] No automated checks for spec markdown — manual review only.

### Manual Verification
- [x] Shared "Condition Verb Evaluation" section is reachable from each verb's anchor link.
- [x] All six verb parameter lines include the cross-reference.
- [x] Pseudo-code in `impl/07_control_flow.md` handles suspend and fatal in all three sites (`IfDriver`, `WhileDriver`, `shouldBreak` + callers).
- [x] No new diagnostic codes or parameter changes introduced.

### Acceptance Criteria Validation

| Criterion | How to Verify | Expected Result |
|-----------|---------------|-----------------|
| Shared behavior note exists | Read `spec/2_verbs.md` § Condition Verb Evaluation | Section covers complete, suspend, fatal |
| Each flow verb references note | Search for "Condition Verb Evaluation" in 2_verbs.md | 6 references (if, while, switch×2, breakif×3) |
| Pseudo-code handles suspend | Read `shouldBreak`, `IfDriver`, `WhileDriver` in 07_control_flow.md | `result is Suspend → return` pattern present |
| Pseudo-code handles fatal | Same three sites | `result.isFatal → return` pattern present |

---

## Rollback Plan

1. Revert the "Condition Verb Evaluation" section in `spec/2_verbs.md`.
2. Revert the cross-reference additions in each verb's parameter line.
3. Revert the three pseudo-code blocks in `impl/07_control_flow.md` to their original form.

---

## Notes

### Risks
- **Spec over-specification:** Embedding suspend/fatal handling in condition semantics could constrain future runtime designs that handle these differently (e.g. compile-time analysis). Mitigation: the note describes runtime behavior, which is the spec's scope; compile-time is a separate layer.

### Open Questions
- None. The `/try` Suspension Transparency principle and the fatal-terminates-context contract are already established.
