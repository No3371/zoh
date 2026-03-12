# Presentation Verb Localization Standard

> **Status:** Draft
> **Created:** 2026-03-13
> **Author:** Agent
> **Related Projex:** 20260312-locale-companion-file-explore.md, 20260312-script-level-localization-proposal.md, 20260313-string-substitution-preprocessor-proposal.md, 20260313-embed-variable-locale-flag-proposal.md

---

## Summary

Standardize which verb parameters and attributes are **localizable** across ZOH's presentation verbs, define the verb driver contract for locale-aware string lookup, and establish `[l10n]` as an attribute for disambiguating identical source strings. No new language features — this proposal codifies conventions that make ZOH stories localizable using the existing `locale` flag, verb driver architecture, and `#strsub` companion files.

---

## Problem Statement

### Current State

ZOH's verb-based architecture already provides the interception point that other narrative scripting languages (e.g., Ink) lack: every user-facing string passes through a verb driver call. A host can already intercept `/converse` and substitute translated text. The `locale` standard flag exists. The `#strsub` preprocessor is proposed.

### Gap

There is no standard answer to:
- **Which parameters are localizable?** A host implementing l10n must reverse-engineer each verb's spec to determine which string parameters carry user-facing text vs. internal identifiers.
- **How does a driver disambiguate identical source strings?** Two `/converse "Yes";` calls in different scenes are distinct translation units but have the same source key.
- **What is the extraction contract?** Tooling that extracts localizable strings needs a declared list of localizable slots per verb.

### Why Now?

The `locale` flag, `#strsub`, and `#embed?` proposals are in draft. Defining the convention layer now ensures these mechanisms compose into a coherent l10n story rather than leaving each host to invent its own rules.

---

## Proposed Change

### Overview

Three additions, all additive:

1. **Localizable slot declarations** on standard presentation verbs — spec metadata marking which parameters and attributes carry user-facing text.
2. **`[l10n]` standard attribute** — optional disambiguation tag for translation units with identical source strings.
3. **Verb driver l10n contract** — standard behavior for locale-aware string resolution.

### 1. Localizable Slot Declarations

Each standard verb that presents text to users declares which of its parameters and attributes are **localizable**. A localizable slot is one whose string value is intended for human display and should be substituted when a locale is active.

#### Presentation Verbs and Their Localizable Slots

| Verb | Localizable Parameters | Localizable Attributes | Notes |
|------|----------------------|----------------------|-------|
| `/converse` | `content` (all instances) | `[By]`, `[Portrait]` | Core dialogue verb. `[Portrait]` localizable for region-specific art. |
| `/choose` | `prompt`, `text` (all instances) | `[By]`, `[Portrait]` | Choice values are NOT localizable (internal identifiers). |
| `/chooseFrom` | `prompt` | `[By]`, `[Portrait]` | Choice texts are in the list data — host resolves those separately. |
| `/prompt` | `prompt` | — | Text input prompt. |
| `/show` | `resource` | — | Resource path — localizable for region-specific images (e.g., signs with text). |
| `/play` | `resource` | — | Resource path — localizable for region-specific audio (e.g., voiced lines). |

**Non-localizable presentation attributes:** `[Style]`, `[Wait]`, `[Fade]`, `[Easing]`, `[Anchor]`, positions, dimensions — these control rendering, not content.

#### Declaration Format in Spec

Each verb's spec section gains a `### Localizable` subsection listing the slots:

```markdown
### Localizable
- Parameters: `content`
- Attributes: `[By]`, `[Portrait]`
```

This is spec documentation, not runtime data. Extraction tooling reads these declarations. Custom verbs can declare their own localizable slots via the same convention.

### 2. The `[l10n]` Attribute

A standard attribute that provides a disambiguation context for translation units sharing the same source string.

```zoh
/converse [l10n:"menu_confirm"] "Yes";
/converse [l10n:"battle_confirm"] "Yes";
```

- Defined in `std_attributes.md`
- Any verb call can carry it
- Value must be a string literal
- No uniqueness requirement (unlike `[address]`) — multiple calls can share the same `[l10n]` value if they should share a translation
- When present, the translation key becomes the composite `(source_string, l10n_context)` rather than `source_string` alone

**When is `[l10n]` needed?** Only when the same source string appears in contexts where different translations are appropriate. In most stories this is rare. The attribute is fully optional — absence means the source string alone is the key.

### 3. Verb Driver L10n Contract

When the `locale` flag is set to a non-empty value, a locale-aware verb driver **should**:

