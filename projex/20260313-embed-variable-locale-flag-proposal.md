# Variable-Aware Embedding and `locale` Standard Flag

> **Status:** Draft
> **Created:** 2026-03-13
> **Author:** Agent
> **Related Projex:** 20260312-script-level-localization-proposal.md, 20260313-statement-locator-system-proposal.md, 20260313-locator-preprocess-phase-explore.md, 20260313-runtime-scoped-flags-proposal.md, 20260313-embed-variable-interpolation-plan.md

---

## Summary

Extend `#embed` to support variable interpolation in file paths, sourcing values from built-in preprocessor variables, runtime-scoped flags, and story metadata. Introduce `#embed?` as an optional embed form. Introduce `locale` as a standard flag set at runtime scope. The runtime-scoped flag system (per companion proposal) bridges the phase gap — runtime flags are available to the preprocessor, eliminating the need for a separate "preprocessor variable" concept.

---

## Problem Statement

### Current State

`#embed` accepts only static file paths:

```zoh
#embed "path/to/file.zoh";
```

There is no way to parameterize the path based on runtime configuration, story metadata, or flags. The preprocessor has access to the story name and metadata entries, but `#embed` can't reference them.

Separately, the `/patch` proposal needs a delivery mechanism for locale companion files. The earlier exploration proposed a custom "silent embed" preprocessor — a one-off mechanism just for locale files.

### Gap

Two gaps converge:
1. `#embed` is static — can't adapt to context (locale, platform, variant)
2. Locale companion file delivery requires custom machinery that could be general-purpose

### Why Now?

The `/patch` proposal is in Draft and needs a delivery mechanism. Rather than building a locale-specific preprocessor, making `#embed` variable-aware solves the delivery problem generically and provides value beyond localization (conditional content, platform variants, difficulty modes).

---

## Proposed Change

### Prerequisite: Runtime-Scoped Flags

This proposal depends on `20260313-runtime-scoped-flags-proposal.md`, which extends the flag system with a runtime scope. Runtime-scoped flags are:
- Set by the runtime API before story loading
- Available to all contexts (with context-level override via shadowing)
- **Available to the preprocessor** — this is the key bridge

Without runtime-scoped flags, the preprocessor has no access to flag values, and a separate "preprocessor variable" concept would be needed. With runtime-scoped flags, flags ARE the preprocessor's variable source for runtime-provided values.

### Variable Interpolation in `#embed`

`#embed` paths support `${}` interpolation:

```zoh
#embed "${filename}.${locale}.zoh";
```

**Resolution order for `${name}`:**
1. **Built-in preprocessor variables** — intrinsic values about the current file:
   - `filename` — base name of the current file without extension (e.g., `"cafe"` for `cafe.zoh`)
2. **Runtime-scoped flags** — flags set by the runtime API (e.g., `locale`, `platform`, `debug`)
3. **Story metadata** — metadata entries from the current story header
4. **Empty string** — if no source has the name, resolves to `""`

### Optional Embed

Introduce an optional embed form:

```zoh
#embed? "${filename}.${locale}.zoh";
```

The `?` suffix makes the embed conditional — if the resolved file doesn't exist, the directive is silently removed (no error). Without `?`, a missing file is a compile error (unchanged behavior).

Essential for locale files: not every story has a companion for every locale. `#embed?` gracefully degrades to no-op.

### The `locale` Standard Flag

Add `locale` to the standard flags in `std_flags.md`:

**locale** — The active locale identifier (BCP 47: `"fr"`, `"ja"`, `"pt-BR"`). Accept `string` or `*string`. Default: `""` (empty — no locale). Typically set at **runtime scope**.

At runtime scope, `locale` is:
- Available to the preprocessor for `#embed` path interpolation
- Available to all contexts for locale-aware verb behavior
- Overridable per-context via `/flag [scope: "context"] "locale", "ja";`

### Locale Delivery Without Special Machinery

```zoh
The Last Coffee Shop
===

#embed? "${filename}.${locale}.zoh";

@opening
/converse [By: "Narrator"] "11:47 PM. The cafe closes at midnight.";
```

`${filename}` → `"cafe"`, `${locale}` → `"fr"` (from runtime flag). Resolves to `#embed? "cafe.fr.zoh";`. If the file exists, it's embedded. If not, silently skipped.

**Alternatively — runtime auto-injection:** The runtime could automatically inject `#embed? "${filename}.${locale}.zoh";` at line 0 of every story body when `locale` is set. This is a runtime convention, not a language feature.

