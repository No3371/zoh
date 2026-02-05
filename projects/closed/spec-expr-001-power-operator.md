# spec-EXPR-001: Power Operator

## Priority
**Medium** - Standardizes arithmetic capabilities.

## Problem
The `**` operator is missing from the language specification and implementation guides, despite being a standard operator in similar languages.

## Proposed Change
Update the ZOH specification to include the `**` operator for exponentiation.

## Specification Updates

### `s:\repos\zoh\impl\04_expressions.md`

#### Grammar Update
In **Expression Grammar (EBNF)** section:
```ebnf
unary_expr      := ( '!' | '-' ) unary_expr | power_expr
power_expr      := primary_expr ( '**' unary_expr )*
primary_expr    := literal | reference | '(' expression ')' | special_form
```

#### Operator Precedence Update
In **Operator Precedence** table:
Add row for `**`:
| Precedence | Operators | Associativity |
|------------|-----------|---------------|
| 1 (highest) | `**` | Right |

Shift other precedence numbers down (Unary becomes 2, etc.).

#### AST Nodes Update
In **Expression AST Nodes**:
Add to `BinaryOp`:
```
  | '**'
```

#### Binary Operation Implementation
In **Binary Operation Implementation** (`applyBinaryOp`):
Add case:
```
      '**':
        return pow(left, right)
```

#### Type Coercion Update
In **Type Coercion** table:
Add:
| Operation | Left Type | Right Type | Result |
|-----------|-----------|------------|--------|
| `**` | int | int (>=0) | int |
| `**` | number | number | double |

### `s:\repos\zoh\spec.md`

Check if `spec.md` contains the full EBNF. If so, apply similar updates there.
(Based on `impl/04_expressions.md` being the detailed guide, `spec.md` might be higher level or contain a duplicate grammar).

## Acceptance Criteria
1. `impl/04_expressions.md` is updated with new grammar rules.
2. `impl/04_expressions.md` precedence table is updated.
3. `impl/04_expressions.md` includes `**` in AST and implementation examples.

## Review: Challenge
1. Is right-associativity (`2 ** 3 ** 2` = `2 ** 9`) definitely the desired behavior for ZOH?
2. How should the operator handle `nothing` operands? Should it be fatal or propagate `nothing`?
3. What is the behavior for `0 ** 0`? (Math says 1 or undefined, usually 1 in programming).
4. Is there any visual ambiguity with pointer dereference `*`? e.g. `* *ptr` vs `**ptr`?
A: Deferencing is not a thing
5. Should we support a function `pow(b, e)` as well for clarity?
A: Not for now
6. Does `**` have higher precedence than unary `-` (`-2**2` = `-4`)? (Yes, per proposal, but verify against C# standard).
7. What about complex numbers? (ZOH only has double/int, so probably NaN for `sqrt(-1)` equivalent).
8. Is `level 1` precedence correct? `unary` is usually very high.
9. Should we allow `**=` assignment operator?
A: Assignment is not a thing
10. Does this affect any existing macros or variable names (e.g. `**` chars in names/strings)?
