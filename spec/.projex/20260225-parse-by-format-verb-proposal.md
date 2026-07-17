# Core.Eval.Scan — Parse-by-Format Verb

> **Status:** Accepted
> **Created:** 2026-02-25
> **Author:** agent
> **Related Projex:** `20260224-expr-spec-fixes-plan.md` (closed — verify spec text only), `20260225-versioning-concept-spec-plan.md`, `2603311700-core-eval-scan-spec-plan.md`
> **Reviewed:** 2026-03-31 — `2603311600-20260225-parse-by-format-verb-proposal-review.md`
> **Review Outcome:** Needs Modification — `trim` redundancy with `Core.Parse`; diagnostic severity (Warning to Info); implementation notes stale for named params.

---

## Summary

Propose **`Core.Eval.Scan`** (alias `/scan`) — the structural inverse of **`Core.Eval.Interpolate` for the simple template subset**: only **`${*ref}`** and **`${*ref+}`** placeholders with literal delimiters. (It does **not** invert formatting, `$#{...}`, `$?{...}`, unrolling, picks, rolls, or other `Core.Eval.Interpolate` extensions.) Where Interpolate substitutes those placeholders with values to produce a string, Scan matches a string against the same placeholder layout and assigns captured substrings into the references. Captures are coerced to each reference’s declared type (`Core.Parse` type-parsing logic, without Parse's unconditional pre-trim); untyped references receive `string`. Named parameters control matching behavior: `nocase` for case-insensitive literal matching, `trim` for whitespace-stripping captures before coercion. Example: `/scan "${*MAJOR}.${*MINOR}.${*PATCH}", "1.2.3"` with integer refs sets `*MAJOR = 1`, `*MINOR = 2`, `*PATCH = 3`.

---

## Problem Statement

### Current State

`Core.Eval.Interpolate` generates strings from templates (including rich sub-forms). For templates that only use **`${*ref}` / `${*ref+}`**-style holes and literals, the inverse — decomposing a structured string back into named parts — has no dedicated verb. Extracting such segments today requires `Core.Evaluate` with string arithmetic, manual index tracking, or regex-style workarounds outside the language model.

`Core.Parse` is adjacent but orthogonal: it converts a string to a scalar typed value (integer, double, boolean). It has no format/template concept.

### Gap / Need / Opportunity

Scripts that deal with versioned identifiers, structured tokens, file path patterns, date strings, or similar structured text have no idiomatic way to destructure them. A format-based scan verb completes **delimiter + `${*ref}`** symmetry with `Core.Eval.Interpolate` for that subset and makes it a first-class operation.

### Why Now?

The versioning concept spec (`20260225-versioning-concept-spec-plan.md`) introduces version string handling as a concrete use case. Symmetry with **`Core.Eval.Interpolate`** on simple templates makes the verb easy to justify and teach next to **`#### Core.Eval.Interpolate`** in `spec/2_verbs.md`.

---

## Proposed Change

### Verb placement

Add **`#### Core.Eval.Scan`** immediately after **`#### Core.Eval.Interpolate`** in `spec/2_verbs.md`. Scan reuses the **same `${*ref}` / `${*ref+}` token shapes** as the interpolate grammar for variable holes, but only those forms in v1 — so it lives in **`Core.Eval`** beside Interpolate rather than as a separate top-level `Core.Scan`.

### Overview

The verb accepts a format string and an input string. The format uses **`${*ref}`** and **`${*ref+}`** only as named capture positions (v1). Literal text between placeholders acts as delimiters. On success, each `*ref` is set to its captured segment, coerced per `Core.Parse` type-parsing logic (without the unconditional pre-trim); untyped references receive `string`. Returns `true` on full match, `false` on mismatch. Named parameters `nocase` and `trim` control case-insensitive literal matching and whitespace-stripping of captures respectively.

```
:: *MAJOR, *MINOR, *PATCH declared as integer
/scan "${*MAJOR}.${*MINOR}.${*PATCH}", "1.2.3";
:: *MAJOR = 1, *MINOR = 2, *PATCH = 3  (integers)
:: returns true
```

### Approach Options

#### Option A: `Core.Eval.Scan` — symmetric with `Core.Eval.Interpolate` (subset)

- **Description:** Named after `scanf`. Format uses `${*ref}` / `${*ref+}` as in Interpolate’s simple holes. Literal segments act as separators. Returns `boolean`.
- **Pros:** Same tokens as Interpolate for variable positions; sits next to Interpolate in the spec. Boolean return enables conditional handling.
- **Cons:** `${*ref}` under `/itpl` means “emit value”; under `/scan` means “capture into ref” — opposite data flow, same surface syntax. Spec should state explicitly: **meaning is determined by the verb**.
- **Effort:** Medium — spec authoring + implementation of format-to-scan-plan compilation (pure string operations; see Implementation Notes).

#### Option B: Distinct capture syntax (e.g., `{MAJOR}.{MINOR}.{PATCH}`)

- **Description:** Use a separate placeholder syntax for capture (no `${*}` wrapping) to distinguish from Interpolate's output-side usage.
- **Pros:** No semantic ambiguity — capture placeholders look different from interpolation placeholders.
- **Cons:** Second placeholder convention. Breaks visual parity with **`Core.Eval.Interpolate`** for simple templates.
- **Effort:** Same as Option A, plus spec for a new syntax form.

#### Option C: Predicate-style, returns a map rather than setting refs

- **Description:** `/scan "${MAJOR}.${MINOR}.${PATCH}", "1.2.3"` returns a map `{MAJOR: "1", MINOR: "2", PATCH: "3"}` instead of setting variables.
- **Pros:** No side-effectful reference assignment. Functional style. Composable.
- **Cons:** Diverges from the direct-assignment model used by `Core.Set`, `Core.Capture`. Requires an extra `/set` step to bind values. Less ergonomic for the primary use case.
- **Effort:** Same.

### Recommended Approach

**Option A** — **`Core.Eval.Scan`** with **`${*ref}` / `${*ref+}`** only in v1.

Dual-use of `${*ref}` is acceptable: **`/itpl`** is output-producing; **`/scan`** is input-consuming; meaning follows the verb. **`Core.Eval.Scan`** next to **`Core.Eval.Interpolate`** reinforces that relationship while scoping Scan to the subset Interpolate does not need to “round-trip” for.

---

## Specification Sketch

### Core.Eval.Scan

Scans an input string against a format template, capturing matched segments into references. **Inverse of `Core.Eval.Interpolate` only for templates expressible with `${*ref}` / `${*ref+}` and literals** — not for full interpolate features (formatting, conditionals, counts, unrolling, etc.).

#### v1 scope
- **Allowed in `format`:** Literal segments (including empty) and placeholders **`${*ref}`** and **`${*ref+}`** only.
- **Not allowed in v1:** Any other `Core.Eval.Interpolate` construct (`${...}` with format specifiers, `$#{...}`, `$?{...}`, unrolling `...`, picks, rolls, etc.). Such tokens are **`invalid_syntax`** in a scan format string, or the plan may treat them as compile-time fatal when parsing the format.

#### Named Parameters
- `nocase`: Whether literal matching ignores letter case. Accept `boolean`/`*boolean`. Optional. Default to `false`. When `true`, literal segments in the format are compared against the input using case-insensitive matching (Unicode case-folding; exact folding algorithm is part of the Plan projex). Does **not** affect type coercion of captures — captured text preserves its original casing.
- `trim`: Whether captured segments are whitespace-trimmed before type coercion. Accept `boolean`/`*boolean`. Optional. Default to `false`. When `true`, leading and trailing whitespace is stripped from each capture before type parsing (or before assignment for untyped string refs). When `false`, captured segments are passed to the type parser as-is — whitespace in the capture can cause coercion failure for numeric/boolean types. Literal delimiters are never affected by `trim` (no whitespace normalization of delimiters). The exact set of characters constituting "whitespace" must align with `Core.Parse`'s trimming definition; exact specification is part of the Plan projex (recommendation: Unicode `\p{White_Space}` or the equivalent used by the runtime's string model). Unlike standalone `Core.Parse` which unconditionally trims, Scan gives the format author explicit control because delimiter-bounded captures may intentionally include whitespace.

