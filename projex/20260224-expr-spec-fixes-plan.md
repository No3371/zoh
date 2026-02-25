# Expression Grammar Spec Fixes

> **Status:** Ready
> **Created:** 2026-02-24
> **Author:** agent
> **Source:** `20260224-expr-spec-eval.md`
> **Related Projex:** `20260224-expr-spec-eval.md`, `20260224-expr-spec-fixes-impl-plan.md`

---

## Summary

Fixes all spec-level issues identified in the expression grammar evaluation: adds the `**` exponentiation operator, corrects `interpolate_form` syntax, updates the operators and type-behavior tables (including Map column and `&&`/`||` row), adds a Truthiness section, adds grammar disambiguation notes, documents edge-case semantics (nothing, division-by-zero, short-circuit, count, out-of-bounds), and fixes misleading/incomplete diagnostics in `spec/2_verbs.md` (Core.Evaluate and Core.Assert).

**Scope:** `spec/expr.md` and `spec/2_verbs.md` only — no implementation files.
**Estimated Changes:** 2 files, ~10 targeted edits.

---

## Objective

### Problem / Gap / Need
`spec/expr.md` has 3 major divergences from the implementation and verb spec, 5 moderate grammar ambiguities, and 8 minor semantic gaps documented in `20260224-expr-spec-eval.md`. These prevent independent implementations from being conformant.

### Success Criteria
- [ ] Main grammar includes `power_expr` and `**` operator; `integer_literal`/`double_literal` have no sign
- [ ] `interpolate_form` grammar matches the syntax in `spec/2_verbs.md` Core.Evaluate (`$"str"` / `$*ref`)
- [ ] Special forms semantics table updated to match corrected `interpolate_form` syntax
- [ ] `index_spec` allows full expressions (not just reference/integer_literal)
- [ ] Grammar disambiguation notes present for `any_form`/`conditional_form` and `roll_form`/`wroll_form`
- [ ] Precedence table has 8 rows with `**` at level 1
- [ ] Type-Specific Behavior table has Map column; `**` row; `<`/`>`/`<=`/`>=` row for strings; `&&`/`||` row
- [ ] New Truthiness section added to `spec/expr.md` with full per-type table
- [ ] New Edge Cases section covers: div-by-zero, nothing operators, short-circuit, `$#()` types, all-nothing any, `$(options)` without brackets
- [ ] `Core.Evaluate` diagnostics updated: misleading example fixed, `division_by_zero` and `invalid_index` added
- [ ] `Core.Assert` truthiness description updated to match full truthiness table

### Out of Scope
- Implementation changes (`impl/04_expressions.md`) — covered in `20260224-expr-spec-fixes-impl-plan.md`
- String interpolation syntax (`${...}`) inside strings — belongs to `spec/0_basic.md`
- Map `+` merge operator semantics — not defined in the implementation; remains N/A

---

## Context

### Current State
`spec/expr.md` grammar has `unary_expr → primary_expr` (no power level), signed literals, `interpolate_form := '$(' expression ')'` (wrong syntax), and `index_spec` limited to `reference | integer_literal`. The operators section has 7 precedence levels and a 5-column type table missing Map, `**`, string comparisons, and `&&`/`||`. No Truthiness section or edge-case semantics exist. `spec/2_verbs.md` Core.Evaluate has a misleading `invalid_type` example and is missing `division_by_zero` / `invalid_index` diagnostics. Core.Assert's truthiness description ("falsy: `false` or `nothing`") is incomplete — the impl defines falsy as also including `0`, `0.0`, `""`, `[]`, `{}`.

### Key Files
| File | Purpose | Changes Needed |
|------|---------|----------------|
| `spec/expr.md` | Expression grammar specification | Grammar, tables, edge cases section |
| `spec/2_verbs.md` | Core verb specs including Core.Evaluate and Core.Assert | Fix `invalid_type` example, add diagnostics; fix Core.Assert truthiness |

### Dependencies
- **Requires:** `20260224-expr-spec-eval.md` (source of all findings — already committed)
- **Blocks:** `20260224-expr-spec-fixes-impl-plan.md` (impl doc fixes reference this spec)

---

## Implementation

### Overview
Ten sequential edits across two files. All changes are documentation — no code. Each edit is self-contained and can be verified by reading the output.

---

