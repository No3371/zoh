# rfc-FLOW-001: Replace Labels with Checkpoints

## Status
**Draft**

## Summary
Replace the `@label` syntax with `@checkpoint` to denote named nodes in a story. Checkpoints serve as the primary targets for `/jump`, `/fork`, and `/call` verbs. They enforce state contracts by allowing optional variable references that must be valid (not `nothing`) upon entry.

## Motivation
Current labels (`@label`) are simple jump targets without any validation of the execution state. This can lead to runtime errors when jumping to a label that expects certain variables to be populated. 

This proposal aims to:
1.  **Formalize Story Structure**: Elevate named nodes to "checkpoints" which implies a safe, recoverable state.
2.  **Enforce State Contracts**: Allow checkpoints to declare required variables, ensuring the context is valid before execution proceeds.
3.  **Unified Node Concept**: Replace loose "labels" with a strictly defined structural element (`@checkpoint`) at the root level of stories.

## Proposal

### Syntax
A checkpoint is denoted by `@` followed by the checkpoint name. It must be defined at the root level of a story and on its own line.

```zoh
@checkpoint_name
```

### Constraints
1.  **Placement**: Must be at the root level (not nested inside blocks).
2.  **Format**: Must be on its own line.
3.  **Naming**: Case-insensitive, no whitespace, no reserved characters.
4.  **Uniqueness**: Names must be unique within each story.

### State Contracts (Parameters)
Checkpoints can optionally be suffixed with a space-separated list of variable references. These serve as a contract for the state required by the checkpoint.

```zoh
@checkpoint_name *var1 *var2
```

**Semantics**:
- When executing across, jumping to, forking to, or calling a checkpoint, the runtime verifies that all referenced variables (`*var1`, `*var2`) resolve to a value other than `nothing`.
- If any required variable is `nothing` or missing, the runtime should emit a fatal error.

### Example

```zoh
Story Main
===

@start
    /set *score, 0;
    /jump @level_1;

@level_1 *score
    /converse "Level 1 Start. Score: ${*score}";
    /set *score, 10;
    /jump @level_2;

@level_2 *score *inventory
    :: valid if *score and *inventory exist
```

## Impact

### Benefits
-   **Safety**: Catches missing state errors earlier (at the jump/transition point).
-   **Clarity**: Explicit requirements for each section of the story.
-   **Tooling**: Easier to analyze story flow and dependencies.
-   **Save/Restore**: "Checkpoints" are natural candidates for save slots.

### Drawbacks
-   **Verbosity**: Requires explicit listing of dependencies (though this is also a benefit).
-   **Breaking Change**: All existing `@label` usage syntax must be updated to reference `@checkpoint` (conceptually) or simply renamed if the keyword changes (though the syntax `@name` remains, the terminology changes, and the strict root-level/line requirement might affect some existing loose label formatting).
-   **Refactoring**: Moving code into a checkpoint requires updating the contract.

### Affected Areas
-   **Spec**: `spec.md` (Labels section needs total rewrite).
-   **Parser**: Needs to enforce root-level and "own line" constraints more strictly if not already. Needs to parse the space-separated parameter list.
-   **Runtime**: `Jump`, `Fork`, `Call` verbs must validate contracts.
-   **Compiler/Validator**: Check uniqueness and format.

## Open Questions
1.  Should we allow type constraints in the contract? e.g. `@name *score:integer`. For now, we are sticking to simple existence check (`not nothing`) to keep syntax clean and dynamic.

## Follow-up Work
1.  Update `spec.md` to remove "Label" and add "Checkpoint".
2.  Update parser to handle state contracts.
3.  Update runtime to validate contracts on transition.
4.  Update all existing tests and examples to use the new structure.
