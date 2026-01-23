# feat-REF-002: Dynamic Path Resolution using Expressions

## Priority
**Medium** - Addresses a gap in the specification for nested paths.

## Problem

The recent addition of Nested Paths (REF-001) allows navigating collections using indices: `*var[index]`.
The specification states that "Each index is evaluated".
However, the specification does not define behavior when the evaluation of an index yields a **Resolvable Type** (specifically `expression` values).

1.  **Strict Typing**: Lists require `Integer` indices. Maps require `String` keys.
2.  **Invalid Usage**: If a variable `*logic` holds an expression `` `1+1` ``, using it as a reference index `*list[*logic]` currently results in a Fatal Error (Invalid Type), because `ExpressionValue` is not an `Integer`.
3.  **Missing Feature**: Users cannot use stored logic or verb calls to dynamically determine a path index inline, forcing verbose workarounds:
    ```zoh
    :: Workaround required currently
    *idx <- [resolve] *logic;
    <- *list[*idx];
    ```

## Flaw Analysis

The specification is "broken" or incomplete because it restricts index types without accounting for resolvable value types (Expressions/Verbs) which are valid in the language.

### Specific Spec Issues

**1. `spec.md` - Strictly typed index definition:**
> `spec.md:339`
> `- Each index navigates into a collection: integers for lists (0-based), strings for map keys.`

This line implies that *only* integers and strings are valid indices. It fails to mention that an attribute (like `[resolve]`) or implicit behavior should resolve other types *into* integers/strings.

**2. `expr.md` - Grammar permits expressions but behavior is undefined:**
> `expr.md:36`
> `index := integer_literal | string_literal | reference | expression`

The expression grammar *allows* an expression to be an index, but `spec.md` doesn't describe how that expression is processed (evaluated vs treated as a literal value) during path navigation. The default assumption in ZOH is that values are passed as-is unless `[resolve]` is used, but `[resolve]` cannot be applied to individual path indices in the current syntax `*var[index]`.

### Conclusion
The spec defines `Expression` and `Verb` as first-class values but renders them unusable in this context, creating a syntax trap where the grammar allows `*var[`expr`]` but the runtime (following spec) would Fatal on "Invalid Type".

## Proposed Change

**Reference Path Resolution should support implicitly resolving `expression` values.**

When navigating a reference path `*var[index1][index2]...`:
1.  Evaluate the index to get a Value (standard behavior).
2.  **NEW**: If the Value is an `ExpressionValue`, **implicitly resolve it** (evaluate) to get the final index value.
    - If the result is another Expression, continue resolving (or limit recursion depth).
    - **Crucially, `VerbValue` is NOT implicitly resolved.** This prevents side-effects from arbitrary verb execution during path navigation.
3.  Use the final resolved value as the index.

### Examples

```zoh
*list <- [10, 20, 30];
*logic <- `1 + 1`;
*getter <- /sequence/ /converse "Getting index..."; /eval `0`; /;;

:: With proposed change:
<- *list[*logic];      :: Resolves logic `1+1` -> 2, returns 30
<- *list[`1+1`];       :: works if parser allows expression literals in path

:: Verbs are NOT resolved:
<- *list[*getter];     :: Fatal error: Invalid Type (VerbValue is not Integer)
```

## Considerations

-   **Safety**: Restricting to Expressions avoids the "surprise side effects" of executing full verbs like `/delete_all_files;` just by accessing a variable. Expressions are generally safer (though they can call read-only special forms).
-   **Syntax**: `*var[`expr`]` closely resembles standard `arr[i+1]` indexing in other languages. `*var[/verb;]` is syntactically awkward and jarring.
-   **Consistency**: While ZOH is verb-centric, path navigation is a "query" operation. Expressions map well to "calculating a value", whereas Verbs map to "performing an action".
-   **Map Keys**: Maps usually stringify keys. If we resolve, we lose the ability to use the *string representation* of an expr as a key. This is acceptable as the utility of dynamic keys outweighs using raw expression code as a map key.

## Conclusion

To fully realize the power of Nested Paths, the spec should be updated to mandate implicit resolution of `Verb` and `Expression` values when used as path indices.
