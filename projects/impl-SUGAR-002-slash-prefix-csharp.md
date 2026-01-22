# C# Runtime: Slash-Prefix Syntax for Evaluate and Interpolate Sugar

**Scope**: C# reference implementation only
**Depends on**: `projects/impl-SUGAR-001-slash-prefix-syntax.md` (spec changes)

Implement the slash-prefix syntactic sugar in the C# runtime to match the updated specification.

| Old | New | Token |
|-----|-----|-------|
| `` $` `` | `` /` `` | `SlashBacktick` |
| `$"` | `/"` | `SlashQuote` |
| `$'` | `/'` | `SlashQuote` |

---

## Tasks

### 1. Update Token Types

**File**: `c#/src/Zoh.Runtime/Lexing/TokenType.cs`

- Add `SlashBacktick` token type
- Add `SlashQuote` token type
- Remove `DollarBacktick` token type
- Remove `DollarQuote` token type

### 2. Update Lexer

**File**: `c#/src/Zoh.Runtime/Lexing/Lexer.cs`

- Add detection for `` /` `` → `SlashBacktick`
- Add detection for `/"` and `/'` → `SlashQuote`
- Remove detection for `$` + backtick/quote combinations

### 3. Update Parser

**File**: `c#/src/Zoh.Runtime/Parsing/Parser.cs`

- Update `ParseStatement()` to handle `SlashBacktick` → `ParseEvalSugar()`
- Update `ParseStatement()` to handle `SlashQuote` → `ParseInterpolateSugar()`

### 4. Update Tests

**Directory**: `c#/tests/Zoh.Tests/`

- Update lexer tests for new token types
- Update parser tests using `` $`...` `` to use `` /`...` ``
- Update parser tests using `$"..."` to use `/"..."`

---

## Verification

- `dotnet build` succeeds
- `dotnet test` passes all tests
- Lexer correctly produces `SlashBacktick` and `SlashQuote` tokens
- Parser correctly desugars to `/evaluate` and `/interpolate` calls
