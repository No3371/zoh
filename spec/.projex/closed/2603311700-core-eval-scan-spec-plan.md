# Core.Eval.Scan — Spec Addition

> **Status:** Complete
> **Created:** 2026-03-31
> **Author:** agent
> **Source:** `20260225-parse-by-format-verb-proposal.md` (Option A, reviewed `2603311600-20260225-parse-by-format-verb-proposal-review.md`)
> **Related Projex:** `20260225-versioning-concept-spec-plan.md`
> **Worktree:** Yes

> **Execution:** `[PATCHED]` — fully executed via `2607171630-core-eval-scan-spec-addition-patch.md`. No objectives remain open.

---

## Summary

Add `#### Core.Eval.Scan` to `spec/2_verbs.md` immediately after `#### Core.Eval.Interpolate`. Scan is the structural inverse of Interpolate for simple `${*ref}` / `${*ref+}` templates — it decomposes a string into named captures rather than composing one.

**Scope:** `spec/2_verbs.md` only — no runtime or implementation changes.
**Estimated Changes:** 1 file, 1 new section (~80 lines).

---

## Objective

### Problem / Gap / Need

No verb exists to decompose a structured string into named parts using the same `${*ref}` template syntax that `Core.Eval.Interpolate` uses for composition. Scripts dealing with version strings, tokens, paths, or date formats must use manual index tracking or expression arithmetic.

### Success Criteria
- [ ] `spec/2_verbs.md` contains `#### Core.Eval.Scan` immediately after `#### Core.Eval.Interpolate`
- [ ] Section follows the same `#####`-level structure as neighboring verbs (Aliases, Named Parameters, Parameters, Diagnostics, Returns, Examples)
- [ ] Named parameters `nocase` and `trim` are documented with types, defaults, and behavior
- [ ] Lazy/greedy placeholder suffixes `${*ref}` / `${*ref+}` are specified
- [ ] Adjacent-placeholder prohibition is stated as Fatal
- [ ] Coercion rules explicitly note: Parse type-parsing logic without Parse's unconditional pre-trim
- [ ] `trim` whitespace definition deferred with recommendation
- [ ] `nocase` case-folding algorithm deferred with recommendation
- [ ] Escaping rule cross-references `Core.Eval.Interpolate`

### Out of Scope
- Runtime / C# implementation (separate plan, separate `.projex/` scope)
- Interpolate sub-forms beyond `${*ref}` / `${*ref+}` (v1 scope)
- Exact Unicode case-folding algorithm definition (deferred to impl plan)
- Exact whitespace character set definition (deferred to impl plan)

---

## Context

### Current State

`spec/2_verbs.md` § `### Evaluation (core.eval)` contains two verbs:

| Line | Verb | Notes |
|------|------|-------|
| 263 | `Core.Eval.Evaluate` | Expression evaluator |
| 329 | `Core.Eval.Interpolate` | String template composition; ends at line 384 |

Line 386 begins `### Control Flow (core.flow)`. The new section inserts between lines 384 and 386.

`Core.Eval.Interpolate` structure (the pattern to follow):
```
#### Core.Eval.Interpolate
[description paragraph]
##### Aliases
##### Named Parameters
##### Parameters
##### Diagnostics
##### Returns
##### Examples
##### Syntactic Sugar Forms
```

Scan mirrors this structure minus Syntactic Sugar Forms, plus additional subsections for side effects and placeholder behavior.

### Key Files

| File | Role | Change Summary |
|------|------|----------------|
| `spec/2_verbs.md` | Core verb specification | Insert `#### Core.Eval.Scan` after Interpolate (after line 384, before line 386) |

