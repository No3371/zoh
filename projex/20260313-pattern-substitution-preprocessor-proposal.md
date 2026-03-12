# Pattern Substitution Preprocessor (`#psub`)

> **Status:** Draft
> **Created:** 2026-03-13
> **Author:** Agent
> **Related Projex:** 20260313-string-substitution-preprocessor-proposal.md, 20260312-locale-companion-file-explore.md, 20260312-script-level-localization-proposal.md, 20260313-embed-variable-locale-flag-proposal.md

---

## Summary

A `#psub` preprocessor directive for compile-time pattern-based string substitution. Where `#strsub` matches exact string content, `#psub` matches string literals against a pattern containing named placeholders (`{*name}`) and rewrites them using a template that references those captures. Runs alongside `#strsub` in the SubPreprocessor at priority 250, with `#strsub` taking precedence for exact matches. Primary use case: localization of strings with variable structure (numbered chapters, parameterized UI text, systematic formatting changes).

---

## Problem Statement

### Current State

The `#strsub` proposal (20260313-string-substitution-preprocessor-proposal.md) covers exact-match string substitution. For localization, this handles the majority: swap `"Hello"` for `"Bonjour"`. But some strings share a common structure with varying content:

```zoh
/converse "Chapter 1: The Arrival";
/converse "Chapter 2: The Search";
/converse "Chapter 3: The Descent";
:: ...30 more chapters
```

With `#strsub`, each chapter title requires its own directive — 33 `#strsub` entries for 33 chapters. If the pattern is `"Chapter N: Title"` → `"Chapitre N: Title"`, only the frame changes. The variable parts (`N`, `Title`) pass through unchanged.

### Gap

No way to express "substitute all strings matching this structure" in a single directive. `#strsub` requires one entry per unique string. For structurally similar strings, this produces bloated companion files and maintenance burden — every new chapter means a new `#strsub` entry in every locale file.

### Why Now

`#psub` is a natural companion to `#strsub`. Defining both together ensures they share processing semantics and interact cleanly. Deferring `#psub` risks `#strsub` companion files growing unwieldy for pattern-heavy content, pushing authors toward `/patch` or custom preprocessors for something the standard pipeline should handle.

---

## Proposed Change

### The `#psub` Directive

```zoh
#psub "Chapter {*n}: {*title}" -> "Chapitre {*n}: {*title}";
```

A preprocessor directive that registers a pattern-to-template substitution. The pattern contains literal text and named placeholders. When a string literal in the source matches the full pattern, it is replaced by the template with captures inserted.

**Syntax:**

```
#psub "pattern with {*name} placeholders"
-> "template with {*name} references";
```

- Starts with `#psub` at the beginning of a line (leading whitespace allowed)
- Pattern string: a ZOH string literal containing literal text and `{*name}` placeholders
- `->` separator (same as `#strsub`)
- Template string: a ZOH string literal containing literal text and `{*name}` back-references
- Terminated with `;`
- Multi-line form supported (same as `#strsub`)

### Placeholder Syntax

| Form | Matches | Captured | In template |
|------|---------|----------|-------------|
| `{*name}` | Minimal non-empty substring | Yes, as `name` | `{*name}` inserts the captured text |
| `{*}` | Minimal non-empty substring | No (anonymous) | Not referenceable |
| `{**name}` | Greedy (maximal) substring | Yes, as `name` | `{**name}` or `{*name}` |
| Literal text | Itself, exactly | N/A | Itself |

- **Minimal (non-greedy) by default:** `{*name}` matches the shortest substring that allows the rest of the pattern to succeed. In `"Chapter {*n}: {*title}"`, `{*n}` matches `"1"` (stops at `": "`), and `{*title}` matches `"The Arrival"`.
- **Greedy with `{**name}`:** `{**name}` matches the longest possible substring. Useful when the boundary is at the end of the string or when minimal matching would be too aggressive.
- **Anonymous `{*}`:** Matches and discards — the captured text is not available in the template. Useful for ignoring variable portions.
- **Names follow ZOH identifier rules:** case-insensitive, no whitespace, no reserved characters.
- **Escaping:** Literal `{*` is written as `\{*`. Literal `\{` is `\\{`.

### Pattern Matching Semantics

1. The pattern is matched against the **entire** string literal content (not a substring). `"Chapter {*n}"` does not match `"See Chapter 3"` — the string must start with `"Chapter "`.
2. Literal portions are matched exactly (case-sensitive).
3. Placeholders match non-empty substrings by default. A placeholder cannot match the empty string.
4. Match is quote-style agnostic (same as `#strsub`).
5. If a pattern cannot produce an unambiguous match (e.g., `"{*a}{*b}"` with no literal separator), it is a compile error: `ambiguous_pattern`.

