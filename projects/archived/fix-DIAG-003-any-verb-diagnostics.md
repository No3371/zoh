# fix-DIAG-003: Missing `invalid_index_type` Diagnostics

## Parent Project
**spec-DIAG-002** - Introduce `invalid_index_type` Diagnostic

## Priority
**Medium-High** - Spec completeness + impl guide fixes for nested path support in mutation verbs.

## Problem

During review of `spec-DIAG-002` execution, a systematic gap was identified:

**Any verb that accepts a reference parameter and resolves it could encounter `invalid_index_type`** because the reference could have a path (e.g., `*data["items"]` where `"items"` is wrong type for the collection).

Per `impl/06_core_verbs.md:32-34`, verbs resolve references themselves:
```
All drivers share a common pattern:
1. Extract and validate parameters
2. Resolve references to values, unless in rare cases references are expected
3. Perform the operation
```

This means the verb is responsible for any errors during resolution, including `invalid_index_type`.

## Verbs Missing `invalid_index_type`

### 1. Core.Type (Line 888)
- Accepts: references (`*var`, could be `*data["key"]`)
- Current: No Diagnostics section at all
- Needs: `invalid_index_type` for path resolution errors

### 2. Core.Has (Line 1127)
- Accepts: `*[list]` or `*{map}` for collection (could be `*data["items"]`)
- Current: Only `invalid_type`
- Needs: `invalid_index_type` for path resolution errors

### 3. Core.Append (Line 1475)
- Accepts: `*[list]` or `*{map}` for collection
- Current: Only `invalid_type`, `key_conflict`
- Needs: `invalid_index_type` for path resolution errors

### 4. Core.Remove (Line 1496)
- Accepts: `*[list]` or `*{map}` for collection, `*integer`/`*"string"` for index
- Current: Only `invalid_type`
- Needs: `invalid_index_type` for path resolution errors (both parameters)

### 5. Core.Insert (Line 1518)
- Accepts: `*[list]` for list, `*integer` for index
- Current: `invalid_type`, `invalid_index`
- Needs: `invalid_index_type` for path resolution errors

### 6. Core.Clear (Line 1539)
- Accepts: `*[list]` or `*{map}` for collection
- Current: Only `invalid_type`
- Needs: `invalid_index_type` for path resolution errors
- **Note**: Impl guide needs fix - should support nested paths like `/clear *data["items"];`

## Additional Fix

### Test Name Misleading

**File**: `c#/tests/Zoh.Tests/Execution/ValueResolverTests.cs`
**Test**: `Resolve_IndexedReference_ExpressionResolvesToInvalidType_ReturnsNothing`

The test name says "ReturnsNothing" but actually asserts a throw. Rename to `...Throws`.

## Proposed Changes

### spec.md Updates

**Core.Type (~line 888)** - Add Diagnostics section:
```markdown
#### Diagnostics
- Fatal: `invalid_index_type`: Path navigation failed due to invalid index type.
```

**Core.Has (~line 1136)** - Add to existing Diagnostics:
```markdown
- Fatal: `invalid_index_type`: Path navigation failed due to invalid index type.
```

**Core.Append (~line 1484)** - Add to existing Diagnostics:
```markdown
- Fatal: `invalid_index_type`: Path navigation failed due to invalid index type.
```

**Core.Remove (~line 1507)** - Add to existing Diagnostics:
```markdown
- Fatal: `invalid_index_type`: Path navigation failed due to invalid index type.
```

**Core.Insert (~line 1528)** - Add to existing Diagnostics:
```markdown
- Fatal: `invalid_index_type`: Path navigation failed due to invalid index type.
```

**Core.Clear (~line 1547)** - Add to existing Diagnostics:
```markdown
- Fatal: `invalid_index_type`: Path navigation failed due to invalid index type.
```

### Test Fix

**File**: `c#/tests/Zoh.Tests/Execution/ValueResolverTests.cs`

```diff
- public void Resolve_IndexedReference_ExpressionResolvesToInvalidType_ReturnsNothing()
+ public void Resolve_IndexedReference_ExpressionResolvesToInvalidType_Throws()
```

```diff
- // List requires Int. Get should logic return Nothing.
+ // List requires Int. Type mismatch throws ZohDiagnosticException.
```

### Walkthrough Update

Update `projects/closed/walkthrough-spec-DIAG-002-invalid-index-type.md`:
- Fix test name reference
- Note that `/has` was correctly included (not incorrectly as previously thought)

## Implementation Guide Updates

