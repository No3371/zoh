# Statement Locator System

> **Status:** Draft
> **Created:** 2026-03-13
> **Author:** Agent
> **Related Projex:** 20260312-script-level-localization-proposal.md, 20260313-verb-driven-locale-hybrid-explore.md, 20260313-locator-preprocess-phase-explore.md

---

## Summary

Define a standardized **statement locator** system as a first-class concept in the ZOH spec. A locator is a structural address that resolves to exactly one compiled statement (or a component within one). Two locator types — checkpoint+offset and named tag — combined with sub-addressing (`#N` for positional parameters, `.name` for named parameters, `[attr#N]` for attributes) form a reusable grammar for any feature that needs to reference compiled story elements. `/patch` is the first consumer; debugging, validation, and tooling are natural extensions. Source-string content matching is explicitly excluded — it is a query, not an address, and belongs to consumers like `/patch` as a verb-specific behavior (see 20260313-locator-preprocess-phase-explore.md).

---

## Problem Statement

### Current State

ZOH has no way to reference a specific compiled statement from outside its execution context. Checkpoints name entry points for control flow (`/jump`, `/fork`, `/call`), but there is no addressing system for individual statements within a checkpoint, nor for specific parameters or attributes on a statement.

### Gap

The `/patch` verb proposal requires a way to locate statements and select targets within them. Designing this as a `/patch` implementation detail buries a general-purpose concept inside one verb. Other features need the same capability:
- **Debugging:** "show me what's at `@opening+3`"
- **Validation:** "assert the string at `@intro+1.1` is not empty"
- **Tooling:** "extract all localizable strings with their locator addresses"
- **Story diffing:** "what changed at `@ending+2[fade#1]` between versions?"

### Why Now?

The `/patch` proposal is in Draft. Defining the locator system independently ensures `/patch` consumes a clean, reusable spec rather than inventing ad-hoc addressing. Other consumers can follow without redesigning the grammar.

---

## Proposed Change

### Overview

A **statement locator** is a string expression that resolves to exactly one compiled statement or a component within one. It has two layers:

1. **Locator** — identifies a statement
2. **Sub-address** — selects a specific component within the located statement

### Locator Types

#### Checkpoint+Offset Locator

Addresses a specific statement by position under a checkpoint.

```
@checkpoint+N
```

- `@checkpoint` — checkpoint name (case-insensitive, standard identifier rules)
- `+N` — Nth statement under the checkpoint (1-indexed)
- Counts **all statements** regardless of verb type
- Resolves to exactly one statement

```zoh
@opening
/show [id:"bg"] "cafe_night.png";          :: @opening+1
/converse [By:"Narrator"] "Line one.";      :: @opening+2
/converse "Line two.";                        :: @opening+3
*trust <- 0;                                  :: @opening+4 (sugar → /set)
====> @next;                                  :: @opening+5 (sugar → /jump)
```

Statement counting happens after preprocessing (sugar transformation, macro expansion). The offset addresses the compiled statement list, not source lines.

#### Named Locator

Addresses a statement by a user-assigned tag via the `[address]` attribute.

```
tag_name
```

- Resolves to the statement carrying `[address: "tag_name"]` in the base script
- Tag names follow standard identifier rules (case-insensitive, no whitespace, no reserved chars)
- Tags must be unique within a story — duplicate `[address]` values produce a compile-time `duplicate_address_tag` error

```zoh
/converse [address:"flashback_echo"] "You're not from around here.";
```

Addressed as: `flashback_echo`

### Sub-Addressing

Once a statement is located, sub-addresses select a component within it.

#### `#N` — Positional Parameter Sub-Address

Selects the Nth parameter on the statement (1-indexed). Counts **all parameters** regardless of type.

```
@opening+2#1      :: 1st param of 2nd statement at @opening
@opening+2#3      :: 3rd param
flashback_echo#1  :: 1st param of the tagged statement
```

```zoh
/choose/
    prompt:"The rain picks up."
    true "Sit across from her" "sit"
    true "Leave" "leave"
/;
:: #1 = "The rain picks up."  (named: prompt)
:: #2 = true
:: #3 = "Sit across from her"
:: #4 = "sit"
:: #5 = true
:: #6 = "Leave"
:: #7 = "leave"
```

The locator addresses compiled structure honestly — all parameters, all types. Consumers (like `/patch`) decide which targets they're willing to act on.

**Default:** `#1` is implied when no sub-address is given. `@opening+2` ≡ `@opening+2#1`.

#### `.name` — Named Parameter Sub-Address

Selects a named parameter by its name.

```
@first_approach+1.prompt   :: the "prompt:" named param on 1st statement at @first_approach
flashback_echo.timeout     :: the "timeout:" named param on the tagged statement
```

