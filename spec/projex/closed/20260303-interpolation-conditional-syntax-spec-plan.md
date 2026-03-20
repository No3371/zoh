# Interpolation Conditional Syntax Update - Spec Plan

> **Status:** Complete
> **Created:** 2026-03-03
> **Walkthrough:** [20260303-interpolation-conditional-syntax-spec-walkthrough.md](20260303-interpolation-conditional-syntax-spec-walkthrough.md)
> **Author:** Antigravity
> **Related Projex:** [csharp/projex/20260303-interpolation-conditional-syntax-csharp-plan.md](../../csharp/projex/20260303-interpolation-conditional-syntax-csharp-plan.md)

---

## Summary

This plan updates the ZOH specification for conditional string interpolation `$?{cond}` to use the colon `:` instead of the pipe `|` for the false branch. It also clarifies that interpolation text format suffixes (like `,width:format`) are permitted alongside feature syntax elements like `$?{cond}`.

**Scope:** `spec/` folder ONLY.
**Estimated Changes:** 1 file modified.

---

## Objective

### Problem / Gap / Need
The interpolated conditional syntax `$?{*cond? *true_case | *false_case}` recorded in the spec is asymmetrical to its expression counterpart `$?(cond ? true : false)`. Furthermore, the specification claims that feature syntaxes (like `$?{}`) cannot be combined with formatting suffixes (`,-width:F2`). This creates an unnecessary user constraint forcing nested interpolations (e.g. `${$?(*cond ? A : B), -4}`) that the language should natively handle.

### Success Criteria
- [ ] `spec/2_verbs.md` defines `$?{cond ? true_case : false_case}`.
- [ ] The clause forbidding formatting with feature interpolation syntaxes is removed/softened so that cases like `$?{cond?A:B, -4}` are valid according to the spec.

### Out of Scope
- Code changes to the C# implementation (covered in the downstream C# plan).

---

## Context

### Current State
`spec/2_verbs.md` instructs using `|` for the else branch in interpolation, and explicitly excludes formatting functionality from special interpolation blocks (like `$?{...}`).

### Key Files
| File | Purpose | Changes Needed |
|------|---------|----------------|
| `spec/2_verbs.md` | Language documentation | Update interpolation examples and restrictions. |

### Dependencies
- **Requires:** None
- **Blocks:** `csharp/projex/20260303-interpolation-conditional-syntax-csharp-plan.md`

### Constraints
- The syntax must clearly match standard C#/ZOH expression syntax where `:` separates ternary paths.

---

## Implementation

### Overview
Modify `spec/2_verbs.md` to establish `:` as the authoritative conditional else separator and allow composition with formatting width/format suffixes.

### Step 1: Update Conditional Spec
**Objective:** Replace `|` with `:` in interpolation conditional docs.
**Files:**
- `spec/2_verbs.md`

**Changes:**
```markdown
// Before:
- Parity of `/if` for `$?{*cond? *true_case | *false_case}`.
    - Example: `/itpl "You $?{*win? "win"|"lose"}."` is `"You win."`

// After:
- Parity of `/if` for `$?{*cond? *true_case : *false_case}`.
    - Example: `/itpl "You $?{*win? "win":"lose"}."` is `"You win."`
```

### Step 2: Remove Formatting Prohibition
**Objective:** Enable formatting on special form interpolation blocks.
**Files:**
- `spec/2_verbs.md`

**Changes:**
```markdown
// Before:
- Formatting: C# composite format parity `${var[,width][:formatString]}`.
    ...
    - This can not be used with the following feature syntaxes.

// After:
- Formatting: C# composite format parity `${var[,width][:formatString]}`.
    ...
    - This formatting suffix can also be applied to the feature syntaxes below (e.g., `$?{*win? 'W' : 'L', -4}`).
```

---

## Verification Plan

### Manual Verification
- [ ] Review `spec/2_verbs.md` renders correctly.
- [ ] Ensure no lingering references to `$?{...|...}` conditional strings exist.

### Acceptance Criteria Validation
| Criterion | How to Verify | Expected Result |
|-----------|---------------|-----------------|
| `:` Used for Ternary | Visual inspection | The `:` properly separates the `then` and `else` parts of interpolation in `spec/2_verbs.md`. |
| Formatting allowed | Visual inspection | The restriction on the formatting clause is reversed. |

---

## Rollback Plan
1. Revert `spec/2_verbs.md`.

---

## Notes
- None.
