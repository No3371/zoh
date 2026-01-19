# Expression Grammar

Expressions are special constructs enclosed in backticks (`` ` ``) that can be evaluated by `/evaluate` at runtime.

## Syntax

```
`expression`
```

Expressions are stored as data until explicitly evaluated. They support variable interpolation and arithmetic/logical operations.

## Grammar (EBNF)

```ebnf
expression      := or_expr

or_expr         := and_expr ( '||' and_expr )*
and_expr        := equality_expr ( '&&' equality_expr )*
equality_expr   := relational_expr ( ( '==' | '!=' ) relational_expr )*
relational_expr := additive_expr ( ( '<' | '>' | '<=' | '>=' ) additive_expr )*
additive_expr   := multiplicative_expr ( ( '+' | '-' ) multiplicative_expr )*
multiplicative_expr := unary_expr ( ( '*' | '/' | '%' ) unary_expr )*
unary_expr      := ( '!' | '-' ) unary_expr | primary_expr
primary_expr    := literal | reference | '(' expression ')' | special_form

literal         := string_literal | integer_literal | double_literal | boolean_literal | nothing_literal
string_literal  := '"' <chars> '"' | "'" <chars> "'"
integer_literal := ['-'] <digits>
double_literal  := ['-'] <digits> '.' <digits>
boolean_literal := 'true' | 'false'
nothing_literal := '?'

reference       := '*' identifier [ '[' index ']' ]
identifier      := <letter_or_underscore> <alphanumeric_or_underscore>*
index           := integer_literal | string_literal | reference | expression
```

## Special Forms

Expressions support special syntax that expands to verb calls during evaluation:

```ebnf
special_form    := interpolate_form | count_form | conditional_form | any_form | indexed_form | roll_form | wroll_form

interpolate_form := '$(' expression ')'
count_form       := '$#(' reference ')'
conditional_form := '$?(' expression '?' expression ':' expression ')'
any_form         := '$?(' option_list ')'
indexed_form     := '$(' option_list ')[' index_spec ']'
roll_form        := '$(' option_list ')[%]'
wroll_form       := '$(' weighted_option_list ')[%]'

option_list         := expression ( '|' expression )*
weighted_option_list := weighted_option ( '|' weighted_option )*
weighted_option     := expression ':' integer_literal
index_spec          := ['!'] ( reference | integer_literal )
```

### Special Form Semantics

| Syntax | Evaluates To |
|--------|--------------|
| `$(*var)` | Interpolated string |
| `$#(*var)` | Count/length of collection or string |
| `$?(*cond? *a : *b)` | `*a` if `*cond` is truthy, else `*b` |
| `$?(*a\|*b\|*c)` | First non-Nothing value |
| `$(1\|2\|3)[*i]` | Element at index `*i` |
| `$(1\|2\|3)[!*i]` | Element at index `*i % count` (wrap-around) |
| `$(a\|b\|c)[%]` | Random element |
| `$(a:1\|b:2)[%]` | Weighted random selection |

## Operators

### Precedence (Highest to Lowest)

| Precedence | Operators | Associativity |
|------------|-----------|---------------|
| 1 | `!` `-` (unary) | Right |
| 2 | `*` `/` `%` | Left |
| 3 | `+` `-` | Left |
| 4 | `<` `>` `<=` `>=` | Left |
| 5 | `==` `!=` | Left |
| 6 | `&&` | Left |
| 7 | `\|\|` | Left |

### Type-Specific Behavior

| Operator | Integer | Double | String | List | Boolean |
|----------|---------|--------|--------|------|---------|
| `+` | Add | Add | Concatenate | Concatenate | N/A |
| `-` | Subtract | Subtract | N/A | N/A | N/A |
| `*` | Multiply | Multiply | N/A | N/A | N/A |
| `/` | Divide (floor) | Divide | N/A | N/A | N/A |
| `%` | Modulo | N/A | N/A | N/A | N/A |
| `==` | Equal | Equal | Equal (case-insensitive) | Deep equal | Equal |
| `!=` | Not equal | Not equal | Not equal | Not deep equal | Not equal |

## Variable Resolution

Within expressions, `*var` references are resolved at evaluation time:
- Story scope is searched first
- Context scope is searched if not found
- **Throws an error if any referenced variable doesn't exist**

## Escaping

- `\*` - Literal asterisk (not a reference)
- `` \` `` - Literal backtick (escape expression delimiter)

## Examples

```zoh
:: Arithmetic
`*count + 1`
`(*price * *quantity) - *discount`
`*total / *items`

:: Comparison
`*health > 0`
`*name == "Alice"`
`*score >= 100 && *lives > 0`

:: Logical
`!*is_dead && *has_key`
`*door_open || *has_lockpick`

:: Special forms
`$#(*inventory)`           :: Count items
`$?(*gold > 100? "rich" : "poor")`  :: Conditional
`$(1|2|3|4|5|6)[%]`        :: Random dice roll
```

## Usage

Expressions are evaluated via `/evaluate` (or sugar `` $`expr` ``):

```zoh
/evaluate `*var + 1`; -> *result;
$`*health - *damage`; -> *new_health;
```