### Metadata as Interpolation Source

Story metadata is the lowest-priority source for `${}` interpolation:

```zoh
My Story
variant: "extended";
===

#embed? "chapter3.${variant}.zoh";
```

Enables story-level parameterization beyond locale — difficulty variants, content editions, platform-specific inclusions.

### Full Example

**Runtime configuration:**
```
runtime.setFlag("locale", "fr")  // runtime-scoped flag
```

**Base story (`cafe.zoh`):**
```zoh
The Last Coffee Shop
===

#embed? "${filename}.${locale}.zoh";

@opening
/converse [By: "Narrator"] "11:47 PM. The cafe closes at midnight.";
/converse "A woman in a red coat sits alone by the window.";
```

**French companion (`cafe.fr.zoh`):**
```zoh
/patch "11:47 PM. The cafe closes at midnight.", "23h47. Le cafe ferme a minuit.";
/patch "A woman in a red coat sits alone by the window.", "Une femme en manteau rouge est assise seule pres de la fenetre.";
/patch [locator: "checkpoint"] "@opening+1[By#1]", "Narrateur";
```

**After preprocessing (locale="fr"):**
```zoh
The Last Coffee Shop
===

/patch "11:47 PM. The cafe closes at midnight.", "23h47. Le cafe ferme a minuit.";
/patch "A woman in a red coat sits alone by the window.", "Une femme en manteau rouge est assise seule pres de la fenetre.";
/patch [locator: "checkpoint"] "@opening+1[By#1]", "Narrateur";

@opening
/converse [By: "Narrator"] "11:47 PM. The cafe closes at midnight.";
/converse "A woman in a red coat sits alone by the window.";
```

---

## Design Rationale

| Decision | Rationale |
|----------|-----------|
| `${filename}` as built-in | Portable self-referencing embed patterns without hardcoding filenames |
| Runtime flags as interpolation source | No separate "preprocessor variable" concept needed — runtime-scoped flags bridge the phase gap cleanly |
| Resolution order: built-in → runtime flag → metadata | Most specific wins; runtime config overrides metadata; built-ins can't be shadowed |
| `#embed?` optional form | Locale files are optional per story. Missing file must not be an error |
| `locale` at runtime scope | Global environment property, not per-context; available to preprocessor |
| Metadata as lowest priority | Metadata is story-specific config; runtime flags and built-ins take precedence |

---

## Impact Analysis

### Affected Areas
- **Spec: `1_concepts.md`** — `#embed` interpolation syntax, `#embed?` optional form, built-in preprocessor variables
- **Spec: `std_flags.md`** — `locale` standard flag definition
- **Impl: `03_preprocessor.md`** — `${}` resolution in embed paths, optional embed logic

### Dependencies
- **Runtime-scoped flags** (per 20260313-runtime-scoped-flags-proposal.md) — provides the runtime flag → preprocessor bridge
- `/patch` verb (per 20260312-script-level-localization-proposal.md) — the patching mechanism locale files use
- Statement locator system (per 20260313-statement-locator-system-proposal.md) — structural addressing for patches

### Risks
| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Empty interpolation producing bad paths | Low | Low | `#embed?` makes missing files non-fatal; empty flag means no embed |
| Metadata key collision with runtime flags | Low | Low | Runtime flags take priority; documented resolution order |
| Circular embed via interpolation | Low | Low | Existing cycle detection still applies post-resolution |

### Breaking Changes
None. `#embed` without `${}` behaves identically. `#embed?` is new syntax. `locale` flag is additive.

---

## Open Questions

- [ ] Should `${}` interpolation be limited to `#embed` paths, or extended to macro arguments and other preprocessor directives?
- [ ] Should the runtime auto-inject `#embed? "${filename}.${locale}.zoh";` by convention, or should authors always write it explicitly?
- [ ] Should `#embed?` support a fallback path? (e.g., `#embed? "${filename}.${locale}.zoh" || "${filename}.en.zoh";`)
- [ ] Should other built-in preprocessor variables be provided? (e.g., `${filepath}`, `${directory}`)

---

## Next Steps

If accepted:
1. Spec: Add `${}` interpolation and `#embed?` to embed section in `1_concepts.md`
2. Spec: Document built-in preprocessor variables (`${filename}`) in `1_concepts.md`
3. Spec: Add `locale` to `std_flags.md`
4. Impl: Update `03_preprocessor.md` with interpolation and optional embed logic
5. Update `/patch` proposal to reference this delivery mechanism
