# Standard String Substitution Preprocessor (`#strsub`)

> **Status:** Draft
> **Created:** 2026-03-13
> **Author:** Agent
> **Related Projex:** 20260313-pattern-substitution-preprocessor-proposal.md, 20260312-locale-companion-file-explore.md, 20260312-script-level-localization-proposal.md, 20260313-embed-variable-locale-flag-proposal.md, 20260313-locator-preprocess-phase-explore.md, 20260313-runtime-scoped-flags-proposal.md

---

## Summary

A new `#strsub` preprocessor directive for compile-time string literal substitution. `#strsub` defines a source-to-replacement mapping that the preprocessor applies to all matching string literals in the story source. Companion files containing `#strsub` directives are delivered via `#embed` (or `#embed?` for optional delivery). This becomes the fourth standard preprocessor in the pipeline, alongside embed, macro, and sugar. Primary use case: localization. Generalizes to content variants, terminology adaptation, and any scenario requiring bulk string substitution before compilation.

---

## Problem Statement

### Current State

ZOH has no compile-time string substitution mechanism. The language has three standard preprocessors (embed, macro, sugar) and an extensible pipeline for custom ones. String literals in verb calls are baked into compiled story data as-written.

The `/patch` verb proposal (20260312-script-level-localization-proposal.md) addresses localization at runtime: `/patch "Hello", "Bonjour";` scans compiled data and replaces matching string values during execution. This is correct for dynamic and conditional patching, but the 95% localization case is static bulk replacement -- every run replaces the same strings with the same translations. That's a text transformation, and text transformations are what preprocessors do.

### Gap

No mechanism exists to substitute string literals at compile time based on an explicit mapping. Authors who want static string replacement must either:
- Use `/patch` at runtime (paying execution cost for a static transformation)
- Maintain separate script copies per language (duplication)
- Build custom preprocessors per runtime (no standard)

### Why Now

The localization design has matured through several explorations and proposals. The `/patch` verb handles the dynamic/structural case. The `#embed?` + `locale` flag delivery mechanism is proposed. The preprocessor pipeline is extensible. What's missing is the standard preprocessor that handles the static bulk case -- the simplest, most common operation in localization.

The locator-preprocess-phase exploration (20260313-locator-preprocess-phase-explore.md) explicitly left this door open: "If performance or static validation becomes a concern, a preprocessor that pre-resolves content matches can be added later without changing the language semantics."

---

## Proposed Change

### The `#strsub` Directive

```zoh
#strsub "source string" -> "replacement string";
```

A preprocessor directive that registers a string-to-string substitution. The preprocessor collects all `#strsub` directives, removes them from the source, then applies the registered substitutions to all matching string literals in the remaining source text.

**Syntax:**
- Starts with `#strsub` at the beginning of a line (leading whitespace allowed)
- Source string: a ZOH string literal (single or double quotes, multiline `"""`)
- `->` separator (reads: "becomes")
- Replacement string: a ZOH string literal
- Terminated with `;`
- Standard ZOH comments (`::`) allowed on the same line
- Multi-line form supported: `->` can start on its own line after the source string

```zoh
:: Single-line form
#strsub "The cafe closes at midnight." -> "Le cafe ferme a minuit.";
#strsub 'Sit across from her' -> "S'asseoir en face d'elle";

:: Multi-line form (natural for long strings)
#strsub "The cafe closes at midnight."
-> "Le cafe ferme a minuit.";

:: Multiline strings
#strsub """
She slides a napkin across the table.
A map, a time, and a single word: "basement".
"""
-> """
Elle glisse une serviette sur la table.
Une carte, une heure, et un seul mot: "sous-sol".
""";
```

### Pipeline Position

Priority 250: after embed (100) and macro (200), before sugar (300).

```
EmbedPreprocessor (100)  ->  MacroPreprocessor (200)  ->  SubPreprocessor (250)  ->  SugarPreprocessor (300)
```

**Why after embed:** `#strsub` directives arrive via `#embed` from companion files. They must be inlined before collection.

**Why after macro:** Macro expansion may produce string literals that need substitution. The preprocessor must see the fully expanded source to match correctly.

