# Walkthrough: spec-CORE-002-interpolation-syntax

**Status**: Completed & Verified
**Date**: 2026-01-25

## 1. Objective

Refined the interpolation syntax in expressions to distinguish between string interpolation (`$"string"`), variable interpolation (`$*var`), and option lists (`$(...)`). This removes the overloading of `$(...)` and aligns with modern language standards.

## 2. File Changes

### `spec.md`

| Location | Change |
|----------|--------|
| ~line 754 | Replaced `$(string)` with `$"string"` and `$*var` in usage list. |
| ~line 800 | Added "Interactions & Edge Cases" section detailing precedence and escaping. |

### `impl/04_expressions.md`

| Location | Change |
|----------|--------|
| ~line 27 | Updated EBNF to add `interpolate_string` (`$"..."`) and `interpolate_var` (`$*...`). |
| ~line 126 | Added `DOLLAR_STRING` and `DOLLAR_REF` tokens. |
| ~line 220 | Updated Parser logic to handle new tokens in `parsePrimary`. |
| ~line 325 | Updated Evaluator comments to reflect new handling. |
| ~line 464 | Updated Testing Checklist with new syntax. |

## 3. Verification

Verified that the specification and implementation guide now clearly distinguish between the new interpolation syntax and the existing option list syntax. The EBNF is unambiguous.

## 4. Key Insights

- Separating interpolation from option lists (`$(...)`) simplifies the grammar significantly.
- `$"..."` is a familiar pattern for string interpolation.
