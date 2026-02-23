# Assert Verb Spec Addition

> **Status:** Complete
> **Created:** 2026-02-23
> **Author:** agent
> **Source:** Direct request
> **Related Projex:** 20260222-specs-nav.md
> **Patch Document:** [20260223-assert-verb-spec-patch.md](file:///s:/repos/zoh/projex/closed/20260223-assert-verb-spec-patch.md)

---

## Summary

Add a new `Core.Assert` verb to the ZOH language spec, placed immediately after the Debug Verbs section in `spec/2_verbs.md`. The verb provides a first-class assertion primitive that emits a fatal diagnostic when a condition is not met, enabling script authors to express invariants that halt context execution on failure.

**Scope:** Spec-only — one section added to `spec/2_verbs.md`
**Estimated Changes:** 1 file, ~40 lines

---

## Objective

### Problem / Gap / Need

ZOH has Debug Verbs (`/info`, `/warning`, `/error`, `/fatal`) for emitting diagnostics and `/try` for error recovery, but no dedicated assertion verb. Currently, expressing an invariant requires a multi-verb pattern:

```zoh
/if `*health < 0`, /fatal "health invariant violated";;
```

A first-class `/assert` verb makes invariants self-documenting, reduces boilerplate, and establishes a clear convention for defensive scripting. It naturally belongs alongside the Debug Verbs as a diagnostic-emitting verb.

### Success Criteria
- [x] [PATCHED] `Core.Assert` section added to `spec/2_verbs.md` after the Debug Verbs section (after L1256)
- [x] [PATCHED] Spec follows the same structure/conventions as other verb specs (Parameters, Diagnostics, Returns, Examples)
- [x] [PATCHED] Diagnostic codes are consistent with existing conventions
- [x] [PATCHED] Parameter acceptance rules follow established spec patterns

### Out of Scope
- Implementation in the C# runtime (separate plan in `c#/projex/`)
- Implementation spec in `impl/` (separate plan in `impl/projex/`)
- Changes to `AGENT.md` quick-reference table (should follow in a separate update)
- Syntactic sugar forms

---

## Context

### Current State

The Debug Verbs section (L1239–1256) is a group spec covering `/info`, `/warning`, `/error`, `/fatal`. These verbs emit diagnostics at varying severity levels. `/fatal` terminates the context.

The verb immediately following Debug Verbs is `Core.Roll` at L1258.

Nearby related verbs:
- `Core.Try` (L179) — downgrade fatals to errors, error recovery
- `Core.Diagnose` (L165) — retrieve diagnostics from last verb call
- `Core.If` (L568) — conditional execution

### Key Files
| File | Purpose | Changes Needed |
|------|---------|----------------|
| `spec/2_verbs.md` | Core verb specifications | Add `Core.Assert` section after Debug Verbs (after L1256) |

### Dependencies
- **Requires:** None — spec addition is self-contained
- **Blocks:** C# runtime implementation plan, implementation spec update

### Constraints
- Must follow existing spec structure for verb documentation
- Parameter acceptance types must use the established notation (`*reference`, `` `expr` ``, `/verb`, etc.)
- Diagnostic codes must follow the `snake_case` convention used throughout

---

## Implementation

### Overview

Insert a new `### Core.Assert` section into `spec/2_verbs.md` immediately after the Debug Verbs code block (after L1256, before the blank line at L1257).

### Step 1: Add Core.Assert Spec Section

**Objective:** Define the assert verb specification

**Files:**
- `spec/2_verbs.md`

**Changes:**

Insert the following after L1256 (the closing ``` of Debug Verbs examples):

```markdown

### Core.Assert

An assert verb checks a condition and emits a fatal diagnostic with the given message if the condition is not met (falsy: `false` or `nothing`). If the condition is met (truthy: anything that is not falsy), no diagnostic is emitted.

#### Named Parameters
- `is`: Value to be compared to the subject. Optional. Default to `true`. Accept `any`. In case of reference, the value is used. In case of `` `expr` ``, it is evaluated.

#### Parameters
- `subject`: The condition to assert. Accept `any`. In case of reference, the value is used. In case of `` `expr` ``, it is evaluated. In case of `/verb`, the returned value is used.
- `message`: The message to emit on failure. Accept `"string"`, `*"string"`, `` `expr` ``, or `` *`expr` ``. In case of reference, the value is used. In case of string, the value is interpolated ONCE. In case of `` `expr` ``, the expression is evaluated. Optional — defaults to `"assertion failed"`.

#### Diagnostics
- Fatal: `assertion_failed`: The asserted condition was falsy. Includes the evaluated message.

#### Returns
A nothing.

#### Examples
```
/assert *is_valid;
/assert *health, "health must be truthy";
/assert `*health > 0`, "health must be positive: ${*health}";
/assert *mode, is: "combat", "expected combat mode";
/assert /has *inventory, "sword";, "player must have sword";
```
```

**Rationale:**
- **Parameter pattern** mirrors `/if` for `subject` + `is:` — familiar to ZOH authors
- **Message parameter** follows Debug Verbs convention (accepts same types, interpolated ONCE)
- **Diagnostic code** `assertion_failed` is a new code specific to this verb, following `snake_case` convention
- **Returns nothing** — consistent with `/fatal` and Debug Verbs
- **Falsy definition** — `false` or `nothing`, consistent with how `/if` evaluates conditions

**Verification:** Read the file and confirm section is syntactically consistent with surrounding sections.

---

## Verification Plan

### Automated Checks
- [ ] N/A — this is a markdown spec change, no code to test

### Manual Verification
- [ ] Read the inserted section and verify heading level (`###`) matches siblings
- [ ] Verify parameter notation follows conventions (backtick-quoted expressions, `*reference` syntax)
- [ ] Verify examples are valid ZOH syntax
- [ ] Verify the section appears between Debug Verbs and Core.Roll
- [ ] Verify diagnostic code follows `snake_case` convention

### Acceptance Criteria Validation
| Criterion | How to Verify | Expected Result |
|-----------|---------------|-----------------|
| Section exists after Debug Verbs | View file around L1257 | `### Core.Assert` heading visible |
| Structure matches convention | Compare with adjacent verb specs | Same subsections: Parameters, Diagnostics, Returns, Examples |
| Diagnostic codes consistent | Grep for `assertion_failed` | Uses `snake_case`, appears in Diagnostics section |

---

## Rollback Plan

If the spec addition needs to be reverted:

1. `git revert <commit>` — single-file change, clean revert

---

## Notes

### Assumptions
- The assert verb belongs in the Core verb category (not Standard), since it is broadly useful across all story types
- `false` and `nothing` are the only falsy values in ZOH (consistent with `/if` behavior)
- The `is:` named parameter provides value-comparison assertions (like `/if`'s `is:` parameter), not just truthy checks

### Risks
- **Semantic overlap with `/fatal`**: Mitigated — `/assert` is conditional (only fires on failure), while `/fatal` always fires. They serve different purposes: invariant checking vs. unconditional termination.

### Open Questions
- None