**Why before sugar:** Sugar transforms (`*var <- "text";`) produce verb calls (`/set *var, "text";`). Substitution should operate on the pre-sugar source where string literals are in their authored form.

### Processing Algorithm

```
SubPreprocessor.process(source):
    1. COLLECT: Scan source for all #strsub directives.
       Parse each into (sourceString, replacementString).
       Remove the directive lines from the source.

    2. VALIDATE:
       - Duplicate source keys -> compile error (ambiguous mapping)
       - Malformed directives -> compile error

    3. REPLACE: Walk the remaining source text.
       For each string literal encountered:
         - Normalize: resolve to canonical form (unescape, strip quotes)
         - Look up in the substitution map
         - If found: replace the entire string literal (including quotes) with the replacement (re-quoted in the original quote style)
         - If not found: leave unchanged

    4. REPORT: Emit diagnostics for #strsub entries that matched nothing
       (stale substitutions -- the source string doesn't exist in the script)

    return modified source
```

**Matching semantics:**
- Match is on **evaluated string content**, quote-style agnostic: `'Hello'` and `"Hello"` match the same `#strsub` key
- Escape sequences are normalized before matching: `"She said \"hello\""` matches by its content, not its raw form
- Multiline strings (`"""..."""`) match by content after standard indent-stripping

### Companion File Pattern

A companion file is a plain file containing `#strsub` directives:

```zoh
:: cafe.fr.zoh -- French translations for The Last Coffee Shop

#strsub "11:47 PM. The cafe closes at midnight."
-> "23h47. Le cafe ferme a minuit.";

#strsub "A woman in a red coat sits alone by the window."
-> "Une femme en manteau rouge est assise seule pres de la fenetre.";

#strsub "The rain picks up outside." -> "La pluie s'intensifie dehors.";
#strsub "Sit across from her" -> "S'asseoir en face d'elle";
#strsub "Order something first" -> "Commander quelque chose d'abord";
#strsub "Leave" -> "Partir";
#strsub "Narrator" -> "Narrateur";
```

Delivered via the variable-aware `#embed?` proposed in 20260313-embed-variable-locale-flag-proposal.md:

```zoh
The Last Coffee Shop
===

#embed? "${filename}.${locale}.zoh";

@opening
/converse [By: "Narrator"] "11:47 PM. The cafe closes at midnight.";
/converse "A woman in a red coat sits alone by the window.";
```

When `locale` is `"fr"`: resolves to `#embed? "cafe.fr.zoh"` -> inlines the `#strsub` directives -> SubPreprocessor applies them -> all matching string literals are replaced before compilation.

When `locale` is `""` or unset: `#embed?` resolves to an empty path or missing file -> silently skipped -> no substitutions -> base strings survive unchanged.

### What Gets Replaced

All string literals in the story body that exactly match a `#strsub` key, regardless of position. This includes:

- Verb parameters: `/converse "Hello";`
- Attribute values: `[By: "Narrator"]`
- Sugar forms: `*greeting <- "Hello";`
- Macro expansion results: strings produced by macro expansion that match a key
- Nested verb parameters: `/if *cond, /converse "Hello";;`

### What Doesn't Get Replaced

- **Preprocessor directive arguments:** `#embed "path.zoh"` -- the embed path is consumed at priority 100, before `#strsub` runs at 250. Even if it weren't, directive arguments are not story body strings.
- **`#strsub` directive strings:** The `#strsub` directives themselves are collected and removed before replacement begins. A `#strsub` source key does not match against other `#strsub` entries.
- **Expression bodies:** Strings inside backtick expressions (`` `$*"text"` ``) are code, not content. The preprocessor should not alter expression semantics. (Note: this requires the preprocessor to recognize backtick boundaries.)
- **Macro definitions:** Macro bodies are already expanded by priority 200. The definitions no longer exist in the source when `#strsub` runs.
- **Comments:** Comment text (`:: ...`, `::: ... :::`) is not a string literal.

### Interpolation Templates

Interpolation templates contain `${...}` placeholders. These are part of the source string key:

