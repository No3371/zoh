# Implement Slash-Prefix Syntax for Evaluate and Interpolate Sugar

**Scope**: Specification and implementation guides only (no runtime code changes)

Change ZOH's syntactic sugar documentation from dollar-prefix to slash-prefix to align with the verb-centric design philosophy. This makes `/evaluate` and `/interpolate` sugar visually consistent with verb calls.

| Current | New | Desugars To |
|---------|-----|-------------|
| `` $`*a + *b` `` | `` /`*a + *b` `` | `` /evaluate `*a + *b`; `` |
| `$"Hello ${*name}"` | `/"Hello ${*name}"` | `/interpolate "Hello ${*name}";` |

In-expression special forms (`$()`, `$#()`, `$?()`) retain `$` prefix - this change only affects statement-level sugar.

**Related**: `projects/rfc-SUGAR-001-slash-prefix-syntax.md`

---

## Tasks

### 1. Update Language Specification

**File**: `spec.md`

- Update syntactic sugar table to show `` /`...` `` and `` /"..." `` forms
- Update all code examples using `` $`...` `` or `` $"..." `` syntax

### 2. Update Lexer Implementation Guide

**File**: `impl/01_lexer.md`

- Replace `DollarBacktick` token with `SlashBacktick`
- Replace `DollarQuote` token with `SlashQuote`
- Update token definitions table

### 3. Update Parser Implementation Guide

**File**: `impl/02_parser.md`

- Update grammar rules:
  ```ebnf
  eval_sugar       := SLASH_BACKTICK expr_content BACKTICK attributes? SEMICOLON
  interpolate_sugar := SLASH_QUOTE string_content QUOTE attributes? SEMICOLON
  ```
- Update any examples using the old syntax

---

## Verification

- All spec examples consistently use new syntax
- Implementation guides accurately describe the new token types
- Grammar rules in parser guide are updated
