# Walkthrough: spec-CORE-001-interpolation-undefined-vars

**Status**: Completed & Verified
**Date**: 2026-01-25

## 1. Objective

Relaxed the strictness of `Core.Interpolate` and `${...}` string interpolation. Previously, referencing an undefined variable caused a fatal error. Now, it resolves to `nothing` (defaulting to `"?"`), allowing for more robust text generation. Introduced a `fallback` parameter to `Core.Interpolate` to customize this replacement value.

## 2. File Changes

### `spec.md`

| Location | Change |
|----------|--------|
| `Core.Interpolate` | Updated description to state undefined variables resolve to `nothing`/fallback. |
| `Core.Interpolate` | Added `fallback` named parameter documentation. |
| `Core.Interpolate` | Removed `undefined_var` from Fatal Diagnostics list. |
| `Core.Interpolate` | Added examples demonstrating leniency and fallback usage. |

### `impl/impl-CORE-001-interpolation-undefined-vars.md`

- Created new implementation guide detailing the logic.
- Clarified the distinction between lenient interpolation (`${*var}`) and strict expression evaluation (`$(*var)`).

## 3. Verification

Verified that the spec changes are consistent:
- `Core.Evaluate` remains strict (no changes made, consistent with goal).
- `expr.md` remains strict (no changes made).
- `Core.Interpolate` examples clearly show the new behavior.

## 4. Key Insights

- **Strictness Boundary**: It was crucial to define that `$(expression)` within `expr.md` remains strict. Leniency only applies when the runtime is purely "interpolating a string template", not "evaluating logic".
- **Fallback Power**: The global `fallback` parameter is a simple but powerful addition for localization or debugging (e.g. setting fallback to `"[MISSING]"`).
