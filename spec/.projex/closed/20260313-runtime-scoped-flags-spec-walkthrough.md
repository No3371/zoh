# Walkthrough: Runtime-Scoped Flags — Spec Changes

> **Execution Date:** 2026-03-13
> **Completed By:** Agent
> **Source Plan:** 20260313-runtime-scoped-flags-spec-plan.md
> **Result:** Success

---

## Summary

Added runtime-scoped flags to the ZOH language spec across 4 files. Flags now have two scopes (runtime and context) with a context-first fallback chain, a `[scope]` attribute on `/flag`, a `locale` standard flag, and explicit preprocessor access to runtime flags. All 5 acceptance criteria passed. No deviations from plan.

---

## Objectives Completion

| Objective | Status | Notes |
|-----------|--------|-------|
| `spec/1_concepts.md` — flag scoping concept section | Complete | `# Flags` inserted before `# Verb` |
| `spec/2_verbs.md` — `/flag` `[scope]` attribute | Complete | Core.Flag definition replaced in full |
| `spec/std_flags.md` — `locale` standard flag | Complete | `## Locale` appended after Pace |
| `spec/3_runtime.md` — runtime flag storage and preprocessor access | Complete | Three sub-changes applied |
| Backwards compatibility | Complete | Existing `/flag` calls unchanged |

---

## Execution Detail

### Step 1: Add Flags Section to `spec/1_concepts.md`

**Planned:** Insert `# Flags` section between `## Attributes` close and `# Verb`.

**Actual:** Inserted after the closing code block of `## Attributes` (after line 180). Section defines the two scopes, fallback semantics, and case-insensitivity of flag names.

**Deviation:** None.

**Files Changed:**
| File | Change Type | Planned? | Details |
|------|-------------|----------|---------|
| `spec/1_concepts.md` | Modified | Yes | ~12 lines inserted before `# Verb` |

**Verification:** Edit applied cleanly. `# Flags` section now sits between `## Attributes` and `# Verb` at the same heading level as `# Variables`.

---

### Step 2: Update Core.Flag in `spec/2_verbs.md`

**Planned:** Replace Core.Flag definition at lines 929–945 with new text adding `[scope]` attribute and fallback semantics.

**Actual:** Exact replacement applied. New definition adds scope documentation, `#### Attributes` subsection with `scope` (string: `context`/`runtime`, default `context`), and updated examples showing all three usage forms.

**Deviation:** None.

**Files Changed:**
| File | Change Type | Planned? | Details |
|------|-------------|----------|---------|
| `spec/2_verbs.md` | Modified | Yes | Lines 929–945 replaced; net +7 lines |

**Verification:** Old text fully replaced; `[scope]` attribute defined with default `context` (backwards compatible).

---

### Step 3: Add `locale` to `spec/std_flags.md`

**Planned:** Append `## Locale` section after the Pace section (after line 18).

**Actual:** Appended as planned. Entry covers BCP 47 format, `string`/`*string` type, empty default, and runtime-scope recommendation.

**Deviation:** None.

**Files Changed:**
| File | Change Type | Planned? | Details |
|------|-------------|----------|---------|
| `spec/std_flags.md` | Modified | Yes | ~7 lines appended |

**Verification:** Format matches existing Interactive/Instant/Pace entries.

---

### Step 4: Update `spec/3_runtime.md`

**Planned:** Three sub-changes — insert Flag Resolution section, update preprocessor description, update State Management.

**Actual:** All three applied:
- 4a: `## Flag Resolution` inserted after `## Variable Resolution` (after line 13) — mirrors Variable Resolution structure.
- 4b: Preprocessor description updated to mention `runtime-scoped flags` alongside existing inputs.
- 4c: State Management fixed typo `exists` → `exist` and appended runtime state enumeration.

**Deviation:** None.

**Files Changed:**
| File | Change Type | Planned? | Details |
|------|-------------|----------|---------|
| `spec/3_runtime.md` | Modified | Yes | ~13 lines added across three locations |

