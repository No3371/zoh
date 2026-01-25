# spec-CORE-003: Fix Interpolation Semantics for References

## Priority
**High** - Corrects the semantics of `$*var` introduced in `spec-CORE-002` before implementation proceeds too far.

## Problem
In `spec-CORE-002`, the syntax `$*var` was defined as "stringifying" the variable. However, resolving a variable to its string representation is often redundant or can be achieved implicitly. The intended utility of explicit `$*var` syntax in expressions is to **interpolate** the value of the variable (treating the value as a template string).

## Proposed Change
Update the specification to define `$*var` as:
1. Resolve the value of `*var`.
2. Treat the value as a string template.
3. Perform interpolation on that template.
4. Return the resulting string.

**Note**: `$"string"` behavior remains unchanged (it is an interpolated string literal).

## Specification Updates

### `spec.md`
- Update the "Interactions & Edge Cases" or "Core.Evaluate" section to clarify that `$*var` triggers interpolation on the variable's value.
- Add example:
  ```
  *template <- "Hello ${*name}";
  *name <- "World";
  /eval `1 + $*template` -> "1Hello World" (Wait, implicit string concat?)
  /eval `$*template` -> "Hello World"
  ```
  *Correction via spec rules*: `1 + "string"` is valid string concat.

### `impl/04_expressions.md`
- Update comments in `ExpressionEvaluator` pseudocode or description for `InterpolateForm`.
- Clarify that for `InterpolateForm` with a reference, the value must be interpolated.

## Acceptance Criteria
1. `spec.md` clearly states `$*var` interpolates the value.
2. `impl/04_expressions.md` reflects this semantic.
