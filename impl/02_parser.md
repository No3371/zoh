# 02: Parser Implementation

## Purpose

The parser transforms a token stream into an Abstract Syntax Tree (AST). The AST represents the structure of a ZOH story for compilation and execution.

---

## AST Node Types

### Story Structure

```
StoryNode:
  name: string              # Story title
  metadata: Map<string,any> # Key-value metadata
  body: List<Statement>     # Top-level statements
  checkpoints: Map<string,Checkpoint> # Name -> Checkpoint Node

Statement: 
  | VerbCall
  | Checkpoint
  | PreprocessorDirective
  | SugarStatement
```

### Verb Call

```
VerbCallNode:
  namespace: string?        # Optional namespace (e.g., "std" in "std.converse")
  name: string              # Verb name
  isBlock: bool             # Block form (/verb/ ... /;)
  attributes: List<Attribute>
  namedParams: Map<string,Value>
  unnamedParams: List<Value>
  position: Position        # Source location
```

### Attribute

```
AttributeNode:
  name: string              # Attribute name
  value: Value?             # Optional value (nothing if absent)
```

### Values

```
Value:
  | NothingValue            # ?
  | BoolValue               # true/false
  | IntegerValue            # 42
  | DoubleValue             # 3.14
  | StringValue             # "hello"
  | ReferenceValue          # *var, *var[index]
  | ChannelValue            # <channel_name>
  | ListValue               # [a, b, c]
  | MapValue                # {"key": value}
  | ExpressionValue         # `*a + *b`
  | VerbCallValue           # /verb; (verb as data)

ReferenceValue:
  name: string              # Variable name
  path: List<Value>         # Path of indexes (for *list[0], *map["k"], *data["a"][0]["b"])
```

### Sugar Statements

```
SugarStatement:
  | SetSugar                # *var <- value;
  | GetSugar                # <- *var;
  | CaptureSugar            # -> *var;
  | JumpSugar               # ====> @label;
  | ForkSugar               # ====+ @label;
  | CallSugar               # <===+ @label;
  | FlagSugar               # #flag name value;
  | EvalSugar               # /`expr`;
  | InterpolateSugar        # /"string";
```

### Checkpoint

```
CheckpointNode:
  name: string              # Checkpoint name (without @)
  contract: List<Reference> # Required variables
  position: Position
```

---

## Grammar (Pseudo-EBNF)

```ebnf
story           := story_header story_body

story_header    := IDENTIFIER NEWLINE (metadata_entry)* STORY_SEP
metadata_entry  := IDENTIFIER COLON value SEMICOLON

story_body      := (statement)*
statement       := verb_call | checkpoint | sugar_statement | preprocessor

checkpoint      := AT IDENTIFIER (reference)*

verb_call       := VERB_START namespaced_id attributes params SEMICOLON
                 | VERB_START namespaced_id VERB_START attributes params BLOCK_END

namespaced_id   := IDENTIFIER (DOT IDENTIFIER)*
                  // Note: IDENTIFIER cannot contain dots. Dots are strictly separators.

attributes      := (attribute)*
attribute       := LBRACKET IDENTIFIER (COLON value)? RBRACKET

params          := (param_item)*
param_item      := named_param | value
named_param     := IDENTIFIER COLON value

value           := nothing | boolean | number | string | reference
                 | list | map | expression | channel | verb_call_value

nothing         := QUESTION
boolean         := TRUE | FALSE
number          := INTEGER | DOUBLE
string          := STRING | MULTILINE_STRING
reference       := ASTERISK IDENTIFIER (LBRACKET value RBRACKET)*
list            := LBRACKET (value (COMMA value)*)? RBRACKET
map             := LBRACE (map_entry (COMMA map_entry)*)? RBRACE
map_entry       := (string | reference) COLON value
expression      := BACKTICK expr_content BACKTICK
channel         := LANGLE IDENTIFIER RANGLE
verb_call_value := verb_call

sugar_statement := set_sugar | get_sugar | capture_sugar
                 | jump_sugar | fork_sugar | call_sugar
                 | flag_sugar | eval_sugar | interpolate_sugar

set_sugar       := ASTERISK IDENTIFIER (LBRACKET value RBRACKET)? 
                   attributes? (ARROW_LEFT value)? SEMICOLON
get_sugar       := ARROW_LEFT ASTERISK IDENTIFIER (LBRACKET value RBRACKET)? 
                   attributes? SEMICOLON
capture_sugar   := ARROW_RIGHT attributes? ASTERISK IDENTIFIER 
                   (LBRACKET value RBRACKET)? SEMICOLON

jump_sugar      := JUMP attributes? AT label_ref (value)* SEMICOLON
fork_sugar      := FORK attributes? AT label_ref (value)* SEMICOLON  
call_sugar      := CALL attributes? AT label_ref (value)* SEMICOLON
label_ref       := (IDENTIFIER COLON)? IDENTIFIER

flag_sugar      := HASH FLAG attributes? IDENTIFIER value SEMICOLON
eval_sugar      := SLASH_BACKTICK expr_content BACKTICK attributes? SEMICOLON
interpolate_sugar := SLASH_QUOTE string_content QUOTE attributes? SEMICOLON
```