### Step 1: Fix main grammar — add `power_expr`, remove signs from literals

**Objective:** Add `**` exponentiation level to the grammar; eliminate sign-in-literal ambiguity with unary `-`.

**Files:** `spec/expr.md`

**Changes:**

```diff
// spec/expr.md — inside the ```ebnf block, lines ~23-30

- unary_expr      := ( '!' | '-' ) unary_expr | primary_expr
- primary_expr    := literal | reference | '(' expression ')' | special_form
+ unary_expr      := ( '!' | '-' ) unary_expr | power_expr
+ power_expr      := primary_expr ( '**' unary_expr )*
+ primary_expr    := literal | reference | '(' expression ')' | special_form

- integer_literal := ['-'] <digits>
- double_literal  := ['-'] <digits> '.' <digits>
+ integer_literal := <digits>
+ double_literal  := <digits> '.' <digits>
```

**Rationale:** `**` is right-associative and binds tighter than unary, which is the standard definition (e.g., `-2**2 = -(2**2) = -4`). Removing `['-']` from literals eliminates the grammar ambiguity where `-5` could parse two ways; unary `-` on `primary_expr` already handles negative numbers.

**Verification:** Grammar block reads correctly; `unary_expr` references `power_expr`; `power_expr` references `primary_expr`; literals have no `['-']`.

---

### Step 2: Fix `interpolate_form` grammar and `index_spec`

**Objective:** Align `interpolate_form` with the syntax described in `spec/2_verbs.md` Core.Evaluate; relax `index_spec` to accept full expressions.

**Files:** `spec/expr.md`

**Changes — inside the second ```ebnf block (Special Forms grammar):**

```diff
- interpolate_form := '$(' expression ')'
+ interpolate_form := '$' string_literal | '$' reference

- index_spec          := ['!'] ( reference | integer_literal )
+ index_spec          := ['!'] expression
```

**Rationale:** The verb spec and impl both use `$"str"` / `$*ref` syntax (no parens) for interpolation. The paren form was creating an ambiguity with `$(option_list)[...]` indexed forms. `index_spec` being limited to `reference | integer_literal` was unnecessarily restrictive — the impl already parses full expressions here (`index = parse()`), and `$(a|b|c)[*offset + 1]` is a natural and useful expression.

**Verification:** `interpolate_form` rule uses `'$' string_literal | '$' reference`; `index_spec` reads `['!'] expression`.

---

### Step 3: Update Special Forms semantics table

**Objective:** Reflect the corrected `interpolate_form` syntax; document the all-nothing result for `any_form`.

**Files:** `spec/expr.md`

**Changes — the semantics table after "### Special Form Semantics":**

```diff
- | `$(*var)` | Interpolated string |
+ | `$*var` | Value of `*var` treated as string template and interpolated |
+ | `$"string"` | String literal interpolated |
  | `$#(*var)` | Count/length of collection or string |
  | `$?(*cond? *a : *b)` | `*a` if `*cond` is truthy, else `*b` |
- | `$?(*a\|*b\|*c)` | First non-nothing value |
+ | `$?(*a\|*b\|*c)` | First non-nothing value; `?` if all are nothing |
  | `$(1\|2\|3)[*i]` | Element at index `*i` |
  | `$(1\|2\|3)[!*i]` | Element at index `*i % count` (wrap-around) |
  | `$(a\|b\|c)[%]` | Random element |
  | `$(a:1\|b:2)[%]` | Weighted random selection |
```

**Rationale:** The old `$(*var)` was the paren-form which is being removed. Two separate rows clearly distinguish string-literal interpolation from variable-reference interpolation. All-nothing result is now documented explicitly.

**Verification:** Table has two interpolate rows (`$*var`, `$"string"`); any_form row ends with `; \`?\` if all are nothing`.

---

### Step 4: Add grammar disambiguation notes

**Objective:** Explain the single-token lookahead rules that distinguish `any_form` from `conditional_form`, and `roll_form` from `wroll_form`.

**Files:** `spec/expr.md`

