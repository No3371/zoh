# spec-CORE-004: Fix Unrolling Syntax

## Priority
**Medium** - Fixes a syntactic inconsistency in the specification and aligns unrolling with standard interpolation syntax.

## Problem

The current specification for unrolling in `/interpolate` uses the `{...}` syntax, which is inconsistent with the standard `${...}` syntax used for variable interpolation.

- `spec.md:707` - Defines unrolling as `{*var..."delim"}`.

This inconsistency risks confusion and deviates from the pattern established by other interpolation features (e.g., `${*var}`, `$#{*var}`).

## Proposed Change

Update the specification to use `${...}` for unrolling, ensuring consistency across all interpolation features.

### Specification Updates

#### `spec.md`
- Update line 707 to define unrolling as `${*var..."delim"}`.

## Acceptance Criteria
1. `spec.md` clearly defines unrolling syntax as `${*var..."delim"}`.
2. No other interpolation syntax is affected.

## Risks & Unknowns
- **Breaking Change**: This changes the documented syntax, but since the feature is relatively new/unstable, it's safer to align it now.