### Ambiguity Rules

Two adjacent placeholders with no literal separator between them are ambiguous — there's no way to determine where one capture ends and the next begins:

```zoh
:: COMPILE ERROR: ambiguous_pattern
#psub "{*first}{*last}" -> "{*last}, {*first}";

:: OK: literal " " separates the captures
#psub "{*first} {*last}" -> "{*last}, {*first}";

:: OK: anonymous placeholder doesn't need a boundary
:: (still ambiguous — but {*} is explicitly "match anything remaining")
:: Actually, this is still ambiguous. Keep it simple:
:: Adjacent placeholders always require a literal separator.
```

**Rule:** Every pair of adjacent placeholders must have at least one literal character between them. The preprocessor validates this at directive parse time.

### Template Semantics

The template string can:
- Reference any named capture from the pattern via `{*name}`
- Contain literal text that differs from the pattern
- Rearrange captures in any order
- Repeat a capture multiple times
- Omit captures (discarding the matched portion)

```zoh
:: Rearrange: "Last, First" → "First Last"
#psub "{*last}, {*first}" -> "{*first} {*last}";

:: Prefix transformation
#psub "Ch. {*n} - {*title}" -> "Chapitre {*n} — {*title}";

:: Drop a portion
#psub "{*speaker} says: {*line}" -> "{*line}";
```

A template referencing a `{*name}` that doesn't exist in the pattern is a compile error: `undefined_capture`.

### Pipeline Position and Interaction with `#strsub`

Both `#strsub` and `#psub` run in the SubPreprocessor at priority 250.

**Processing order within the SubPreprocessor:**

```
1. Collect all #strsub and #psub directives; remove from source
2. Validate both sets (duplicates, malformed, ambiguous patterns)
3. Apply #strsub (exact match) — each matched string is marked as replaced
4. Apply #psub (pattern match) — only to strings NOT already replaced by #strsub
5. Report diagnostics for unmatched entries
```

**`#strsub` takes precedence.** If a string matches both a `#strsub` key and a `#psub` pattern, the `#strsub` replacement wins. This allows specific overrides:

```zoh
:: Pattern: translate chapter headers
#psub "Chapter {*n}: {*title}" -> "Chapitre {*n}: {*title}";

:: Override: special title for Chapter 7
#strsub "Chapter 7: The Descent" -> "Chapitre 7: La Chute";
```

"Chapter 7: The Descent" matches both, but `#strsub` wins — it gets the hand-crafted translation. All other chapters fall through to `#psub`.

**Among `#psub` directives:** First matching pattern wins (declaration order). If multiple `#psub` patterns match the same string, the first declared `#psub` is applied.

### Companion File Pattern

`#psub` directives live in companion files alongside `#strsub`:

```zoh
:: cafe.fr.zoh

:: Exact replacements
#strsub "Narrator" -> "Narrateur";
#strsub "The cafe closes at midnight." -> "Le cafe ferme a minuit.";

:: Pattern replacements
#psub "Chapter {*n}: {*title}" -> "Chapitre {*n}: {*title}";
#psub "{*name} whispers: {*line}" -> "{*name} chuchote: {*line}";
```

### What Gets Replaced

Same scope as `#strsub`: all string literals in the story body (verb parameters, attribute values, sugar forms, macro expansion results). Does NOT replace strings inside expressions, comments, or preprocessor directive arguments.

---

## Approach Options

### Option A: Named Placeholder Patterns (Recommended)

The `{*name}` syntax described above. ZOH-native, readable, self-documenting.

**Pros:**
- Reads like ZOH: `{*name}` echoes `*reference` and `${*var}` interpolation conventions
- Named captures are self-documenting — pattern and template are readable without regex knowledge
- Ambiguity is statically detectable (adjacent placeholders without separators)
- No ReDoS risk — matching is deterministic given the separator constraint
- Translators can author patterns without learning regex

**Cons:**
- Less expressive than regex: no character classes, quantifiers, alternation
- Cannot match "strings containing a digit" or "strings ending in a question mark" without literal framing
- For complex patterns, authors fall back to custom preprocessors or `/patch`

### Option B: Regex Patterns

```zoh
#psub /Chapter (\d+): (.+)/ -> "Chapitre $1: $2";
```

**Pros:**
- Maximum expressiveness: character classes, quantifiers, alternation, anchors
- Familiar to developers

**Cons:**
- Regex in a storytelling language is a mismatch — translators are not developers
- ReDoS risk on pathological patterns
- New syntax: `/pattern/` delimiters are not ZOH
- Numbered captures (`$1`, `$2`) are less readable than named ones
- Error messages for regex failures are opaque to non-technical users

