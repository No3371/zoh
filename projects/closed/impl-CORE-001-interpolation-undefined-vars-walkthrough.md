# Walkthrough: impl-CORE-001-interpolation-undefined-vars

**Status**: Completed & Verified
**Date**: 2026-01-25

## 1. Objective
Executed the implementation guide updates for "Relaxed Interpolation & Undefined Variables". This project ensures that `Core.Interpolate` is implemented to treat undefined variables as `nothing` (or fallbacks) rather than crashing, while maintaining strictness for expressions.

## 2. File Changes

### `impl/06_core_verbs.md`
- Updated `InterpolateDriver.execute` pseudocode to accept `fallback` parameter.
- Updated `parseInterpolation` and `parseBasicInterpolation` signatures to pass `fallback`.
- Added logic comments to handle undefined variables by returning fallback.

### `impl/impl-CORE-001-interpolation-undefined-vars.md`
- Restored/Created the detailed implementation guide for this specific feature as a reference for C# implementation.

## 3. Verification
- Checked `impl/06_core_verbs.md`: The pseudocode now accurately reflects the spec change regarding `fallback` and parsing.
- Checked `impl/impl-CORE-001...`: Consistent with the project description.

## 4. Key Insights
- The separation of "Interpolation Parser" vs "Expression Evaluator" is key. The implementation must ensure that strict "undefined variable" checks are skipped/caught only within the `${...}` parsing context, not globally.
