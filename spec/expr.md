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
unary_expr      := ( '!' | '-' ) unary_expr | power_expr
power_expr      := primary_expr ( '**' unary_expr )*
primary_expr    := literal | reference | '(' expression ')' | special_form

literal         := string_literal | integer_literal | double_literal | boolean_literal | nothing_literal
string_literal  := '"' <chars> '"' | "'" <chars> "'"
integer_literal := <digits>
double_literal  := <digits> '.' <digits>
boolean_literal := 'true' | 'false'
nothing_literal := '?'

reference       := '*' identifier ( '[' index ']' )*
identifier      := <letter_or_underscore> <alphanumeric_or_underscore>*
index           := integer_literal | string_literal | reference | expression
```

## Special Forms

Expressions support special syntax that expands to verb calls during evaluation:

```ebnf
special_form    := interpolate_form | count_form | conditional_form | any_form | indexed_form | roll_form | wroll_form

interpolate_form := '$' string_literal | '$' reference
count_form       := '$#(' reference ')'
conditional_form := '$?(' expression '?' expression ':' expression ')'
any_form         := '$?(' option_list ')'
indexed_form     := '$(' option_list ')[' index_spec ']'
roll_form        := '$(' option_list ')[%]'
wroll_form       := '$(' weighted_option_list ')[%]'

option_list         := expression ( '|' expression )*
weighted_option_list := weighted_option ( '|' weighted_option )*
weighted_option     := expression ':' integer_literal
index_spec          := ['!'] expression
```

> **Disambiguation — `any_form` vs `conditional_form`:** Both start with `$?(`. They are distinguished by the token after the first expression: `?` signals conditional form; `|` or `)` signals any form.
>
> **Disambiguation — `roll_form` vs `wroll_form`:** Both use `$(options)[%]`. A form is weighted (`wroll_form`) when any option includes a `:` weight suffix (`expression ':' integer_literal`). Detection happens during parsing of the option list.
>
> **`$(options)` without suffix:** `$(a|b|c)` not followed by `[index]` or `[%]` is a parse error. Use `$?(a|b|c)` for first-non-nothing selection.

### Special Form Semantics

| Syntax | Evaluates To |
|--------|--------------|
| `$*var` | Value of `*var` treated as string template and interpolated |
| `$"string"` | String literal interpolated |
| `$#(*var)` | Count/length of collection or string |
| `$?(*cond? *a : *b)` | `*a` if `*cond` is truthy, else `*b` |
| `$?(*a\|*b\|*c)` | First non-nothing value; `?` if all are nothing |
| `$(1\|2\|3)[*i]` | Element at index `*i` |
| `$(1\|2\|3)[!*i]` | Element at index `*i % count` (wrap-around) |
| `$(a\|b\|c)[%]` | Random element |
| `$(a:1\|b:2)[%]` | Weighted random selection |

## Operators

### Precedence (Highest to Lowest)

| Precedence | Operators | Associativity |
|------------|-----------|---------------|
| 1 | `**` | Right |
| 2 | `!` `-` (unary) | Right |
| 3 | `*` `/` `%` | Left |
| 4 | `+` `-` | Left |
| 5 | `<` `>` `<=` `>=` | Left |
| 6 | `==` `!=` | Left |
| 7 | `&&` | Left |
| 8 | `\|\|` | Left |

### Type-Specific Behavior

| Operator | Integer | Double | String | List | Map | Boolean |
|----------|---------|--------|--------|------|-----|---------|
| `**` | Power¹ | Power | N/A | N/A | N/A | N/A |
| `+` | Add | Add | Concatenate | Concatenate | N/A | N/A |
| `-` | Subtract | Subtract | N/A | N/A | N/A | N/A |
| `*` | Multiply | Multiply | N/A | N/A | N/A | N/A |
| `/` | Divide (floor) | Divide | N/A | N/A | N/A | N/A |
| `%` | Modulo | N/A | N/A | N/A | N/A | N/A |
| `<` `>` `<=` `>=` | Compare | Compare | Lexicographic | N/A | N/A | N/A |
| `==` | Equal | Equal | Equal (case-sensitive) | Deep equal | Deep equal | Equal |
| `!=` | Not equal | Not equal | Not equal | Not deep equal | Not deep equal | Not equal |
| `&&` `\|\|` | →bool² | →bool² | →bool² | →bool² | →bool² | →bool |

¹ Result type: `integer ** integer` (exponent ≥ 0) → `integer`; any other numeric combination → `double`.
² Operands coerced to boolean via truthiness rules (see §Truthiness); result is always `boolean`.

### Truthiness

Boolean coercion is used by `&&`, `||`, `!`, and all conditional forms (`$?`). The following table defines what is truthy and falsy for each type:

| Type | Truthy | Falsy |
|------|--------|-------|
| `nothing` | — | Always falsy |
| `boolean` | `true` | `false` |
| `integer` | Non-zero | `0` |
| `double` | Non-zero | `0.0` |
| `string` | Non-empty | `""` |
| `list` | Non-empty | `[]` |
| `map` | Non-empty | `{}` |
| `channel` | Always truthy | — |
| `verb` | Always truthy | — |
| `expression` | Always truthy | — |

## Edge Cases

### Division and Modulo by Zero

Division or modulo with a zero divisor is a fatal `division_by_zero` error.

```zoh
`*x / 0`   :: fatal: division_by_zero
`*x % 0`   :: fatal: division_by_zero
```

### `nothing` in Operator Expressions

The `?` nothing literal supports equality comparison only:

| Expression | Result |
|------------|--------|
| `*a == ?` | `true` if `*a` is `nothing`, `false` otherwise |
| `*a != ?` | `false` if `*a` is `nothing`, `true` otherwise |

All other operators with a `nothing` operand produce fatal `invalid_type`.

### Short-Circuit Evaluation

`&&` and `||` are short-circuit (lazy):
- `false && X` — `X` is never evaluated; result is `false`
- `true || X` — `X` is never evaluated; result is `true`

Errors in unevaluated branches are not raised.

### `$#()` Count by Type

| Type | Result |
|------|--------|
| `list` | Number of elements |
| `map` | Number of key-value pairs |
| `string` | Number of Unicode code points |
| All other types | Fatal `invalid_type` |

### `$(options)` Indexed Form Out-of-Bounds

Accessing `$(a|b|c)[5]` with an out-of-bounds index is a fatal `invalid_index` error. Wrap-around (`[!5]`) uses modulo and is never out-of-bounds.

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
