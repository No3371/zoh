# Patch: Fix Spec/Impl Inconsistencies (Finding 4)

**Source:** `projex/20260207-spec-impl-redteam.md`
**Subject:** Finding 4 (Spec/Impl Inconsistencies)
**Status:** Applied

## Changes

### 1. Fix `impl/08` Pull Behavior
- **Issue:** Impl guide stated `pull` returns a result object, but Spec and Code confirm it returns values or throws errors.
- **Fix:** Updated `impl/08` pseudo-code to reflect error-based behavior.

### 2. Fix `impl/03` Macro Syntax Text
- **Issue:** Intro text mentioned `#macro`, but ZOH uses `|%` syntax.
- **Fix:** Removed contradictory `#macro` reference from `impl/03` introduction.

## Verification
- Verified against `spec.md` and `Zoh.Runtime` source code.
- See `projex/20260207-finding4-verification-report.md` for details.