### Dependencies
- **Requires:** `Core.Eval.Interpolate` spec (stable, defines `${*ref}` syntax)
- **Requires:** `Core.Var.Parse` spec (stable, defines type-parsing logic referenced by Scan's coercion)
- **Blocks:** Runtime implementation plan (C# `.projex/` scope)

### Constraints
- Follow the exact `#####`-level subsection ordering used by neighboring verbs
- Diagnostic code names must use `snake_case` and severity levels must match existing patterns (Fatal, Error, Info — no Warning)
- No `##### Side Effects` subsection (no verb in the spec uses one) — document side effects in the main description paragraph

### Assumptions
- `Core.Eval.Interpolate` section ends at line 384 with Syntactic Sugar Forms code block — verify at execution time
- `### Control Flow (core.flow)` begins at line 386 — verify at execution time
- String escaping for `${` in interpolation contexts follows the rules on lines 326–327 of the Evaluate section (`\${` inside `$"..."` expression strings) — Scan reuses whatever Interpolate uses, cross-referenced rather than redefined

### Impact Analysis
- **Direct:** `spec/2_verbs.md` — new section inserted
- **Adjacent:** `Core.Eval.Interpolate` — cross-referenced (no edits to Interpolate section)
- **Downstream:** Future runtime implementation plan will consume this spec

---

## Implementation

### Overview

Single insertion of a new `#### Core.Eval.Scan` section into `spec/2_verbs.md`. The section text is derived from the proposal's Specification Sketch, adapted to match the spec file's conventions (subsection ordering, diagnostic format, parameter documentation style).

### Step 1: Insert Core.Eval.Scan section

**Objective:** Add the complete `Core.Eval.Scan` specification section after `Core.Eval.Interpolate`.
**Confidence:** High
**Depends on:** None

**Files:**
- `spec/2_verbs.md`

**Changes:**

Insert after line 384 (end of Interpolate's Syntactic Sugar Forms), before `### Control Flow (core.flow)`:

```markdown
#### Core.Eval.Scan

A scan verb matches an input string against a format template, capturing matched segments into references. Structural inverse of `Core.Eval.Interpolate` for the simple template subset only: format strings may contain `${*ref}` and `${*ref+}` placeholders with literal delimiters. Interpolate sub-forms (formatting specifiers, `$#{...}`, `$?{...}`, unrolling, picks, rolls) are not supported and produce `invalid_syntax`.

On successful match, each placeholder's captured segment is assigned to its reference, coerced to the reference's declared type using `Core.Parse` type-parsing logic — **without** `Core.Parse`'s unconditional pre-trim. Trimming is controlled by the `trim` named parameter. Untyped references receive the captured segment as `string`. If any capture fails type coercion, the verb returns `false` and no references are modified. On mismatch, no references are modified.

Literal segments between placeholders must match the input exactly (case-sensitive by default; see `nocase`). Escaping rules for producing a literal `${` in the format follow the same conventions as `Core.Eval.Interpolate`.

##### Aliases
- `/scan`

##### Named Parameters
- `nocase`: Whether literal matching ignores letter case. Accept `boolean`/`*boolean`. Optional. Default to `false`. When `true`, literal segments are compared case-insensitively using simple Unicode case-folding (no locale). Captured text preserves its original casing.
- `trim`: Whether captured segments are whitespace-trimmed before type coercion. Accept `boolean`/`*boolean`. Optional. Default to `false`. When `true`, leading and trailing whitespace is stripped from each capture before coercion (or before assignment for untyped string refs). When `false`, the raw captured segment is passed to the type parser as-is. The set of characters constituting whitespace aligns with `Core.Parse`'s trimming definition.

##### Parameters
- `format`: The format template. Accept `string`/`*string`. Must contain at least one `${*ref}` or `${*ref+}` placeholder. Two adjacent placeholders with no literal separator between them are `invalid_syntax`.
- `value`: The string to scan. Accept `string`/`*string`.

##### Diagnostics
- Fatal: `invalid_syntax`: Malformed format string (unclosed `${`, invalid reference name, adjacent placeholders with no separator, or unsupported Interpolate sub-forms).
- Info: `invalid_input`: Input did not match the format pattern. Returns `false`.
- Info: `type_coercion_failed`: A captured segment could not be parsed to the declared type of its reference. Returns `false`; no refs modified.

##### Returns
`true` if the entire input was matched and all captures were assigned. `false` on mismatch or type coercion failure. Never returns `nothing`.

##### Placeholder Behavior
- `${*ref}` (default) — Lazy: captures the shortest segment that allows the remainder of the format to match.
- `${*ref+}` — Greedy: captures the longest segment before the next literal delimiter.
- A capture may be the empty string if the delimiter structure still matches (e.g., format `"a${*X}b"` on input `"ab"` yields `*X = ""`).

##### Examples
```
:: Basic — integer captures
:: *MAJOR, *MINOR, *PATCH declared as integer
/scan "${*MAJOR}.${*MINOR}.${*PATCH}", "1.2.3";
:: *MAJOR = 1, *MINOR = 2, *PATCH = 3 → true

:: Untyped references → string captures
/scan "v${*VER}-${*ENV}", "v2.0.1-prod";
:: *VER = "2.0.1", *ENV = "prod" → true

:: No match
/scan "${*A}.${*B}", "hello";
:: no '.' in input → false; *A and *B unchanged

:: Type coercion failure
:: *COUNT declared as integer
/scan "count=${*COUNT}", "count=abc";
:: "abc" cannot parse as integer → false; *COUNT unchanged

:: Conditional usage
*ok <- /scan "${*MAJOR}.${*MINOR}.${*PATCH}", *version_str;
/if *ok {
    /show "Major: ${*MAJOR}";
};

:: Case-insensitive literal matching
/scan nocase: true, "status=${*CODE}", "Status=OK";
:: literals "status=" matched case-insensitively → *CODE = "OK" → true

:: Whitespace — default (no trim)
:: *NUM declared as integer
/scan "x=${*NUM}", "x= 42 ";
:: captured " 42 " passed as-is → integer parse fails → false

:: Whitespace — trim opt-in
/scan trim: true, "x=${*NUM}", "x= 42 ";
:: captured " 42 " trimmed to "42" → *NUM = 42 → true

:: trim with untyped reference
/scan trim: true, "name: ${*N}", "name:  Alice ";
:: captured " Alice " trimmed → *N = "Alice" → true

:: Greedy vs lazy
/scan "${*A}:${*B+}", "a:b:c";
:: lazy *A captures "a" (first ':'), greedy *B+ captures "b:c" (last ':') → true
```
```

**Rationale:** Follows the exact subsection ordering of `Core.Eval.Interpolate` (Aliases → Named Parameters → Parameters → Diagnostics → Returns → Examples) with an additional `##### Placeholder Behavior` subsection to document lazy/greedy semantics and empty captures — information that doesn't fit naturally into any standard subsection.

**Verification:** After insertion, confirm:
1. The section appears between Interpolate and Control Flow
2. Heading levels are consistent (`####` for the verb, `#####` for subsections)
3. No duplicate section headers in the file
4. Diagnostic format matches neighbors (e.g., `- Fatal: \`code\`: Description.`)

**If this fails:** Remove the inserted text. Single insertion — clean rollback.

---

## Verification Plan

### Automated Checks
- [ ] No syntax or formatting issues in the markdown (heading levels, code blocks properly closed)

### Manual Verification
- [ ] `#### Core.Eval.Scan` appears immediately after `#### Core.Eval.Interpolate`'s last subsection
- [ ] `### Control Flow (core.flow)` immediately follows the new section
- [ ] Named parameter documentation matches the style of `fallback:` in Interpolate (type, optional, default)
- [ ] Diagnostic codes are snake_case, severities are Fatal/Info (no Warning)
- [ ] Examples compile mentally — each shows format + input + expected result

### Acceptance Criteria Validation
| Criterion | How to Verify | Expected Result |
|-----------|---------------|-----------------|
| Section exists at correct position | Search `spec/2_verbs.md` for `#### Core.Eval.Scan` | Found between Interpolate and Control Flow |
| Follows verb structure | Compare subsection headers with Interpolate | Same ordering (Aliases, Named Params, Params, Diagnostics, Returns, Examples) |
| `nocase` / `trim` documented | Read Named Parameters subsection | Both present with type, default, description |
| Coercion note present | Read description paragraph | Mentions Parse logic without pre-trim |
| Adjacent placeholder rule | Read Diagnostics | `invalid_syntax` includes "adjacent placeholders" |

---

## Rollback Plan

1. Revert the inserted section from `spec/2_verbs.md` (single contiguous block — delete from `#### Core.Eval.Scan` to the line before `### Control Flow`).

---

## Notes

### Risks
- **Line numbers shift between plan and execution**: Mitigated by searching for content anchors (`##### Syntactic Sugar Forms` end / `### Control Flow`) rather than hard line numbers.

### Open Questions
None — all design decisions resolved in the proposal and review.
