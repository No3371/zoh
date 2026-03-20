# Walkthrough: Variable-Aware `#embed` and Optional Embed (`#embed?`)

> **Execution Date:** 2026-03-13
> **Completed By:** Agent
> **Source Plan:** [20260313-embed-variable-interpolation-plan.md](20260313-embed-variable-interpolation-plan.md)
> **Duration:** ~25 minutes
> **Result:** Success

---

## Summary

Successfully expanded the `#embed` syntax to support variable interpolation via `${name}` syntax, resolving variables across built-ins, runtime flags, and story metadata. Implemented the `#embed?` optional directive that explicitly suppresses file-not-found errors to support dynamic or fallback script inclusions (such as language packs).

---

## Objectives Completion

| Objective | Status | Notes |
|-----------|--------|-------|
| Document `${}` interpolation syntax in `spec/1_concepts.md` | Complete | Includes variable resolution order across sources. |
| Document `#embed?` optional syntax in `spec/1_concepts.md` | Complete | Delineates behavior on missing files. |
| Document built-in `${filename}` preprocessor variable | Complete | Listed in proper resolution context. |
| Specify interpolation context algorithms in `impl/03_preprocessor.md` | Complete | Inserted a new step for interpolation path building. |
| Specify optional block handling in `impl/03_preprocessor.md` | Complete | Adapted the `processEmbeds` logic block appropriately. |
| Expand syntax definitions and test checklists | Complete | Expanded existing `#embed` checklists and sections to include new cases. |

---

## Execution Detail

> **NOTE:** This section documents what ACTUALLY happened, derived from git history and execution notes.

### Step 1: Expand Embed Section in `spec/1_concepts.md`

**Planned:** Update the Embed section (L243-259) to include the Variable Interpolation subsection, the Optional Embed subsection, and an example. Also fix typographical errors in the original text ("absoulute").

**Actual:** Replaced `spec/1_concepts.md` lines 248-261 appropriately. `absoulute` was fixed to `absolute`. The subsections for `Variable Interpolation`, `Optional Embed`, and the accompanying Example were seamlessly integrated.

**Deviation:** None

**Files Changed (ACTUAL):**
| File | Change Type | Planned? | Details |
|------|-------------|----------|---------|
| `spec/1_concepts.md` | Modified | Yes | Expanded the Embed section to fully detail new interpolation and optional behaviors. |

**Verification:** Validated that static `#embed "path";` specification remained unaffected and variable interpolation resolution order was highly visible as described.

**Issues:** None

---

### Step 2: Update Embed Processing in `impl/03_preprocessor.md`

**Planned:** Update directive syntax, insert a new `Step 2: Embed Path Interpolation` block referencing the new `InterpolationContext`, update `processEmbeds` to evaluate the optional `?` and process string paths through `interpolatePath`, shift indexing of subsequent steps, and augment the test checklist.

**Actual:** Implemented exact string replacements matching Step 2 of the plan. Added `InterpolationContext` data structure definition, the `interpolatePath` string interpolation rule, updated `processEmbeds` to inject context variables and bypass file lookup errors securely. Correctly shifted numbering logic from Step 3 to Step 6, and appended testing checklists securely.

**Deviation:** None

**Files Changed (ACTUAL):**
| File | Change Type | Planned? | Details |
|------|-------------|----------|---------|
| `impl/03_preprocessor.md` | Modified | Yes | Included the `InterpolationContext`, updated pipeline numbers, and adjusted testing tasks. |

**Verification:** Confirmed `#embed(\??)` syntax evaluation checks. Tested logic matches all algorithm definitions requested.

**Issues:** None

---

## Complete Change Log

> **Derived from:** `git diff --stat main..HEAD`

### Files Modified
| File | Changes | Lines Affected | In Plan? |
|------|---------|----------------|----------|
| `impl/03_preprocessor.md` | Updated embed resolution, step numbers, and testing checklists | +89, -19 | Yes |
| `spec/1_concepts.md` | Added interpolation context variables, example blocks, and syntax details | +39, -1 | Yes |
| `projex/20260313-embed-variable-interpolation-plan.md` | Status marked Complete | +1, -1 | Yes |