Cleaner than positional indexing for verbs with many parameters where order isn't obvious.

#### `[attr#N]` — Attribute Sub-Address

Selects the Nth attribute of a given type on the statement (1-indexed).

```
@opening+2[By#1]     :: value of 1st [By] attribute on 2nd statement at @opening
@opening+2[fade#1]   :: value of 1st [fade] attribute
flashback_echo[By#1] :: value of 1st [By] attribute on tagged statement
```

Attribute names are case-insensitive (standard ZOH rules). Multiple attributes of the same type on one statement are indexed left-to-right.

### Grammar

```ebnf
locator_expr       := positional_locator | named_locator
positional_locator := '@' identifier '+' integer sub_address?
named_locator      := identifier sub_address?
sub_address        := positional_param | named_param | attr_sub
positional_param   := '#' integer
named_param        := '.' identifier
attr_sub           := '[' identifier '#' integer ']'
```

### Resolution Rules

1. **Checkpoint+offset locators** resolve to exactly one statement. Sub-addressing selects within it.
2. **Named locators** resolve to exactly one statement. Sub-addressing selects within it.
3. If a locator resolves to nothing, the consumer decides behavior (e.g., `/patch` emits a diagnostic).
4. Both locator types are deterministic — they point to exactly one slot. Content-based matching (source-string search) is not a locator; it is a consumer-specific behavior (e.g., `/patch`'s default content-match mode).

### The `[address]` Attribute

The `[address: "tag"]` attribute is a **standard attribute** that assigns a stable name to a statement for external reference. It is consumed by the locator system and has no effect on verb execution.

- Defined in `std_attributes.md`
- Any verb call can carry it
- Value must be a string literal (not a variable reference)
- Must be unique within a story

**Naming note:** `[address]` reflects the primary use case (localization tagging). If the locator system is adopted for broader purposes, the attribute could be aliased or renamed (e.g., `[tag]`, `[id]`). This is an open question.

---

## Design Rationale

| Decision | Rationale |
|----------|-----------|
| Count all statements for offset | Stable regardless of verb classification; adding a `/set` before a `/converse` shifts offsets predictably |
| `#N` counts all params (any type) | Locator addresses structure honestly; consumers decide what they'll act on |
| No source-string locator type | Source-string matching is a content query (zero-or-more, no sub-addressing), not a structural address. It belongs to consumers like `/patch` as a verb-specific behavior, not the locator system. See 20260313-locator-preprocess-phase-explore.md. |
| Named tags via attribute, not syntax | Fits ZOH's attribute model; no new language syntax needed |
| Tags unique per story | Prevents ambiguity; named locators must resolve to exactly one statement |
| Locator is a string expression | Can be passed as parameters, stored in variables, composed dynamically |

---

## Impact Analysis

### Affected Areas
- **Spec: `1_concepts.md`** — New "Statement Locator" concept section
- **Spec: `std_attributes.md`** — `[address]` attribute definition
- **Spec: `2_verbs.md`** — `/patch` verb references locator system (per companion proposal)
- **Impl spec** — Locator resolution algorithm, compiled-story indexing

### Dependencies
- Compiled story data structure must support indexed access by checkpoint+offset
- Verb drivers may declare parameter semantics (for consumer filtering, not for locator resolution)
- `[address]` uniqueness enforced at compile time

### Consumers

| Consumer | How it uses locators |
|----------|---------------------|
| `/patch` | Locate targets to modify (per companion proposal) |
| Diagnostics | "Error at `@opening+3`" — structured error locations |
| Debugging/inspection | `/inspect @opening+2` — examine compiled statement |
| Tooling | Extract locator addresses for string catalogs, diffing |
| Validation | Assert properties of specific compiled elements |

### Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Offset fragility on base script edits | Medium | Medium | Named locators (`[address]`) for stability; offsets for precision |
| Verb driver must declare localizable slots | Low | Low | Standard verbs ship with declarations; custom verbs default to "all string params" |
| `[address]` tag maintenance burden | Low | Low | Tags are optional; only needed for named locator use |

### Breaking Changes
None. Fully additive — no existing behavior is modified.

---

## Open Questions

- [ ] Should `[address]` be renamed to something broader (e.g., `[tag]`, `[id]`) given the system is not localization-specific?
- [ ] Should checkpoint+offset support ranges? (`@opening+2..5` for statements 2 through 5)
- [ ] Should locator expressions be a distinct type in the type system, or always represented as strings?

---

## Next Steps

If accepted:
1. Spec: Add "Statement Locator" section to `1_concepts.md` with grammar and resolution rules
2. Spec: Add `[address]` to `std_attributes.md`
3. Update `/patch` proposal to reference locator system spec instead of defining its own addressing
4. Impl: Define locator resolution algorithm and compiled-story index structures
