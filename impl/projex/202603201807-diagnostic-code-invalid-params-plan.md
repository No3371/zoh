# Diagnostic Code: Update `no_choices` → `invalid_params` in Impl Spec

> **Status:** In Progress
> **Created:** 2026-03-20
> **Author:** Agent
> **Source:** Direct request
> **Related Projex:** 202603201757-diagnostic-code-invalid-params-plan.md

---

## Summary

One line in the implementation spec uses `"no_choices"` as a diagnostic code. This plan aligns the spec with the C# runtime plan that replaces `"no_choices"` with `"invalid_params"`.

**Scope:** `impl/10_std_verbs.md` only. No language spec (`spec/`) changes needed — neither `no_choices` nor `arg_count` appear there.
**Estimated Changes:** 1 file, 1 line.

---

## Objective

### Problem / Gap / Need

`impl/10_std_verbs.md` line 125 (ChooseDriver pseudocode) emits `WARNING("no_choices", "No visible choices")`. After the companion C# runtime plan executes, the implementation no longer matches the spec.

### Success Criteria

- [ ] `impl/10_std_verbs.md` line 125 uses `"invalid_params"` instead of `"no_choices"`
- [ ] `grep -r "no_choices" impl/` returns no matches
- [ ] Change committed to base branch

### Out of Scope

- Language spec (`spec/`) — no diagnostic code strings appear there
- `arg_count` — not referenced in any spec or impl doc

---

## Context

### Current State

```
# impl/10_std_verbs.md, line 124-125 (ChooseDriver)
    if choices.isEmpty():
        return Complete { Nothing, [Diagnostic(WARNING, "no_choices", "No visible choices")] }
```

### Key Files

| File | Role | Change Summary |
|------|------|----------------|
| `impl/10_std_verbs.md` | Impl spec for standard verbs | Line 125: `"no_choices"` → `"invalid_params"` |

### Dependencies

- **Requires:** Nothing (editorial only; can execute before or after the C# plan)
- **Blocks:** Nothing

---

## Implementation

### Step 1: Update `no_choices` to `invalid_params`

**Objective:** Replace the single occurrence of `"no_choices"` in the ChooseDriver pseudocode.
**Confidence:** High
**Depends on:** None

**File:** `impl/10_std_verbs.md`

**Change:**

```diff
-        return Complete { Nothing, [Diagnostic(WARNING, "no_choices", "No visible choices")] }
+        return Complete { Nothing, [Diagnostic(WARNING, "invalid_params", "No visible choices")] }
```

**Verification:** `grep -n "no_choices" impl/10_std_verbs.md` returns no matches.

---

## Verification Plan

### Automated Checks

- [ ] `grep -r "no_choices" impl/` — no matches

### Acceptance Criteria Validation

| Criterion | How to Verify | Expected Result |
|-----------|---------------|-----------------|
| Code updated | `grep "no_choices" impl/10_std_verbs.md` | No output |

---

## Rollback Plan

1. `git revert` the commit or `git checkout HEAD -- impl/10_std_verbs.md`
