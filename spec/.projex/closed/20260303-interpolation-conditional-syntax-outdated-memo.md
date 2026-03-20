# Memo: Interpolation Conditional Syntax Outdated

> **Date:** 2026-03-03
> **Author:** Antigravity
> **Source Type:** Issue
> **Origin:** Noticed during execution of string interpolation formatting plan

---

## Source

In `impl/06_core_verbs.md`, the "Special Syntax in Strings" section lists the conditional interpolation syntax as:

`$?{*a? *b | *c}`

However, the `ExpressionTests.cs` and C# expression parser use the parenthesis syntax, which matches the standard expression forms:

`$?(*score >= 10 ? 'Win' : 'Lose')`

---

## Context

While fixing a bug in the string interpolation formatting implementation, a test failed because it was written using the curly brace syntax (`$?{...}`) mixed with the ternary operator (`? :`). The correct syntax for the expression parser is parenthesis-based (`$?(...)`). 

The `impl/06_core_verbs.md` specification still lists the old `{}` syntax in its table, which causes confusion and leads to malformed tests (like the one that failed during the formatting implementation). The spec needs to be updated to reflect the actual parenthesis-based syntax used by the parser.

---

## Related Projex

- [20260225-string-interpolation-formatting-plan.md](20260225-string-interpolation-formatting-plan.md)
