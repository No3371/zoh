# Walkthrough: spec-FLOW-001-checkpoints

**Status**: Completed & Verified
**Date**: 2026-01-23

## 1. Objective

Replaced the concept of "Labels" with "Checkpoints" across the ZOH Specification and Implementation Guides. Checkpoints elevate named locations in a story to first-class structural elements that define a state contract (required variables), improving runtime safety and clarity.

## 2. File Changes

### `spec.md`

- **Glossary/Types**: "Labels" replaced with "Checkpoints".
- **Story Structure**: defined `@checkpoint *contract` syntax.
- **core.Jump**, **core.Fork**, **core.Call**:
    - Terminology update.
    - Added **Validation** section specifying that contract violations (missing required variables) result in Fatal errors.
- **Example (The Last Coffee Shop)**: Converted `@label` usage to `@checkpoint` style (conceptual update, syntax `@name` remains valid for no-contract checkpoints).

### `impl/02_parser.md`

- **AST**: Replaced `LabelNode` with `CheckpointNode`.
    - Added `contract: List<Reference>` to the node.
- **Grammar**: Updated `label` rule to `checkpoint := AT IDENTIFIER (reference)*`.
- **Logic**: Updated `parseLabel` to `parseCheckpoint` and added parameter parsing loop.

### `impl/09_runtime.md`

- **CompiledStory**: Replaced `labels: Map<string, int>` with `checkpoints: Map<string, CompiledCheckpoint>`.
    - `CompiledCheckpoint` stores the contract and statement index.
- **Execution**: Logic updated to skip `CompiledCheckpoint` (markers) but contract validation logic is implicitly required by the Spec update (actual code implementation pending in next project).

## 3. Verification

Verified via `grep` that "Label" usage in `spec.md` is minimal/appropriately legacy or non-existent in critical definitions. Verified `impl` guides define the new structures correctly.

## 4. Key Insights

- The transition from Label to Checkpoint is largely compatible with existing syntax (`@name`) if no contract is used, making it a non-breaking semantic shift for simple stories, but a breaking one for Runtime implementers who must now handle the contract.
- Syntax sugar for Jump/Fork/Call (`====>`, `====+`, `<===+`) works naturally with Checkpoints.
