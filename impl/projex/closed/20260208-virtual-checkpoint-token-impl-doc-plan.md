# Virtual Checkpoint Token - Implementation Guide Update Plan

> **Status:** Complete
> **Created:** 2026-02-08
> **Author:** Antigravity
> **Source:** Direct request / [Analysis](../../../checkpoint-parsing-analysis.md)
> **Related Projex:** 
> - Blocks: [C# Implementation Plan](../../../csharp/projex/20260208-virtual-checkpoint-token-csharp-plan.md)

---

## Summary

This plan outlines the updates required for the Implementation Guides (`impl/`) to formally specify the "Virtual Checkpoint-Ending Token" mechanism. This mechanism resolves ambiguity in checkpoint parsing by making newlines significant in the context of checkpoint definitions.

**Scope:** `impl/01_lexer.md`, `impl/02_parser.md`
**Estimated Changes:** 2 files

---

## Objective

### Problem / Gap / Need
The current specification relies on parser lookahead to determine the end of a checkpoint definition (`@label *param`). This is fragile and complicates the parser. The proposed solution is to introduce a virtual token (`CHECKPOINT_END`) emitted by the Lexer when it encounters a newline while in a "Checkpoint Definition" state.

### Success Criteria
- [ ] `01_lexer.md` describes the `CHECKPOINT_END` token and the stateful lexing logic.
- [ ] `02_parser.md` grammar reflects that `checkpoint` is terminated by `CHECKPOINT_END`.
- [ ] Documentation is clear enough for a new runtime implementer to follow.

### Out of Scope
- Implementation in C# (Covered by separate plan)

---

## Context

### Current State
- `01_lexer.md`: Defines `LABEL` as `@identifier`. Does not mention stateful lexing or significant newlines.
- `02_parser.md`: Defines `checkpoint := AT IDENTIFIER (contract_param)*`. Implies greedy consumption or lookahead.

### Key Files
| File | Purpose | Changes Needed |
|------|---------|----------------|
| `impl/01_lexer.md` | Lexer Spec | Add `CHECKPOINT_END` token, describe newline handling logic. |
| `impl/02_parser.md` | Parser Spec | Update grammar and parsing steps to use `CHECKPOINT_END`. |

### Dependencies
- **Blocks:** C# Implementation (technically can proceed in parallel, but spec should lead)

---

## Implementation

### Step 1: Update Lexer Guide

**Objective:** Define the virtual token and state requirements.

**Files:**
- `impl/01_lexer.md`

**Changes:**
1.  Add `CHECKPOINT_END` to **Token Categories**.
2.  Add a section (or update "Lexing Logic") to describe **Context-Sensitive Newlines**:
    *   When `@` is followed by an `identifier`, enter `CheckPointMode`.
    *   In `CheckPointMode`, a newline emits `CHECKPOINT_END` and exits `CheckPointMode`.

**Rationale:** Explicitly documenting this state is crucial for consistent implementation across runtimes.

**Verification:** Manual review of the rendered markdown.

---

### Step 2: Update Parser Guide

**Objective:** Update grammar and parsing logic.

**Files:**
- `impl/02_parser.md`

**Changes:**
1.  Update **Grammar** to: `checkpoint := AT IDENTIFIER (contract_param)* CHECKPOINT_END` (or equivalent).
2.  Update **Implementation Steps** (Step 3 or equivalent) to show `consume(CHECKPOINT_END)` (or checking for it).

**Rationale:** The parser logic becomes simpler (no lookahead), and the spec should reflect this simplification.

**Verification:** Manual review of the rendered markdown.

---

## Verification Plan

### Manual Verification
- [ ] Read `impl/01_lexer.md` and ensure the state machine logic is unambiguous.
- [ ] Read `impl/02_parser.md` and ensure the grammar is correct and consistent with the lexer changes.

---

## Notes

### Risks
- **Backward Compatibility**: This technically changes the spec to *enforce* single-line checkpoints. Although existing stories likely follow this, it's a constraint that should be noted (though the spec already implies newlines separate statements, this makes it strict for checkpoints).
