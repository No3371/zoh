# 01: Lexer Implementation

## Purpose

The lexer (tokenizer) converts ZOH source code into a stream of tokens. This is the foundation of the parsing pipeline.

---

## Token Categories

### 1. Keywords & Reserved Words

| Token Type | Pattern | Example |
|------------|---------|---------|
| `VERB_START` | `/` | `/converse` |
| `BLOCK_END` | `/;` | Block terminator |
| `LABEL` | `@identifier` | `@opening` |
| `NOTHING` | `?` | Default/null value |
| `TRUE` | `true` (case-insensitive) | |
| `FALSE` | `false` (case-insensitive) | |

### 2. Punctuation & Operators

| Token Type | Pattern | Notes |
|------------|---------|-------|
| `SEMICOLON` | `;` | Statement terminator |
| `COMMA` | `,` | Parameter separator |
| `COLON` | `:` | Named parameter, attribute value |
| `ARROW_LEFT` | `<-` | Set operator |
| `ARROW_RIGHT` | `->` | Capture operator |
| `ASTERISK` | `*` | Reference prefix |
| `LBRACKET` | `[` | Attribute/index start |
| `RBRACKET` | `]` | Attribute/index end |
| `LBRACE` | `{` | Map literal start |
| `RBRACE` | `}` | Map literal end |
| `LPAREN` | `(` | Expression grouping |
| `RPAREN` | `)` | Expression grouping |
| `LANGLE` | `<` | Channel start |
| `RANGLE` | `>` | Channel end |
| `BACKTICK` | `` ` `` | Expression delimiters |

### 3. Syntactic Sugar Tokens

| Token Type | Pattern | Expands To |
|------------|---------|------------|
| `JUMP` | `====>` | `/jump` |
| `FORK` | `====+` | `/fork` |
| `CALL` | `<===+` | `/call` |
| `STORY_SEP` | `===` | Story header separator |
| `HASH_DIRECTIVE` | `#embed`, `#macro`, `#expand`, `#flag` | Preprocessor |
| `DOLLAR_BACKTICK` | `` $` `` | `/evaluate` sugar |
| `DOLLAR_QUOTE` | `$"` or `$'` | `/interpolate` sugar |

### 4. Literals

| Token Type | Pattern | Example |
|------------|---------|---------|
| `INTEGER` | `[+-]?[0-9]+` | `42`, `-7` |
| `DOUBLE` | `[+-]?[0-9]+\.[0-9]+` | `3.14`, `-0.5` |
| `STRING` | `"..."` or `'...'` | `"hello"` |
| `MULTILINE_STRING` | `"""..."""` or `'''...'''` | Block strings |

### 5. Identifiers

| Token Type | Pattern | Example |
|------------|---------|---------|
| `IDENTIFIER` | `[a-zA-Z_][a-zA-Z0-9_]*` | `var_name`, `Jack` |
| `NAMESPACED_ID` | `namespace.name` | `std.converse` |

### 6. Comments

| Token Type | Pattern | Notes |
|------------|---------|-------|
| `COMMENT` | `::` to EOL | Single-line |
| `MULTILINE_COMMENT` | `:::` to `:::` | Multi-line block |

---

## Implementation Steps

### Step 1: Character Stream

```
Input: Source code string
Output: Character iterator with position tracking

Interface:
- peek(): char        # Look at current char without consuming
- advance(): char     # Consume and return current char
- isAtEnd(): bool     # Check if at end of input
- position: Position  # Current line, column, offset
```

**Position Tracking:**
- Line number (1-indexed)
- Column number (1-indexed)
- Byte offset (0-indexed)

### Step 2: Token Data Structure

```
Token:
  type: TokenType      # Enum of all token types
  lexeme: string       # Raw text that matched
  literal: any         # Parsed value (for literals)
  position: Position   # Start position in source
  length: int          # Length in characters
```

### Step 3: Scanner Implementation

```
Scanner:
  source: string
  tokens: List<Token>
  start: int           # Start of current lexeme
  current: int         # Current position
  line: int
  column: int

  scanTokens(): List<Token>
  scanToken(): void
  isAtEnd(): bool
  advance(): char
  peek(): char
  peekNext(): char
  match(expected: char): bool
  addToken(type: TokenType): void
  addToken(type: TokenType, literal: any): void
```

### Step 4: Lexing Logic

#### Main Loop

```
while not isAtEnd():
    start = current
    scanToken()
return tokens
```

#### Token Recognition (Priority Order)

