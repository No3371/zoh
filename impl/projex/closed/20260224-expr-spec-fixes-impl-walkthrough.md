# Walkthrough: Expression Spec Fixes — Impl Doc

> **Execution Date:** 2026-02-28
> **Completed By:** agent
> **Source Plan:** `20260224-expr-spec-fixes-impl-plan.md`
> **Duration:** ~7 minutes
> **Result:** Success

---

## Summary

Two targeted documentation bugs in `impl/04_expressions.md` were corrected: the EBNF grammar comment for the `conditional` form had a wrong separator (`'|'` instead of `':'`), and `parseOptionList()` had an unintended fallthrough that silently returned `AnyForm(options)` instead of emitting a parse error for bare `$(...)` syntax. Both fixes are documentation-only with no runtime impact on the existing C# implementation.

---

## Objectives Completion

| Objective | Status | Notes |
|-----------|--------|-------|
| Fix conditional EBNF grammar comment (`'|'` → `':'`) | Complete | Line 30 in `impl/04_expressions.md` |
| Fix `parseOptionList()` to error on bare `$(options)` | Complete | Line 255 in `impl/04_expressions.md` |

---

## Execution Detail

> **NOTE:** This section documents what ACTUALLY happened, derived from git history and execution notes.
> Differences from the plan are explicitly called out.

### Step 1: Fix conditional grammar comment

**Planned:** Change `'|'` to `':'` in the `conditional` EBNF rule on the grammar comment line.

**Actual:** Edited `impl/04_expressions.md` line 30 — replaced `'|'` with `':'` in the `conditional` rule.

**Deviation:** None. Applied exactly as planned.

**Files Changed (ACTUAL):**
| File | Change Type | Planned? | Details |
|------|-------------|----------|---------|
| `impl/04_expressions.md` | Modified | Yes | Line 30: `'?' expr '|' expr` → `'?' expr ':' expr` |

**Verification:** Read line 30 post-edit — confirmed `conditional := '$?(' expr '?' expr ':' expr ')'`.

**Issues:** None.

---

### Step 2: Fix `parseOptionList()` fallthrough

**Planned:** Replace `return AnyForm(options)` at the end of `parseOptionList()` with an `error(...)` call that guides authors to use `$?(` instead.

**Actual:** Edited `impl/04_expressions.md` line 255 — replaced `return AnyForm(options)` with:
```
error("'$(' option list requires '[index]' or '[%]' suffix; did you mean '$?(' for first-non-nothing selection?")
```

**Deviation:** Steps 1 and 2 were committed in a single commit (both edits in the same file — logically atomic).

**Files Changed (ACTUAL):**
| File | Change Type | Planned? | Details |
|------|-------------|----------|---------|
| `impl/04_expressions.md` | Modified | Yes | Line 255: `return AnyForm(options)` → `error(...)` |

**Verification:** Read lines 233–255 — confirmed no `return AnyForm(options)` fallthrough; final statement is `error(...)` with correct message referencing `$?(`.

**Issues:** None.

---

## Complete Change Log

> **Derived from:** `git diff --stat main..HEAD`

### Files Created
| File | Purpose | Lines | In Plan? |
|------|---------|-------|----------|
| `impl/projex/20260224-expr-spec-fixes-impl-log.md` | Execution log | 46 | No (execution artifact) |

### Files Modified
| File | Changes | Lines Affected | In Plan? |
|------|---------|----------------|----------|
| `impl/04_expressions.md` | Two targeted edits (grammar comment + parseOptionList) | 30, 255 | Yes |
| `impl/projex/20260224-expr-spec-fixes-impl-plan.md` | Status updated to Complete; criteria checked off | Header + criteria | Yes (status update) |

### Files Deleted
_None._

### Planned But Not Changed
_All planned changes were made._

---

## Success Criteria Verification

### Criterion 1: Grammar comment for `conditional` uses `':'` not `'|'`

