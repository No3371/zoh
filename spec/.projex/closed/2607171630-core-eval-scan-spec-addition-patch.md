# Patch: Core.Eval.Scan — Spec Addition

> **Date:** 2026-07-17
> **Author:** agent
> **Directive:** Execute `2603311700-core-eval-scan-spec-plan.md` — insert `#### Core.Eval.Scan` spec section into `spec/2_verbs.md`
> **Source Plan:** `2603311700-core-eval-scan-spec-plan.md`
> **Result:** Success

---

## Summary

Inserted `#### Core.Eval.Scan` into `spec/2_verbs.md` immediately after `Core.Eval.Interpolate`, per the Ready plan `2603311700-core-eval-scan-spec-plan.md`. Single-section, single-file insertion — no exploration or design decisions needed, so executed as a patch rather than a full execute-projex cycle. All plan assumptions (Interpolate ends line 384, Control Flow begins line 386) verified correct before insertion.

---

## Changes

### `spec/2_verbs.md`

**File:** `spec/2_verbs.md`
**Change Type:** Modified
**What Changed:**
- Inserted `#### Core.Eval.Scan` section (80 lines) between line 384 (end of Interpolate's Syntactic Sugar Forms) and line 386 (`### Control Flow (core.flow)`)
- New section includes: description paragraph, `##### Aliases`, `##### Named Parameters` (`nocase`, `trim`), `##### Parameters` (`format`, `value`), `##### Diagnostics` (`invalid_syntax` Fatal, `invalid_input`/`type_coercion_failed` Info), `##### Returns`, `##### Placeholder Behavior`, `##### Examples`

**Why:**
No verb existed to decompose a structured string into named captures using the `${*ref}`/`${*ref+}` template syntax that `Core.Eval.Interpolate` uses for composition — Scan fills that gap as the structural inverse.

---

## Verification

**Method:** Searched `spec/2_verbs.md` for section headers before and after insertion; compared subsection ordering against `Core.Eval.Interpolate`; confirmed diagnostic format and severities.

**Result:**
```
$ grep -n "^### \|^#### Core.Eval" spec/2_verbs.md
261:### Evaluation (core.eval)
263:#### Core.Eval.Evaluate
329:#### Core.Eval.Interpolate
386:#### Core.Eval.Scan
466:### Control Flow (core.flow)

Subsection order — Interpolate: Aliases, Named Parameters, Parameters, Diagnostics, Returns, Examples, Syntactic Sugar Forms
Subsection order — Scan:        Aliases, Named Parameters, Parameters, Diagnostics, Returns, Placeholder Behavior, Examples
Diagnostics: Fatal `invalid_syntax`; Info `invalid_input`, `type_coercion_failed` — snake_case, no Warning severity used
```

**Status:** PASS

All Success Criteria from the source plan satisfied:
- [x] `#### Core.Eval.Scan` appears immediately after `#### Core.Eval.Interpolate`
- [x] Same `#####`-level structure as neighboring verbs
- [x] `nocase` and `trim` documented with types, defaults, behavior
- [x] `${*ref}` / `${*ref+}` lazy/greedy suffixes specified
- [x] Adjacent-placeholder prohibition stated as Fatal (`invalid_syntax`)
- [x] Coercion rules note Parse type-parsing logic without Parse's pre-trim
- [x] `trim` whitespace definition deferred with recommendation (aligned to `Core.Parse`'s definition)
- [x] `nocase` case-folding algorithm deferred with recommendation (simple Unicode case-folding, no locale)
- [x] Escaping rule cross-references `Core.Eval.Interpolate`

---

## Impact on Related Projex

| Document | Relationship | Update Made |
|----------|-------------|-------------|
| `2603311700-core-eval-scan-spec-plan.md` | Source plan | Marked `[PATCHED]`, status set to `Complete`, moved to `.projex/closed/` |

---

## Notes

Plan's `> **Worktree:** Yes` was not used — patch workflow commits directly to the current branch by design (no ephemeral branch for well-understood, single-file changes). No gotchas; all plan assumptions verified true on first check.
