# spec-DIAG-002: Introduce `invalid_index_type` Diagnostic

## Parent Project
**feat-REF-002** - Dynamic Path Resolution using Expressions

## Priority
**High** - Blocks correct completion of feat-REF-002; provides clearer error diagnostics.

## Problem

The `invalid_index` diagnostic is overloaded for two distinct error conditions:

1. **Path navigation failure**: Element doesn't exist or index out of bounds
2. **Type mismatch**: Index resolved to wrong type (e.g., `Double` for list, `Integer` for map)

With feat-REF-002 introducing implicit expression resolution in path indices, type mismatches become more likely (an expression could evaluate to any type). Users need distinct error codes to diagnose issues.

### Current Spec (Overloaded)

| Line | Verb | Diagnostic | Message |
|------|------|------------|---------|
| 454 | `/set` | `invalid_index` | "The index provided is invalid" |
| 530 | `/get` | `invalid_index` | "path navigation failed due to invalid index type" |
| 562 | `/drop` | `invalid_index` | "path navigation failed due to invalid index type" |
| 584 | `/capture` | `invalid_index` | "path navigation failed due to invalid index type" |

Note: Lines 530, 562, 584 already mention "invalid index **type**" but use the generic `invalid_index` code.

## Proposed Change

Introduce `invalid_index_type` as a distinct fatal diagnostic for type mismatches during path resolution.

### Diagnostic Definitions

| Code | Condition | Examples |
|------|-----------|----------|
| `invalid_index` | Path element missing, out of bounds | `*list[999]`, `*map["no_key"]` |
| `invalid_index_type` | Index is wrong type for collection | `*list["key"]`, `*list[1.5]`, `*map[123]` |

### Error Behavior

Type mismatches are **always fatal** regardless of operation context (query or mutation). This is a programmer error, not a data absence condition.

## Spec Updates

### Reference Type Section (~line 339)

Add after the implicit resolution bullet:

```markdown
- **Index Type Validation**: After resolution, indices must match collection type:
    - Lists: integer required (fatal `invalid_index_type` otherwise)
    - Maps: string required (fatal `invalid_index_type` otherwise)
```

### Verb Diagnostics Matrix

Each verb with path resolution needs both diagnostics defined:

| Verb | Line | `invalid_index` (missing/bounds) | `invalid_index_type` (wrong type) |
|------|------|----------------------------------|-----------------------------------|
| `/set` | 454 | Fatal: intermediate path missing | Fatal: index type mismatch |
| `/get` | 530 | *(returns `?` - query semantic)* | Fatal: index type mismatch |
| `/drop` | 562 | Fatal: intermediate path missing | Fatal: index type mismatch |
| `/capture` | 584 | Fatal: intermediate path missing | Fatal: index type mismatch |
| `/count` | 910 | *(returns 0 - query semantic)* | Fatal: index type mismatch |
| `/increase` | 1576 | Fatal: path missing | Fatal: index type mismatch |
| `/decrease` | 1601 | Fatal: path missing | Fatal: index type mismatch |
| `/insert` | 1521 | Error: out of bounds | Fatal: index type mismatch |
| `/remove` | 1539 | *(silent if missing)* | Fatal: index type mismatch |
| `/has` | ~1476 | *(returns false)* | Fatal: index type mismatch |

**Key distinction**:
- **Missing path**: Behavior varies by verb (query returns nothing, mutation fatals)
- **Type mismatch**: Always fatal - this is a programmer error, not data absence

## Implementation Guide Updates

### impl/05_type_system.md

Update `resolveReference` pseudocode to use explicit fatal with diagnostic code:

```python
if baseValue is ListValue:
    if index is not Integer:
        return fatal("invalid_index_type", "List index must be integer, got: " + index.getType())

if baseValue is MapValue:
    if index is not StringValue:
        return fatal("invalid_index_type", "Map key must be string, got: " + index.getType())
```

### impl/06_core_verbs.md

Update `getAtPath`, `setAtPath`, and related helpers to distinguish between:
- `fatal("invalid_index", ...)` for missing path elements
- `fatal("invalid_index_type", ...)` for type mismatches

## Acceptance Criteria

1. `invalid_index_type` is documented as a fatal diagnostic in spec.md
2. All verb diagnostics correctly distinguish between missing paths and type mismatches
3. impl/05_type_system.md uses `fatal("invalid_index_type", ...)` for type errors
4. impl/06_core_verbs.md is consistent with the new diagnostic code
5. feat-REF-002 addendum is updated to reference `invalid_index_type`
