# Assert Verb Implementation Spec

> **Status:** Ready
> **Created:** 2026-02-23
> **Author:** agent
> **Source:** Direct request (followup for @[impl])
> **Related Projex:** [20260223-assert-verb-spec-patch.md](file:///s:/repos/zoh/projex/closed/20260223-assert-verb-spec-patch.md), 20260222-specs-nav.md

---

## Summary

Add the implementation specification for the new `Core.Assert` verb to `impl/06_core_verbs.md`. The spec will outline the `AssertDriver.execute` pseudocode, handling the condition evaluation, `is:` named parameter matching, and fatal diagnostic emission with optional string interpolation.

**Scope:** `impl` documentation only — one file modified
**Estimated Changes:** 1 file, ~40 lines

---

## Objective

### Problem / Gap / Need
The language specification was recently updated to include `Core.Assert` (in `spec/2_verbs.md`). The implementation specification in `impl/` needs to match this addition so runtime authors have a reference for its internal behavior (such as `resolveValue`, comparison semantics, and diagnostic emission).

### Success Criteria
- [ ] `Core.Assert` section added to `impl/06_core_verbs.md` after the `Debug Verbs` section.
- [ ] Pseudocode clearly defines how `subject` and `is` parameters are evaluated and compared.
- [ ] Pseudocode defines how the `message` string is resolved and interpolated exactly ONCE on failure.
- [ ] `Core.Assert` (and optionally Debug Verbs) added to the `Testing Checklist` at the end of the file.

### Out of Scope
- Actually implementing `Core.Assert` in the C# runtime.
- Modifying any other verb implementation.

---

## Context

### Current State
`Core.Assert` is documented in `spec/2_verbs.md` but missing from `impl/06_core_verbs.md`. `If` and `Debug` verbs have implementation specs that demonstrate how boolean checking, equality comparison, and message interpolation are handled.

### Key Files
| File | Purpose | Changes Needed |
|------|---------|----------------|
| `impl/06_core_verbs.md` | Core verb implementations | Add `Core.Assert` block and testing checklist items. |

### Dependencies
- **Requires:** `projex/closed/20260223-assert-verb-spec-patch.md` (Completed)
- **Blocks:** C# Runtime implementation plan (to be created later in `c#/projex/`)

### Constraints
- Pseudocode must follow the conventions set in `impl/`.
- Ensure the `is` parameter defaults to `true` (boolean), matching the language specification and `If` verb approach.

---

## Implementation

### Overview
Insert the `.Assert` implementation block into `impl/06_core_verbs.md` after the `Debug Verbs` section (around L751). Also update the `Testing Checklist` at the bottom of the file to add a `Debug & Assert Verbs` category to ensure runtimes create compliance tests.

### Step 1: Add Core.Assert Implementation Spec

**Objective:** Add the pseudocode for `/assert`.

**Files:**
- `impl/06_core_verbs.md`

**Changes:**
Insert the following after the `Debug Verbs` section (before `Core.Has`):

```markdown
---

## Core.Assert

**Purpose**: Assert a condition matches a value (truthy by default) and emit a fatal diagnostic on failure.

### Signature
```
/assert subject, is:value?, message?;
```

### Implementation

```
AssertDriver.execute(call, context):
    subject = resolveValue(call.params[0], context)
    compareValue = getNamedParam(call, "is", BoolValue(true))
    
    # Evaluate subject if needed
    if subject is VerbValue:
        subject = executeVerb(subject, context)
    if subject is ExpressionValue:
        subject = evaluate(subject, context)
        
    # Validate subject type when comparing to default (true)
    if compareValue == BoolValue(true):
        if subject is not BoolValue and not subject.isNothing():
            return fatal("invalid_type", "Condition must be boolean or nothing, got: " + subject.getType())

    compareValue = resolveValue(compareValue, context)
    matches = equals(subject, compareValue)
    
    if not matches:
        message = "assertion failed"
        if call.params.length > 1:
            msgParam = resolve(call.params[1], context)
            if msgParam is ExpressionValue:
                msgParam = evaluate(msgParam, context)
            if msgParam is StringValue:
                message = interpolate(msgParam.value, context)
            else:
                message = msgParam.toString()
                
        return fatal("assertion_failed", message)
        
    return ok()
```
```

**Rationale:** The pseudocode correctly demonstrates that `subject` evaluation, type checking against default `true`, and equality checks mirror the `If` verb semantics. The message evaluation and interpolation mirror the `Debug` verbs semantics.

**Verification:** Read the file to ensure the section is formatted correctly and placed appropriately.

### Step 2: Add Testing Checklist Items

**Objective:** Ensure Assert and Debug verbs are represented in the compliance checklist.

**Files:**
- `impl/06_core_verbs.md`

**Changes:**
Add the following to the `Testing Checklist` section at the end of the file:

```markdown
### Debug & Assert Verbs
- [ ] Debug verbs emit appropriate diagnostic severities
- [ ] Debug verbs interpolate message once
- [ ] Assert passes on truthy (or matching `is:`)
- [ ] Assert fatals on falsy (or mismatching `is:`)
- [ ] Assert evaluates and interpolates message only on failure
- [ ] Assert returns `assertion_failed` fatal code
```

**Rationale:** Runtimes need to know what compliance tests to write for these verbs since they were omitted.

**Verification:** Ensure it is added properly to the list.

---

## Verification Plan

### Automated Checks
- [ ] N/A for markdown changes

### Manual Verification
- [ ] Read `impl/06_core_verbs.md` to ensure `Core.Assert` appears after `Debug Verbs`
- [ ] Verify checklist renders correctly in Markdown

### Acceptance Criteria Validation
| Criterion | How to Verify | Expected Result |
|-----------|---------------|-----------------|
| `Core.Assert` section added | `cat` `impl/06_core_verbs.md` | Contains `## Core.Assert` and pseudocode block |
| `Testing Checklist` updated | `cat` `impl/06_core_verbs.md` | Contains `### Debug & Assert Verbs` checklist |

---

## Rollback Plan
1. Revert the file edits using `git checkout` or reverse patches.

---

## Notes

### Assumptions
- The `is` parameter defaults to a boolean `true`, similar to `IfDriver`.
- It is acceptable to add Debug verbs to the testing checklist in the same step.
### Risks
- None
### Open Questions
- None