---

## Success Criteria Verification

### Criterion 1: `spec/1_concepts.md` documents `${}` interpolation syntax and resolution order

**Verification Method:** Exact source string review.

**Evidence:**
```markdown
Embed paths support `${}` interpolation:
...
Resolution order for `${name}`:
1. **Built-in preprocessor variables**...
2. **Runtime-scoped flags**...
3. **Story metadata**...
4. **Empty string**...
```

**Result:** PASS

---

### Criterion 2: `spec/1_concepts.md` documents `#embed?` optional form

**Verification Method:** Exact source string review.

**Evidence:**
```markdown
`#embed?` is the optional form — if the resolved file does not exist, the directive is silently removed (no error).
```

**Result:** PASS

---

### Criterion 3: `spec/1_concepts.md` documents built-in `${filename}`

**Verification Method:** Exact source string review.

**Evidence:**
```markdown
1. **Built-in preprocessor variables** — intrinsic values about the current file:
   - `filename` — base name of the current file without extension (e.g., `"cafe"` for `cafe.zoh`)
```

**Result:** PASS

---

### Criterion 4: `impl/03_preprocessor.md` specifies interpolation resolution algorithm

**Verification Method:** Exact source string review.

**Evidence:**
```markdown
interpolatePath(path: string, ctx: InterpolationContext): string
    return path.replaceAll(/\$\{(\w+)\}/, (match, name) =>
        if name in ctx.builtIns:
...
```

**Result:** PASS

---

### Criterion 5: `impl/03_preprocessor.md` specifies optional embed processing

**Verification Method:** Exact source string review.

**Evidence:**
```markdown
            if not fileExists(absPath):
                if optional:
                    continue    # #embed? — silently skip
                else:
                    error("File not found: " + absPath)
```

**Result:** PASS

---

### Criterion 6: Test checklists cover new functionality

**Verification Method:** Verification of checkboxes included.

**Evidence:**
```markdown
- [ ] Interpolation: `${filename}` resolves to current file base name
...
- [ ] Optional embed: `#embed?` skips silently when file not found
```

**Result:** PASS

---

### Acceptance Criteria Summary

| Criterion | Method | Result | Evidence |
|-----------|--------|--------|----------|
| `spec/1_concepts.md` `${}` syntax | Manual Review | Pass | Verified |
| `spec/1_concepts.md` `#embed?` syntax | Manual Review | Pass | Verified |
| `spec/1_concepts.md` built-in `${filename}` | Manual Review | Pass | Verified |
| `impl/03_preprocessor.md` interpolate algorithm | Manual Review | Pass | Verified |
| `impl/03_preprocessor.md` optional embed skip | Manual Review | Pass | Verified |
| `impl/03_preprocessor.md` test checklist | Manual Review | Pass | Verified |

**Overall:** 6/6 criteria passed

---

## Deviations from Plan

None.

---

## Issues Encountered

None. Execution ran completely smoothly based directly on the provided specs.

---

## Key Insights

### Technical Insights
- Separating the logic pipeline in `impl/03_preprocessor.md` correctly identified and isolated interpolation resolution (Step 2) prior to the path loading mechanisms logic step (Step 3).

---

## Recommendations

### Immediate Follow-ups
- [ ] Incorporate these expanded testing checklists into runtime testing tasks.
- [ ] Utilize this new feature branch implementation for `/patch` mechanism evaluations.

---

## Related Projex Updates

### Documents to Update
| Document | Update Needed |
|----------|---------------|
| `20260313-embed-variable-locale-flag-proposal.md` | Ensure it references this walkthrough. Wait to close until after validation of the script-level localization proposal. |

---

## Appendix

### References
- Spec definitions: [spec/1_concepts.md](../../spec/1_concepts.md)
- Impl definitions: [impl/03_preprocessor.md](../../impl/03_preprocessor.md)
