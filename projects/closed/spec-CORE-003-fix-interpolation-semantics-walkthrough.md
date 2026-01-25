# Walkthrough: spec-CORE-003-fix-interpolation-semantics

**Status**: Completed & Verified
**Date**: 2026-01-25

## 1. Objective

Updated the semantics of `$*var` in the interpolation specification from "stringification" to "dynamic interpolation". This means that when `$*var` is used in an expression, the value of the variable is treated as a template string and interpolated.

## 2. File Changes

### `spec.md`

| Location | Change |
|----------|--------|
| ~line 803 | Updated "Interactions & Edge Cases" to define `$*var` as treating value as template string and interpolating it. Added example. |

### `impl/04_expressions.md`

| Location | Change |
|----------|--------|
| ~line 325 | Updated comments in `ExpressionEvaluator` pseudocode to reflect that `$*...` (ReferenceExpr) values should be interpolated recursively. |

## 3. Verification

Verified that `spec.md` explicitly describes the new behavior and distinguishes it from simple stringification. The implementation guide now directs the `ExpressionEvaluator` to handle this case correctly.

## 4. Key Insights

- This change makes `$*var` much more powerful for templating scenarios, allowing variables to contain dynamic content that is resolved at evaluation time.
- Implicit recursion: If `*a` contains `${*b}`, and `*b` contains `${*c}`, evaluating `$*a` should resolve all the way down.
