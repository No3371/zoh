# spec-CORE-002: Refine Interpolation Syntax in Expressions

## Priority
**High** - Current syntax `$(...)` is overloaded and verbose, leading to user confusion and parsing complexity.

## Problem
The `$(...)` token is currently overloaded in expressions. It is used for both:
1. Interpolation: `$("Hello ${*name}")`
2. Option Lists: `$(1|2|3)[%]`

This violates the principle of least surprise and makes the grammar harder to reason about. Additionally, `$("...")` is verbose for simple string interpolation compared to modern languages.

## Proposed Change
Refactor the specification to use:
- `$"string"` for string literals with interpolation.
- `$*var` for variable reference interpolation.
- Retain `$(...)` **only** for Option Lists (grouping of alternatives).

This aligns ZOH with C# string interpolation (`$"..."`) and provides a cleaner syntax.

**Reference:** `projects/proposal-core-002-interpolation-syntax.md`

## Specification Updates

### `spec.md`
- Update "Core.Evaluate" section (~line 754) to remove `$(string)` interpolation syntax.
- Add `$"..."` and `$*...` to Expression Grammar.
- Ensure all examples reflect the new syntax.
- Add "Interactions" section detailing precedence and escaping.

### `impl/04_expressions.md`
- Update "Expression Grammar (EBNF)" to replace `interpolate := '$(' string | reference ')'` with `interpolate := '$' string | '$' reference`.
- Update "Implementation Steps" (Lexer/Parser) to reflect new tokens.
- Update "Testing Checklist" with new syntax examples.

## Acceptance Criteria
1. `spec.md` reflects the new syntax clearly.
2. `impl/04_expressions.md` EBNF and pseudocode parser implementation are updated.
3. No ambiguity exists in the spec between Option Lists and Interpolation.
