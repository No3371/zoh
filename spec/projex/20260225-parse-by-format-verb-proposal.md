# Core.Scan — Parse-by-Format Verb

> **Status:** Draft
> **Created:** 2026-02-25
> **Author:** agent
> **Related Projex:** 20260224-expr-spec-fixes-plan.md

---

## Summary

Propose `Core.Scan` — the structural inverse of `Core.Interpolate`. Where `Interpolate` substitutes `${*ref}` placeholders with variable values to produce a string, `Scan` matches a string against a `${*ref}` pattern and sets the captured segments into the referenced variables. Captured segments are coerced to the declared type of each reference (using the same rules as `Core.Parse`); untyped references receive `string`. Example: `/scan "${*MAJOR}.${*MINOR}.${*PATCH}", "1.2.3"` where MAJOR/MINOR/PATCH are `integer` references sets `*MAJOR = 1`, `*MINOR = 2`, `*PATCH = 3`.

---

## Problem Statement

### Current State

`Core.Interpolate` generates strings from templates. Its inverse — decomposing a structured string back into named parts — has no dedicated verb. Currently, extracting structured segments from a string requires `Core.Evaluate` with string arithmetic, manual index tracking, or regex-style workarounds outside the language model.

`Core.Parse` is adjacent but orthogonal: it converts a string to a scalar typed value (integer, double, boolean). It has no format/template concept.

### Gap / Need / Opportunity

Scripts that deal with versioned identifiers, structured tokens, file path patterns, date strings, or similar structured text have no idiomatic way to destructure them. A format-based scan verb completes the `Interpolate` ↔ `Scan` symmetry and makes this a first-class operation in the language.

### Why Now?

The versioning concept spec (`20260225-versioning-concept-spec-plan.md`) introduces version string handling as a concrete use case. The symmetry with `Core.Interpolate` makes the verb immediately useful and consistently motivated within the existing verb set.

---

## Proposed Change

### Overview

Add `Core.Scan` to `spec/2_verbs.md`. The verb accepts a format string and an input string. The format string uses `${*ref}` placeholders (same syntax as `Core.Interpolate`) as named capture positions. Literal text between placeholders acts as delimiters. On a successful match, each `*ref` is set to its captured segment coerced to the reference's declared type; untyped references receive `string`. Returns `true` on full match, `false` on mismatch.

```
:: *MAJOR, *MINOR, *PATCH declared as integer
/scan "${*MAJOR}.${*MINOR}.${*PATCH}", "1.2.3";
:: *MAJOR = 1, *MINOR = 2, *PATCH = 3  (integers)
:: returns true
```

### Approach Options

#### Option A: `Core.Scan` — symmetric with `Core.Interpolate`

- **Description:** Named after `scanf`. The format string uses `${*ref}` notation from Interpolate. Literal segments act as separators. Returns `boolean`.
- **Pros:** Consistent with existing `${*ref}` syntax. Immediately readable to anyone who knows `Interpolate`. Clean symmetry in the verb set. Boolean return enables conditional handling.
- **Cons:** `${*ref}` in `Interpolate` reads as "insert the value of `*ref` here"; in `Scan` it reads as "capture into `*ref` here" — opposite semantics, same syntax. This is a dual-use of the notation.
- **Effort:** Medium — spec authoring + implementation of format-to-scan-plan compilation (pure string operations; see Implementation Notes).

#### Option B: Distinct capture syntax (e.g., `{MAJOR}.{MINOR}.{PATCH}`)

- **Description:** Use a separate placeholder syntax for capture (no `${*}` wrapping) to distinguish from Interpolate's output-side usage.
- **Pros:** No semantic ambiguity — capture placeholders look different from interpolation placeholders.
- **Cons:** Introduces a second placeholder convention. Users must learn both. Breaks the "Scan is the inverse of Interpolate" visual symmetry.
- **Effort:** Same as Option A, plus spec for a new syntax form.

#### Option C: Predicate-style, returns a map rather than setting refs