#### Parameters
- `format`: The format template. Accept `string` or `*string`. Must contain at least one `${*ref}` or `${*ref+}` placeholder. Literal text between placeholders must match exactly by default (**case-sensitive** unless `nocase: true`; **no whitespace normalization**). Literal matching uses the **same rules as string equality for literal substrings** in ZOH (document precisely in the plan: e.g. Unicode scalar sequence vs UTF-16 code units — must match whatever the rest of the string model uses).
- `value`: The string to scan. Accept `string` or `*string`.

#### Literals and escaping
- Format compilation must define how a **literal** can contain **`${`** without starting a placeholder. **Recommendation:** Align with **`Core.Eval.Interpolate`** / string-literal escaping (e.g. the same `\`-escape rules that produce a literal `${` in interpolate contexts, if applicable). If Interpolate uses a different story for “literal brace,” Scan should **reuse it** so one mental model applies. Final rules are part of the Plan projex.

#### Empty captures
- A capture may be the **empty string** if the delimiter structure still matches (e.g. `"a${*X}b"` on input `"ab"` yields `*X <- ""`). Type coercion still applies (`Core.Parse` on `""` per ref type).

#### Returns
`true` if the entire input was matched and all captures were assigned. `false` if the input did not match the format. Does not return `nothing` — result is always boolean.

#### Side Effects
On successful match, each `${*ref}` placeholder in the format is set in the current scope to the corresponding captured segment. For typed references, the captured segment is coerced using the same type-parsing logic as `Core.Parse` — **but without `Core.Parse`'s unconditional pre-trim**. Trimming is controlled exclusively by the `trim` named parameter: when `trim: false` (default), the raw captured segment is passed to the type parser as-is; when `trim: true`, leading/trailing whitespace is stripped first. If a captured segment cannot be coerced to the declared type (e.g. whitespace preventing integer parse when `trim: false`), the verb returns `false` and no references are set. For untyped references, the captured value is assigned as `string` (also subject to `trim`). On failure (no match or type coercion failure), **no references are set** — existing values are preserved.

#### Placeholder Suffixes
- **`${*ref}`** (default): Lazy — captures the shortest segment that allows the remainder of the format to match.
- **`${*ref+}`**: Greedy — captures the longest segment before the next literal.

#### Diagnostics
- Fatal: `invalid_syntax` — Malformed format string (unclosed `${`, invalid reference name, adjacent placeholders with no separator).
- Info: `match_failed` — Input did not match format pattern. Returns `false`.
- Info: `type_coercion_failed` — A captured segment could not be parsed to the declared type of its reference. Returns `false`; no refs modified.

#### Examples
```
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

:: Whitespace in captures — no trim (default)
:: *NUM declared as integer
/scan "x=${*NUM}", "x= 42 ";
:: captured " 42 " passed as-is → integer parse fails → false; *NUM unchanged

:: Whitespace in captures — trim opt-in
/scan trim: true, "x=${*NUM}", "x= 42 ";
:: captured " 42 " trimmed to "42", parsed as integer → *NUM = 42 → true

:: trim with untyped (string) reference
/scan trim: true, "name: ${*N}", "name:  Alice ";
:: captured " Alice " trimmed to "Alice" → *N = "Alice" → true

:: Combined
/scan nocase: true, trim: true, "env:${*E}", "ENV: production ";
:: *E = "production" → true
```

#### Adjacent Placeholders

Two `${*ref}` placeholders with no literal separator between them (`"${*A}${*B}"`) are ambiguous — there is no delimiter to separate the captures. This is a `Fatal: invalid_syntax` error. At least one literal character must appear between any two adjacent placeholders.

---

## Impact Analysis

### Affected Areas
- `spec/2_verbs.md`: Add **`#### Core.Eval.Scan`** after **`#### Core.Eval.Interpolate`**. Add alias `/scan`.
- Expression semantics: No change — `Core.Eval.Scan` does not introduce new expression forms.
- Runtime implementations: New verb to implement (compile format to match pattern, apply, assign refs).

### Dependencies
- **`Core.Eval.Interpolate`** (defines `${*ref}` and related syntax; Scan reuses **only** the simple hole subset in v1) — no Interpolate changes required; cross-link in spec.
- **`20260224-expr-spec-fixes-plan.md`** is **closed**; verify current `spec/2_verbs.md` / expression spec text when authoring the Plan so `${*ref}` in Scan is described consistently.

### Risks
| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Dual-use `${*ref}` confuses new users | Medium | Low | Clear spec: under **`/itpl`** substitute from ref; under **`/scan`** capture into ref. Verb name and **`Core.Eval`** placement imply input direction |
| Adjacent placeholder ambiguity silent failure | Medium | High | Require Fatal diagnostic; enforce at parse/compile time |
| Greedy vs. lazy semantics produce surprising captures | Medium | Medium | Default to lazy (shortest match); document with examples |
| Scope pollution from partial match failure | Low | Medium | Specify explicitly: on failure, no refs are modified |
| `nocase` + Unicode edge cases (locale-dependent folding) | Low | Medium | Use simple Unicode case-folding (no locale); document in Plan |
| `trim` interaction with empty captures | Low | Low | Trimming `""` stays `""`; trimming `"  "` yields `""`; coercion rules apply as normal |

### Breaking Changes
None. New verb only.

---

## Open Questions

- [ ] Should **`Core.Eval.Scan`** support any **`Core.Eval.Interpolate`** sub-forms beyond basic `${*ref}`, e.g., `$#{*ref}` or `$?{...}`? (Recommendation: **No** — v1 is `${*ref}` / `${*ref+}` + literals only; see **v1 scope** above.)
- [x] ~~Should captured values be automatically parsed to typed values?~~ **Resolved:** Captures are coerced to the declared type of the reference (same rules as `Core.Parse`). Untyped references receive `string`. Type coercion failure returns `false` with no refs set.
- [ ] Should **`Core.Eval.Scan`** support an optional `fallback` or default value for non-matching captures? (Recommendation: No defaults — match is all-or-nothing.)
- [x] ~~**How should lazy vs greedy be selected?**~~ **Resolved:** Per-placeholder suffix — `${*ref+}` for greedy, `${*ref}` for lazy (default). No verb-level attribute. Optional (`?`) capture deferred. Full regex capturing is out of scope for **`Core.Eval.Scan`**; a future verb (e.g. **`Core.Eval.ScanRegex`**) can handle that.
- [x] ~~**Should case-sensitivity be configurable?**~~ **Resolved:** `nocase: true` named parameter. Literal matching is case-sensitive by default; `nocase` opts in to case-insensitive comparison. `trim` added for whitespace-trimming captures before coercion.
- [ ] Alias: `/scan` is the natural alias. Is there a shorter one worth documenting? (`/fmt`? No — too ambiguous.)
- [ ] Should additional named parameters be considered for v1? (e.g. `limit: integer` to cap capture length, `start: integer` to begin scanning at an offset.) Recommendation: `nocase` and `trim` only for v1; others deferred.

---

## Next Steps

If accepted:
1. Create a Plan projex to add **`Core.Eval.Scan`** to `spec/2_verbs.md` (placement after **`Core.Eval.Interpolate`**) with full spec: named parameters (`nocase`, `trim`), positional parameters, returns, diagnostics, examples, adjacent-placeholder rule, **v1 scope**, **literal escaping**, **Unicode / equality** for literals (including case-folding algorithm for `nocase`), **empty captures**.
2. Reconcile wording with **current** spec (post-`20260224-expr-spec-fixes-plan.md`) so `${*ref}` in Scan matches formal interpolate terminology.
3. Add implementation spec for runtime: format → match pattern compilation, lazy/greedy flag, scope assignment.

---

## Implementation Notes

### No Regex Required

The **v1** format subset (only `${*ref}` / `${*ref+}` and literals) plus the adjacent-placeholder prohibition makes the scan algorithm implementable with pure string operations. No regex engine is needed.

**Algorithm:**

1. **Compile format** — Parse the format string into an alternating sequence: `[literal₀, ref₁, literal₁, ref₂, literal₂, ..., refN, literalN]`. This is a one-time parse of the format template.
2. **Verify prefix** — Input must start with `literal₀` (if `nocase: true`, compare case-insensitively). Advance past it.
3. **For each `refᵢ` / `literalᵢ` pair:**
   - Find `literalᵢ` in the remaining input (if `nocase: true`, find case-insensitively).
   - **Lazy (`${*ref}`, default):** take the *first* occurrence — capture everything before it.
   - **Greedy (`${*ref+}`):** take the *last* occurrence — capture everything before it.
   - Assign the captured segment to `refᵢ` (pending trim and type coercion).
   - Advance past `literalᵢ`.
4. **Verify suffix** — After the last placeholder, the remaining input must be exactly `literalN` (if `nocase: true`, compare case-insensitively). If not, return `false`.
5. **Trim** — If `trim: true`, strip leading and trailing whitespace from each captured segment.
6. **Type coercion** — Attempt to parse each (possibly trimmed) captured segment to its reference's declared type, using `Core.Parse` type-parsing logic without its unconditional pre-trim. If any fails, return `false` and apply no assignments.
7. **Assign** — All coercions succeeded; set each ref in current scope.

The only subtlety is when a separator literal appears inside a captured value (e.g., format `"${*A}:${*B+}"`, input `"a:b:c"`). Lazy `${*A}` gives `A="a"`, `B="b:c"`; greedy `${*A+}` gives `A="a:b"`, `B="c"`. Both are deterministic without regex.

---

## Appendix

### Prior Art

- **C `scanf`** — Reads formatted input, assigns captures to pointers. No named captures; positional only. **`Core.Eval.Scan`** improves on this with named `${*ref}` capture and structured literal delimiters.
- **Python `str.partition` / `re.match` with named groups** — Named capture groups (`(?P<name>...)`) are regex-based. **`Core.Eval.Scan`** reuses the same **simple** template tokens as **`Core.Eval.Interpolate`** for variable holes.
- **Rust `scan_fmt`** — Format-string scanning crate. Similar concept. **`Core.Eval.Scan`** is narrower: v1 holes are `${*ref}` / `${*ref+}` only; typing comes from ref declarations + `Core.Parse` rules.

### Alternatives Considered

- **Regex verb** (`Core.Match` with regex): Full regex support would subsume **`Core.Eval.Scan`** but introduces a complex new syntax domain. Scan is intentionally narrower and more accessible.
- **Core.Parse extension**: Extending `Core.Parse` with a `format` parameter was considered. Rejected — scan has multi-ref side effects vs. single typed return and would over-complicate **`Core.Parse`**.
