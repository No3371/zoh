# RFC: Slash-Prefix Syntax for Evaluate and Interpolate Sugar

**Status**: Proposal
**Impact**: Breaking change to syntactic sugar
**Scope**: Lexer, Parser, Spec, Documentation
**Related**: `spec.md` (lines 714-771), `expr.md`, `impl/02_parser.md`

---

## Executive Summary

This proposal explores changing ZOH's syntactic sugar for `/evaluate` and `/interpolate` from dollar-prefix (`$`) to slash-prefix (`/`):

| Current | Proposed | Desugars To |
|---------|----------|-------------|
| `` $`*a + *b` `` | `` /`*a + *b` `` | `` /evaluate `*a + *b`; `` |
| `$"Hello ${*name}"` | `/"Hello ${*name}"` | `/interpolate "Hello ${*name}";` |

The motivation is **consistency with ZOH's verb-centric design**: since `/evaluate` and `/interpolate` are verbs, their sugar forms should visually resemble verb calls.

However, this change creates a **split syntax** where statement-level sugar uses `/` but in-expression special forms (`$()`, `$#()`, `$?()`) retain `$`. This trade-off must be weighed against the consistency gains.

---

## Current State

### Statement-Level Sugar (Target of This Change)

```zoh
:: Evaluate sugar
$`*health - *damage`;                    :: Desugars to /evaluate
$`*count + 1` [scope:"context"];         :: With attributes

:: Interpolate sugar
$"Hello, ${*name}!";                     :: Desugars to /interpolate
$'Welcome to ${*location}';              :: Single quotes supported
```

### In-Expression Special Forms (NOT Changed)

Inside backtick expressions, special forms use `$` prefix:

```zoh
/evaluate `$(*greeting) + " " + $(*name)`;     :: $() interpolation
/evaluate `$#(*items)`;                         :: $#() count
/evaluate `$?(*cond? *a : *b)`;                 :: $?() conditional
/evaluate `$(*a|*b|*c)[%]`;                     :: Random selection
```

These are **evaluated within expressions**, not statement-level constructs.

---

## Motivation

### 1. Verb-Centric Consistency

ZOH's philosophy is "everything is a verb." Current sugar forms:

| Sugar | Prefix | Feels Like |
|-------|--------|------------|
| `*var <- value;` | `*` | Variable operation |
| `====> @label;` | `=` | Flow control |
| `$`expr`;` | `$` | **Something else** |
| `$"itpl";` | `$` | **Something else** |

The `$` prefix stands out as non-verb-like. Changing to `/` aligns with verb syntax:

```zoh
/converse "Hello";      :: Verb
/evaluate `1 + 1`;      :: Verb (explicit)
/`1 + 1`;               :: Verb sugar (proposed)
```

### 2. Visual Grouping

With `/` prefix, all executable statements share a common visual anchor:

```zoh
:: Proposed - unified visual flow
/set *name, "Alice";
/"Hello, ${*name}!";
/`$#(*items) > 0`;
/if *ready, /proceed;;
```

### 3. Reduced Cognitive Load

One less prefix character to learn for statement-level constructs.

---

## Proposed Changes

### Syntax Transformation

```zoh
:: CURRENT                          :: PROPOSED
$`*a + *b`;                         /`*a + *b`;
$`expr` [attr];                     /`expr` [attr];
$"Hello ${*name}";                  /"Hello ${*name}";
$'Single ${*quote}';                /'Single ${*quote}';
```

### Grammar Update

```ebnf
:: Current (impl/02_parser.md)
eval_sugar       := DOLLAR_BACKTICK expr_content BACKTICK attributes? SEMICOLON
interpolate_sugar := DOLLAR_QUOTE string_content QUOTE attributes? SEMICOLON

:: Proposed
eval_sugar       := SLASH_BACKTICK expr_content BACKTICK attributes? SEMICOLON
interpolate_sugar := SLASH_QUOTE string_content QUOTE attributes? SEMICOLON
```

### Token Changes

| Current Token | New Token | Pattern |
|---------------|-----------|---------|
| `DollarBacktick` | `SlashBacktick` | `` /` `` |
| `DollarQuote` | `SlashQuote` | `/"` or `/'` |

---

## The Split Syntax Problem

### What Stays the Same

In-expression special forms **retain `$` prefix**:

```zoh
/`$(*var)`;              :: Interpolation in expression
/`$#(*list)`;            :: Count in expression
/`$?(*cond? *a : *b)`;   :: Conditional in expression
/`$(*a|*b|*c)[%]`;       :: Random selection
```

### Why This Creates Inconsistency

After the change, a script would have both prefixes:

```zoh
:: Statement level uses /
/"Player ${*name} has ${*hp} HP";

:: But inside expressions, $ remains
/`$(*name) + " dealt " + $(*damage) + " damage"`;
```

### Rationale for Keeping `$` in Expressions

1. **Different context**: Statement sugar vs. in-expression evaluation
2. **Parser complexity**: Changing `$()` inside expressions requires expression grammar changes
3. **Disambiguation**: `$` clearly marks "special evaluation" within expression context
4. **Scope of change**: Limiting to statement-level keeps change manageable

---

## Impact Analysis

### Breaking Changes

| Impact | Description |
|--------|-------------|
| **All existing scripts** | `$`...`` and `$"..."` statements must be updated |
| **Documentation** | spec.md, impl/*.md, tutorials need rewrite |
| **Tooling** | Syntax highlighters, linters need updates |

### Lexer Changes

Current `/` handling (Lexer.cs lines 122-127):
```csharp
case '/':
    if (Peek() == ';') { Advance(); return Token(TokenType.SlashSemicolon); }
    if (Peek() == '/') { Advance(); return Token(TokenType.DoubleSlash); }
    return Token(TokenType.Slash);