```zoh
#strsub "Hello, ${*name}. Welcome back." -> "Bonjour, ${*name}. Bon retour.";
```

The `${*name}` is literal text in the source -- the preprocessor matches and replaces the entire template string. The interpolation engine processes the result after compilation. Translators must preserve `${...}` placeholders in the replacement string; omitting one causes graceful degradation (the placeholder is absent, not a crash).

---

## Approach Options

### Option A: Blanket Replacement (Recommended)

Replace every string literal whose content matches a `#strsub` key, regardless of its syntactic position (verb parameter, attribute value, sugar form, etc.).

**Pros:**
- Simplest implementation: string-level scan, no verb or syntax awareness needed
- Most general: works for any string substitution, not just localization
- Author-controlled: the mapping is explicitly authored, so false positives are the author's responsibility
- Matches gettext philosophy: source text is the key, replacement is unconditional

**Cons:**
- Could replace strings in non-presentation positions (e.g., a `/set *role, "Narrator";` storing a string for logic, not display)
- Mitigation: authors control the mapping -- don't include strings you don't want replaced

### Option B: Verb-Context-Aware Replacement

Replace string literals only when they appear as parameters to known presentation verbs (`converse`, `choose`, `show`, etc.) or as values of known presentation attributes (`[By]`, `[prompt]`).

**Pros:**
- No false positives on non-presentation strings
- Safer for scripts that reuse strings in both display and logic positions

**Cons:**
- Requires the preprocessor to parse verb call structure (lightweight but non-trivial)
- Verb classification burden: which verbs are "presentation"? Requires a standard list + extensibility mechanism
- Less general: can't substitute strings in non-presentation positions (e.g., system messages, log strings)
- The earlier locale companion file exploration proposed this but the design evolved away from it toward simpler models

### Recommendation

**Option A.** The mapping is explicitly authored -- an author who puts `"Narrator"` in their `#strsub` file intends it to be replaced everywhere it appears. This is the gettext model, battle-tested across decades. The rare case where the same string serves both display and logic purposes is better handled by giving one instance a different string value, not by teaching the preprocessor about verb semantics.

For the 5% of cases needing positional precision, `/patch` with structural locators (`[locator: "checkpoint"]`, `[locator: "named"]`) is the right tool.

---

## Design Rationale

| Decision | Rationale |
|----------|-----------|
| Preprocessor directive, not verb | String substitution is a text transformation. Preprocessors transform text; verbs operate on compiled data. |
| `#strsub` naming | "string substitute" — explicit about what it matches (exact strings). Pairs with `#psub` (pattern substitute). The `str` prefix distinguishes exact-match from pattern-match. |
| `->` separator | Reads naturally as "becomes". Echoes ZOH's capture sugar (`-> *var;` = "store into"). Does not collide with any existing preprocessor syntax. Supports multi-line form where `->` starts its own line. |
| Priority 250 | After embed (directives arrive) and macro (strings finalized), before sugar (authored form). |
| Blanket replacement | Simplest correct approach. Author controls the mapping. Verb-awareness adds complexity for marginal safety. |
| Collect-then-replace (two-pass) | Directive position in source doesn't matter. Cleaner than order-dependent replacement. |
| Quote-style agnostic matching | `'text'` and `"text"` are equivalent in ZOH. The substitution map should respect this. |
| Compile-time stale-key diagnostic | Catches orphaned translations early. The base script changed but the companion file wasn't updated. |
| No regex support | Exact match only. Pattern-based replacement is `#psub`'s domain (see 20260313-pattern-substitution-preprocessor-proposal.md). |

### Why Not Just `/patch`?

| | `#strsub` (preprocessor) | `/patch` (runtime) |
|---|---|---|
| **Phase** | Compile-time | Execution-time |
| **Cost** | Zero runtime cost | O(N*M) scan per execution |
| **Validation** | Compile-time stale-key warning | Runtime diagnostic only |
| **Matching** | Exact string content | Content-match, checkpoint, named |
| **Conditional** | No | Yes (`/if *formal, /patch ...;`) |
| **Dynamic** | No | Yes (runtime locale switching) |
| **Structural** | No (content only) | Yes (sub-addressing: `#N`, `.name`, `[attr#N]`) |
| **Generality** | Any string literal | Compiled statement parameters |
| **Complexity** | Minimal | Locator system, compiled-data mutation |

