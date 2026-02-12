# Checkpoint Type Contract Proposal

> **Status:** Accepted
> **Created:** 2026-02-08
> **Author:** Antigravity (on behalf of User)
> **Related Projex:**
>   - [Spec Plan](projex/20260208-checkpoint-type-contract-spec-plan.md)
>   - [Impl Spec Plan](impl/projex/20260208-checkpoint-type-contract-impl-spec-plan.md)
>   - [C# Impl Plan](c%23/projex/20260208-checkpoint-type-contract-impl-plan.md)

---

## Summary

This proposal introduces optional type annotations for checkpoint parameters (e.g., `@checkpoint *var1 [ *var2:type ] ...`), enabling the runtime to enforce type safety at story boundaries. This ensures that variables transferred or required by a checkpoint match the expected types, preventing late runtime errors.

---

## Problem Statement

### Current State
Currently, checkpoints define a "contract" by listing required variables (`@checkpoint *var1 *var2`). The runtime validates that these variables exist and are not `nothing` when jumping/calling/forking to the checkpoint.

### Gap / Need / Opportunity
However, the current contract only guarantees *presence*, not *type*.
If a story expects `*count` to be an integer but receives a string "10", the error will only occur when `*count` is used in a mathematical operation later in the story.
This deferred failure makes debugging harder, especially across story boundaries where context is transferred.
Checking types at the boundary (the checkpoint) follows the "fail fast" principle and makes the story's interface explicit.

### Why Now?
As stories grow in complexity and number, rigorous interface definitions become crucial for maintainability and correctness.

---

## Proposed Change

### Overview
Extend the checkpoint syntax to support optional type contracts for variables.

### Syntax
The syntax for checkpoint definition is extended:
```zoh
@checkpoint *var1 [ *var2:type ] ...
```

Where `type` is a case-insensitive string matching standard ZOH types:
- `string`
- `integer`
- `double`
- `boolean`
- `list`
- `map`
- `channel`
- `verb`
- `expression`

### Examples

**Mixed typed and untyped:**
```zoh
@checkpoint *name:string *age:integer *metadata
```

**All typed:**
```zoh
@checkpoint *id:integer *is_active:boolean
```

### Semantics

1.  **Validation Timing**:
    - During `/jump`, `/call`, or `/fork` (any operation targeting a checkpoint).
    - When execution flow naturally reaches a checkpoint (fall-through).

2.  **Validation Logic**:
    - **Existence**: (Existing behavior) variable must exist and not be `nothing`.
    - **Type**: (New behavior) If a type is specified, the runtime checks if the variable's value matches the specified type.
        - `integer` matches `integer` values.
        - `double` matches `double` values.
        - `string` matches `string` values.
        - etc.

3.  **Failure**:
    - If type mismatch occurs, raise a **Fatal** diagnostic: `checkpoint_violation`.
    - The diagnostic message should specify which variable failed and the expected vs. actual type.

### Approach Options

#### Option A: Colon Syntax (Recommended)
Use `*var:type`.
- **Pros**: Familiar to many languages (TypeScript, Python hints, etc.), concise.
- **Cons**: Might be confused with map literal syntax if not careful (though map keys are strings), but here it is strictly variable names.

#### Option B: Separate Attribute
Use attributes like `[type:"integer"] *var`.
- **Pros**: Uses existing attribute syntax.
- **Cons**: Verbose. `@checkpoint [type:"string"] *name [type:"integer"] *age` is hard to read.

#### Option C: Keyword
`@checkpoint *var as integer`.
- **Pros**: Readable.
- **Cons**: Adds new keywords `as` to the grammar in this specific context.

### Recommended Approach
**Option A (Colon Syntax)** is recommended for its brevity and intuitive mapping to "variable of type".

---

## Impact Analysis

### Affected Areas
- **Spec**: `spec.md` (Checkpoint section).
- **Runtime**:
    - **Parser**: Needs to parse `*var:type` in checkpoint definitions.
    - **Compiler/Model**: Store the expected type in the checkpoint model.
    - **Validator/Runtime**: `Jump`/`Call`/`Fork` logic needs to perform the type check.

### Dependencies
- None.

### Risks
| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Syntax ambiguity | Low | Low | The `:` is tokenized clearly. Ensure it doesn't conflict with other uses (e.g. map keys). |
| Performance | Low | Low | Type checking is a simple O(1) op per variable. |

### Breaking Changes
- **None**: This is an additive change. Existing checkpoints without types continue to work as "any" (or "not nothing").

---

## Open Questions

- [ ] Should we support union types? (e.g. `*var:string|integer`) -> *Defer for now. Keep it simple.*
- [ ] Should we support `any` explicitly? -> *`*var` without type implies `any` (but not nothing).*
- [ ] Does `nothing` satisfy any type? -> *No, `nothing` is already rejected by the existence check.*

---

## Next Steps

If accepted:
1.  Create a Plan Projex to implement this.
2.  Update `spec.md`.
3.  Implement in C# Runtime.
