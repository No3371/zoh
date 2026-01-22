# 04: Expression Evaluation

## Purpose

Expressions in ZOH are enclosed in backticks (`` ` ``) and can contain arithmetic, logical, and special operations. They are stored as data until explicitly evaluated via `/evaluate`.

---

## Expression Grammar (EBNF)

```ebnf
expression      := or_expr

or_expr         := and_expr ( '||' and_expr )*
and_expr        := equality_expr ( '&&' equality_expr )*
equality_expr   := relational_expr ( ( '==' | '!=' ) relational_expr )*
relational_expr := additive_expr ( ( '<' | '>' | '<=' | '>=' ) additive_expr )*
additive_expr   := mult_expr ( ( '+' | '-' ) mult_expr )*
mult_expr       := unary_expr ( ( '*' | '/' | '%' ) unary_expr )*
unary_expr      := ( '!' | '-' ) unary_expr | primary_expr
primary_expr    := literal | reference | '(' expression ')' | special_form

literal         := string | integer | double | boolean | nothing
reference       := '*' identifier ( '[' index ']' )*
special_form    := interpolate | count | conditional | any | indexed | roll | wroll

interpolate     := '${' reference '}'                      # /interpolate (Vars only)
count           := '$#(' reference ')'                     # /count
conditional     := '$?(' expr '?' expr ':' expr ')'        # /if ternary
any             := '$?(' option_list ')'                   # /first non-nothing  
indexed         := '$(' option_list ')[' index ']'         # /get by index
roll            := '$(' option_list ')[%]'                 # /roll random
wroll           := '$(' weighted_opts ')[%]'               # /wroll weighted

option_list     := expression ( '|' expression )*
weighted_opts   := weighted ( '|' weighted )*
weighted        := expression ':' integer
index           := '!'? ( reference | integer )            # '!' = wrap-around
```

---

## Expression AST Nodes

```
ExprNode:
  | BinaryExpr     { left: ExprNode, op: BinaryOp, right: ExprNode }
  | UnaryExpr      { op: UnaryOp, operand: ExprNode }
  | LiteralExpr    { value: any }
  | ReferenceExpr  { name: string, path: List<ExprNode> }
  | GroupExpr      { inner: ExprNode }
  | SpecialExpr    { form: SpecialForm, ... }

BinaryOp: 
  '+' | '-' | '*' | '/' | '%' 
  | '==' | '!=' | '<' | '>' | '<=' | '>=' 
  | '&&' | '||'

UnaryOp:
  '!' | '-'

SpecialForm:
  | InterpolateForm { ref: ReferenceExpr }
  | CountForm       { ref: ReferenceExpr }
  | ConditionalForm { cond: ExprNode, then: ExprNode, else: ExprNode }
  | AnyForm         { options: List<ExprNode> }
  | IndexedForm     { options: List<ExprNode>, index: ExprNode, wrap: bool }
  | RollForm        { options: List<ExprNode> }
  | WRollForm       { options: List<(ExprNode, int)> }
```

---

## Operator Precedence

| Precedence | Operators | Associativity |
|------------|-----------|---------------|
| 1 (highest) | `!` `-` (unary) | Right |
| 2 | `*` `/` `%` | Left |
| 3 | `+` `-` | Left |
| 4 | `<` `>` `<=` `>=` | Left |
| 5 | `==` `!=` | Left |
| 6 | `&&` | Left |
| 7 (lowest) | `\|\|` | Left |

---

## Implementation Steps

### Step 1: Expression Lexer

Tokenize expression content (inside backticks):

```
ExpressionLexer:
  source: string     # The expression text (without enclosing backticks)
  
  scanTokens(): List<Token>
  # Token types: IDENTIFIER, INTEGER, DOUBLE, STRING, 
  #              ASTERISK, LPAREN, RPAREN, LBRACKET, RBRACKET,
  #              PLUS, MINUS, STAR, SLASH, PERCENT,
  #              EQ_EQ, BANG_EQ, LT, GT, LT_EQ, GT_EQ,
  #              AND_AND, OR_OR, BANG, PIPE, COLON, QUESTION,
  #              DOLLAR_BRACE, DOLLAR_PAREN, DOLLAR_HASH_PAREN, DOLLAR_QUESTION_PAREN,
  #              PERCENT_RBRACKET (for [%])
```

### Step 2: Expression Parser

Recursive descent parser following precedence:

```
ExpressionParser:
  tokens: List<Token>
  current: int
  
  parse(): ExprNode
    return parseOr()
  
  parseOr(): ExprNode
    left = parseAnd()
    while match('||'):
      right = parseAnd()
      left = BinaryExpr(left, '||', right)
    return left
  
  parseAnd(): ExprNode
    left = parseEquality()
    while match('&&'):
      right = parseEquality()
      left = BinaryExpr(left, '&&', right)
    return left
  
  parseEquality(): ExprNode
    left = parseRelational()
    while match('==', '!='):
      op = previous()
      right = parseRelational()
      left = BinaryExpr(left, op, right)
    return left
  
  parseRelational(): ExprNode
    left = parseAdditive()
    while match('<', '>', '<=', '>='):
      op = previous()
      right = parseAdditive()
      left = BinaryExpr(left, op, right)
    return left
  
  parseAdditive(): ExprNode
    left = parseMultiplicative()
    while match('+', '-'):
      op = previous()
      right = parseMultiplicative()
      left = BinaryExpr(left, op, right)
    return left
  
  parseMultiplicative(): ExprNode
    left = parseUnary()
    while match('*', '/', '%'):
      op = previous()
      right = parseUnary()
      left = BinaryExpr(left, op, right)
    return left
  
  parseUnary(): ExprNode
    if match('!', '-'):
      op = previous()
      operand = parseUnary()
      return UnaryExpr(op, operand)
    return parsePrimary()
  
  parsePrimary(): ExprNode
    if match(INTEGER):
      return LiteralExpr(previous().literal)
    if match(DOUBLE):
      return LiteralExpr(previous().literal)
    if match(STRING):
      return LiteralExpr(previous().literal)
    if match(TRUE):
      return LiteralExpr(true)
    if match(FALSE):
      return LiteralExpr(false)
    if match(QUESTION):
      return LiteralExpr(Nothing)
    if match(ASTERISK):
      return parseReference()
    if match(LPAREN):
      expr = parse()
      consume(RPAREN, "Expected ')' after expression")
      return GroupExpr(expr)
    if match(DOLLAR_BRACE):
      return parseInterpolate()
    if match(DOLLAR_PAREN):
      return parseOptionList()
    if match(DOLLAR_HASH_PAREN):
      return parseCountForm()
    if match(DOLLAR_QUESTION_PAREN):
      return parseConditionalOrAny()
    
    error("Expected expression")
```

### Step 3: Special Form Parsing

```
parseInterpolate(): ExprNode
    # ${ *var }
    ref = parseReference()
    consume(RBRACE)
    return InterpolateForm(ref)

parseOptionList(): ExprNode
    # $( a | b )
    options = [parse()]
    while match(PIPE):
        options.add(parse())
    consume(RPAREN)
    
    # Check for index or roll
    if match(LBRACKET):
        if match(PERCENT):
            consume(RBRACKET)
            # Check for weighted
            if hasWeights(options):
                return WRollForm(parseWeighted(options))
            return RollForm(options)
        else:
            # Indexed access
            wrap = match('!')
            index = parse()
            consume(RBRACKET)
            return IndexedForm(options, index, wrap)
    
    return AnyForm(options)

parseConditionalOrAny(): ExprNode
    # $?(cond ? then : else) or $?(a|b|c)
    first = parse()
    
    if match(QUESTION):
        # Conditional ternary
        thenExpr = parse()
        consume(COLON, "Expected ':' in conditional")
        elseExpr = parse()
        consume(RPAREN)
        return ConditionalForm(first, thenExpr, elseExpr)
    
    # Any form (first non-nothing)
    options = [first]
    while match(PIPE):
        options.add(parse())
    consume(RPAREN)
    return AnyForm(options)

parseCountForm(): ExprNode
    # $#(*var)
    ref = parseReference()
    consume(RPAREN)
    return CountForm(ref)
```

### Step 4: Expression Evaluator

```
ExpressionEvaluator:
  context: Context    # Access to variables
  
  evaluate(expr: ExprNode): any
    match expr:
      LiteralExpr:
        return expr.value
      
      ReferenceExpr:
        value = context.getVariable(expr.name)
        if value == null:
          error("Undefined variable: " + expr.name)
        for i, pathElement in enumerate(expr.path):
          index = evaluate(pathElement)
          value = getIndexed(value, index)
          if value is Nothing:
            error("Invalid path at index " + i + ": element does not exist")
        return value
      
      UnaryExpr:
        operand = evaluate(expr.operand)
        match expr.op:
          '!': return not toBool(operand)
          '-': return negate(operand)
      
      BinaryExpr:
        left = evaluate(expr.left)
        right = evaluate(expr.right)
        return applyBinaryOp(expr.op, left, right)
      
      GroupExpr:
        return evaluate(expr.inner)
      
      InterpolateForm:
        value = evaluate(expr.ref)
        return value   # Just return the value, do NOT stringify here? 
                       # Or return value to be stringified by caller?
                       # If expr is "${*x}", it evaluates to val(*x).
      
      CountForm:
        value = evaluate(expr.ref)
        return count(value)
      
      ConditionalForm:
        cond = evaluate(expr.cond)
        if toBool(cond):
          return evaluate(expr.then)
        else:
          return evaluate(expr.else)
      
      AnyForm:
        for opt in expr.options:
          value = evaluate(opt)
          if value != Nothing:
            return value
        return Nothing
      
      IndexedForm:
        values = expr.options.map(evaluate)
        index = toInt(evaluate(expr.index))
        if expr.wrap:
          index = index % values.length
        return values[index]
      
      RollForm:
        values = expr.options.map(evaluate)
        return values[randomInt(0, values.length)]
      
      WRollForm:
        return weightedRandom(expr.options, context)
```

### Step 5: Binary Operation Implementation

```
applyBinaryOp(op: string, left: any, right: any): any
    match op:
      # Arithmetic (int/double)
      '+':
        if isString(left) or isString(right):
          return toString(left) + toString(right)
        if isList(left) and isList(right):
          return concatenate(left, right)
        return left + right
      '-':
        return left - right
      '*':
        return left * right
      '/':
        if isInt(left) and isInt(right):
          return floor(left / right)
        return left / right
      '%':
        return left % right
      
      # Comparison
      '==':
        if isString(left) and isString(right):
          return left == right  # Case-sensitive per spec
        if isList(left) and isList(right):
          return deepEquals(left, right)
        return left == right
      '!=':
        return not applyBinaryOp('==', left, right)
      '<':
        return left < right
      '>':
        return left > right
      '<=':
        return left <= right
      '>=':
        return left >= right
      
      # Logical
      '&&':
        return toBool(left) and toBool(right)
      '||':
        return toBool(left) or toBool(right)
```

---

## Type Coercion

| Operation | Left Type | Right Type | Result |
|-----------|-----------|------------|--------|
| `+` | int | int | int |
| `+` | int | double | double |
| `+` | double | double | double |
| `+` | string | any | string (concat) |
| `+` | any | string | string (concat) |
| `+` | list | list | list (concat) |
| `-,*,/,%` | int | int | int |
| `-,*,/` | any | any | double |
| `/` | int | int | int (floor) |
| `==` | string | string | bool (case-sensitive) |
| `<,>,<=,>=` | number | number | bool |
| `&&,\|\|` | any | any | bool |

---

## Error Handling

| Error | Cause |
|-------|-------|
| Undefined variable | `*var` doesn't exist |
| Invalid path | `*data["missing"]["key"]` - path element doesn't exist |
| Type mismatch | `"string" - 5` |
| Division by zero | `*x / 0` |
| Index out of bounds | `$(a\|b)[5]` with 2 options |

---

## Testing Checklist

### Arithmetic
- [ ] Integer addition: `` `1 + 2` `` → `3`
- [ ] Double division: `` `5.0 / 2` `` → `2.5`
- [ ] Integer floor division: `` `5 / 2` `` → `2`
- [ ] Modulo: `` `7 % 3` `` → `1`
- [ ] Negation: `` `-*x` ``
- [ ] Precedence: `` `1 + 2 * 3` `` → `7`
- [ ] Grouping: `` `(1 + 2) * 3` `` → `9`

### Comparison
- [ ] Equality: `` `*a == *b` ``
- [ ] String case-sensitive: `` `"HELLO" == "hello"` `` → `false`
- [ ] Relational: `` `*x > 5` ``

### Logical
- [ ] AND: `` `*a && *b` ``
- [ ] OR: `` `*a || *b` ``
- [ ] NOT: `` `!*flag` ``
- [ ] Short-circuit evaluation

### Special Forms
- [ ] Interpolate Invocation: `$("string")` (e.g. `$("Hello ${*name}")`) ``
- [ ] Count: `` `$#(*list)` ``
- [ ] Conditional: `` `$?(*cond? *a : *b)` ``
- [ ] Any: `` `$?(*a|*b|*c)` `` → first non-nothing
- [ ] Indexed: `` `$(a|b|c)[1]` `` → `b`
- [ ] Wrap-around: `` `$(a|b)[!5]` `` → `$(a|b)[1]` → `b`
- [ ] Roll: `` `$(1|2|3)[%]` `` → random
- [ ] Weighted roll: `` `$(rare:1|common:9)[%]` ``

### Variables
- [ ] Simple reference: `` `*count + 1` ``
- [ ] Indexed reference: `` `*list[0]` ``
- [ ] Map reference: `` `*map["key"]` ``
- [ ] Nested path: `` `*data["users"][0]["name"]` ``
- [ ] Dynamic nested index: `` `*matrix[*i][*j]` ``
- [ ] Undefined variable error
- [ ] Invalid path error: `` `*data["missing"]["key"]` `` → fatal
