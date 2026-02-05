# Plan: Pipe-Delimited Macro System (Spec & Docs)

> **Status:** Complete
> **Created:** 2026-02-06
> **Source:** [proposal-core-002-macro-pipe-syntax.md](./proposal-core-002-macro-pipe-syntax.md)
> **Related Projex:** [20260206-proposal-core-002-macro-pipe-syntax-review.md](./20260206-proposal-core-002-macro-pipe-syntax-review.md)

---

## Summary

Phase 1 of the Macro System Redo. Updates the language specification (`spec.md`) and implementation notes (`impl/03_preprocessor.md`) to define the new pipe-delimited `|%...|%|` syntax.

**Scope:** `spec.md`, `impl/03_preprocessor.md`
**Estimated Changes:** 2 files modified

---

## Objective

### Problem / Gap / Need

The current spec references `#macro`/`#expand` syntax which is being replaced. The specification must be updated as the single source of truth before implementation begins.

### Success Criteria

- [ ] `spec.md`: Existing "Macro" section updated to define `|%...|%|` syntax
- [ ] `impl/03_preprocessor.md`: Implementation guide updated to reflect new syntax and logic
- [ ] No references to legacy `#macro`/`#expand` syntax remain in docs

### Out of Scope

- C# Runtime implementation (covered in separate plan)
- Test updates (covered in separate plan)

---

## Context

### Key Files

| File | Purpose | Changes Needed |
|------|---------|----------------|
| `spec.md` | Language Specification | Update Macro section (lines ~1827) |
| `impl/03_preprocessor.md` | Implementation Guide | Replace logic descriptions |

---

## Implementation Steps

### Step 1: Update Language Specification

**File:** `spec.md`

Replace the "Macro" section with:

```markdown
## Macro

The language supports story body templating with macros.

Macros are defined using the pipe-delimited syntax `|%NAME%|...|%NAME%|`.

### Definition
```zoh
|%MACRO_NAME%|
<body with placeholders>
|%MACRO_NAME%|
```

### Expansion
```zoh
|%MACRO_NAME|arg0|arg1|...|%|
```

### Placeholders

| Pattern | Meaning |
|---------|---------|
| `|%N|` | Argument at position N (0-indexed) |
| `|%|` | Next argument (auto-increment) |
| `|%+N|` | Relative: current + N |
| `|%-N|` | Relative: current - N |
| `\|` | Escaped pipe (literal) |
```

### Step 2: Update Implementation Spec

**File:** `impl/03_preprocessor.md`

- Update "Directive Syntax" section to replace `#macro` and `#expand` with new forms.
- Update "Macro Collection" logic description (regex patterns for `|%NAME%|`).
- Update "Macro Expansion" logic description (pipe splitting, placeholder replacement).
- Update "Placeholder Syntax" table.
- Update "Error Handling" examples.

---

## Verification Plan

### Manual Verification
- Review diff of `spec.md` to ensure clarity and correctness.
- Review diff of `impl/03_preprocessor.md` to ensure logic descriptions match the new spec.

---

## Rollback Plan

- Revert file changes using git.
