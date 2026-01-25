# spec-CORE-005: Conditional Interpolation Syntax Update

## Priority
**Low** - Syntactic consistency improvement.

## Problem

The current conditional interpolation syntax `$?{condition ? trueCase : falseCase}` uses `:` as a separator. This is inconsistent with other choice-based syntax in ZOH (like `any` which uses `|`) and potentially conflicts with future format string extensions where `:` is standard.

- `spec.md:707`: `$?{*var1? *var2 : *var3}`
- `impl/04_expressions.md:29`: `$?(' expr '?' expr ':' expr ')'`

## Proposed Change

Change the separator from `:` to `|`.

New Syntax: `$?{condition ? trueCase | falseCase}`

### Rationale
1. **Consistency**: Aligns with `$?{a|b|c}` (first non-nothing) and `${a|b|c}[index]` (choice).
2. **Ambiguity Reduction**: `:` is commonly used for formatting (e.g., `${var:format}`). Using `|` avoids parsing ambiguity if we introduce formatting inside interpolation blocks in the future.

### Specification Updates

#### `spec.md`
- Update line 707 and relevant examples to use `|` instead of `:`.

#### `impl/04_expressions.md`
- Update grammar rule: `conditional := '$?(' expr '?' expr '|' expr ')'`
- Update examples.

#### `impl/06_core_verbs.md`
- Update table and examples.

## Acceptance Criteria
1. `$?{true ? "Yes" | "No"}` returns "Yes".
2. `$?{false ? "Yes" | "No"}` returns "No".
3. `$?{true ? "Yes" : "No"}` is treated as invalid syntax (or deprecated if we want grace period, but strict is preferred).

## Risks & Unknowns
- Breaking change for any existing code using conditional interpolation (feature is new, so low risk).
