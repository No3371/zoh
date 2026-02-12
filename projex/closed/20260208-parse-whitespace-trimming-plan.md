# Plan: Parse Verb Whitespace Trimming (Spec)

> **Reviewed:** 2026-02-12 - [20260212-parse-whitespace-trimming-plan-review.md](20260212-parse-whitespace-trimming-plan-review.md)
> **Review Outcome:** Valid. Mandatory trimming for `/parse` is beneficial for consistency and CRLF resilience.


> **Status:** Complete
> **Created:** 2026-02-08
> **Author:** Agent
> **Source:** [Finding 7 in Red Team Report](../projex/20260207-spec-impl-redteam.md)  
> **Related Projex:** [Plan: C# Implementation](../projex/20260208-parse-whitespace-trimming-csharp-plan.md)

---

## Summary

This plan addresses a specification ambiguity regarding whitespace handling in the `/parse` verb. It mandates that `/parse` must trim leading and trailing whitespace from the input string before parsing or inferring types. This ensures consistent behavior across runtimes and prevents unexpected failures when parsing user input or indented strings.

**Scope:** Documentation (`spec.md`, `impl/`).
**Estimated Changes:** 2 files.

---

## Objective

### Problem / Gap / Need
The current specification effectively ignores whitespace handling for `/parse`. This leads to ambiguity and potential inconsistency across runtimes.

### Success Criteria
- [ ] `spec.md` explicitly states that input to `/parse` is trimmed.
- [ ] `impl/06_core_verbs.md` pseudo-code reflects trimming.

### Out of Scope
- Runtime implementation (covered in [C# Plan](../projex/20260208-parse-whitespace-trimming-csharp-plan.md)).

---

## Context

### Current State
- **Spec:** Silent on whitespace.
- **Impl Guide:** `inferType(str)` uses regex `^-?\d+$` which implies no whitespace allowed.

### Key Files
| File | Purpose | Changes Needed |
|------|---------|----------------|
| `spec.md` | Language Specification | Add "Input string is trimmed" rule to `/parse`. |
| `impl/06_core_verbs.md` | Implementation Guide | Update `ParseDriver` pseudo-code to include `trim()`. |

---

## Implementation

### Step 1: Update Specification
**Objective:** Codify the whitespace trimming rule.

**Files:**
- `s:\repos\zoh\spec.md`

**Changes:**
Add to `/parse` section:
"The verb first trims any leading and trailing whitespace from the input string."

### Step 2: Update Implementation Guide
**Objective:** Align pseudo-code with spec.

**Files:**
- `s:\repos\zoh\impl\06_core_verbs.md`

**Changes:**
```python
ParseDriver.execute(call, context):
    # Trim whitespace first
    str = resolve(call.params[0], context).toString().trim()
```

---

## Verification Plan

### Manual Verification
- [ ] Review `spec.md` changes.
- [ ] Review `impl/06_core_verbs.md` changes.

---