**Verification:** Flag Resolution section flows naturally after Variable Resolution. Preprocessor contract now explicit. State Management sentence is grammatically correct and complete.

---

## Complete Change Log

> **Derived from:** `git diff --stat main..HEAD`

### Files Created
| File | Purpose | In Plan? |
|------|---------|----------|
| `projex/20260313-runtime-scoped-flags-spec-plan-log.md` | Execution log | No (standard artifact) |

### Files Modified
| File | Changes | In Plan? |
|------|---------|----------|
| `spec/1_concepts.md` | Added `# Flags` section (runtime/context scopes, fallback, case-insensitivity) | Yes |
| `spec/2_verbs.md` | Replaced Core.Flag with `[scope]` attribute, fallback docs, updated examples | Yes |
| `spec/std_flags.md` | Appended `## Locale` flag entry | Yes |
| `spec/3_runtime.md` | Added Flag Resolution section; updated preprocessor contract; updated State Management | Yes |
| `projex/20260313-runtime-scoped-flags-spec-plan.md` | Status In Progress → Complete; criteria checked | No (standard) |

### Files Deleted
None.

### Planned But Not Changed
None.

---

## Success Criteria Verification

### Criterion 1: `spec/1_concepts.md` documents flag scoping with fallback semantics

**Verification Method:** Read inserted section.

**Evidence:** `# Flags` section defines runtime scope (shared, preprocessor-accessible, runtime lifetime) and context scope (context lifetime, copied to forks), with explicit fallback: context checked first, runtime second; context shadows runtime.

**Result:** PASS

---

### Criterion 2: `spec/2_verbs.md` `/flag` accepts `[scope]` attribute, defaults to `"context"`

**Verification Method:** Read updated Core.Flag definition.

**Evidence:** `#### Attributes` subsection states: `scope (string: context/runtime): the scope to set the flag to. Defaults to context if not specified.`

**Result:** PASS

---

### Criterion 3: `spec/std_flags.md` includes `locale` standard flag

**Verification Method:** Read appended section.

**Evidence:** `## Locale` entry present with BCP 47 examples, `string`/`*string` type, `""` default, runtime-scope recommendation.

**Result:** PASS

---

### Criterion 4: `spec/3_runtime.md` documents runtime flag storage, preprocessor access, and flag resolution

**Verification Method:** Read updated file.

**Evidence:** `## Flag Resolution` section present after `## Variable Resolution`; preprocessor description includes `runtime-scoped flags`; State Management enumerates runtime-scoped flags as part of runtime state.

**Result:** PASS

---

### Criterion 5: All changes are backwards compatible — existing `/flag` calls unchanged

**Verification Method:** `[scope]` attribute defaults to `context`; no existing `/flag` syntax altered.

**Evidence:** Default scope is `context`, so `/flag "name", value;` (no `[scope]`) behaves identically to before.

**Result:** PASS

---

### Acceptance Criteria Summary

| Criterion | Result |
|-----------|--------|
| Flag scoping in `1_concepts.md` | Pass |
| `/flag` `[scope]` attribute in `2_verbs.md` | Pass |
| `locale` flag in `std_flags.md` | Pass |
| Runtime flag storage + preprocessor in `3_runtime.md` | Pass |
| Backwards compatibility | Pass |

**Overall:** 5/5 criteria passed.

---

## Deviations from Plan

None.

---

## Issues Encountered

**Stash required for clean working state.** `projex/20260304-runtime-spec-gaps-memo.md` had unstaged modifications at execution start. Stashed with label `pre-execute stash: runtime-scoped-flags-spec-plan` and restored after branch finalization.

---

## Key Insights

### Pattern Discoveries

1. **Flag scoping mirrors variable scoping exactly** — both use `[scope]` attribute, both have a named default scope, both use shadowing semantics. This consistency makes the spec learnable: understanding one teaches the other.

### Recommendations

#### Immediate Follow-ups
- [ ] Execute `20260313-runtime-scoped-flags-impl-plan.md` — impl spec changes that depend on this spec
- [ ] Embed variable interpolation plan can now proceed (depends on preprocessor access to runtime flags)
