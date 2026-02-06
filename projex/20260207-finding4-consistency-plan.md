# Resolve Spec/Impl Inconsistencies (Finding 4)

> **Status:** Ready
> **Created:** 2026-02-07
> **Author:** Agent
> **Source:** [20260207-spec-impl-redteam.md](./20260207-spec-impl-redteam.md) - Finding 4
> **Related Projex:** None

---

## Summary

This plan addresses "Finding 4: Spec/Impl Inconsistencies" from the Red Team analysis. It focuses on reconciling the Macro syntax between the Specification and Implementation/Code (adopting the working `|%...%|` syntax), clarifying `/pull` return values, and defining map key ordering behaviors.

**Scope:** Documentation updates (`spec.md`, `impl/*.md`). No runtime code changes.
**Estimated Changes:** 2 files (`spec.md`, `impl/03_preprocessor.md`).

---

## Objective

### Problem / Gap / Need
The Spec and Impl guides contradict each other on several key features, causing confusion for implementers and story authors.
1.  **Macro Syntax:** Spec uses `#macro`, but Code/Impl uses `|%...%|`.
2.  **Channel Pull:** Code returns a result object (status), Spec says "Error".
3.  **Map Order:** Implementation-defined behavior is not documented in Impl guide.

### Success Criteria
- [ ] `spec.md` Preprocessor directive section matches the actual `|%...%|` syntax implemented in C# runtime.
- [ ] `impl/03_preprocessor.md` is cleaned up to remove conflicting `#macro` references.
- [ ] `spec.md` Channel `/pull` return value matches the `PullResult` pattern (non-fatal closed state).
- [ ] `impl/08_concurrency.md` explicitly documents map key order behavior for C# runtime.

### Out of Scope
- Changing C# runtime macro implementation (Logic: Code is "truth" for now, Spec should reflect reality).

---

## Context

### Current State
- `spec.md` describes `#macro NAME` which doesn't exist in code.
- `MacroPreprocessor.cs` implements a complex pipe-based syntax `|%Name|Args|%|`.
- `impl/03_preprocessor.md` describes the pipe syntax in detail but header mentions `#macro`.

### Key Files
| File | Purpose | Changes Needed |
|------|---------|----------------|
| `spec.md` | Language Specification | Replace `#macro` section with `|%...%|` syntax details. Update `/pull` return value. |
| `impl/03_preprocessor.md` | Preprocessor Implementation Guide | Remove `#macro` mentions, ensure consistency. |
| `impl/08_concurrency.md` | Concurrency/Channel Guide | Clarify Map order behavior (if mentioned) or add note. |

---

## Implementation

### Step 1: Reconcile Macro Syntax

**Objective:** Update Spec to match the pipe-based macro syntax used in Code.

**Files:**
- `spec.md`
- `impl/03_preprocessor.md`

**Changes:**
1.  **`spec.md`**: Rewrite "Preprocessor Directives" section.
    *   Remove `#macro` / `#expand`.
    *   Document `|%DEF%| ... |%DEF%|` and expansion `|%DEF|args|%|`.
2.  **`impl/03_preprocessor.md`**: Remove conflicting references to `#macro` in the introduction.

**Rationale:** The C# runtime already implements the pipe syntax robustly. Changing the spec is safer than rewriting the preprocessor.

**Verification:** Manual review of `spec.md` against `MacroPreprocessor.cs` logic.

### Step 2: Clarify Channel Pull Behavior

**Objective:** Update Spec to reflect that `/pull` returns a status/result object, not just a bare value or fatal error.

**Files:**
- `spec.md`

**Changes:**
1.  **`spec.md`**: Update `/pull` section.
    *   Return value: `PullResult` map `{"status": "ok"|"closed"|"empty", "value": ?}` OR clarify that it acts as a value retrieval that *can* fail gracefully.
    *   Actually, `PullResult` is an internal runtime concept. The verb might return `nothing` or `status`.
    *   *Correction*: My previous Finding 1 work made `/pull` return a `PullResult` object (Map). The Spec should say `/pull` returns a Map with `status` and `value`.

**Verification:** Verify against `PullDriver.cs` (from Finding 1 work).

### Step 3: Document Map/Collection Implementation Details

**Objective:** Explicitly state the "Implementation Defined" behaviors in the Impl guide.

**Files:**
- `impl/08_concurrency.md` (or generic `impl/00_overview.md` if exists, sticking to 08 or relevant impl doc). (Actually `impl/08` is concurrency. Maybe create `impl/09_types.md` or add to `impl/01_types.md` if exists).
- *Check:* I'll check if `impl/01_types.md` exists. If not, I'll add a section to `spec.md` "Implementation Notes" or similar.
- *Decision:* I will update `spec.md` "Map" section to reiterate "unordered", and check if `impl/04_types.md` exists.

---

## Verification Plan

### Manual Verification
- [ ] **Review Spec:** Read the new Macro section in `spec.md`.
- [ ] **Review Impl:** Read `impl/03_preprocessor.md` to ensure no `#macro` residue.
- [ ] **Cross-Check:** Compare `spec.md` macro examples with `MacroPreprocessor.cs` regexes.

### Acceptance Criteria Validation
| Criterion | Method | Expected Result |
|-----------|--------|-----------------|
| Macro Syntax | Visual Inspection | `spec.md` uses `\|%...%\|` |
| Pull Return | Visual Inspection | `spec.md` documents Map return |
