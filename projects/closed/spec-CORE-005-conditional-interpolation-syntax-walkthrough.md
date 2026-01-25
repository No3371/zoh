# Walkthrough: spec-CORE-005-conditional-interpolation-syntax

**Status**: Completed & Verified
**Date**: 2026-01-25

## 1. Objective

Updated the ZOH specification to standardize the conditional interpolation syntax. Changed the separator from `:` to `|` (e.g., `$?{c? a|b}`) to align with the `any` syntax and prevent future conflicts with format strings.

## 2. File Changes

### `spec.md`
- **Line 710**: Updated syntax definition to `$?{*cond? *true_case | *false_case}`.

### `impl/04_expressions.md`
- **Line 29**: Updated grammar rule: `conditional := '$?(' expr '?' expr '|' expr ')'`.

### `impl/06_core_verbs.md`
- **Line 315**: Updated table example to `$?{*a? *b | *c}`.

## 3. Verification

Verified the changes by reading the files:
- `spec.md` confirms `|` usage in description and examples.
- `impl/04_expressions.md` confirms `|` in EBNF grammar.
- `impl/06_core_verbs.md` confirms `|` in syntax table.

## 4. Key Insights

- This change unifies the "choice" operator syntax in ZOH (`|` is now consistently used for alternatives in `any` and `if/else`).
- `impl` code (C#) will need to be updated next (`impl-CORE-005`) to enforce this new syntax.