1. For each localizable parameter/attribute on the current verb call:
   a. Read the source string value
   b. If the verb call carries `[l10n]`, form the lookup key as `(source_string, l10n_value)`; otherwise use `source_string` alone
   c. Look up the key in the host's string table for the active locale
   d. If a translation is found, use it in place of the source string
   e. If no translation is found, fall through to the source string (source-as-key model)
2. Proceed with normal verb execution using the resolved strings

The string table format and storage are host-defined. The `#strsub` companion file format (see 20260313-string-substitution-preprocessor-proposal.md) is the recommended interchange format:

```zoh
:: fr.l10n.zoh — French translations for story
#strsub "Hello, stranger." -> "Bonjour, inconnu.";
#strsub "Yes" [l10n:"menu_confirm"] -> "Oui";
#strsub "Yes" [l10n:"battle_confirm"] -> "Ouais";
```

The `[l10n:"..."]` annotation in `#strsub` directives mirrors the `[l10n]` attribute on verb calls, allowing extraction tools to round-trip disambiguation context.

### Extraction Workflow

1. **Extract**: Tooling scans compiled story, reads localizable slot declarations per verb, emits `#strsub` directives with source strings (and `[l10n]` context where present)
2. **Translate**: Translators fill in the `->` targets in the `#strsub` file (or convert to/from PO, XLIFF, etc.)
3. **Apply**: Either:
   - **Compile-time**: `#embed` the `#strsub` file — preprocessor substitutes at compile time (zero runtime cost, one compiled story per locale)
   - **Runtime**: Host loads the string table and verb drivers resolve at call time (single compiled story, dynamic locale switching)

Both paths use the same source format. The choice is a host deployment decision, not a language concern.

---

## Design Rationale

| Decision | Rationale |
|----------|-----------|
| Convention layer, not new syntax | ZOH already has the architecture (verb drivers + flags). The gap is standardization, not mechanism. |
| Source-as-key model | Matches gettext philosophy. Scripts remain readable in the source language. No opaque string IDs in story files. |
| `[l10n]` as attribute, not parameter | Attributes are optional metadata — perfect fit. Does not change verb signatures. |
| No uniqueness constraint on `[l10n]` | Unlike `[address]`, multiple calls may intentionally share a translation context. |
| Resource paths as localizable | Region-specific art/audio is a real l10n need (signs, voice acting). Hosts that don't need it simply don't translate those keys. |
| `#strsub` as interchange format | Dual-purpose: both a working preprocessor directive AND a string catalog format. No new tooling format to invent. |
| `/chooseFrom` list texts excluded from declaration | List data is constructed programmatically — the host must localize at the data level, not the verb call level. |

---

## Impact Analysis

### Affected Areas
- **Spec: `std_verbs.md`** — Add `### Localizable` subsection to each presentation verb
- **Spec: `std_attributes.md`** — Add `[l10n]` attribute definition
- **Spec: `std_flags.md`** — No changes (locale flag already defined)
- **Tooling (future)** — String extraction tool reads localizable declarations

### Dependencies
- `locale` standard flag (spec'd in `std_flags.md`)
- `#strsub` preprocessor (proposed in 20260313-string-substitution-preprocessor-proposal.md) — for interchange format; not required for runtime l10n

### Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Hosts ignore the convention | Medium | Low | Convention is lightweight; even partial adoption improves interop |
| `[l10n]` proliferation clutters scripts | Low | Low | Only needed for disambiguation; most strings are unique in context |
| Custom verbs lack localizable declarations | Medium | Low | Default: all string params on custom verbs are localizable (safe default) |
| Expression/interpolated strings resist extraction | Medium | Medium | Tooling extracts the template; runtime interpolation happens after lookup |

### Breaking Changes

None. Fully additive.

---

## Open Questions

- [ ] Should `#strsub` support the `[l10n:"..."]` context annotation syntax, or should disambiguation be handled purely by the host's string table format?
- [ ] Should the spec recommend a file naming convention for l10n companion files (e.g., `{story}.{locale}.l10n.zoh`)?
- [ ] Should `/chooseFrom` list-based texts have a recommended localization pattern (e.g., a helper verb or convention for localizable list construction)?

---

## Next Steps

If accepted:
1. Add `### Localizable` subsections to presentation verbs in `std_verbs.md`
2. Add `[l10n]` attribute to `std_attributes.md`
3. Update `#strsub` proposal to include `[l10n]` context annotation syntax
4. Define extraction algorithm for tooling (future plan)
