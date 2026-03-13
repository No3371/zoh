# Statement Locator System — Spec Integration

> **Status:** Ready
> **Created:** 2026-03-13
> **Author:** Agent
> **Source:** 20260313-statement-locator-system-proposal.md
> **Related Projex:** 20260312-script-level-localization-proposal.md, 20260313-statement-locator-system-proposal.md
> **Worktree:** No

---

## Summary

Add the Statement Locator System to the ZOH language spec as a first-class concept. This integrates the addressing grammar (checkpoint+offset, ranges, named tags, sub-addressing) into `1_concepts.md` and adds the `[address]` standard attribute to `std_attributes.md`.

**Scope:** Spec files only — `spec/1_concepts.md` and `spec/std_attributes.md`
**Estimated Changes:** 2 files

---

## Objective

### Problem / Gap / Need

The statement locator system is fully designed in the proposal (20260313-statement-locator-system-proposal.md) but exists only as a projex document. The language spec has no concept of statement addressing. `/patch` (20260312-script-level-localization-proposal.md) and future consumers need a spec-level definition to reference.

### Success Criteria

- [ ] `spec/1_concepts.md` contains a "Statement Locator" section with locator types, sub-addressing, range syntax, grammar (EBNF), and resolution rules
- [ ] `spec/std_attributes.md` defines the `[address]` attribute
- [ ] New spec content follows existing conventions (heading levels, formatting, example style)
- [ ] Grammar covers: single locator (`@cp+N`), range locator (`@cp+N..M`, `@cp+N..`), named locator (`tag`), and all three sub-address types (`#N`, `.name`, `[attr#N]`)

### Out of Scope

- `/patch` verb definition (separate proposal, goes in `2_verbs.md`)
- `[locator]` attribute (belongs to `/patch` verb spec, not the locator system)
- Implementation spec / C# runtime changes
- Locator resolution algorithm details (impl-level concern)

---

## Context

### Current State

`spec/1_concepts.md` ends with the Checkpoint section (line 371–394). Checkpoints define named entry points (`@name`) for control flow but have no mechanism to address individual statements under them. `spec/std_attributes.md` defines 19 standard attributes (Scope through Inline, lines 9–127) with no `[address]` attribute.

### Key Files

| File | Role | Change Summary |
|------|------|----------------|
| `spec/1_concepts.md` | Language concepts spec | Append "Statement Locator" section after Checkpoint |
| `spec/std_attributes.md` | Standard attributes spec | Append `[address]` attribute definition |

### Dependencies

- **Requires:** 20260313-statement-locator-system-proposal.md (Accepted or Draft-with-consensus — provides the design)
- **Blocks:** `/patch` verb spec integration (needs locator system defined first)

### Constraints

- Must follow existing spec formatting conventions (heading levels, example style, terse descriptions)
- Concepts in `1_concepts.md` use `##` headings (e.g., `## Checkpoint`); sub-sections use `###`
- Attributes in `std_attributes.md` use `##` headings with 1–3 line descriptions

### Assumptions

- The proposal's grammar, resolution rules, and range syntax are accepted as designed
- `[address]` keeps its current name (open question in proposal acknowledged but not blocking)

---

## Implementation

### Overview

Two sequential additions: (1) a new concept section in `1_concepts.md` and (2) a new attribute entry in `std_attributes.md`. Both are pure appends — no existing content is modified.

### Step 1: Add Statement Locator section to `spec/1_concepts.md`

**Objective:** Define the locator system as a language concept, immediately after Checkpoint.
**Confidence:** High
**Depends on:** None

**File:** `spec/1_concepts.md`

**Changes:** Append after line 394 (`@checkpoint *var1 *var2:integer *var3:string` + closing triple-backtick). The section follows the same pattern as Checkpoint: definition, rules, sub-sections for each variant, grammar, and examples.

```markdown
## Statement Locator

A statement locator is a string expression that resolves to one or more compiled statements, or a component within a single statement.

A locator has two layers:
1. **Locator** — identifies one statement or a contiguous range of statements
2. **Sub-address** — selects a component within a single located statement (not valid on ranges)

### Checkpoint+Offset Locator

Addresses a statement by position under a checkpoint.

`@checkpoint+N` — Nth statement under the checkpoint (1-indexed). Counts all statements regardless of verb type. Resolves to exactly one statement.

```
@opening
/show [id:"bg"] "cafe_night.png";          :: @opening+1
/converse [By:"Narrator"] "Line one.";      :: @opening+2
/converse "Line two.";                        :: @opening+3
*trust <- 0;                                  :: @opening+4 (sugar → /set)
====> @next;                                  :: @opening+5 (sugar → /jump)
```

Statement counting happens after preprocessing (sugar transformation, macro expansion). The offset addresses the compiled statement list, not source lines.

### Checkpoint+Offset Range Locator

Addresses a contiguous sequence of statements under a checkpoint.

`@checkpoint+N..M` — statements N through M inclusive (1-indexed, N ≤ M). Resolves to an ordered list of statements. Sub-addressing is not valid on ranges.

`@checkpoint+N..` — open-ended range, statements N through the last statement under the checkpoint.

```
@opening
/show [id:"bg"] "cafe_night.png";          :: ┐
/converse [By:"Narrator"] "Line one.";      :: ├─ @opening+1..3
/converse "Line two.";                        :: ┘
*trust <- 0;                                  :: @opening+4
====> @next;                                  :: @opening+5
```

If the range start exceeds the statement count, the range resolves to an empty list. If the end exceeds the statement count (or is open-ended), it clamps to the last statement.

### Named Locator

Addresses a statement by a user-assigned tag via the `[address]` attribute.

The tag name follows standard identifier rules (case-insensitive, no whitespace, no reserved characters). Tags must be unique within a story — duplicate `[address]` values produce a compile-time `duplicate_address_tag` error.

```
/converse [address:"flashback_echo"] "You're not from around here.";
```

Addressed as: `flashback_echo`

### Sub-Addressing

Once a single statement is located, a sub-address selects a component within it.

#### `#N` — Positional Parameter

