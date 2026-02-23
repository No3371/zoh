# Patch: Assert Verb Spec Addition

> **Date:** 2026-02-23
> **Author:** agent
> **Directive:** patch-projex@[s:\repos\zoh\projex\20260223-assert-verb-spec-plan.md]
> **Source Plan:** [20260223-assert-verb-spec-plan.md](file:///s:/repos/zoh/projex/20260223-assert-verb-spec-plan.md)
> **Result:** Success

---

## Summary

Added a new `Core.Assert` verb specification to `spec/2_verbs.md`, placed immediately after the Debug Verbs section. This provides a first-class assertion primitive that emits a fatal diagnostic when a condition is not met.

---

## Changes

### Specification Addition

**File:** `spec/2_verbs.md`
**Change Type:** Modified
**What Changed:**
- Added `### Core.Assert` section.
- Defined `is`, `subject` and `message` parameters.
- Added `assertion_failed` fatal diagnostic documentation.
- Appended example usage blocks.

**Why:**
To establish a clear convention for defensive scripting and avoid relying only on `/if` and `/fatal` combinations.

---

## Verification

**Method:** Manual inspection of the markdown file structure.

**Result:**
```
Section correctly placed between Debug Verbs and Core.Roll. Follows the verb spec convention with Named Parameters, Parameters, Diagnostics, Returns, and Examples sections. Diagnostic code is `assertion_failed`.
```

**Status:** PASS

---

## Impact on Related Projex

| Document | Relationship | Update Made |
|----------|-------------|-------------|
| [20260223-assert-verb-spec-plan.md](file:///s:/repos/zoh/projex/closed/20260223-assert-verb-spec-plan.md) | Source plan | Marked objective as [PATCHED], updated status to Complete, and moved to `closed/` directory. |

---

## Notes

The syntax additions follow standard interpolation patterns and use the identical definition for 'falsy' (`false` or `nothing`) as the `/if` verb.