**Verification Method:** Read `impl/04_expressions.md` EBNF grammar block at line 30.

**Evidence:**
```ebnf
conditional     := '$?(' expr '?' expr ':' expr ')'        # /if ternary
```

**Result:** PASS

---

### Criterion 2: `parseOptionList()` raises a parse error when `$(...)` is not followed by `[index]` or `[%]`

**Verification Method:** Read `parseOptionList()` pseudocode at line 255.

**Evidence:**
```
error("'$(' option list requires '[index]' or '[%]' suffix; did you mean '$?(' for first-non-nothing selection?")
```

**Result:** PASS

---

### Criterion 3: Error message clearly explains the mistake and suggests `$?()` as the correct form

**Verification Method:** Read the error message text in the code above.

**Evidence:** Message explicitly names `$?(` as the correct alternative.

**Result:** PASS

---

### Acceptance Criteria Summary

| Criterion | Method | Result | Evidence |
|-----------|--------|--------|----------|
| Conditional comment uses `':'` | Read EBNF grammar block | **PASS** | Line 30 |
| `parseOptionList()` emits error | Read pseudocode | **PASS** | Line 255 |
| Error message references `$?(` | Read error text | **PASS** | Line 255 |

**Overall: 3/3 criteria passed**

---

## Deviations from Plan

### Deviation 1: Steps 1 and 2 committed together
- **Planned:** Implied separate commits per step
- **Actual:** Single commit covering both edits (`3fa71af`)
- **Reason:** Both edits target the same file; a single atomic commit is cleaner
- **Impact:** None — all changes present and correct
- **Recommendation:** Plan could note single-commit preference when steps share a file

---

## Issues Encountered

### Issue 1: Plan file not committed before execution
- **Description:** The plan file (`20260224-expr-spec-fixes-impl-plan.md`) was untracked on `main` at execution start — workflow requires it to be committed first
- **Severity:** Low
- **Resolution:** Committed plan file to `main` before creating ephemeral branch
- **Time Impact:** Minimal (~1 minute)
- **Prevention:** Commit impl plan files immediately after creation

### Issue 2: Unrelated working tree changes
- **Description:** `CLAUDE.md` and `spec/2_verbs.md` were modified in the working tree, preventing a clean ephemeral branch creation
- **Severity:** Low
- **Resolution:** Stashed those files before execution, restored after completion
- **Time Impact:** Minimal
- **Prevention:** Keep working tree clean before starting execution

---

## Key Insights

### Lessons Learned

1. **Commit plan files immediately after creation**
   - Context: Plan was untracked at execution time — required an extra commit step before branching
   - Insight: Plan files should be committed right after `plan-projex` produces them, not left as untracked
   - Application: Make committing the plan file the final step of `plan-projex`

### Technical Insights

- The `parseConditionalOrAny()` implementation code was already correct (uses `consume(COLON, ...)`); only the EBNF comment was wrong — illustrates a documentation-code drift risk
- `parseOptionList()` returning `AnyForm` was dead code even at time of writing — `$?(...)` is the correct any-form and is handled by a different parse path. The fallthrough was never exercised

---

## Recommendations

### Immediate Follow-ups
- [ ] Verify the concrete C# `ExpressionParser.cs` implementation does not have an equivalent `AnyForm` fallthrough in its `parseOptionList` equivalent

### Future Considerations
- Consider adding a doc-linting step that cross-checks EBNF grammar comments against implementation code for separator mismatches

---

## Related Projex

### Documents Updated
| Document | Update |
|----------|--------|
| `20260224-expr-spec-fixes-impl-plan.md` | Status: `Complete`; all criteria checked |

### Cross-references
| Document | Relationship |
|----------|-------------|
| `20260224-expr-spec-eval.md` | Source evaluation that identified F3 and F4 |
| `20260224-expr-spec-fixes-plan.md` | Companion spec-level plan (fixes spec files) |
