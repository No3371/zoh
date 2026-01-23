# Walkthrough: feat-REF-002-dynamic-path-resolution

**Status**: Completed & Verified
**Date**: 2026-01-23

## 1. Objective
Implement implicit resolution of `Expression` values when used as indices in Reference Paths (e.g. `*var[*expr]`), enabling dynamic keys and indices.

## 2. Specification Changes

### `spec.md`
Modified **Reference Paths** section (~Line 340):
- Added rule: **Implicit Resolution**: If an index evaluates to an `expression`, it is automatically evaluated (recursively) until a non-expression value is matched.
- Documented **Recursion Depth** limit (implementation detail enforced by runtime).

## 3. Implementation Guide Updates

### `impl/05_type_system.md`
Updated `resolveReference` pseudocode to include the resolution loop:
```python
    # Implicitly resolve expressions in path
    while index is ExpressionValue:
        index = evaluate(index.ast, context)
```

### `impl/06_core_verbs.md`
Updated `getAtPath` and `setAtPath` helpers with similar logic to ensure consistency across verbs.

## 4. C# Runtime Implementation

### `Zoh.Runtime.Helpers.CollectionHelpers`
Modified `GetAtPath` and `SetAtPath` to implement the resolution loop with safety guards.

**File**: `c#\src\Zoh.Runtime\Helpers\CollectionHelpers.cs`
**Changes**:
- Added `const int MAX_RECURSION_DEPTH = 20;`
- Implemented loop:
  ```csharp
  int depth = 0;
  while (index is ZohExpr expr)
  {
      if (++depth > MAX_RECURSION_DEPTH) 
          throw new ZohDiagnosticException("runtime_error", "Maximum recursion depth exceeded...");
      index = ValueResolver.Resolve(expr.ast, context);
  }
  ```

## 5. Verification

### Unit Tests
**File**: `c#\tests\Zoh.Tests\Execution\ValueResolverTests.cs`

1.  **Infinite Recursion Test**: `Resolve_IndexedReference_InfiniteRecursion_Throws`
    -   Setup: `*loop` variable containing expression `*loop` (self-reference).
    -   Action: Resolve `*loop[*loop]`.
    -   Assert: Throws `ZohDiagnosticException` (`runtime_error`) with "Maximum recursion depth exceeded".
    -   Status: **Passed**.

## 6. Conclusion
The feature is fully implemented. Dynamic paths are now supported with robust protection against infinite recursion loops.
