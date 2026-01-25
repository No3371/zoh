# Walkthrough: spec-CORE-004-unrolling-syntax

**Status**: Completed & Verified
**Date**: 2026-01-25

## 1. Objective

Updated the ZOH specification to rationalize the unrolling syntax in `/interpolate`. Changed the inconsistent `{...}` syntax to `${...}` to align with standard variable interpolation.

## 2. File Changes

### `spec.md`

| Location | Change |
|----------|--------|
| ~line 707 | Updated unrolling syntax definition from `{*var..."delim"}` to `${*var..."delim"}`. |

## 3. Verification

Verified the change in `spec.md` by reading the file.

```markdown
707: - Unrolling: `${*var..."delim"}` to expand `*[list]` and `*{map}` into `{element}{delim}{element}...` where `element`s are the list elements or map k:v pairs.
```

## 4. Key Insights

- This change unifies the interpolation syntax, making the language more consistent.
- The C# implementation (`impl-CORE-004`) will need to be updated to support this new syntax and likely drop support for the old one (or keep it as deprecated if we cared about backward comp, but here we decided to fix it as a bug).
