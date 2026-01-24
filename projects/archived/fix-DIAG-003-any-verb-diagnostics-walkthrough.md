# Walkthrough: fix-DIAG-003-any-verb-diagnostics

**Status**: Completed & Verified
**Date**: 2026-01-23

## 1. Objective

Add missing `invalid_index_type` diagnostics to verbs that accept reference parameters, and fix mutation verbs in impl guides to support nested paths.

## 2. Specification Changes

### spec.md

Added `invalid_index_type` diagnostic to the following verbs:

| Verb | Line | Change |
|------|------|--------|
| Core.Type | ~893 | Added Diagnostics section with `invalid_index_type` |
| Core.Has | ~1137 | Added `invalid_index_type` to existing Diagnostics |
| Core.Append | ~1486 | Added `invalid_index_type` to existing Diagnostics |
| Core.Remove | ~1513 | Added `invalid_index_type` to existing Diagnostics |
| Core.Insert | ~1535 | Added `invalid_index_type` to existing Diagnostics |
| Core.Clear | ~1556 | Added `invalid_index_type` to existing Diagnostics |

## 3. Implementation Guide Updates

### impl/06_core_verbs.md

**TypeDriver (~line 384)**: Added comment noting `resolve()` can propagate `invalid_index_type`.

**HasDriver (~line 729)**: Added comment noting `resolve()` can propagate `invalid_index_type`.

**AppendDriver (~line 835)**: Refactored to use `getAtPath`/`setAtPath` for nested path support.

**RemoveDriver (~line 878)**: Refactored to use `getAtPath`/`setAtPath` for nested path support. Changed type mismatch error from `invalid_type` to `invalid_index_type`.

**InsertDriver (~line 921)**: Refactored to use `getAtPath`/`setAtPath` for nested path support.

**ClearDriver (~line 959)**: Refactored to use `getAtPath`/`setAtPath` for nested path support.

## 4. C# Runtime Implementation

### Driver Updates

**File**: `c#/src/Zoh.Runtime/Verbs/Core/CollectionDrivers.cs`
- Line 86: Changed `"invalid_type"` to `"invalid_index_type"` for map key type mismatch in RemoveDriver

**File**: `c#/src/Zoh.Runtime/Verbs/Core/AnyDriver.cs`
- Line 51: Changed `"invalid_type"` to `"invalid_index_type"` for map key type mismatch

**File**: `c#/src/Zoh.Runtime/Verbs/Core/HasDriver.cs`
- Line 43: Changed `"invalid_type"` to `"invalid_index_type"` for map key type mismatch

### Test Updates

**File**: `c#/tests/Zoh.Tests/Execution/ValueResolverTests.cs`
- Renamed `Resolve_IndexedReference_ExpressionResolvesToInvalidType_ReturnsNothing` to `Resolve_IndexedReference_ExpressionResolvesToInvalidType_Throws`
- Fixed misleading comment

**File**: `c#/tests/Zoh.Tests/Runtime/MapStringKeyTests.cs`
- Updated all 3 tests to expect `"invalid_index_type"` instead of `"invalid_type"`:
  - `Map_IntegerIndex_Fails`
  - `Has_MapWithIntegerSubject_Fails`
  - `Remove_MapWithIntegerKey_Fails`

## 5. Walkthrough Updates

**File**: `projects/closed/walkthrough-spec-DIAG-002-invalid-index-type.md`
- Fixed test name reference from `...ReturnsNothing` to `...Throws`
- Added note about additional verbs updated in fix-DIAG-003

## 6. Verification

```
dotnet clean && dotnet build && dotnet test
```

**Result**: 384 tests passed, 0 failed.

## 7. Key Insight

During review, discovered that **any verb resolving a reference parameter is responsible for `invalid_index_type` errors** from path resolution. This is because verbs resolve references themselves (per `impl/06_core_verbs.md:32-34`), not the runtime framework.

Also identified that mutation verbs (Append, Remove, Insert, Clear) need `getAtPath`/`setAtPath` to support nested paths like `/clear *data["items"];`.