Selects the Nth parameter (1-indexed). Counts all parameters regardless of type. `#1` is implied when no sub-address is given.

```
@opening+2#1      :: 1st param of 2nd statement at @opening
flashback_echo#1  :: 1st param of the tagged statement
```

#### `.name` — Named Parameter

Selects a named parameter by its name.

```
@first_approach+1.prompt   :: the prompt: named param
flashback_echo.timeout     :: the timeout: named param
```

#### `[attr#N]` — Attribute

Selects the Nth attribute of a given type (1-indexed). Attribute names are case-insensitive. Multiple attributes of the same type on one statement are indexed left-to-right.

```
@opening+2[By#1]     :: value of 1st [By] attribute
@opening+2[fade#1]   :: value of 1st [fade] attribute
```

### Grammar

```ebnf
locator_expr       := range_locator | single_locator
single_locator     := positional_locator | named_locator
positional_locator := '@' identifier '+' integer sub_address?
named_locator      := identifier sub_address?
range_locator      := '@' identifier '+' integer '..' integer?
sub_address        := positional_param | named_param | attr_sub
positional_param   := '#' integer
named_param        := '.' identifier
attr_sub           := '[' identifier '#' integer ']'
```

The trailing integer in a range locator is optional — when omitted (`@cp+N..`), the range extends to the last statement under the checkpoint.

### Resolution Rules

1. Checkpoint+offset locators resolve to exactly one statement. Sub-addressing selects within it.
2. Range locators resolve to an ordered list of statements. Sub-addressing is not valid on ranges.
3. Named locators resolve to exactly one statement. Sub-addressing selects within it.
4. If a locator resolves to nothing, the consumer decides behavior.
5. All locator types are deterministic.
```

**Rationale:** Placed after Checkpoint because locators extend the checkpoint concept with per-statement addressing. The section mirrors the Checkpoint section's structure: definition, naming rules, variants, examples.

**Verification:** Read `spec/1_concepts.md` from line 394 onward; confirm the new section follows Checkpoint without gaps or formatting breaks.

**If this fails:** Remove the appended content; `1_concepts.md` returns to its original state.

---

### Step 2: Add `[address]` attribute to `spec/std_attributes.md`

**Objective:** Define `[address]` as a standard attribute for statement tagging.
**Confidence:** High
**Depends on:** Step 1 (conceptually; the attribute definition references the locator system)

**File:** `spec/std_attributes.md`

**Changes:** Append after line 127 (end of `## Inline` section):

```markdown

## Address

Assigns a stable name to a statement for external reference via the statement locator system. Accept `"string"`. Value must be a string literal, not a variable reference. Must be unique within a story — duplicates produce a compile-time `duplicate_address_tag` error.

Used by: statement locator system, `/patch`.
```

**Rationale:** Follows the existing attribute format: `## Name`, description, type, constraints, behavior. Placed at the end alongside other metaprogramming attributes (Required, Resolve, Clone, Inline).

**Verification:** Read `spec/std_attributes.md` from line 125 onward; confirm `## Address` follows `## Inline` with consistent formatting.

**If this fails:** Remove the appended content; `std_attributes.md` returns to its original state.

---

## Verification Plan

### Manual Verification

- [ ] `spec/1_concepts.md` — Statement Locator section appears after Checkpoint, uses `##` for main heading and `###` for sub-sections
- [ ] Grammar in spec matches the EBNF in the proposal (including range_locator production)
- [ ] Resolution rules cover: single checkpoint+offset, range, named, error/empty cases
- [ ] Sub-addressing rules cover: `#N`, `.name`, `[attr#N]`, default `#1` behavior
- [ ] Range semantics: `N..M` inclusive, `N..` open-ended, clamping, empty-list on out-of-bounds
- [ ] `spec/std_attributes.md` — `[address]` entry follows attribute format conventions
- [ ] No existing content in either file is modified

### Acceptance Criteria Validation

| Criterion | How to Verify | Expected Result |
|-----------|---------------|-----------------|
| Locator concept in spec | Read `1_concepts.md` tail | `## Statement Locator` section with all subsections present |
| `[address]` in attributes | Read `std_attributes.md` tail | `## Address` section with type, constraints, uniqueness rule |
| Conventions followed | Compare heading levels and description style with neighbors | Consistent with `## Checkpoint` and `## Inline` |
| Grammar completeness | Compare EBNF against proposal | All productions match, including `range_locator` |

---

## Rollback Plan

Both changes are pure appends. Rollback = delete the appended lines from each file.

---

## Notes

### Risks

- **Proposal still Draft:** The locator system proposal hasn't been formally Accepted. Spec integration assumes consensus on the design. If the proposal is revised, the spec section must be updated.

### Open Questions

(None blocking execution — remaining open questions from the proposal are about future extensions, not the core spec content.)