**Changes — add after the closing ` ``` ` of the Special Forms grammar block, before "### Special Form Semantics":**

```markdown
> **Disambiguation — `any_form` vs `conditional_form`:** Both start with `$?(`. They are distinguished by the token after the first expression: `?` signals conditional form; `|` or `)` signals any form.
>
> **Disambiguation — `roll_form` vs `wroll_form`:** Both use `$(options)[%]`. A form is weighted (`wroll_form`) when any option includes a `:` weight suffix (`expression ':' integer_literal`). Detection happens during parsing of the option list.
>
> **`$(options)` without suffix:** `$(a|b|c)` not followed by `[index]` or `[%]` is a parse error. Use `$?(a|b|c)` for first-non-nothing selection.
```

**Rationale:** Disambiguation rules are not derivable from EBNF alone. These notes let an implementer correctly resolve all LL(1) conflicts without reading the implementation.

**Verification:** Three disambiguation notes present after the special forms grammar block.

---

### Step 5: Update precedence table — add `**` at level 1

**Objective:** Insert `**` as highest-precedence operator, shift all others down by one.

**Files:** `spec/expr.md`

**Changes — the `### Precedence (Highest to Lowest)` table:**

```diff
- | 1 | `!` `-` (unary) | Right |
- | 2 | `*` `/` `%` | Left |
- | 3 | `+` `-` | Left |
- | 4 | `<` `>` `<=` `>=` | Left |
- | 5 | `==` `!=` | Left |
- | 6 | `&&` | Left |
- | 7 | `\|\|` | Left |
+ | 1 | `**` | Right |
+ | 2 | `!` `-` (unary) | Right |
+ | 3 | `*` `/` `%` | Left |
+ | 4 | `+` `-` | Left |
+ | 5 | `<` `>` `<=` `>=` | Left |
+ | 6 | `==` `!=` | Left |
+ | 7 | `&&` | Left |
+ | 8 | `\|\|` | Left |
```

**Verification:** Table has 8 rows; row 1 is `**` Right; all previous rows shifted down by one.

---

### Step 6: Replace Type-Specific Behavior table

**Objective:** Add Map column; add `**` row; add `<`/`>`/`<=`/`>=` row with string lexicographic support; add `&&`/`||` row using truthiness coercion.

**Files:** `spec/expr.md`

**Changes — replace the entire `### Type-Specific Behavior` table:**

```diff
- | Operator | Integer | Double | String | List | Boolean |
- |----------|---------|--------|--------|------|---------|
- | `+` | Add | Add | Concatenate | Concatenate | N/A |
- | `-` | Subtract | Subtract | N/A | N/A | N/A |
- | `*` | Multiply | Multiply | N/A | N/A | N/A |
- | `/` | Divide (floor) | Divide | N/A | N/A | N/A |
- | `%` | Modulo | N/A | N/A | N/A | N/A |
- | `==` | Equal | Equal | Equal (case-sensitive) | Deep equal | Equal |
- | `!=` | Not equal | Not equal | Not equal | Not deep equal | Not equal |
+ | Operator | Integer | Double | String | List | Map | Boolean |
+ |----------|---------|--------|--------|------|-----|---------|
+ | `**` | Power¹ | Power | N/A | N/A | N/A | N/A |
+ | `+` | Add | Add | Concatenate | Concatenate | N/A | N/A |
+ | `-` | Subtract | Subtract | N/A | N/A | N/A | N/A |
+ | `*` | Multiply | Multiply | N/A | N/A | N/A | N/A |
+ | `/` | Divide (floor) | Divide | N/A | N/A | N/A | N/A |
+ | `%` | Modulo | N/A | N/A | N/A | N/A | N/A |
+ | `<` `>` `<=` `>=` | Compare | Compare | Lexicographic | N/A | N/A | N/A |
+ | `==` | Equal | Equal | Equal (case-sensitive) | Deep equal | Deep equal | Equal |
+ | `!=` | Not equal | Not equal | Not equal | Not deep equal | Not deep equal | Not equal |
+ | `&&` `\|\|` | →bool² | →bool² | →bool² | →bool² | →bool² | →bool |
+
+ ¹ Result type: `integer ** integer` (exponent ≥ 0) → `integer`; any other numeric combination → `double`.
+ ² Operands coerced to boolean via truthiness rules (see §Truthiness); result is always `boolean`.
```

**Rationale:** Map `==`/`!=` deep equality was implied but unspecified — `map` should behave the same as `list` for equality. Lexicographic string comparison (`<`, `>`, etc.) is a common need that was previously unspecified. `**` type behavior directly mirrors the implementation's type coercion table. `&&`/`||` are now explicit in the table, referencing the new §Truthiness section. All N/A cells produce fatal `invalid_type` at runtime.