1. **Whitespace**: Skip (but track newlines for line count)
2. **Comments**: `::` or `:::`
3. **Multi-char operators**: `====>`, `====+`, `<===+`, `===`, `<-`, `->`
4. **String literals**: `"`, `'`, `"""`, `'''`
5. **Sugar prefixes**: `$"`, `$'`, `` $` ``
6. **Preprocessor**: `#embed`, `#macro`, `#expand`, `#flag`
7. **Numbers**: digits possibly with `.`
8. **Identifiers/Keywords**: letters, underscore, then alphanumeric
9. **Single-char tokens**: `;`, `,`, `:`, `[`, `]`, etc.
10. **Expression backticks**: `` ` ``

#### String Lexing

```
scanString(quote: char):
    # Check for multiline (""" or ''')
    if peek() == quote and peekNext() == quote:
        advance(); advance()  # Consume remaining quotes
        return scanMultilineString(quote)
    
    # Single-line string
    while peek() != quote and not isAtEnd():
        if peek() == '\n':
            # Multi-line allowed in single quotes too
            line++; column = 0
        if peek() == '\\':
            advance()  # Escape next char
        advance()
    
    if isAtEnd():
        error("Unterminated string")
        return
    
    advance()  # Closing quote
    value = unescape(source[start+1 : current-1])
    addToken(STRING, value)
```

#### Expression Lexing

```
scanExpression():
    # Expressions are enclosed in backticks
    depth = 1  # Handle nested backticks via escaping
    while depth > 0 and not isAtEnd():
        if peek() == '\\':
            advance()  # Skip escape
            advance()
        elif peek() == '`':
            depth--
        else:
            advance()
    
    if isAtEnd():
        error("Unterminated expression")
        return
    
    content = source[start+1 : current]
    advance()  # Closing backtick
    addToken(EXPRESSION, content)
```

#### Number Lexing

```
scanNumber():
    while isDigit(peek()):
        advance()
    
    isDouble = false
    if peek() == '.' and isDigit(peekNext()):
        isDouble = true
        advance()  # Consume '.'
        while isDigit(peek()):
            advance()
    
    text = source[start : current]
    if isDouble:
        addToken(DOUBLE, parseDouble(text))
    else:
        addToken(INTEGER, parseInt(text))
```

---

## Edge Cases

### 1. Escaped Characters in Strings

| Escape | Result |
|--------|--------|
| `\\` | Literal backslash |
| `\"` | Literal double quote |
| `\'` | Literal single quote |
| `\n` | Newline |
| `\t` | Tab |
| `\{` | Literal `{` (in interpolation) |
| `\}` | Literal `}` (in interpolation) |

### 2. Multiline String Indentation

For `"""..."""` strings, track leading whitespace of the closing delimiter and strip that amount from each line.

### 3. Context-Sensitive Tokens

Some tokens change meaning based on context:
- `[` after `*identifier` = index access
- `[` at verb position = attribute start
- `<` and `>` around identifier = channel

### 4. Lookahead Requirements

| Pattern | Lookahead |
|---------|-----------|
| `====>` | 5 chars |
| `====+` | 5 chars |
| `<===+` | 5 chars |
| `===` | 3 chars |
| `"""` / `'''` | 3 chars |
| `::` vs `:::` | 3 chars |

---

## Error Handling

### Recoverable Errors

- Unterminated string → report and skip to next line
- Unknown character → report and skip

### Fatal Errors

All of the following errors must halt lexing immediately:

- **Unterminated string** - String literal without closing delimiter
- **Unknown character** - Character not part of any valid token
- **Unbalanced expression backticks** - Mismatched `` ` `` delimiters
- **Invalid escape sequence** - Invalid `\x` escape in string
- **Unterminated multiline comment** - `:::` without matching closing `:::`

### Error Message Format

```
[ERROR] file.zoh:10:5 - Unterminated string literal
    10 | /converse "Hello, world!
       |           ^
```

---

## Testing Checklist

- [ ] Basic verb tokenization: `/converse;`
- [ ] Attribute tokenization: `[By: "John"]`
- [ ] String escape sequences: `"Hello \"World\""`
- [ ] Multiline strings: `"""..."""`
- [ ] Numbers: integers, doubles, negatives
- [ ] Expression backticks: `` `*a + *b` ``
- [ ] Sugar tokens: `====>`, `====+`, `<===+`
- [ ] Comments: inline `::`, block `:::`
- [ ] Reference index: `*list[0]`, `*map["key"]`
- [ ] Channel syntax: `<channel_name>`
- [ ] Preprocessor directives: `#embed`, `#macro`
- [ ] Edge case: strings containing quotes
- [ ] Edge case: expressions with escapes
- [ ] Error recovery: unterminated constructs

---

## Output Format

The tokenizer should output a list of tokens suitable for the parser:

```
Token { type: VERB_START, lexeme: "/", line: 1, col: 1 }
Token { type: IDENTIFIER, lexeme: "converse", line: 1, col: 2 }
Token { type: LBRACKET, lexeme: "[", line: 1, col: 11 }
Token { type: IDENTIFIER, lexeme: "By", line: 1, col: 12 }
Token { type: COLON, lexeme: ":", line: 1, col: 14 }
Token { type: STRING, lexeme: "\"John\"", literal: "John", line: 1, col: 16 }
Token { type: RBRACKET, lexeme: "]", line: 1, col: 22 }
Token { type: STRING, lexeme: "\"Hello!\"", literal: "Hello!", line: 1, col: 24 }
Token { type: SEMICOLON, lexeme: ";", line: 1, col: 32 }
Token { type: EOF, lexeme: "", line: 1, col: 33 }
```
