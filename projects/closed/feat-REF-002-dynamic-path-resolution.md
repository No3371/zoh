# feat-REF-002: Dynamic Path Resolution using Expressions

## Priority
**Medium** - Addresses a gap in the specification for nested paths.

## Problem

The recent addition of Nested Paths (REF-001) allows navigating collections using indices: `*var[index]`.
The specification states that "Each index is evaluated".
However, the specification does not define behavior when the evaluation of an index yields a **Resolvable Type** (specifically `expression` values).

1.  **Strict Typing**: Lists require `Integer` indices. Maps require `String` keys.
2.  **Invalid Usage**: If a variable `*logic` holds an expression `` `1+1` ``, using it as a reference index `*list[*logic]` currently results in a Fatal Error (Invalid Type), because `ExpressionValue` is not an `Integer`.
3.  **Missing Feature**: Users cannot use stored logic or verb calls to dynamically determine a path index inline, forcing verbose workarounds.

## Flaw Analysis

The specification is "broken" or incomplete because it restricts index types without accounting for resolvable value types (Expressions/Verbs) which are valid in the language.

### Specific Spec Issues

**1. `spec.md` - Strictly typed index definition:**
> `spec.md:339`
> `- Each index navigates into a collection: integers for lists (0-based), strings for map keys.`

This line implies that *only* integers and strings are valid indices. It fails to mention that implicit resolution should resolve other types *into* integers/strings.

**2. `expr.md` - Grammar permits expressions but behavior is undefined:**
> `expr.md:36`
> `index := integer_literal | string_literal | reference | expression`

The grammar allows expressions, but the runtime processing isn't defined to recursively evaluate them during path navigation.

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

## Spec & Impl Updates

### `spec.md`

Update **Reference Paths** section:
```markdown
- Each index navigates into a collection:
    - Lists: Requires integer (0-based).
    - Maps: Requires string key.
- **Implicit Resolution**: If an index evaluates to an `expression`, it is automatically evaluated (recursively) until a non-expression value is matched.
```

### `impl/05_type_system.md`

Update `resolveReference` pseudocode:

```diff
    # Handle indexed access
    index = evaluate(ref.index, context)
+   
+   # Implicitly resolve expressions in path
+   while index is ExpressionValue:
+       index = evaluate(index.ast, context)
    
    if baseValue is ListValue:
```

### `impl/06_core_verbs.md`

Update `getAtPath` and `setAtPath` helpers conceptually (though they delegate to `resolve`, `resolve` usually only handles `Reference` -> `Value`. Path resolution logic needs the extra expression evaluation step explicitly added or defined in the `resolve` helper for indices).

**Guidance**: The implementation should centralize this "Path Index Resolution" logic to ensure `set`, `get`, `drop` and `eval` all behave consistently.

## Considerations

-   **Safety**: Restricting to Expressions avoids the "surprise side effects" of executing full verbs.
-   **Syntax**: `*var[`expr`]` closely resembles standard `arr[i+1]` indexing.
-   **Consistency**: Maps dynamic navigation to "calculating" a location rather than "acting" to find a location.

## Conclusion

To fully realize the power of Nested Paths, the spec should be updated to mandate implicit resolution of `Expression` values when used as path indices.

## Addendum

### Note on Existing Spec Usage
The specification already contained examples of implicit expression resolution in syntactic sugar, specifically `` <- *var[\`*index+1\`];; ``, which is a bug introduced by previous projects that adapt the verbs from dedicated index parameters to embeded path in references. This project fills the gap.

### Error Handling
- **Recursion Depth**: The runtime MUST enforce a maximum recursion depth (e.g., 20) during implicit resolution to prevent infinite loops (e.g., explicit or implicit self-reference).
- **Invalid Types**: If resolution yields an invalid index type (e.g., `Double` for a List), it produces a Fatal `invalid_index_type` diagnostic.
  *Update*: `spec-DIAG-002` standardized this across all relevant core verbs: `/set`, `/get`, `/drop`, `/capture`, `/count`, `/increase`, `/decrease`, `/has`, `/any`.
- **Runtime Safety**: Driver execution is guarded to catch unhandled exceptions (like recursion limit violations or invalid type operations) and report them as Fatal runtime errors (ZOH Diagnostics with Fatal severity), ensuring the host application does not crash.