### impl/06_core_verbs.md

The following drivers use `resolve()` which can propagate `invalid_index_type`:

**TypeDriver (~line 384)** - Add comment noting resolve can fatal:
```python
TypeDriver.execute(call, context):
    # resolve() may return fatal("invalid_index_type", ...) if path has wrong index type
    value = resolve(call.params[0], context)
    return ok(StringValue(value.getType()))
```

**HasDriver (~line 729)** - Already has `invalid_index_type` at line 741 for subject type check, but needs note about path resolution:
```python
HasDriver.execute(call, context):
    # resolve() may return fatal("invalid_index_type", ...) for path errors
    collection = resolve(call.params[0], context)
    subject = resolve(call.params[1], context)
    ...
```

**AppendDriver (~line 833)** - Fix to support nested paths:
```python
AppendDriver.execute(call, context):
    collectionRef = call.params[0]
    if collectionRef is not ReferenceValue:
        return fatal("invalid_type", "Expected reference")

    # Use getAtPath to support nested paths
    collection = getAtPath(context, collectionRef.name, collectionRef.path)
    # getAtPath may return fatal("invalid_index_type", ...) for path errors

    value = resolve(call.params[1], context)
    # resolve() may also return fatal("invalid_index_type", ...)

    # ... append logic ...

    return setAtPath(context, collectionRef.name, collectionRef.path, collection)
```

**RemoveDriver (~line 874)** - Fix to support nested paths:
```python
RemoveDriver.execute(call, context):
    collectionRef = call.params[0]
    if collectionRef is not ReferenceValue:
        return fatal("invalid_type", "Expected reference")

    # Use getAtPath to support nested paths
    collection = getAtPath(context, collectionRef.name, collectionRef.path)
    # getAtPath may return fatal("invalid_index_type", ...) for path errors

    index = resolve(call.params[1], context)
    # resolve() may also return fatal("invalid_index_type", ...)

    # ... remove logic ...

    return setAtPath(context, collectionRef.name, collectionRef.path, collection)
```

**InsertDriver (~line 915)** - Fix to support nested paths:
```python
InsertDriver.execute(call, context):
    listRef = call.params[0]
    if listRef is not ReferenceValue:
        return fatal("invalid_type", "Expected reference")

    # Use getAtPath to support nested paths
    list = getAtPath(context, listRef.name, listRef.path)
    # getAtPath may return fatal("invalid_index_type", ...) for path errors

    index = resolve(call.params[1], context)
    value = resolve(call.params[2], context)
    # resolve() may also return fatal("invalid_index_type", ...)

    # ... insert logic ...

    return setAtPath(context, listRef.name, listRef.path, list)
```

**ClearDriver (~line 951)** - Fix to support nested paths:
```python
ClearDriver.execute(call, context):
    collectionRef = call.params[0]
    if collectionRef is not ReferenceValue:
        return fatal("invalid_type", "Expected reference")

    # Use getAtPath to support nested paths like *data["items"]
    collection = getAtPath(context, collectionRef.name, collectionRef.path)
    # getAtPath may return fatal("invalid_index_type", ...) for path errors

    if collection is ListValue:
        collection.elements.clear()
    elif collection is MapValue:
        collection.entries.clear()
    else:
        return fatal("invalid_type", "Expected list or map")

    # Use setAtPath to write back to nested path
    return setAtPath(context, collectionRef.name, collectionRef.path, collection)
```

### Note on Path Support (Implementation Gap)

The mutation verbs (Append, Remove, Insert, Clear) currently use `context.get(collectionRef.name)` which only resolves the variable name, not nested paths. This is **incorrect** - `/clear *data["items"];` should clear the nested list, not operate on `*data`.

**Required fix**: These verbs should use `getAtPath`/`setAtPath` helpers (or equivalent) to properly support nested paths. This brings them in line with `/set` behavior and makes `invalid_index_type` applicable.

This implementation fix is **in scope** for this project since it's required for the diagnostics to be meaningful.

## Acceptance Criteria

1. All 6 verbs have `invalid_index_type` in their Diagnostics sections in spec.md
2. impl/06_core_verbs.md updated:
   - TypeDriver, HasDriver: Add comments noting `invalid_index_type` propagation from `resolve()`
   - AppendDriver, RemoveDriver, InsertDriver, ClearDriver: Fix to use `getAtPath`/`setAtPath` for nested path support
3. Test method renamed to reflect actual behavior
4. Walkthrough updated with correct test name
5. All tests still pass
