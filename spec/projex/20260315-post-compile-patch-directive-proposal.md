# Post-Compilation Patching via `#patch` Directive

> **Status:** Draft
> **Created:** 2026-03-15
> **Author:** Agent
> **Related Projex:** 20260312-script-level-localization-proposal.md, 20260313-statement-locator-system-proposal.md, 20260313-string-substitution-preprocessor-proposal.md, 20260315-strsub-filter-mechanism-proposal.md

---

## Summary

A new `#patch` directive applies structural modifications to compiled story data at build time. Unlike `#strsub` (pre-compile text substitution) or `/patch` (runtime verb), `#patch` operates post-compilation: directives are collected during preprocessing and applied to compiled data structures after the tokenizer/parser/compiler pipeline completes. This gives `#patch` the structural precision of `/patch` (locator-based addressing, typed values, sub-addressing) with zero runtime cost and compile-time validation. It fills the gap between blanket string substitution and dynamic runtime patching.

---

## Problem Statement

### Current State

The localization and story-modification design has converged on two mechanisms at opposite ends of the pipeline:

- **`#strsub` (pre-compile, proposed)** — Blanket string literal substitution in the preprocessor pipeline. Operates on source text. Content-match only. Handles the 95% static localization case.
- **`/patch` (runtime, proposed)** — Verb-based mutation of compiled story data during execution. Uses locators (checkpoint+offset, named) and sub-addresses (`#N`, `.name`, `[attr#N]`). Conditional, dynamic, structurally precise.

### Gap

Between pre-compile text substitution and runtime verb execution, there is no mechanism for structural, build-time patching.

| Scenario | `#strsub` | `/patch` | Need |
|----------|-----------|----------|------|
| Replace a specific parameter at `@opening+2#1` | No structural awareness | Works, but runtime cost | Build-time structural targeting |
| Patch attribute value `[By#1]` on a specific statement | Operates on text, not compiled structure | Works, but runtime cost | Build-time attribute patching |
| Replace a string that serves both display and logic, but only display | Blanket replacement hits both | Works with locators | Build-time precision |
| Set a numeric parameter per build variant | String-only | Works | Build-time typed values |
| Validate patches against compiled structure before execution | No compiled structure exists yet | Runtime-only validation | Compile-time validation |

These scenarios share a pattern: **static modifications that require compiled-structure awareness**. The modification is known at build time (not conditional on runtime state), but needs locator-based precision that source-text operations can't provide.

### Why Now

The locator system and `/patch` verb are in Draft. Adding `#patch` now ensures the locator grammar is shared across all three tiers from the start. The preprocessor pipeline is extensible, and the deferred-directive pattern that `#patch` introduces is a clean extension of existing concepts.

---

## Proposed Change

### The `#patch` Directive

```zoh
#patch target -> value;
```

A preprocessor directive that declares a post-compilation modification. The preprocessor collects and removes `#patch` directives from the source. After compilation, a patch-application phase resolves each target against the compiled story data and applies the replacement.

- **`target`** — a locator expression or a quoted string for content-match
- **`->`** — separator (consistent with `#strsub`)
- **`value`** — any ZOH literal (string, integer, double, boolean, `?`)
- Terminated with `;`
- Standard ZOH comments (`::`) allowed on the same line

### Locator Types