**Verification:** Table has 7 columns (Operator + 6 types including Map); 10 rows (including `**`, `<>` row, equality rows, and `&&`/`||` row); two footnotes present.

---

### Step 7: Add Truthiness section to `spec/expr.md`

**Objective:** Add a dedicated `### Truthiness` subsection under `## Operators` containing the full per-type truthiness table. This gives `&&`/`||`, `!`, and all conditional forms a canonical reference within the expression spec.

**Files:** `spec/expr.md`

**Changes — insert after the Type-Specific Behavior table, before `## Variable Resolution`:**

```markdown
### Truthiness

Boolean coercion is used by `&&`, `||`, `!`, and all conditional forms (`$?`). The following table defines what is truthy and falsy for each type:

| Type | Truthy | Falsy |
|------|--------|-------|
| `nothing` | — | Always falsy |
| `boolean` | `true` | `false` |
| `integer` | Non-zero | `0` |
| `double` | Non-zero | `0.0` |
| `string` | Non-empty | `""` |
| `list` | Non-empty | `[]` |
| `map` | Non-empty | `{}` |
| `channel` | Always truthy | — |
| `verb` | Always truthy | — |
| `expression` | Always truthy | — |
```

**Rationale:** The spec currently has no canonical truthiness definition — it was scattered across Core.Assert (incomplete) and the impl. Since `&&`/`||` now appear in the type table referencing §Truthiness, the section must exist in `spec/expr.md`. The table matches `impl/05_type_system.md` exactly.

**Verification:** `### Truthiness` subsection present under `## Operators`; table has 10 rows covering all types.

---

### Step 8: Add Edge Cases section

**Objective:** Document runtime semantics for division-by-zero, nothing in expressions, short-circuit evaluation, `$#()` count by type, all-nothing any form, and the illegal `$(options)` bare form.

**Files:** `spec/expr.md`

**Changes — insert a new `## Edge Cases` section between the Operators section and `## Variable Resolution`:**

```markdown
## Edge Cases

### Division and Modulo by Zero

Division or modulo with a zero divisor is a fatal `division_by_zero` error.

```zoh
`*x / 0`   :: fatal: division_by_zero
`*x % 0`   :: fatal: division_by_zero
```

### `nothing` in Operator Expressions

The `?` nothing literal supports equality comparison only:

| Expression | Result |
|------------|--------|
| `*a == ?` | `true` if `*a` is `nothing`, `false` otherwise |
| `*a != ?` | `false` if `*a` is `nothing`, `true` otherwise |

All other operators with a `nothing` operand produce fatal `invalid_type`.

### Short-Circuit Evaluation

`&&` and `||` are short-circuit (lazy):
- `false && X` — `X` is never evaluated; result is `false`
- `true || X` — `X` is never evaluated; result is `true`

Errors in unevaluated branches are not raised.

### `$#()` Count by Type

| Type | Result |
|------|--------|
| `list` | Number of elements |
| `map` | Number of key-value pairs |
| `string` | Number of Unicode code points |
| All other types | Fatal `invalid_type` |

### `$(options)` Indexed Form Out-of-Bounds

Accessing `$(a|b|c)[5]` with an out-of-bounds index is a fatal `invalid_index` error. Wrap-around (`[!5]`) uses modulo and is never out-of-bounds.
```

**Rationale:** Each of these was previously unspecified, forcing implementations to guess. Standardizing these semantics ensures conformant behavior across runtimes.

**Verification:** New `## Edge Cases` section appears between Operators and Variable Resolution; contains five subsections (division-by-zero, nothing, short-circuit, count, out-of-bounds).

---

### Step 9: Fix `spec/2_verbs.md` — Core.Evaluate diagnostics and Core.Assert truthiness

**Objective:** Fix the misleading `invalid_type` example; add `division_by_zero` and `invalid_index` diagnostics to Core.Evaluate; expand Core.Assert's incomplete truthiness description to match the full truthiness table.

**Files:** `spec/2_verbs.md`

**Changes — in `### Core.Evaluate` → `#### Diagnostics`:**

