# Story Header Virtual Token Plan

> **Status:** Complete
> **Created:** 2026-02-09
> **Author:** Antigravity
> **Completed:** 2026-02-09
> **Walkthrough:** [20260209-story-header-virtual-token-walkthrough.md](20260209-story-header-virtual-token-walkthrough.md)
> **Related Projex:** `s:\repos\zoh\projex\20260208-newline-handling-explore.md`

---

## Summary

This plan updates the **Implementation Guide** to formally define a virtual `STORY_NAME_END` token. This token delineates the end of a story name in the header, allowing story names to contain spaces/multiple words while still being robustly parsed. This mirrors the `CHECKPOINT_END` virtual token mechanism.

**Scope:** Implementation Documentation (`impl/`)
**Estimated Changes:** 2 files.

---

## Objective

### Problem / Gap / Need
The spec states: *"Story name can contain whitespaces"*.
However, the current implementation guide and runtime parses only the first identifier.
To support multi-word story names without complex lookahead or ambiguity, we need a distinct terminator for the story name line.

### Success Criteria
- [ ] `01_lexer.md` documents `STORY_NAME_END` virtual token.
- [ ] `02_parser.md` grammar uses `STORY_NAME_END` in `story_header`.

### Out of Scope
- Code changes (handled in `s:\repos\zoh\c#\projex\20260209-story-header-parsing-plan.md`).

---

## Context

### Current State
- `01_lexer.md` has `CHECKPOINT_END`.
- `02_parser.md` has `story_header := IDENTIFIER NEWLINE ...`. This implies a single identifier.

### Key Files
| File | Purpose | Changes Needed |
|------|---------|----------------|
| `s:\repos\zoh\impl\01_lexer.md` | Lexer Guide | Add `STORY_NAME_END` logic (emit on newline in header). |
| `s:\repos\zoh\impl\02_parser.md` | Parser Guide | Update `story_header` to consume until `STORY_NAME_END`. |

---

## Implementation

### Step 1: Update Lexer Guide

**Objective:** Define `STORY_NAME_END`.

**Files:**
- `s:\repos\zoh\impl\01_lexer.md`

**Changes:**
1.  Add `STORY_NAME_END` to **Virtual Tokens** table in "1. Keywords & Reserved Words" (or create a Virtual Tokens section).
    - Pattern: `\n` (in StoryHeaderMode)
    - Description: Virtual token for story name end.
2.  Update "Logic" section to include **Context-Sensitive Newlines (StoryHeaderMode)** similar to `CheckPointMode`.
    - Enters state at start of file.
    - Exits state after emitting `STORY_NAME_END`.

### Step 2: Update Parser Guide

**Objective:** Use `STORY_NAME_END` in grammar.

**Files:**
- `s:\repos\zoh\impl\02_parser.md`

**Changes:**
1.  Update `story_header` grammar:
    ```ebnf
    story_header    := (IDENTIFIER|STRING)+ STORY_NAME_END (metadata_entry)* STORY_SEP
    ```
    (Or similar, indicating multiple tokens referencing the name are consumed until the virtual terminator).

---

## Verification Plan

### Manual Verification
- [x] Review rendered markdown to ensure clarity and consistency with `CHECKPOINT_END`.