- **Description:** `/scan "${MAJOR}.${MINOR}.${PATCH}", "1.2.3"` returns a map `{MAJOR: "1", MINOR: "2", PATCH: "3"}` instead of setting variables.
- **Pros:** No side-effectful reference assignment. Functional style. Composable.
- **Cons:** Diverges from the direct-assignment model used by `Core.Set`, `Core.Capture`. Requires an extra `/set` step to bind values. Less ergonomic for the primary use case.
- **Effort:** Same.

### Recommended Approach

**Option A** — `Core.Scan` with `${*ref}` placeholders.

The dual-use of `${*ref}` is acceptable because the direction of data flow is determined by the verb, not the syntax. `Interpolate` is always output-producing; `Scan` is always input-consuming. Users already hold the mental model of `${*ref}` as "this is a variable position." The symmetric naming (`Interpolate` ↔ `Scan`) reinforces intent. The boolean return is ergonomic for conditional destructuring flows.

---

## Specification Sketch

### Core.Scan

Scans an input string against a format template, capturing matched segments into references. Structural inverse of `Core.Interpolate`.

#### Parameters
- `format`: The format template. Accept `string` or `*string`. Must contain at least one `${*ref}` or `${*ref+}` placeholder. Literal text between placeholders must match exactly (case-sensitive, no whitespace normalization).
- `value`: The string to scan. Accept `string` or `*string`.

#### Returns
`true` if the entire input was matched and all captures were assigned. `false` if the input did not match the format. Does not return `nothing` — result is always boolean.

#### Side Effects
On successful match, each `${*ref}` placeholder in the format is set in the current scope to the corresponding captured segment, coerced to the reference's declared type using the same rules as `Core.Parse`. If a captured segment cannot be coerced to the declared type, the verb returns `false` and no references are set. For untyped references, the captured value is assigned as `string`. On failure (no match or type coercion failure), **no references are set** — existing values are preserved.

#### Placeholder Suffixes
- **`${*ref}`** (default): Lazy — captures the shortest segment that allows the remainder of the format to match.
- **`${*ref+}`**: Greedy — captures the longest segment before the next literal.

#### Diagnostics
- Fatal: `invalid_syntax` — Malformed format string (unclosed `${`, invalid reference name, adjacent placeholders with no separator).
- Warning: `match_failed` — Input did not match format pattern. Returns `false`.
- Warning: `type_coercion_failed` — A captured segment could not be parsed to the declared type of its reference. Returns `false`; no refs modified.

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
```

#### Adjacent Placeholders

Two `${*ref}` placeholders with no literal separator between them (`"${*A}${*B}"`) are ambiguous — there is no delimiter to separate the captures. This is a `Fatal: invalid_syntax` error. At least one literal character must appear between any two adjacent placeholders.

---

## Impact Analysis

### Affected Areas
- `spec/2_verbs.md`: Add `Core.Scan` verb definition. Add alias `/scan`.
- Expression semantics: No change — `Core.Scan` does not introduce new expression forms.
- Runtime implementations: New verb to implement (compile format to match pattern, apply, assign refs).

### Dependencies
- `Core.Interpolate` spec (defines `${*ref}` syntax that `Scan` reuses) — no changes needed, just reference.
- No dependency on `20260224-expr-spec-fixes-plan.md`, though that plan's `interpolate_form` syntax clarification may affect how `${*ref}` in `Scan` is formally described.

### Risks
| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Dual-use `${*ref}` confuses new users | Medium | Low | Clear spec language ("capture into ref" vs "substitute from ref"); verb name `Scan` strongly implies input direction |
| Adjacent placeholder ambiguity silent failure | Medium | High | Require Fatal diagnostic; enforce at parse/compile time |
| Greedy vs. lazy semantics produce surprising captures | Medium | Medium | Default to lazy (shortest match); document with examples |
| Scope pollution from partial match failure | Low | Medium | Specify explicitly: on failure, no refs are modified |

### Breaking Changes
None. New verb only.

---

## Open Questions

- [ ] Should `Scan` support any `Interpolate` sub-forms beyond basic `${*ref}`, e.g., `$#{*ref}` for count capture or `$?{...}` conditional forms? (Recommendation: No — keep v1 to basic ref capture only.)
- [x] ~~Should captured values be automatically parsed to typed values?~~ **Resolved:** Captures are coerced to the declared type of the reference (same rules as `Core.Parse`). Untyped references receive `string`. Type coercion failure returns `false` with no refs set.
- [ ] Should `Scan` support an optional `fallback` or default value for non-matching captures? (Recommendation: No defaults — match is all-or-nothing.)
- [x] ~~**How should lazy vs greedy be selected?**~~ **Resolved:** Per-placeholder suffix — `${*ref+}` for greedy, `${*ref}` for lazy (default). No verb-level attribute. This is fine-grained and carries no blunt uniformity problem. Optional (`?`) capture deferred — separator coupling makes it non-intuitive. Full regex capturing is out of scope for `Core.Scan`; a future `Core.ScanRegex` verb will handle that use case.
- [ ] Alias: `/scan` is the natural alias. Is there a shorter one worth documenting? (`/fmt`? No — too ambiguous.)

