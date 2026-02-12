# Update Impl Docs with Checkpoint Type Contracts

> **Status:** Created
> **Completed:** 2026-02-08
> **Walkthrough:** [Walkthrough](20260208-checkpoint-type-contract-walkthrough.md)
> **Created:** 2026-02-08
> **Author:** Antigravity (on behalf of User)
> **Source:** [Proposal](projex/20260208-checkpoint-type-contract-proposal.md)
> **Related Projex:**
>   - [Spec Plan](projex/20260208-checkpoint-type-contract-spec-plan.md)
>   - [C# Impl Plan](c%23/projex/20260208-checkpoint-type-contract-impl-plan.md)

---

## Summary

This plan updates the implementation specification documents (`impl/`) to reflect the new checkpoint type contracts feature.

**Scope:** `impl/02_parser.md`, `impl/09_runtime.md`.
**Estimated Changes:** 2 files, ~20 lines.

---

## Objective

### Problem / Gap / Need
The `impl` docs describe checkpoint contracts as `List<string>` (variable names only). To align with the new spec, these must be updated to support typed contracts.

### Success Criteria
- [ ] `impl/02_parser.md` `CheckpointNode.contract` updated to `List<ContractParam>`.
- [ ] `impl/09_runtime.md` `CompiledCheckpoint.contract` updated to `List<ContractParam>`.
- [ ] Contract validation logic documented.

### Out of Scope
- Spec changes (separate plan).
- C# implementation (separate plan).

---

## Context

### Key Files
| File | Purpose | Changes Needed |
|------|---------|----------------|
| `impl/02_parser.md` | Parser implementation | Update `CheckpointNode` definition (L83-90). |
| `impl/09_runtime.md` | Runtime architecture | Update `CompiledCheckpoint` definition (L296-299). Add validation notes. |

---

## Implementation

### Step 1: Update `impl/02_parser.md`

**Objective:** Define `ContractParam` and update `CheckpointNode`.

**Changes:**

```markdown
// Before (Lines 83-90):
### Checkpoint

```
CheckpointNode:
  name: string              # Checkpoint name (without @)
  contract: List<Reference> # Required variables
  position: Position
```

// After:
### Checkpoint

```
ContractParam:
  name: string              # Variable name (without *)
  type: string?             # Optional type constraint (e.g. "integer", "string")

CheckpointNode:
  name: string              # Checkpoint name (without @)
  contract: List<ContractParam> # Required variables with optional type constraints
  position: Position
```
```

**Also update:**
- Grammar (line ~105): `checkpoint := AT IDENTIFIER (contract_param)*`
- Add: `contract_param := ASTERISK IDENTIFIER (COLON IDENTIFIER)?`

### Step 2: Update `impl/09_runtime.md`

**Objective:** Update `CompiledCheckpoint` and add validation logic.

**Changes:**

```markdown
// Before (Lines 296-299):
CompiledCheckpoint:
  name: string
  contract: List<string>          # Required variable names
  statementIndex: int             # Index in statements list

// After:
ContractParam:
  name: string                    # Variable name
  type: string?                   # Type constraint (null = any non-nothing)

CompiledCheckpoint:
  name: string
  contract: List<ContractParam>   # Required variables with optional types
  statementIndex: int             # Index in statements list
```

**Add validation logic section** (after Blocking Operations or as new subsection):

```markdown
## Checkpoint Contract Validation

Navigation verbs (`/jump`, `/call`, `/fork`) AND the runtime execution loop (when falling through to a checkpoint) must validate checkpoint contracts before transferring control.

```
validateContract(checkpoint: CompiledCheckpoint, context: Context):
    for param in checkpoint.contract:
        val = context.get(param.name)
        
        if val is Nothing:
            FATAL "checkpoint_violation": "Required variable '*{param.name}' is nothing"
        
        if param.type != null:
            if not matchesType(val, param.type):
                FATAL "checkpoint_violation": "Variable '*{param.name}' expected {param.type}, got {typeOf(val)}"

matchesType(val: Value, expectedType: string): bool:
    match expectedType.toLowerCase():
        "integer" -> val is ZohInteger
        "double"  -> val is ZohDouble
        "string"  -> val is ZohString
        "boolean" -> val is ZohBool
        "list"    -> val is ZohList
        "map"     -> val is ZohMap
        "channel" -> val is ZohChannel
        "verb"    -> val is ZohVerb
        "expression" -> val is ZohExpression
        _ -> false
```
```

---

## Verification Plan

### Manual Verification
- [ ] Review updated `impl/02_parser.md` for consistency.
- [ ] Review updated `impl/09_runtime.md` for consistency.

---

## Notes
- This is a documentation-only change to the impl specs.
