# Walkthrough: spec-DIAG-002-invalid-index-type

**Status**: Completed & Verified
**Date**: 2026-01-23

## 1. Objective
Standardize the error behavior for Invalid Index Types (e.g. using a `Double` to index a `List`) by introducing a dedicated Fatal diagnostic code `invalid_index_type`, separating it from missing path errors (`invalid_index`).

## 2. Specification Changes

### `spec.md`
- Added **Index Type Validation** rule to Reference Paths section.
- Updated Diagnostics for the following verbs to declare `Fatal: invalid_index_type`:
    - `/set`, `/get`, `/drop`, `/capture`
    - `/count` (Split from `invalid_index`)
    - `/increase`, `/decrease` (Split from `invalid_index`)
    - `/any`
    - Additional verbs updated in fix-DIAG-003: `/type`, `/has`, `/append`, `/remove`, `/insert`, `/clear`

## 3. Implementation Guide Updates

### `impl/05_type_system.md` & `impl/06_core_verbs.md`
Updated pseudocode to return `fatal("invalid_index_type", ...)` instead of generic errors or `invalid_index` when type checks fail.

## 4. C# Runtime Implementation

### Architecture: Exception-to-Diagnostic Mapping
To avoid massive refactoring of `ValueResolver` signatures, we implemented a robust Exception handling pattern.

1.  **`ZohDiagnosticException.cs`** (New File): Created to carry specific ZOH diagnostic codes (e.g. `invalid_index_type`) from deep execution stacks.

2.  **`Zoh.Runtime.Execution.ZohRuntime`**:
    -   Updated `ExecuteVerb` and `Run` to wrap execution in `try-catch`.
    -   Catches `ZohDiagnosticException` -> Converts to Fatal Diagnostic with specified code.
    -   Catches generic `Exception` -> Converts to Fatal `runtime_error` (Safety Net).

### `Zoh.Runtime.Helpers.CollectionHelpers`
Refactored `GetAtPath` to perform **Explicit Type Validation** before access, ensuring specific error messages.

**File**: `c#\src\Zoh.Runtime\Helpers\CollectionHelpers.cs`
**Changes**:
-   `GetAtPath`:
    ```csharp
    // Type Validation
    if (value is ZohList && index is not ZohInt)
        throw new ZohDiagnosticException("invalid_index_type", $"List index must be integer, got {index.Type}");
    // ... same for Map
    ```
-   `GetIndex`: Reverted to return `ZohValue.Nothing` on default fallthrough, ensuring it remains a safe primitive accessor.

## 5. Verification

### Unit Tests
**File**: `c#\tests\Zoh.Tests\Execution\ValueResolverTests.cs`

1.  **Invalid Type Test**: `Resolve_IndexedReference_ExpressionResolvesToInvalidType_Throws`
    -   Action: Resolve `*list[*double]`.
    -   Assert: Throws `ZohDiagnosticException`.
    -   Assert Code: `invalid_index_type`.
    -   Assert Message: "List index must be integer".
    -   Status: **Passed**.

2.  **Recursion Test**: Verified `runtime_error` for infinite loops.

## 6. Conclusion
The error handling is now fully spec-compliant, robust, and safe. Type mismatches produce clear, fatal diagnostics without crashing the host application.