### Option C: Hybrid — Named Placeholders with Optional Type Hints

```zoh
#psub "Chapter {*n:int}: {*title}" -> "Chapitre {*n}: {*title}";
```

Where `{*n:int}` constrains the placeholder to match only digit sequences.

**Pros:**
- Named readability of Option A with slightly more precision
- Type hints mirror ZOH's typed variable system (`*var:integer`)

**Cons:**
- Adds complexity for marginal benefit
- Which types? `int`, `word`, `any`? Becomes a mini-regex
- Can be added later as an extension to Option A without breaking changes

### Recommendation

**Option A.** Named placeholders cover the practical use cases (structured localization, systematic rewriting) without introducing regex complexity into a storytelling language. The constraint that adjacent placeholders need literal separators eliminates ambiguity statically. Option C's type hints can be added later if demand warrants.

For the rare case needing regex-level pattern matching, a custom preprocessor (the pipeline is extensible) or `/patch` with runtime logic is the right tool.

---

## Design Rationale

| Decision | Rationale |
|----------|-----------|
| `#psub` naming | "pattern sub" — mirrors `#strsub` (string sub). The `p` prefix signals pattern matching. |
| `{*name}` placeholder syntax | Echoes ZOH's `*reference` convention. `{...}` mirrors `${...}` interpolation braces. Reads naturally in both pattern and template. |
| Non-greedy default | Minimal matching is predictable. Greedy matching requires `{**name}` opt-in — explicit over implicit. |
| Adjacent-placeholder prohibition | Eliminates ambiguity statically. No backtracking needed. Simple implementation. |
| `#strsub` takes precedence | Specific overrides should win over general patterns. Predictable, no surprises. |
| Declaration-order tiebreaker for `#psub` | Author controls priority through ordering. Simpler than specificity rules or priority attributes. |
| No regex | Translators are not developers. Pattern safety and readability outweigh expressiveness. Custom preprocessors exist for power users. |
| Full-string matching only | `#psub` replaces entire string literals, not substrings within them. Prevents surprising partial rewrites. Consistent with `#strsub`. |

---

## Impact Analysis

### Affected Areas

- **Spec: `1_concepts.md`** — `#psub` directive alongside `#strsub`, `#embed`, and macro
- **Impl: `03_preprocessor.md`** — Pattern matching within SubPreprocessor

### Dependencies

- **`#strsub`** (20260313-string-substitution-preprocessor-proposal.md) — `#psub` shares the SubPreprocessor and the `->` syntax convention. `#strsub` must be defined first; `#psub` extends it.
- **`#embed?` and variable interpolation** (20260313-embed-variable-locale-flag-proposal.md) — delivery mechanism for companion files containing `#psub` directives.

### Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Pattern-order sensitivity surprises | Medium | Low | Document that first match wins; companion file authors control ordering |
| Ambiguous patterns slipping through | Low | Medium | Static validation rejects adjacent placeholders without separators |
| Placeholder name typos (`{*titl}` vs `{*title}`) | Medium | Low | `undefined_capture` compile error catches template references to non-existent captures |
| Overly broad patterns matching unintended strings | Low | Medium | `#strsub` precedence allows specific overrides; stale-pattern diagnostics flag zero-match patterns |
| Greedy vs non-greedy confusion | Low | Low | Non-greedy is the safe default; greedy requires explicit `{**name}` opt-in |

### Breaking Changes

None. Fully additive.

---

## Examples

### Chapter Headers

**Base script:**
```zoh
My Novel
===

@ch1
/converse [By: "Narrator"] "Chapter 1: The Arrival";
/converse "The train pulled into the station at dawn.";

@ch2
/converse [By: "Narrator"] "Chapter 2: The Search";
/converse "She had been looking for three days.";
```

**Companion (`novel.fr.zoh`):**
```zoh
:: One pattern covers all chapter headers
#psub "Chapter {*n}: {*title}" -> "Chapitre {*n}: {*title}";

:: Individual lines still need #strsub
#strsub "The train pulled into the station at dawn."
-> "Le train est arrive en gare a l'aube.";
#strsub "She had been looking for three days."
-> "Elle cherchait depuis trois jours.";
#strsub "Narrator" -> "Narrateur";
```

**After preprocessing (locale="fr"):**
```zoh
@ch1
/converse [By: "Narrateur"] "Chapitre 1: The Arrival";
/converse "Le train est arrive en gare a l'aube.";

@ch2
/converse [By: "Narrateur"] "Chapitre 2: The Search";
/converse "Elle cherchait depuis trois jours.";
```

Note: `{*title}` captures pass through unchanged — only the frame (`"Chapter"` → `"Chapitre"`) is translated by the pattern. Individual chapter titles remain in the source language unless separately addressed by `#strsub`. This is intentional: chapter titles may be proper nouns that don't translate, or they may have `#strsub` overrides for specific titles.