---

## Implementation Steps

### Step 1: Parser State

```
Parser:
  tokens: List<Token>
  current: int              # Current token index
  
  # Navigation
  peek(): Token             # Current token
  previous(): Token         # Last consumed token
  advance(): Token          # Consume and return current
  isAtEnd(): bool
  check(type: TokenType): bool
  match(...types): bool     # Check and consume if match
  consume(type, message): Token  # Consume or error
  
  # Synchronization
  synchronize(): void       # Recover from error
```

### Step 2: Story Parsing

```
parseStory():
    name = parseStoryName()
    metadata = parseMetadata()
    consume(STORY_SEP, "Expected '===' after metadata")
    
    checkpoints = Map<string, CheckpointNode>()
    statements = []
    
    while not isAtEnd():
        stmt = parseStatement()
        if stmt is CheckpointNode:
            checkpoints[stmt.name] = stmt
        statements.add(stmt)
    
    return StoryNode { name, metadata, statements, checkpoints }
```

### Step 3: Statement Parsing

```
parseStatement():
    if check(AT):
        return parseCheckpoint()
    if check(VERB_START):
        return parseVerbCall()
    if check(ASTERISK):
        return parseSetSugar()
    if check(ARROW_LEFT):
        return parseGetSugar()
    if check(ARROW_RIGHT):
        return parseCaptureSugar()
    if check(JUMP):
        return parseJumpSugar()
    if check(FORK):
        return parseForkSugar()
    if check(CALL):
        return parseCallSugar()
    if check(HASH):
        return parsePreprocessor()
    if check(SLASH_BACKTICK):
        return parseEvalSugar()
    if check(SLASH_QUOTE):
        return parseInterpolateSugar()
    
    error("Unexpected token")
    synchronize()
```

### Step 4: Verb Call Parsing

```
parseVerbCall():
    consume(VERB_START, "Expected '/'")
    name = parseNamespacedId()
    
    # Check for block form
    isBlock = match(VERB_START)
    
    attributes = parseAttributes()
    namedParams, unnamedParams = parseParameters(isBlock)
    
    if isBlock:
        consume(BLOCK_END, "Expected '/;' to close block verb")
    else:
        consume(SEMICOLON, "Expected ';' after verb call")
    
    return VerbCallNode { name, isBlock, attributes, namedParams, unnamedParams }
```

### Step 5: Attribute Parsing

```
parseAttributes():
    attrs = []
    while check(LBRACKET):
        advance()  # Consume '['
        name = consume(IDENTIFIER, "Expected attribute name")
        value = Nothing
        if match(COLON):
            value = parseValue()
        consume(RBRACKET, "Expected ']' after attribute")
        attrs.add(AttributeNode { name, value })
    return attrs
```

### Step 6: Parameter Parsing

