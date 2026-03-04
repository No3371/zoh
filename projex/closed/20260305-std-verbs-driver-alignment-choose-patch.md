# Patch: Align ChooseDriver with Verb Driver Model

> **Date:** 2026-03-05
> **Author:** Antigravity
> **Directive:** patch-projex @s:\Repos\zoh\projex\20260304-std-verbs-driver-alignment-plan.md
> **Source Plan:** 20260304-std-verbs-driver-alignment-plan.md
> **Result:** Success

---

## Summary

The `impl/10_std_verbs.md` file was previously updated to align with the verb driver model, but `ChooseDriver.execute` was left behind still calling `context.runtime.presentChoice`. This patch updates `ChooseDriver.execute` to resolve choices and return `Suspend { Host { timeoutMs } }` to match the rest of the presentation verbs, fulfilling the final missing piece of the plan.

---

## Changes

### Std Verbs Implementation

**File:** `impl/10_std_verbs.md`
**Change Type:** Modified
**What Changed:**
- Replaced the `ChooseDriver` body. Removed `PresentationRequest` and the call to `context.runtime.presentChoice`. Now returns `Suspend { Host { timeoutMs } }` with `onFulfilled` logic matching `/converse`, `/chooseFrom`, and `/prompt`.

**Why:**
To fully satisfy the `20260304-std-verbs-driver-alignment-plan.md` objective that presentation verb driver bodies must not call runtime methods directly and must instead return `Suspend { Host { timeoutMs } }`.

---

## Verification

**Method:** Manual inspection of `impl/10_std_verbs.md`.

**Result:**
```
ChooseDriver now ends with:
    return Suspend {
        continuation: Continuation { ... }
    }
```

**Status:** PASS 

---

## Impact on Related Projex

| Document | Relationship | Update Made |
|----------|-------------|-------------|
| 20260304-std-verbs-driver-alignment-plan.md | Source plan | Updated status to "Complete" and added note referencing this patch for partial execution completion. |

---

## Notes

The plan was mostly already implemented in a prior step, so patching was the perfect path for replacing `ChooseDriver` without requiring a branch or full execute-walkthrough lifecycle.