### Speaker Attribution

```zoh
#psub "{*name} says: {*line}" -> "{*name} dit: {*line}";
#psub "{*name} whispers: {*line}" -> "{*name} chuchote: {*line}";
#psub "{*name} shouts: {*line}" -> "{*name} crie: {*line}";
```

Covers any speaker with these attribution patterns, regardless of name or dialogue content.

### Rearrangement

Some languages reorder sentence components:

```zoh
:: English: "Welcome to {place}"
:: Japanese: "{place}へようこそ"
#psub "Welcome to {*place}" -> "{*place}へようこそ";
```

### Anonymous Wildcards

```zoh
:: Strip the room number, keep the description
#psub "Room {*}: {*desc}" -> "{*desc}";
```

`{*}` matches the room number but discards it in the template.

### Precedence: `#strsub` Over `#psub`

```zoh
#psub "Welcome to {*place}" -> "Bienvenue a {*place}";
#strsub "Welcome to The Last Coffee Shop"
-> "Bienvenue au Dernier Cafe";
```

`"Welcome to The Last Coffee Shop"` matches both. `#strsub` wins — the hand-crafted translation with the correct French article (`au` not `a`) is used. All other `"Welcome to X"` strings fall through to the `#psub` pattern.

---

## Open Questions

- [ ] Should `{*name}` be allowed to match the empty string with an explicit modifier (e.g., `{*name?}`)? Current rule: non-empty only. Empty-match support adds edge cases.
- [ ] Should `#psub` support a `[count:N]` attribute to limit how many strings a pattern can match? Useful for catching overly broad patterns.
- [ ] Should type hints (`{*n:int}`, `{*name:word}`) be included in the initial spec, or deferred as an extension?
- [ ] Should unmatched `#psub` patterns produce a warning or be silent? Patterns are inherently more speculative than exact `#strsub` keys — a zero-match pattern might be intentional (covering content that doesn't exist yet).
- [ ] Should `#psub` support alternation in literal portions? e.g., `"Chapter|Ch. {*n}"` matching both `"Chapter 1"` and `"Ch. 1"`. This approaches regex territory — may be better left to a future extension.
- [ ] Naming: `#psub` vs `#patsub` vs `#ptrn`? `#psub` is concise and pairs with `#strsub`.

---

## Next Steps

If accepted:
1. Spec: Add `#psub` directive to `1_concepts.md` alongside `#strsub`
2. Impl: Extend SubPreprocessor in `03_preprocessor.md` with pattern matching
3. Document `#strsub`/`#psub` interaction and precedence rules
4. Update companion file examples across related proposals

---

## Appendix

### Pattern Grammar

```ebnf
psub_directive := '#psub' ws pattern_string ws '->' ws template_string ';'
pattern_string := string_literal  (* contains placeholder syntax *)
template_string := string_literal  (* contains capture references *)

placeholder    := '{*' name '}'       (* non-greedy named capture *)
                | '{**' name '}'      (* greedy named capture *)
                | '{*}'               (* anonymous non-greedy *)
                | '{**}'              (* anonymous greedy *)
name           := identifier          (* ZOH identifier rules *)

(* Validation: no two adjacent placeholders without intervening literal text *)
```

### Matching Algorithm

```
match(pattern, string):
    Convert pattern to segments: [literal, placeholder, literal, ...]

    Walk the string left-to-right:
        For each literal segment: match exactly at current position
        For each placeholder:
            If next segment is a literal:
                Find next occurrence of that literal in the string
                Capture everything between current position and that literal
                (non-greedy: find FIRST occurrence; greedy: find LAST)
            If this is the last segment:
                Capture the remainder of the string
            If capture is empty: match fails

    If entire string is consumed and all segments matched: success
    Otherwise: no match
```

This is a linear scan — no backtracking, no exponential blowup. The adjacent-placeholder prohibition guarantees every placeholder is bounded by a literal (or string boundary) on at least one side.

### Alternatives Considered

- **Regex-based patterns:** Maximum power, but mismatched with ZOH's non-developer audience. ReDoS risk. Foreign syntax. Rejected for the standard preprocessor; available via custom preprocessors for power users.

- **Glob patterns (`*` wildcards):** Simpler than named placeholders but no capture — can't reference matched portions in the template. Insufficient for reordering or selective insertion.

- **Separate `#pfilter` + `#strsub` two-step:** A filter directive that scopes which strings `#strsub` acts on. Rejected: conflates filtering with substitution, requires author to coordinate two directives for one operation, and can't express structural transformations (rearrangement, selective capture).
