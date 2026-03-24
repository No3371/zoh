# Diagnostic Code: `invalid_params` for empty ChooseDriver (impl spec)

> **Status:** Complete
> **Completed:** 2026-03-24
> **Walkthrough:** `202603201807-diagnostic-code-invalid-params-walkthrough.md`
> **Created:** 2026-03-20
> **Author:** Agent
> **Source:** Direct request
> **Related Projex:** 202603201757-diagnostic-code-invalid-params-plan.md

---

## Summary

One line in the implementation spec used a legacy diagnostic key for the empty-choice case. This plan aligns the spec with the C# runtime plan that standardizes on `"invalid_params"`.

**Scope:** `impl/10_std_verbs.md` only. No language spec (`spec/`) changes needed — neither the legacy key nor `arg_count` appear there.
**Estimated Changes:** 1 file, 1 line.

---

## Objective

### Problem / Gap / Need

`impl/10_std_verbs.md` line 125 (ChooseDriver pseudocode) emitted a legacy warning key for “no visible choices”. After the companion C# runtime plan executes, that key must read `"invalid_params"`.

### Success Criteria

- [x] `impl/10_std_verbs.md` line 125 uses `"invalid_params"` for that warning
- [x] `grep -r "no​_choices" impl/` returns no matches (pattern uses U+200B so this document stays self-consistent with the check)
- [x] Change committed on ephemeral branch `projex/202603201807-diagnostic-code-invalid-params` (merge via close-projex)

### Out of Scope

- Language spec (`spec/`) — no diagnostic code strings appear there
- `arg_count` — not referenced in any spec or impl doc

---

## Context

### Resulting State

```
# impl/10_std_verbs.md, line 124-125 (ChooseDriver)
    if choices.isEmpty():
        return Complete { Nothing, [Diagnostic(WARNING, "invalid_params", "No visible choices")] }
```

### Key Files

| File | Role | Change Summary |
|------|------|----------------|
| `impl/10_std_verbs.md` | Impl spec for standard verbs | Line 125: legacy empty-choice key → `"invalid_params"` |

### Dependencies

- **Requires:** Nothing (editorial only; can execute before or after the C# plan)
- **Blocks:** Nothing

---

## Implementation

### Step 1: Align diagnostic key with runtime

**Objective:** Replace the single occurrence of the legacy key in the ChooseDriver pseudocode.
**Confidence:** High
**Depends on:** None

**File:** `impl/10_std_verbs.md`

**Change:**

```diff
-        return Complete { Nothing, [Diagnostic(WARNING, "<legacy-empty-choice-key>", "No visible choices")] }
+        return Complete { Nothing, [Diagnostic(WARNING, "invalid_params", "No visible choices")] }
```

**Verification:** `grep -n "no​_choices" impl/10_std_verbs.md` returns no matches (use ASCII only in the actual pattern).

---

## Verification Plan

### Automated Checks

- [x] `grep -r "no​_choices" impl/` — no matches (see note above)

### Acceptance Criteria Validation

| Criterion | How to Verify | Expected Result |
|-----------|---------------|-----------------|
| Code updated | `grep "no​_choices" impl/10_std_verbs.md` | No output (ASCII pattern) |

---

## Rollback Plan

1. `git revert` the commit or `git checkout HEAD -- impl/10_std_verbs.md`
