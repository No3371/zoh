# RFC: Raw Block Syntax for Macro Parameters

## Status
**Proposed**

## Summary

This RFC introduces "Raw Block" syntax `|#param|value#|` for passing complex content to macros. The current `param:value` syntax breaks when values contain commas, quotes, or multi-line code. Raw blocks solve this by using unambiguous delimiters that capture content verbatim.

**Impact**: Enables macros to accept code blocks, expressions with operators, and structured data without escaping. This is essential for ZOH's macro system to support real-world templating patterns.

| Syntax | Purpose |
| :--- | :--- |
| `name:value` | Simple values: literals, numbers, identifiers |
| `\|#name\|value#\|` | Complex values: code blocks, expressions, lists, nested structures |

Both syntaxes can be mixed freely in a single `#expand` call.

---

## Motivation

ZOH uses commas to delimit macro parameters. This breaks when passing values that contain commas:

```zoh
#expand MACRO list:[1, 2, 3];
:: Parser sees "list:[1", then "2", then "3]" as separate parameters
```

Current workarounds (nested quoting, escape sequences) are error-prone and unreadable. Macros are central to ZOH's design philosophy—they need first-class support for injecting arbitrary code.

---

## Specification

### Syntax

```zoh
#expand MACRO_NAME
    simple:value
    |#block_param| content here #|;
```

**Opening delimiter**: `|#name|` where `name` is the parameter identifier (no surrounding whitespace allowed)

**Closing delimiter**: `#|`

**Content**: Everything between delimiters is captured verbatim, then normalized (see below)

### Properties

1. **Comma Immunity**: Commas are content, not delimiters
2. **Quote Immunity**: No escaping needed for `"` or `'`
3. **Operator Immunity**: `||`, `|`, `#` alone do not trigger delimiters
4. **Visual Consistency**: Mirrors macro placeholder syntax `|#name#|`

### Delimiter Escaping

If content must contain the literal sequence `#|`, use extended delimiters by adding `#` characters:

| Delimiter Level | Opening | Closing |
| :--- | :--- | :--- |
| Standard | `\|#name\|` | `#\|` |
| Extended (1) | `\|##name\|` | `##\|` |
| Extended (2) | `\|###name\|` | `###\|` |

The number of `#` in the opening must match the closing. Use the minimum level needed to avoid conflicts with content.

```zoh
#expand DOCS
    |##example|
        :: Standard raw blocks use #| to close
    ##|;
```

### Empty Parameters

Empty content is valid: `|#param|#|` passes an empty string.

---

## Examples

### Code Blocks

```zoh
#macro REPEAT count, action;
/loop |#count#|, |#action#|;;
#macro;

#expand REPEAT
    count:5
    |#action|
        /print "Hello";
        /play "sound.ogg";
    #|;
```

### Expressions with Operators

```zoh
#expand CHECK_STATE
    |#condition| *health < 10 || *stamina < 5 #|;
```

### Structured Data

```zoh
#expand INIT_DATA
    |#items| [1, 2, 3] #|
    |#config| {"id": 1, "name": "test"} #|;
```

---

## Multi-line Content Handling

### The Problem

Users indent content for readability:

```zoh
#expand DIALOGUE
    |#lines|
        /say "Hello";
        /say "World";
    #|;
```

Literal capture would include 8 spaces of indentation on each line. Injecting this into a macro produces misaligned or over-indented output.

### Indentation Normalization

Apply these rules in order:

1. **Strip boundary newlines**: Remove the first character if it's a newline (immediately after `|#name|`). Remove the last character if it's a newline (immediately before `#|`).

2. **Calculate base indentation**: Count leading whitespace on the line containing `#|`. This is the "base indent".

3. **Strip base indent**: Remove exactly that many leading whitespace characters from each content line. Lines with less indentation than base are left-aligned (no negative indent).

4. **Preserve trailing whitespace**: Trailing whitespace on lines is preserved.

### Examples

**Closing delimiter at column 0:**
```zoh
|#code|
    /line1;
    /line2;
#|
```
Base indent = 0. Result:
```
    /line1;
    /line2;
```

**Closing delimiter indented:**
```zoh
|#code|
    /line1;
    /line2;
    #|
```
Base indent = 4. Result:
```
/line1;
/line2;
```

**Inline content (no newlines):**
```zoh
|#expr| *a + *b #|
```
Result: ` *a + *b ` (leading/trailing spaces preserved since no boundary newlines)

---

## Implementation

### Lexer Changes

When scanning `#expand` arguments:

1. Detect `|` followed immediately by one or more `#` characters
2. Count the `#` characters to determine delimiter level
3. Read identifier until next `|`
4. Enter raw capture mode
5. Scan until matching closing delimiter (same `#` count followed by `|`)
6. Apply normalization to captured content

If `|#` sequence not found, fall back to standard `param:value` parsing.

### Source Mapping

Multi-line parameters complicate error reporting. The source map must track:

- Which output lines come from macro body vs. parameter substitution
- For substituted content, the original line number within the `#expand` block

**Data structure**: Each parameter value carries a `source_offset` indicating its starting line within the `#expand` call. During expansion, output lines are tagged with `(source_file, base_line + offset)`.

**Error example**: If `/line1;` (from the example above) causes an error, report it as the specific line in the source file where `/line1;` appears, not the `#expand` line.

---

## Alternatives Considered

1. **Heredoc syntax** (`<<EOF...EOF`): Familiar but doesn't integrate with parameter naming
2. **Triple-quote strings** (`"""..."""`): Conflicts with potential future string syntax
3. **Backslash escaping**: Error-prone, reduces readability
4. **Parenthesis counting**: Fragile with nested structures

The `|#...#|` syntax was chosen for visual consistency with existing macro placeholders and unambiguous parsing.