```

Proposed (add cases for backtick/quote):
```csharp
case '/':
    if (Peek() == '`') { Advance(); return Token(TokenType.SlashBacktick); }
    if (Peek() == '"') { Advance(); return Token(TokenType.SlashQuote); }
    if (Peek() == '\'') { Advance(); return Token(TokenType.SlashQuote); }
    if (Peek() == ';') { Advance(); return Token(TokenType.SlashSemicolon); }
    if (Peek() == '/') { Advance(); return Token(TokenType.DoubleSlash); }
    return Token(TokenType.Slash);
```

### Parser Changes

Update `ParseStatement()` to handle new tokens:
```csharp
// Current
case TokenType.DollarBacktick: return ParseEvalSugar();
case TokenType.DollarQuote: return ParseInterpolateSugar();

// Proposed
case TokenType.SlashBacktick: return ParseEvalSugar();
case TokenType.SlashQuote: return ParseInterpolateSugar();
```

---

## Files Affected

| File | Change Type | Description |
|------|-------------|-------------|
| `spec.md` | Documentation | Update lines 714-771, all examples |
| `expr.md` | Documentation | Add note distinguishing statement vs expression forms |
| `impl/01_lexer.md` | Implementation spec | Update token definitions |
| `impl/02_parser.md` | Implementation spec | Update grammar rules |
| `c#/src/Zoh.Runtime/Lexing/Lexer.cs` | Code | Add SlashBacktick/SlashQuote detection |
| `c#/src/Zoh.Runtime/Lexing/TokenType.cs` | Code | Add new token types |
| `c#/src/Zoh.Runtime/Parsing/Parser.cs` | Code | Update ParseStatement switch |
| `c#/tests/Zoh.Tests/**` | Tests | Update all tests using `$` sugar |

---

## Pros and Cons

### FOR the Change

| Benefit | Explanation |
|---------|-------------|
| **Verb consistency** | All statement-level operations start with `/` or `*` or flow symbols |
| **Visual unity** | Scripts have cleaner visual rhythm |
| **Philosophy alignment** | "Everything is a verb" extends to sugar forms |
| **Discoverability** | Users expect `/` for operations |

### AGAINST the Change

| Concern | Explanation |
|---------|-------------|
| **Split syntax** | `$` in expressions, `/` in statements creates two mental models |
| **Breaking change** | All existing scripts need migration |
| **Marginal benefit** | Current `$` syntax works fine, change is aesthetic |
| **Expression confusion** | `/"..."` could be confused with `/interpolate "..."` |

---

## Alternatives Considered

### Alternative A: Keep Current Syntax

Accept `$` as the "expression/evaluation" prefix. Document the distinction:
- `/` = verb calls
- `$` = expression evaluation (both statement sugar and in-expression forms)

**Verdict**: Simpler, no breaking changes, but misses consistency opportunity.

### Alternative B: Change Everything to `/`

Also change in-expression forms:
```zoh
/`/(*var) + /(*other)`;    :: Instead of $(*var)
/`/#(*list)`;               :: Instead of $#()
```

**Verdict**: More consistent but requires expression grammar rewrite. The `/` character inside expressions conflicts visually with division.

### Alternative C: New Unified Prefix

Choose a different character for all evaluation forms:
```zoh
@`*a + *b`;                :: Statement sugar
/evaluate `@(*var)`;       :: In-expression form
```

**Verdict**: Even more breaking changes, unclear benefit over current `$`.

---

## Migration Path

### If Approved

1. **Deprecation phase**: Accept both `$` and `/` forms, emit warnings for `$`
2. **Migration tool**: Script to convert `$`...`` to `/`...``
3. **Documentation update**: Update all examples
4. **Removal phase**: Remove `$` statement sugar (keep in expressions)

### Migration Script Pseudocode

```
For each .zoh file:
    Replace `$`` with `/``
    Replace `$"` with `/"`
    Replace `$'` with `/'`
    (Do NOT replace $( inside expressions)
```

---

## Open Questions

1. **Is the split syntax acceptable?** Statement `/` vs expression `$`
2. **Is the benefit worth the breaking change?** Consistency vs stability
3. **Should we support both during transition?** Complexity vs ease of migration
4. **Does `/"..."` look too much like a verb call?** Visual disambiguation concern

---

## Recommendation

### Option 1: Proceed with Change (Recommended if consistency is valued)

The `/` prefix aligns with ZOH's verb-centric philosophy. The split syntax is justifiable:
- **Statement level**: `/` for actions that execute
- **Expression level**: `$` for values that evaluate

Accept this as a documented design distinction, similar to how `*` marks references everywhere.

### Option 2: Keep Current Syntax (Recommended if stability is valued)

The `$` prefix is established, works well, and avoids breaking changes. Document `$` as the "evaluation prefix" distinct from `/` verb calls.

---

## Decision

**Pending review.**