---

## Next Steps

If accepted:
1. Create a Plan projex to add `Core.Scan` to `spec/2_verbs.md` with full spec (parameters, returns, diagnostics, examples, adjacent-placeholder rule).
2. Coordinate with `20260224-expr-spec-fixes-plan.md` to ensure `interpolate_form` clarifications in that plan are consistent with `Scan`'s reuse of `${*ref}` syntax.
3. Add implementation spec for runtime: format → match pattern compilation, lazy/greedy flag, scope assignment.

---

## Implementation Notes

### No Regex Required

The adjacent-placeholder prohibition (at least one literal character between any two `${*ref}` placeholders) makes the scan algorithm implementable with pure string operations. No regex engine is needed.

**Algorithm:**

1. **Compile format** — Parse the format string into an alternating sequence: `[literal₀, ref₁, literal₁, ref₂, literal₂, ..., refN, literalN]`. This is a one-time parse of the format template.
2. **Verify prefix** — Input must start with `literal₀`. Advance past it.
3. **For each `refᵢ` / `literalᵢ` pair:**
   - Find `literalᵢ` in the remaining input.
   - **Lazy (`${*ref}`, default):** take the *first* occurrence — capture everything before it.
   - **Greedy (`${*ref+}`):** take the *last* occurrence — capture everything before it.
   - Assign the captured segment to `refᵢ` (pending type coercion).
   - Advance past `literalᵢ`.
4. **Verify suffix** — After the last placeholder, the remaining input must be exactly `literalN`. If not, return `false`.
5. **Type coercion** — Attempt to parse each captured segment to its reference's declared type. If any fails, return `false` and apply no assignments.
6. **Assign** — All coercions succeeded; set each ref in current scope.

The only subtlety is when a separator literal appears inside a captured value (e.g., format `"${*A}:${*B+}"`, input `"a:b:c"`). Lazy `${*A}` gives `A="a"`, `B="b:c"`; greedy `${*A+}` gives `A="a:b"`, `B="c"`. Both are deterministic without regex.

---

## Appendix

### Prior Art

- **C `scanf`** — Reads formatted input, assigns captures to pointers. No named captures; positional only. `Scan` improves on this with named `${*ref}` capture and structured literal delimiters.
- **Python `str.partition` / `re.match` with named groups** — Named capture groups (`(?P<name>...)`) are regex-based, requiring a separate syntax. `Scan` uses the same template notation as `Interpolate`, making it more accessible.
- **Rust `scan_fmt`** — Format-string scanning crate. Similar concept. `Scan` is simpler: no type specifiers, always captures as string.

### Alternatives Considered

- **Regex verb** (`Core.Match` with regex): Full regex support would subsume `Scan` but introduces a complex new syntax domain. `Scan` is intentionally narrower and more accessible.
- **Core.Parse extension**: Extending `Core.Parse` with a `format` parameter was considered. Rejected — `Scan` has fundamentally different semantics (multi-variable side effects vs. single typed return value) and would over-complicate `Parse`.
