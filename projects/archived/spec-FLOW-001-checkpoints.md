# spec-FLOW-001-checkpoints: Replace Labels with Checkpoints

## Priority
**High** - Current label system lacks safety and validation capabilities required for robust story flow control.

## Problem

Current labels (`@label`) are simple jump targets without any validation of the execution state. This leads to:
1.  **Runtime Errors**: Jumping to a label that expects specific variables (set in a previous segment) to be populated, but they are missing or `nothing`.
2.  **Implicit Dependencies**: It is opaque what state a section of the story relies on without reading the entire code block.
3.  **Loose Structure**: Labels can be placed anywhere, even recursively in blocks (though not indexed), leading to confusing flow.

### Specific Issues
**1. `spec.md:1187` - Core.Jump**
> A jump verb instruct the context to jump to a label in a story.

The spec does not define "Label" firmly (other than as a syntax element and jump target). There is no mechanism to ensure the target state is valid.

## Proposed Change

Replace the concept of **Labels** with **Checkpoints**. A checkpoint is a named node at the root level of a story that strictly defines its required state.

### Syntax
```zoh
@checkpoint_name *required_var1 *required_var2
```

### Semantics
1.  **Root Level Only**: Checkpoints must be defined at the root level of the story structure.
2.  **State Contract**: The optional list of variables after the checkpoint name defines the **Contract**.
3.  **Validation**: When `Jump`, `Fork`, or `Call` targets a checkpoint, the runtime parses the contract. For each variable in the contract, the runtime verifies that the variable exists in the *current context* (or is being passed in the transition using the verb's `*var` params) and is not `nothing`. If verification fails, a fatal error is raised *before* the transition completes.

## Spec Updates

### `spec.md`
- **Glossary/Types**: Remove "Label" if present, add "Checkpoint".
- **Story Structure** (~line 243): Replace "Labels" consideration with "Checkpoints". Define syntax: `@name [contract...]`.
- **Core.Jump** (~line 1187): Update description. Mention that `Jump` validates the target checkpoint's contract.
- **Core.Fork** (~line 1238): Update description. Validates contract.
- **Core.Call** (~line 1289): Update description. Validates contract.

## Implementation Guide Updates

### `impl/02_parser.md`
- **AST Node Types**: Replace `LabelNode` with `CheckpointNode`. Add `parameters: List<Value>` (specifically References) to the node structure.
- **ParseStatement** (~line 104): Ensure `@` starting lines are only parsed at root level (already seemingly true but reinforce).
- **parseLabel -> parseCheckpoint**: Update to parse optional Space-delimited Reference list after the identifier.

### `impl/09_runtime.md`
- **CompiledStory** (Story Structure): Update `CompiledStory` to store `Checkpoints` (name + contract) instead of just `Labels` (name + position).
- **Context Structure**: Update terminology if applicable.
- **Execution**: Logic for `Jump`, `Fork`, `Call` must include contract validation.

## Acceptance Criteria

1.  **Strict Terminology**: `spec.md` and `impl/*.md` no longer use the term "Label" except perhaps in legacy notes/examples illustrating changes or in a dedicated "Migration" section.
2.  **Formal Syntax**: `spec.md` clearly defines `@name *contract` syntax.
3.  **Behavior Definition**: `spec.md` explicitly states that jumping to a checkpoint without satisfying its contract results in a fatal error.
4.  **Updated Guides**: `impl/02_parser.md` and `impl/09_runtime.md` reflect the new data structures and logic.