```
parseParameters(isBlock: bool):
    named = Map<string, Value>()
    unnamed = []
    
    # Named and unnamed params can be freely interleaved
    # In block form, spaces and newlines separate params (no commas needed)
    # In standard form, commas separate params
    
    while not checkTerminator(isBlock):
        if isNamedParam():
            name = consume(IDENTIFIER)
            consume(COLON)
            value = parseValue()
            named[name] = value
        else:
            value = parseValue()
            unnamed.add(value)
        
        # Consume separator
        if not isBlock:
            if not check(SEMICOLON):
                match(COMMA)
    
    return (named, unnamed)

isNamedParam():
    # Lookahead: identifier followed by colon
    return check(IDENTIFIER) and peekNext().type == COLON

checkTerminator(isBlock: bool):
    if isBlock:
        return check(BLOCK_END)
    return check(SEMICOLON) or check(ARROW_RIGHT)
```

### Step 7: Value Parsing

```
parseValue():
    if match(QUESTION):
        return NothingValue()
    if match(TRUE):
        return BoolValue(true)
    if match(FALSE):
        return BoolValue(false)
    if match(INTEGER):
        return IntegerValue(previous().literal)
    if match(DOUBLE):
        return DoubleValue(previous().literal)
    if match(STRING):
        return StringValue(previous().literal)
    if match(ASTERISK):
        return parseReference()
    if check(LBRACKET):
        return parseList()
    if check(LBRACE):
        return parseMap()
    if match(BACKTICK):
        return parseExpression()
    if check(LANGLE):
        return parseChannel()
    if check(VERB_START):
        return parseVerbCallValue()
    
    error("Expected value")
```

### Step 8: Reference Parsing

```
parseReference():
    # ASTERISK already consumed
    name = consume(IDENTIFIER, "Expected variable name")

    path = []
    while match(LBRACKET):
        index = parseValue()
        consume(RBRACKET, "Expected ']' after index")
        path.add(index)

    return ReferenceValue { name, path }
```

### Step 9: Collection Parsing

```
parseList():
    consume(LBRACKET)
    elements = []
    
    if not check(RBRACKET):
        elements.add(parseValue())
        while match(COMMA):
            if check(RBRACKET):  # Trailing comma
                break
            elements.add(parseValue())
    
    consume(RBRACKET, "Expected ']' after list")
    return ListValue(elements)

parseMap():
    consume(LBRACE)
    entries = Map<Value, Value>()
    
    if not check(RBRACE):
        key, value = parseMapEntry()
        entries.add(key, value)
        while match(COMMA):
            if check(RBRACE):  # Trailing comma
                break
            key, value = parseMapEntry()
            entries.add(key, value)
    
    consume(RBRACE, "Expected '}' after map")
    return MapValue(entries)

parseMapEntry():
    key = parseValue()  # Must be string or reference
    consume(COLON, "Expected ':' in map entry")
    value = parseValue()
    return (key, value)
```

---

## Sugar Transformation

Sugar statements should be transformed to standard verb calls:

| Sugar | Standard Form |
|-------|---------------|
| `*var;` | `/set "var";` |
| `*var [attr];` | `/set [attr] "var";` |
| `*var <- value;` | `/set "var", value;` |
| `*var [attr] <- value;` | `/set [attr] "var", value;` |
| `*var[idx] <- value;` | `/set *var[idx], value;` |
| `*var[idx] [attr] <- value;` | `/set [attr] *var[idx], value;` |
| `*var[a][b][c] <- value;` | `/set *var[a][b][c], value;` |
| `<- *var;` | `/get "var";` |
| `<- *var [attr];` | `/get [attr] "var";` |
| `<- *var[idx];` | `/get *var[idx];` |
| `<- *var[a][b][c];` | `/get *var[a][b][c];` |
| `-> *var;` | `/capture "var";` |
| `-> [attr] *var;` | `/capture [attr] "var";` |
| `-> *var[idx];` | `/capture *var[idx];` |
| `-> *var[a][b][c];` | `/capture *var[a][b][c];` |
| `====> @label;` | `/jump ?, "label";` |
| `====> [attr] @label;` | `/jump [attr] ?, "label";` |
| `====> @story:label;` | `/jump "story", "label";` |
| `====> @label *var1 *var2;` | `/jump ?, "label", *var1, *var2;` |
| `====+ @label;` | `/fork ?, "label";` |
| `====+ [attr] @label;` | `/fork [attr] ?, "label";` |
| `====+ @story:label *var;` | `/fork "story", "label", *var;` |
| `<===+ @label;` | `/call ?, "label";` |
| `<===+ [attr] @label;` | `/call [attr] ?, "label";` |
| `<===+ @story:label *var;` | `/call "story", "label", *var;` |
| `` /`expr`; `` | `/evaluate `expr`;` |
| `` /`expr` [attr]; `` | `/evaluate [attr] `expr`;` |
| `/"string";` | `/interpolate "string";` |
| `#flag name value;` | `/flag "name", value;` |
| `#flag [attr] name value;` | `/flag [attr] "name", value;` |