The target format determines locator type — no attribute declaration needed (unlike `/patch`'s `[locator]`):

**Content-match** — quoted string key, matches all compiled string parameters with this content:

```zoh
#patch "The cafe closes at midnight." -> "Le cafe ferme a minuit.";
#patch "Sit across from her" -> "S'asseoir en face d'elle";
```

**Checkpoint+offset** — `@checkpoint+N` with optional sub-address:

```zoh
#patch @opening+2#1 -> "Le cafe ferme a minuit.";
#patch @opening+1[By#1] -> "Narrateur";
#patch @first_approach+1.prompt -> "La pluie s'intensifie dehors.";
```

**Named** — `[address]` tag with optional sub-address:

```zoh
#patch flashback_echo#1 -> "Vous n'etiez pas d'ici...";
#patch greeting.prompt -> "Bienvenue.";
```

Sub-addressing follows the statement locator system spec:

| Sub-address | Targets | Example |
|-------------|---------|---------|
| `#N` | Nth parameter (any type) | `@opening+2#1` |
| `.name` | Named parameter by name | `@first_approach+1.prompt` |
| `[attr#N]` | Nth attribute value of type `attr` | `@opening+1[By#1]` |
| *(none)* | `#1` implied | `@opening+2` = `@opening+2#1` |

Ranges (`@checkpoint+N..M`) are excluded. Patching replaces a specific value at a specific target — ranges select multiple statements without sub-addressing and would need per-statement iteration. Individual `#patch` directives are clearer.

### Three-Tier Patching Model

`#patch` completes a three-tier model where each tier trades simplicity for capability:

```
Source Text
  | #embed (100)                    — assemble source
  | Macros (200)                    — expand templates
  | #strsub + #patch collect (250)  — PRE-COMPILE: substitute strings, collect patches
  | Sugar (300)                     — desugar syntax
  | Tokenizer -> Parser -> Compiler -> Validator
  |
  v COMPILED STORY DATA
  | #patch application              — POST-COMPILE: structural patching
  |
  v PATCHED COMPILED DATA
  | /patch                          — RUNTIME: dynamic patching (during execution)
```

| | `#strsub` | `#patch` | `/patch` |
|---|---|---|---|
| **Phase** | Pre-compile | Post-compile | Runtime |
| **Operates on** | Source text | Compiled data | Compiled data |
| **Matching** | String content (blanket) | Locators + content | Locators + content |
| **Structural awareness** | None | Full (params, attrs, verbs) | Full |
| **Value types** | String -> String | Any -> Any | Any -> Any |
| **Conditional** | No | No | Yes |
| **Dynamic** | No | No | Yes |
| **Validation** | Stale-key warning | Compile-time error on bad locator | Runtime diagnostic |
| **Runtime cost** | Zero | Zero | Per-execution |
| **Use case** | Bulk string replacement | Structural precision, build-time | Dynamic/conditional |

#### Pre-Compile Phase (`#strsub`)

Operates on source text before tokenization. Blanket string replacement — every matching string literal, regardless of syntactic position. No structural awareness: can't distinguish a display string from a logic string with the same content. Cheapest and simplest tier.

**Strengths:** Zero complexity, zero cost, self-documenting (source text in, translated text out). Handles the common case where strings are unique and replacement is unconditional.

**Limitations:** String-only values. No positional targeting. No attribute targeting. Can produce false positives when the same string serves both display and logic purposes.

#### Post-Compile Phase (`#patch`)

Operates on compiled data structures after the full compilation pipeline. Has access to the statement tree: parameter types, attribute values, checkpoint indices, named tags. Uses locators for precision. Validated at build time — a bad locator (`@nonexistent+99`) fails the build, not the runtime. Zero runtime cost since patches are baked into the compiled output.

**Strengths:** Structural precision without runtime cost. Typed values (patch a number, not just strings). Compile-time validation. Can target specific parameters/attributes on specific statements.

**Limitations:** Static — can't respond to runtime state. Requires the author to know (or look up) locator addresses.

#### Runtime Phase (`/patch`)

Operates on compiled data during verb execution. Same addressing as `#patch`, plus conditional execution (`/if *formal, /patch ...;`), dynamic values (from variables/expressions), and responsiveness to runtime state. Pays a cost per execution.

**Strengths:** Full dynamic capability. Can be conditional, computed, responsive to player state.

**Limitations:** Runtime cost. Late validation (errors surface during execution). Patches pollute the execution trace with setup work for what may be static modifications.

#### Tier Interaction

The tiers compose cleanly:

1. `#strsub` modifies source text -> compilation produces data reflecting those substitutions
2. `#patch` modifies compiled data -> runtime sees the patched result as the "original"
3. `/patch` modifies compiled data at runtime -> overrides anything from earlier tiers

A `#patch` content-match sees the **post-`#strsub`** compiled string. If `#strsub` changed `"Hello"` to `"Bonjour"`, a `#patch "Hello" -> ...` won't match — the compiled data contains `"Bonjour"`.

### Deferred Directive Pattern

`#patch` introduces a new pattern: **deferred directives**. Unlike standard directives (`#embed`, `#strsub`) that transform source text immediately, `#patch` is collected during preprocessing but applied after compilation.

**Collection phase** (preprocessor, priority 250):
1. Scan source for `#patch` directives
2. Parse each: extract target (locator expression or content key) and value
3. Store in a `PatchSet` attached to the story's compilation context
4. Remove directive lines from source

**Application phase** (post-compilation):
1. Compilation produces the story data structure
2. Iterate the `PatchSet` in source order
3. Resolve each target against the compiled data
4. Apply replacements
5. Validate: unresolvable locators produce compile-time errors; unmatched content keys produce warnings

### Content-Match vs `#strsub`

Both support string-key matching. The difference is phase and precision:

| | `#strsub` | `#patch` content-match |
|---|---|---|
| Phase | Pre-compile (source text) | Post-compile (compiled data) |
| Scope | All string literals in source | Compiled statement parameters only |
| False positives | Can replace logic strings | Only touches compiled parameters |

For most localization, `#strsub` is sufficient and simpler. `#patch` content-match is useful when a string appears in both display and logic positions — `#strsub` replaces both; `#patch` content-match replaces only compiled parameters.

### Override Order

`#patch` directives apply in source order (top to bottom, including across `#embed`-ed files). Later directives targeting the same element overwrite earlier ones.

```zoh
#patch "Continue" -> "Continuer";                :: all instances
#patch @ending+3#1 -> "Continuer l'histoire";    :: refines one instance
```

### Example

**Base script (`cafe.zoh`):**
```zoh
The Last Coffee Shop
===

#embed? "${filename}.${locale}.zoh";
#embed? "${filename}.${locale}.patch.zoh";

@opening
/show [id: "bg"] "cafe_night.png";
/converse [By: "Narrator"] "11:47 PM. The cafe closes at midnight.";
/converse "A woman in a red coat sits alone by the window.";
====> @first_approach;

@first_approach
/choose prompt: "The rain picks up outside."
    true "Sit across from her" "sit"
    true "Leave" "leave"
; -> *choice;
```

**Pre-compile substitutions (`cafe.fr.zoh`):**
```zoh
:: Bulk translations — #strsub (pre-compile, blanket)
#strsub "11:47 PM. The cafe closes at midnight." -> "23h47. Le cafe ferme a minuit.";
#strsub "A woman in a red coat sits alone by the window."
-> "Une femme en manteau rouge est assise seule pres de la fenetre.";
#strsub "The rain picks up outside." -> "La pluie s'intensifie dehors.";
#strsub "Sit across from her" -> "S'asseoir en face d'elle";
#strsub "Leave" -> "Partir";
```

**Post-compile patches (`cafe.fr.patch.zoh`):**
```zoh
:: Structural patches — #patch (post-compile, locator-based)

:: Attribute value: "Narrator" also appears as a logic string elsewhere,
:: so #strsub would cause false positives. #patch targets only this attribute.
#patch @opening+2[By#1] -> "Narrateur";

:: Difficulty tuning — non-string value, impossible with #strsub
#patch @combat+1#2 -> 50;

:: Accessibility override — attribute value
#patch @opening+1[fade#1] -> 3.0;
```

### Generalization Beyond Localization

Like `/patch`, `#patch` generalizes to any build-time story modification:

```zoh
:: Difficulty variant (delivered via #embed? "variants/${variant}.patch.zoh")
#patch @combat+1#2 -> 50;
#patch @combat+1#3 -> 10;

:: Accessibility variant
#patch @opening+1[fade#1] -> 3.0;
#patch @intro+1[speed#1] -> 0.5;

:: Content variant (director's cut)
#patch @ending+1#1 -> "The director's cut ending text.";
```

---

## Design Rationale

| Decision | Rationale |
|----------|-----------|
| `#` sigil (directive, not verb) | Signals build-time. Same convention as `#embed`, `#strsub`. |
| `->` separator | Consistent with `#strsub`. Reads as "becomes." |
| Deferred application (post-compile) | Locators reference compiled structure — can't resolve against source text. |
| Locator type inferred from format | `@` prefix = checkpoint, quoted string = content-match, bare identifier = named. No `[locator]` attribute needed — directives don't carry attributes. |
| No range support | Patching replaces a specific value at a specific target. Ranges select multiple statements without sub-addressing. Use individual `#patch` directives. |
| Content-match included | Mirrors `/patch` semantics. Compiled-data aware (unlike `#strsub`). Optional — authors choose based on need. |
| Compile-time validation | Bad locators fail the build. Key advantage over `/patch`. |
| Source-order application | Predictable, matches `#strsub` behavior. Later directives refine earlier ones. |

### Why Not Just `/patch`?

`/patch` executes during story playback. For static modifications known at build time:
- Every story load pays the patching cost
- Errors surface only at runtime
- Patches pollute the execution trace with setup work

`#patch` resolves all modifications during the build. The runtime sees a clean, pre-patched story.

### Why Not Just `#strsub`?

`#strsub` operates on source text and can only match/replace string content. It cannot:
- Target a specific parameter at a specific structural position
- Replace non-string values (numbers, booleans)
- Patch attribute values with positional precision
- Distinguish between a display string and a logic string with the same content

---

## Impact Analysis

### Affected Areas
- **Spec: `1_concepts.md`** — `#patch` directive definition, deferred directive concept
- **Impl: `03_preprocessor.md`** — `#patch` collection in SubPreprocessor (priority 250)
- **Impl: new section** — Post-compilation patch application phase (`PatchApplier`)

### Dependencies
- **Statement locator system** (20260313-statement-locator-system-proposal.md) — `#patch` is a consumer
- **`#embed?` and variable interpolation** — delivery mechanism for patch companion files
- **Runtime-scoped flags** — provides `${locale}`, `${variant}` for `#embed?` path resolution
- **Compiled story data** must expose mutable access to parameters and attribute values

### Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Deferred directive adds pipeline complexity | Low | Medium | Minimal pattern: collect -> store -> apply. Single new phase. |
| Content-match overlap with `#strsub` | Medium | Low | Clear guidance: `#strsub` for bulk, `#patch` for precision |
| Checkpoint offset fragility | Medium | Medium | Named locators for stability (same as `/patch`) |
| `#strsub` + `#patch` interaction edge cases | Low | Medium | Clear ordering: `#strsub` modifies source, `#patch` sees compiled result |
| Locator syntax ambiguity in directive context | Low | Low | `@` prefix, quoted strings, and bare identifiers are unambiguous |

### Breaking Changes
None. Fully additive.

---

## Open Questions

- [ ] Should `#patch` content-match be excluded entirely, keeping content-matching in `#strsub` and locator-based matching in `#patch`? Eliminates overlap at the cost of forcing locator use for all `#patch` operations.
- [ ] Should the deferred directive pattern be formalized as a general concept (allowing future `#` directives to defer), or kept as a `#patch`-specific implementation detail?
- [ ] Should `#patch` support a `?` modifier (`#patch?`) that warns instead of errors on unresolvable locators? Useful for patch files that target optional content.
- [ ] Should the post-compile phase validate type compatibility (e.g., patching a string parameter with a number)?
- [ ] Naming: is `#patch` clear enough alongside `/patch`, or does the sigil difference (`#` vs `/`) sufficiently disambiguate?
- [ ] Should `#patch` support a reset-to-original mechanism, or is that exclusively `/patch`'s domain (since `#patch` modifications are baked into the compiled output)?

---

## Next Steps

If accepted:
1. Spec: Define `#patch` directive in `1_concepts.md` alongside `#embed` and `#strsub`
2. Spec: Define post-compilation patch application phase
3. Impl: Extend SubPreprocessor to collect `#patch` directives at priority 250
4. Impl: Define `PatchApplier` post-compilation phase
5. Update `/patch` and `#strsub` proposals to reference `#patch` as complementary mechanism

---

## Appendix

### Tier Selection Guide

| Need | Use |
|------|-----|
| Replace all occurrences of a string | `#strsub` |
| Replace a string at a specific structural location | `#patch` |
| Replace a non-string value at a specific location | `#patch` |
| Conditionally replace based on runtime state | `/patch` |
| Validate patches before the story runs | `#patch` |
| Maximize runtime performance | `#strsub` or `#patch` |

### Prior Art

| System | Mechanism | Relation to `#patch` |
|--------|-----------|---------------------|
| Game modding (xdelta, BPS) | Binary patching by offset | `#patch` is the typed, structural equivalent |
| Unity Addressables | Pre-build asset remapping | Similar phase: resolved during build, not at runtime |
| Ren'Py translate | Label-scoped positional matching | `#patch` checkpoint+offset is structurally equivalent |
| Webpack DefinePlugin | Compile-time constant replacement | Similar: collected from config, applied during build |

### Alternatives Considered

- **`/patch` only (runtime):** Sufficient but suboptimal for static modifications. Pays runtime cost, validates late.
- **`#strsub` only (pre-compile):** Sufficient for string replacement but can't target structure, attributes, or non-string values.
- **Post-compile verb execution phase:** Run `/patch` verbs in a special pre-execution pass. Rejected: conflates build-time and runtime concepts. The `#` sigil correctly signals "this is not executed as a verb."
- **Compiler plugin/hook:** Allow custom post-compile passes via API. More general but less standardized. `#patch` provides a spec-level mechanism.