```diff
- - Fatal: `invalid_syntax`: Malformed expression syntax.
- - Fatal: `invalid_type`: Type error during expression evaluation (e.g., adding string to integer).
- - Fatal: `undefined_var`: Any variable used in the expression is undefined.
+ - Fatal: `invalid_syntax`: Malformed expression syntax.
+ - Fatal: `invalid_type`: Type error during expression evaluation (e.g., subtracting integer from string, or using arithmetic on `nothing`).
+ - Fatal: `undefined_var`: Any variable used in the expression is undefined.
+ - Fatal: `division_by_zero`: Division or modulo operation with a zero divisor.
+ - Fatal: `invalid_index`: Index out of bounds in an `$(options)[n]` indexed form.
```

**Changes — in `### Core.Assert` description, the truthiness parenthetical:**

```diff
- An assert verb checks a condition and emits a fatal diagnostic with the given message if the condition is not met (falsy: `false` or `nothing`). If the condition is met (truthy: anything that is not falsy), no diagnostic is emitted.
+ An assert verb checks a condition and emits a fatal diagnostic with the given message if the condition is not met (falsy: `false`, `nothing`, `0`, `0.0`, `""`, `[]`, `{}`). If the condition is met (truthy: anything that is not falsy), no diagnostic is emitted.
```

**Rationale:** The `invalid_type` example was misleading — `string + integer` is valid concatenation. Core.Assert's "falsy: `false` or `nothing`" is the only place the spec defines truthiness but it omits all the zero/empty cases present in `impl/05_type_system.md`, leaving `&&`/`||` and `/if` semantics underspecified for non-boolean types.

**Verification:** `invalid_type` example says "subtracting integer from string"; two new Fatal diagnostics present in Core.Evaluate; Core.Assert falsy list includes `0`, `0.0`, `""`, `[]`, `{}`.

---

## Verification Plan

### Manual Verification
- [ ] Read through `spec/expr.md` end-to-end — grammar, special forms, and edge cases are internally consistent
- [ ] Confirm `interpolate_form` grammar matches Core.Evaluate verb spec examples (`$"..."`, `$*var`)
- [ ] Confirm precedence table has 8 rows with `**` at level 1 (highest)
- [ ] Confirm type table has Map column, `**` row, and `&&`/`||` row
- [ ] Confirm `### Truthiness` subsection present under `## Operators` with 10-row table
- [ ] Confirm Edge Cases section is present and covers all 5 topics
- [ ] Confirm `spec/2_verbs.md` Core.Evaluate diagnostics list includes `division_by_zero` and `invalid_index`
- [ ] Confirm `spec/2_verbs.md` Core.Assert falsy list includes `0`, `0.0`, `""`, `[]`, `{}`

### Acceptance Criteria Validation
| Criterion | How to Verify | Expected Result |
|-----------|---------------|-----------------|
| `**` in grammar | Read EBNF block | `power_expr` rule present, `unary_expr` references it |
| `interpolate_form` | Read EBNF block | `'$' string_literal \| '$' reference` |
| Precedence table | Count rows | 8 rows, `**` first |
| Type table | Count columns, rows | 7 columns including Map; 10 rows including `&&`/`||` |
| Truthiness section | Read §Truthiness | Table with 10 type rows |
| Core.Evaluate diagnostics | Read spec/2_verbs.md §Core.Evaluate | 5 fatal diagnostics including `division_by_zero` |
| Core.Assert truthiness | Read spec/2_verbs.md §Core.Assert | Falsy list includes `0`, `0.0`, `""`, `[]`, `{}` |

---

## Rollback Plan

All changes are to Markdown documentation. Rollback via `git revert` on the commit.

---

## Notes

### Assumptions
- Map `+` (merge) is intentionally left N/A — the operation is not defined in the implementation or verb spec
- Lexicographic string comparison (`<`, `>`, etc.) follows Unicode code point ordering — the same ordering as most host languages' default string comparison
- `**` negative-integer-exponent behavior (integer → double) matches the impl type coercion table
- `nothing != nothing` returns `false` (i.e., `nothing == nothing` returns `true`)

### Risks
- The `<`/`>`/`<=`/`>=` string comparison row is new behavior — if any existing implementation has been treating it as `invalid_type`, this is a breaking change. Risk is low given the spec was previously silent (no conforming implementation could depend on either behavior).