They are complementary:
- `#strsub` handles the static bulk case (the 95% of localization)
- `/patch` handles dynamic, conditional, and structurally-addressed cases (the 5%)

A project can use either, both, or neither. Using both: `#strsub` for base translations, `/patch` for runtime overrides or context-specific adjustments.

---

## Impact Analysis

### Affected Areas

- **Spec: `1_concepts.md`** -- New `#strsub` directive in the Embed/Macro/Sugar section
- **Impl: `03_preprocessor.md`** -- SubPreprocessor implementation (priority 250)

### Dependencies

- **`#embed?` and variable interpolation** (20260313-embed-variable-locale-flag-proposal.md) -- delivery mechanism for companion files. `#strsub` works without it (manual `#embed` or inline directives), but `#embed?` enables the clean `"${filename}.${locale}.zoh"` pattern.
- **Runtime-scoped `locale` flag** (20260313-runtime-scoped-flags-proposal.md) -- provides the `${locale}` value for `#embed?` path resolution. Again, `#strsub` works without it, but the flag enables automatic locale file selection.

### Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| False-positive replacement (logic string replaced) | Low | Medium | Author controls the mapping; use distinct strings for logic vs display |
| Stale companion file after script edits | Medium | Low | Compile-time warning on unmatched `#strsub` keys |
| Performance on large mapping files | Low | Low | HashMap lookup is O(1) per string; thousands of entries are fine |
| Interaction with expression strings | Low | Medium | Preprocessor must recognize backtick boundaries to skip expressions |
| Ordering ambiguity with other priority-250 preprocessors | Low | Low | Standard preprocessors have reserved priorities; custom ones should avoid 250 |

### Breaking Changes

None. Fully additive. Existing scripts without `#strsub` directives behave identically. No syntax is repurposed.

---

## Examples

### Minimal Localization

**Base script (`cafe.zoh`):**
```zoh
The Last Coffee Shop
===

#embed? "${filename}.${locale}.zoh";

@opening
/converse [By: "Narrator"] "11:47 PM. The cafe closes at midnight.";
/converse "A woman in a red coat sits alone by the window.";

@first_approach
/choose prompt: "The rain picks up outside."
    true "Sit across from her" "sit"
    true "Leave" "leave"
; -> *choice;
```

**Companion (`cafe.fr.zoh`):**
```zoh
:: French translations
#strsub "Narrator" -> "Narrateur";
#strsub "11:47 PM. The cafe closes at midnight." -> "23h47. Le cafe ferme a minuit.";
#strsub "A woman in a red coat sits alone by the window."
-> "Une femme en manteau rouge est assise seule pres de la fenetre.";
#strsub "The rain picks up outside." -> "La pluie s'intensifie dehors.";
#strsub "Sit across from her" -> "S'asseoir en face d'elle";
#strsub "Leave" -> "Partir";
```

**After preprocessing (locale="fr"):**
```zoh
The Last Coffee Shop
===

@opening
/converse [By: "Narrateur"] "23h47. Le cafe ferme a minuit.";
/converse "Une femme en manteau rouge est assise seule pres de la fenetre.";

@first_approach
/choose prompt: "La pluie s'intensifie dehors."
    true "S'asseoir en face d'elle" "sit"
    true "Partir" "leave"
; -> *choice;
```

Note: `"sit"` and `"leave"` are choice identifiers (logic strings) and are NOT in the companion file, so they survive unchanged.

### Non-Localization Use: Content Variant

```zoh
My Story
===

#embed? "variants/${variant}.zoh";

@intro
/converse "Welcome to the standard edition.";
```

**`variants/deluxe.zoh`:**
```zoh
#strsub "Welcome to the standard edition."
-> "Welcome to the deluxe edition. Bonus content awaits.";
```

### Combined with `/patch`

Static translations via `#strsub`, runtime refinement via `/patch`:

