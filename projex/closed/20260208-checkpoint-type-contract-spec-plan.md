# Update Spec with Checkpoint Type Contracts

> **Status:** Closed
> **Walkthrough:** [Walkthrough](file:///s:/repos/zoh/projex/closed/20260208-checkpoint-type-contract-spec-walkthrough.md)
> **Created:** 2026-02-08
> **Author:** Antigravity (on behalf of User)
> **Source:** [Proposal](file:///s:/repos/zoh/projex/20260208-checkpoint-type-contract-proposal.md)
> **Related Projex:** None

---

## Summary

This plan updates the ZOH language specification (`spec.md`) to include optional type contracts in checkpoint definitions. This formalizes the feature's syntax and semantics, enabling rigorous implementation.

**Scope:** `spec.md` only.
**Estimated Changes:** 1 file, ~30 lines.

---

## Objective

### Problem / Gap / Need
The current checkpoint definition in `spec.md` only documents variable existence checks. The accepted proposal introduces optional type checking to improve safety and debuggability at story boundaries. The spec must reflect this new feature to guide implementation.

### Success Criteria
- [ ] `spec.md` documents the new syntax: `@checkpoint *var:type`.
- [ ] `spec.md` documents the validation logic for type checking.
- [ ] `spec.md` includes examples of typed checkpoints.
- [ ] `spec.md` specifies the `checkpoint_violation` fatal diagnostic.

### Out of Scope
- Implementation in runtimes (covered by separate plan).
- Changes to existing checkpoints in the spec (unless they are examples).

---

## Context

### Current State
`spec.md` section `## Checkpoint` describes `@checkpoint *var1 *var2` syntax and mentions validation for variable existence.

### Key Files
| File | Purpose | Changes Needed |
|------|---------|----------------|
| `spec.md` | Language specification | Update `## Checkpoint` section with new syntax and semantics. |

### Dependencies
- **Requires:** Proposal acceptance (Completed).
- **Blocks:** [C# Implementation Plan](file:///s:/repos/zoh/c#/projex/20260208-checkpoint-type-contract-impl-plan.md)

### Constraints
- Must align exactly with the accepted proposal.

---

## Implementation

### Overview
Update the `## Checkpoint` section in `spec.md` to include the new syntax, validation rules, and examples.

### Step 1: Update Checkpoint Section

**Objective:** Document the extended syntax and validation rules.

**Files:**
- `s:\repos\zoh\spec.md`

**Changes:**

```markdown
// Before:
A checkpoint, denoted as `@checkpoint`, is a named node in a stoty, that enables `/jump`, `/fork` and `/call`.
...
A checkpoint can be suffixed with variable references. All referenced variables must not be resolved to `nothing` when the context is about to execute across or jump/fork/call to the checkpoint.

### Examples
```
@checkpoint *var1 *var2 *var3
```

// After:
A checkpoint, denoted as `@checkpoint`, is a named node in a stoty, that enables `/jump`, `/fork` and `/call`.
...
A checkpoint can be suffixed with variable references. These references define the contract for the checkpoint.
The syntax for contract variables is `*var` or `*var:type`.
- `*var`: The variable must exist and not be `nothing`.
- `*var:type`: The variable must exist, not be `nothing`, and match the specified type.
Supported types are: `string`, `integer`, `double`, `boolean`, `list`, `map`, `channel`, `verb`, `expression`.

Validation occurs when a context jumps, calls, or forks to the checkpoint, OR when execution naturally reaches it.
- If a required variable is missing or `nothing`, a fatal `checkpoint_violation` is raised.
- If a typed variable does not match the specified type, a fatal `checkpoint_violation` is raised.

### Examples
```
@checkpoint *var1 *var2:integer *var3:string
```
```

**Rationale:** Directly incorporates the proposal's syntax and semantics.

**Verification:** Manual review of the rendered markdown.

---

## Verification Plan

### Manual Verification
- [ ] Review `spec.md` to ensure the new section is clear and correct.
- [ ] Verify examples cover both typed and untyped cases.

### Acceptance Criteria Validation
| Criterion | How to Verify | Expected Result |
|-----------|---------------|-----------------|
| Syntax Documented | Read `spec.md` | Contains `*var:type` description. |
| Validation Rules | Read `spec.md` | Mentions type checking and `checkpoint_violation`. |

---

## Rollback Plan

If changes are incorrect:
1. Revert `spec.md` to previous commit.

---

## Notes

### Risks
- None. Documentation change only.

### Open Questions
- None.