---

## Error Recovery

### Synchronization Points

1. After `;` (statement boundary)
2. After `/;` (block end)
3. At `@` (label start)
4. At line containing `===` variants

### Panic Mode Recovery

```
synchronize():
    advance()  # Skip error token
    
    while not isAtEnd():
        if previous().type == SEMICOLON:
            return
        if previous().type == BLOCK_END:
            return
        
        match peek().type:
            VERB_START, AT, JUMP, FORK, CALL, HASH:
                return
        
        advance()
```

### Error Message Format

```
[FATAL] file.zoh:10:5 - Unexpected token 'INTEGER', expected 'SEMICOLON'
    10 | /set "x", 42
       |             ^ error here
```

---

## Testing Checklist

### Basic Parsing
- [ ] Story header with name
- [ ] Metadata entries (string, int, list, map)
- [ ] Empty story body
- [ ] Simple verb call: `/verb;`

### Verb Calls
- [ ] Verb with attributes: `/verb [attr];`
- [ ] Verb with named params: `/verb name:value;`
- [ ] Verb with unnamed params: `/verb p1, p2;`
- [ ] Mixed params: `/verb name:value, p1, p2;`
- [ ] Block form verb
- [ ] Nested verb calls: `/if *cond, /do something;;`

### Values
- [ ] All literals: nothing, bool, int, double, string
- [ ] References: simple, single index, nested path (*data["a"][0]["b"])
- [ ] Lists: empty, single, multiple
- [ ] Maps: empty, single, multiple
- [ ] Expressions
- [ ] Channels

### Checkpoints
- [ ] Checkpoint definition
- [ ] Checkpoint with contract: `@name *var1 *var2`
- [ ] Checkpoint reference in jump/fork/call

### Sugar
- [ ] Set sugar (all variants)
- [ ] Get sugar
- [ ] Capture sugar
- [ ] Jump/Fork/Call sugar
- [ ] Eval sugar
- [ ] Interpolate sugar

### Error Cases
- [ ] Missing semicolon
- [ ] Unbalanced brackets
- [ ] Invalid token in value position
- [ ] Duplicate checkpoints (should error)

---

## Output: AST Example

Input:
```zoh
Test Story
author: "Jane";
===

@start
*name <- "World";
/converse [By: "Narrator"] "Hello, ${*name}!";
====> @end;

@end *name
/exit;
```

Output AST:
```
StoryNode {
  name: "Test Story",
  metadata: { "author": "Jane" },
  checkpoints: { 
      "start": CheckpointNode{name: "start", contract: []}, 
      "end": CheckpointNode{name: "end", contract: ["name"]} 
  },
  body: [
    CheckpointNode { name: "start", contract: [] },
    VerbCallNode {
      name: "set",
      attributes: [],
      namedParams: {},
      unnamedParams: ["name", "World"]
    },
    VerbCallNode {
      name: "converse",
      attributes: [{ name: "By", value: "Narrator" }],
      unnamedParams: ["Hello, ${*name}!"]
    },
    VerbCallNode {
      name: "jump",
      unnamedParams: [Nothing, "end"]
    },
    CheckpointNode { name: "end", contract: ["name"] },
    VerbCallNode {
      name: "exit",
      attributes: [],
      unnamedParams: []
    }
  ]
}
```