```zoh
The Last Coffee Shop
===

:: Static bulk translations
#embed? "${filename}.${locale}.zoh";

:: Runtime contextual override (formal register for this scene)
/if *formal_context,
    /patch [locator: "named"] "greeting#1", "Veuillez vous asseoir, s'il vous plait.";
;

@lobby
/converse [address: "greeting"] "Please, have a seat.";
```

Here `#strsub` handles the base French translation. `/patch` conditionally overrides one specific string with a formal register variant at runtime.

---

## Open Questions

- [ ] Should `#strsub` support an optional key/tag for disambiguation (e.g., `#strsub [key:"flashback"] "source" -> "replacement";`)? Or is disambiguation exclusively `/patch`'s domain?
- [ ] Should unmatched `#strsub` keys produce a warning (recommended) or an error? Warnings allow incremental translation; errors enforce completeness.
- [ ] Should `#strsub` skip strings inside `#embed` path arguments? (Likely yes -- embed paths are consumed at priority 100, before `#strsub` runs. But if a future preprocessor at priority < 250 introduces path-like strings, the boundary matters.)
- [x] ~~Naming~~ Resolved: `#strsub` (string substitute) pairs with `#psub` (pattern substitute). See 20260313-pattern-substitution-preprocessor-proposal.md.
- [ ] Should `->` be the only separator, or should `=` be accepted as an alias? `->` is more ZOH-native; `=` is more familiar to translators coming from `.strings` / `.po` workflows.
- [ ] Should companion files support `#embed` themselves (for splitting large translation files into chapters)? The embed preprocessor runs first, so nested embeds in companion files would resolve naturally.
- [ ] Should `#strsub` support a `?` (optional) modifier like `#embed?`? e.g., `#strsub? "text" -> "other";` that silently no-ops if the source string isn't found. This would suppress the stale-key diagnostic for intentionally optional entries.

---

## Next Steps

If accepted:
1. Spec: Add `#strsub` directive to `1_concepts.md` alongside `#embed` and macro
2. Impl: Add SubPreprocessor to `03_preprocessor.md` at priority 250
3. Update `/patch` proposal to reference `#strsub` as the complementary static mechanism
4. Update `#embed?` proposal to reference `#strsub` as the primary consumer of locale companion files

---

## Appendix

### Prior Art

| System | Mechanism | Relation to `#strsub` |
|--------|-----------|-------------------|
| GNU gettext | `msgid`/`msgstr` in `.po` files, compiled to `.mo` | Same source-as-key philosophy. gettext operates at runtime; `#strsub` at compile time. |
| C/C++ `#define` | Preprocessor text substitution | `#strsub` is string-literal-aware (not raw text), avoiding macro pitfalls. |
| Ren'Py `translate` | Block-structured translation tied to labels | More tightly coupled to script structure. `#strsub` is position-independent. |
| Android `strings.xml` | Key-based string resources | Key indirection model. `#strsub` uses source-as-key (no IDs to manage). |
| iOS `.strings` / `.stringsdict` | `"key" = "value";` format | Similar file format. `#strsub` integrates as a preprocessor directive, not an external resource system. |

### Alternatives Considered

- **Preprocessor-only localization (20260312-locale-companion-file-explore.md):** Explored a verb-context-aware preprocessor with a dedicated companion file format (`"source" = "translation";` in a story body). This proposal simplifies: `#strsub` is a directive, blanket replacement, no verb-awareness needed. The companion file is just a list of `#strsub` directives.

- **Runtime-only via `/patch`:** Sufficient but suboptimal for the static case. `/patch` scans compiled data at execution time for every locale load. `#strsub` resolves at compile time, once.

- **Dedicated locale file format:** A story-like file with `locale:` metadata and `"source" = "translation";` body entries. Rejected: introduces new body syntax and a format that exists only for localization. `#strsub` directives are general-purpose and reuse the existing preprocessor directive model.

- **`=` separator instead of `->` :** Considered for familiarity with `.strings` / `.po` formats. `->` chosen because it echoes ZOH's capture sugar, reads naturally as "becomes", and avoids introducing a new operator character into preprocessor syntax.
