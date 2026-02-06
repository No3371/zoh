# Plan: Macro Spec Verification & Alignment

> **Status:** Complete
> **Completed:** 2026-02-06
> **Walkthrough:** [closed/20260206-macro-spec-verification-walkthrough.md](./closed/20260206-macro-spec-verification-walkthrough.md)
> **Created:** 2026-02-06
> **Author:** Agent
> **Source:** [proposal-macro-redo.md](./proposal-macro-redo.md)
> **Related Projex:** 
> - [c#/projex/20260206-macro-impl-verification-plan.md](../c#/projex/20260206-macro-impl-verification-plan.md) (C# implementation)

---

## Summary

Verifies and aligns spec documents (`spec.md`, `impl/03_preprocessor.md`) against the original macro proposal and user decisions. Ensures specification is complete before implementation work proceeds.

**Scope:** `spec.md`, `impl/03_preprocessor.md`
**Estimated Changes:** 2 files modified

---

## Objective

### Problem / Gap / Need

The original macro proposal (`proposal-macro-redo.md`) defines features that may not be fully or accurately documented in the specs. Additionally, specific behaviors for trimming and escaping need to be codified based on user review.

1. **Indentation preservation** — In proposal and `spec.md`, but not in `impl/03_preprocessor.md`
2. **Symmetric trimming** — Smart trimming logic needs to be documented.
3. **`\%` escaping** — Required for safety.
4. **Multiline `|PARAM` syntax** — In proposal, partially in spec.
5. **Missing Argument Behavior** — Needs to be clearly defined as empty string.

### Success Criteria

- [ ] `spec.md` macro section covers all proposal features
- [ ] `impl/03_preprocessor.md` matches `spec.md` 
- [ ] No ambiguity between spec documents
- [ ] Document `\%` escape 
- [ ] Document indentation preservation in impl spec
- [ ] Document symmetric trimming logic (`min(head, tail)`)

### Out of Scope

- C# runtime implementation (covered in separate projex)
- Test updates (covered in separate projex)

---

## Context

### Key Files

| File | Purpose | Changes Needed |
|------|---------|----------------|
| `spec.md` | Language spec | Add trimming, clarify missing arg behavior, document `\%` |
| `impl/03_preprocessor.md` | Implementation guide | Add indentation, add multiline syntax, align missing arg, trimming, escaping |

---

## Implementation

### Step 1: Clarify Missing Argument Behavior

**Decision Required:** The proposal says missing args -> empty string. The impl spec says -> `?`.

**Recommendation:** Follow proposal (empty string) as it's simpler and matches common template behavior.

**Files:** `spec.md`, `impl/03_preprocessor.md`

**Changes:**
- `spec.md` line 1858: Clarify "replaced with empty string"
- `impl/03_preprocessor.md` line 105, 230: Change `?` to empty string

---

### Step 2: Add Indentation Preservation to Impl Spec

**File:** `impl/03_preprocessor.md`

**Changes:** Add new section after line 237:

```markdown
### Indentation Preservation

The macro expansion preserves the indentation of the call site. All lines in the expanded body are prefixed with the whitespace found before the `|%` token.

```zoh
    |%LOGIC|%|
```

If `LOGIC` macro body is:
```zoh
/if *condition,
    /do_something;
;
```

Expanded result:
```zoh
    /if *condition,
        /do_something;
    ;
```
```

---

### Step 3: Add Multiline Parameter Syntax to Impl Spec

**File:** `impl/03_preprocessor.md`

**Changes:** Add documentation after line 105:

```markdown
### Multiline Parameters

Parameters can span multiple lines. Each continuation line starts with `|`:

```zoh
|%MACRO|arg1
|arg2
|arg3|%|

or

|%MACRO|arg1
|arg2
|arg3
|%|
```

This is equivalent to `|%MACRO|arg1|arg2|arg3|%|`.
```

---

### Step 4: Add Symmetric Trimming Documentation

**Files:** `spec.md`, `impl/03_preprocessor.md`

**Changes:** Document the **Smart Symmetric Trimming** logic:
- Calculate `leading_spaces` count.
- Calculate `trailing_spaces` count.
- `trim_amount = min(leading_spaces, trailing_spaces)`.
- Remove `trim_amount` from both start and end of the argument string.

**Example:**
```zoh
|%MACRO|  A  |%|  :: "  A  " (2 lead, 2 trail) -> Trim 2 -> "A"
|%MACRO| A |%|    :: " A "   (1 lead, 1 trail) -> Trim 1 -> "A"
|%MACRO| A  |%|   :: " A  "  (1 lead, 2 trail) -> Trim 1 -> "A "
|%MACRO|  A |%|   :: "  A "  (2 lead, 1 trail) -> Trim 1 -> " A"
```

---

### Step 5: Document `\%` Escape

**Files:** `spec.md`, `impl/03_preprocessor.md`

**Changes:** 
- Document that `\%` is processed as a literal `%` and avoids triggering macro syntax if necessary.
- Helps escape sequences meant to look like macro closers (`%|`).

---

## Verification Plan

### Manual Verification

1. Read both spec files after changes.
2. Verify all 5 implementation steps are accurately reflected.

### Acceptance Criteria Validation

| Criterion | How to Verify | Expected Result |
|-----------|---------------|-----------------|
| Indentation in impl | Read impl spec | Section exists |
| Smart Trimming | Read both specs | Defined as `min(L, T)` |
| `\%` Escape | Read both specs | Documented |
| Missing Args | Read both specs | "Empty string" |

---

## Rollback Plan

- Revert changes: `git checkout -- spec.md impl/03_preprocessor.md`
