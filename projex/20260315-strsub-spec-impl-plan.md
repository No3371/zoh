# Add `#strsub` Directive to Spec and Implementation Spec

> **Status:** Ready
> **Created:** 2026-03-15
> **Author:** Agent
> **Source:** 20260313-string-substitution-preprocessor-proposal.md, 20260315-strsub-filter-mechanism-proposal.md
> **Related Projex:** 20260313-string-substitution-preprocessor-proposal.md, 20260315-strsub-filter-mechanism-proposal.md, 20260315-post-compile-patch-directive-proposal.md, 20260312-script-level-localization-proposal.md
> **Worktree:** No

---

## Summary

Add the `#strsub` preprocessor directive to the ZOH language spec (`spec/1_concepts.md`) and implementation spec (`impl/03_preprocessor.md`). This covers the base directive (blanket string literal substitution) and the optional `where` clause (regex-powered prefix/suffix filtering for selective substitution).

**Scope:** Two spec files — language spec and implementation spec only. No runtime code.
**Estimated Changes:** 2 files

---

## Objective

### Problem / Gap / Need

The `#strsub` directive has two accepted proposals (base + filter mechanism) but no spec or implementation spec entries. The language spec defines `#embed` and macros but has no string substitution directive. The implementation spec defines the preprocessor pipeline but lacks the SubPreprocessor at priority 250.

### Success Criteria

- [ ] `spec/1_concepts.md` has a `## String Substitution` section defining `#strsub` syntax, semantics, matching rules, and the `where` clause
- [ ] `impl/03_preprocessor.md` pipeline diagram includes SubPreprocessor at priority 250
- [ ] `impl/03_preprocessor.md` has `#strsub` directive syntax documentation
- [ ] `impl/03_preprocessor.md` has SubPreprocessor implementation steps (collect, validate, replace with filter support)
- [ ] `impl/03_preprocessor.md` has `#strsub` testing checklist
- [ ] All content consistent with both proposals

### Out of Scope

- `#patch` directive (separate proposal, separate plan)
- `/patch` verb changes
- Runtime C# implementation
- `#pstrsub` (pattern substitution — separate proposal)
- Resolving open questions in the base `#strsub` proposal (optional key/tag, warning vs error for unmatched keys, `?` modifier)

---

## Context

### Current State

**`spec/1_concepts.md`:** The Embed section (`## Embed`, line 243) covers `#embed`, variable interpolation, optional embed, and an example. The Macro section (`## Macro`) starts at line 298. No string substitution directive exists between them.

**`impl/03_preprocessor.md`:** The pipeline diagram (lines 13–44) shows three preprocessors: EmbedPreprocessor (100), MacroPreprocessor (200), SugarPreprocessor (300). The Directive Syntax section (line 68) documents `#embed` and macro syntax. Implementation Steps (line 132) define Steps 1–6. Testing Checklist (line 394) covers `#embed`, `#macro`, `#expand`, and source mapping.

### Key Files

| File | Role | Change Summary |
|------|------|----------------|
| `spec/1_concepts.md` | Language spec | Add `## String Substitution` section between Embed and Macro |
| `impl/03_preprocessor.md` | Implementation spec | Add SubPreprocessor to pipeline, directive syntax, impl steps, testing |

### Dependencies

- **Requires:** Both proposals accepted (20260313-string-substitution-preprocessor-proposal.md, 20260315-strsub-filter-mechanism-proposal.md)
- **Blocks:** C# runtime implementation of SubPreprocessor

### Constraints

- Spec section style must match existing `## Embed` and `## Macro` sections
- Implementation steps must follow the existing pseudocode style (Steps 1–6)
- Pipeline diagram must remain readable ASCII art

### Assumptions

- Both proposals are accepted as-is (Draft status, but content is stable)
- Open questions in the base proposal remain open — the spec documents current decided behavior only
- The `where` clause regex subset (portable, no lookbehind, no backreferences) will be described conceptually; exact subset definition deferred to a future spec pass

### Impact Analysis

- **Direct:** `spec/1_concepts.md`, `impl/03_preprocessor.md`
- **Adjacent:** None — these are spec documents, not runtime code
- **Downstream:** C# implementation will consume these specs

---

## Implementation

### Overview

Two files, four insertion points:
1. Spec: new section in `1_concepts.md` between Embed and Macro
2. Impl: update pipeline diagram, add directive syntax, add implementation steps (renumber existing Step 6), add testing checklist

### Step 1: Add `## String Substitution` section to `spec/1_concepts.md`

**Objective:** Define the `#strsub` directive in the language spec, including the `where` clause.
**Confidence:** High
**Depends on:** None

**Files:**
- `spec/1_concepts.md`

**Changes:**

Insert a new `## String Substitution` section after the Embed example block (line 296) and before `## Macro` (line 298). Content:

```markdown
## String Substitution

The language supports compile-time string literal substitution via the `#strsub` directive.

### `#strsub` Directive

```zoh
#strsub "source string" -> "replacement string";
```

`#strsub` registers a string-to-string mapping. The preprocessor collects all `#strsub` directives, removes them from the source, then replaces all matching string literals in the remaining source text.

- Starts with `#strsub` (leading whitespace allowed)
- Source and replacement are ZOH string literals (single quotes, double quotes, or `"""` multiline)
- `->` separator
- Terminated with `;`
- Multi-line form: `->` can start on its own line after the source string

```zoh
:: Single-line
#strsub "Hello" -> "Bonjour";

:: Multi-line
#strsub "A long source string."
-> "A long replacement string.";

:: Multiline strings
#strsub """
Multi-line source.
Second line.
"""
-> """
Multi-line replacement.
Deuxieme ligne.
""";
```

### Matching Semantics

- Match is on **evaluated string content**, quote-style agnostic: `'Hello'` and `"Hello"` match the same key
- Escape sequences are normalized before matching: `"She said \"hello\""` matches by its content
- Multiline strings (`"""..."""`) match by content after standard indent-stripping
- Duplicate source keys produce a compile error (ambiguous mapping)

### What Gets Replaced

All string literals in the story body that exactly match a `#strsub` key, regardless of position:

- Verb parameters: `/converse "Hello";`
- Attribute values: `[By: "Narrator"]`
- Sugar forms: `*greeting <- "Hello";`
- Macro expansion results
- Nested verb parameters: `/if *cond, /converse "Hello";;`

### What Doesn't Get Replaced

- **Preprocessor directive arguments:** `#embed "path.zoh"` — consumed before `#strsub` runs
- **`#strsub` directive strings:** Collected and removed before replacement begins
- **Expression bodies:** Strings inside backtick expressions are code, not content
- **Macro definitions:** Already expanded before `#strsub` runs
- **Comments:** Comment text is not a string literal

### Interpolation Templates

`${...}` placeholders are literal text in the source key. The preprocessor matches and replaces the entire template string. Interpolation processes the result after compilation.

```zoh
#strsub "Hello, ${*name}." -> "Bonjour, ${*name}.";
```

### Selective Substitution (`where` clause)

An optional `where` clause restricts which occurrences get replaced. Without a clause, replacement is blanket (all matches). With a clause, the preprocessor checks the surrounding source text against regex patterns before replacing.

```zoh
#strsub "source" -> "replacement" where <filter>;
```

#### Filter Predicates

Predicates match the source text surrounding the string literal on the same line.

**`prefix /pattern/`** — The regex must match somewhere in the text from the start of the current line to the start of the matched string literal.

```zoh
#strsub "Narrator" -> "Narrateur" where prefix /\[By\s*:/;
```

**`suffix /pattern/`** — The regex must match somewhere in the text from the end of the matched string literal to the end of the current line.

```zoh
#strsub "Continue" -> "Continuer" where suffix /\s*;/;
```

#### Combining Filters

Filters compose with `,` (OR — replace if any predicate matches):

```zoh
#strsub "X" -> "Y" where prefix /\/converse/, prefix /\/choose/;
```

Regex alternation within a single predicate achieves the same:

```zoh
#strsub "X" -> "Y" where prefix /\/(converse|choose)/;
```

#### Regex

Regex patterns use `/pattern/` delimiters with optional flags. Case-sensitive by default; `/i` for case-insensitive. Patterns operate on single lines. A portable subset is used: no lookbehind, no backreferences.

#### Scope

Filters are line-scoped. For multi-line statements, only the line containing the matched string is checked. For cases requiring cross-line or structural precision, `#patch` (post-compile) provides exact targeting.

### Pipeline Position

Priority 250: after embed (100) and macro (200), before sugar (300).

- **After embed:** `#strsub` directives arrive via `#embed` from companion files
- **After macro:** Macro expansion may produce string literals that need substitution
- **Before sugar:** Substitution operates on authored form before sugar transformation

### Diagnostics

Unmatched `#strsub` keys (source string not found in the script) produce a compile-time diagnostic. This catches stale companion files where the base script changed but translations weren't updated.
```

**Rationale:** Placed between Embed and Macro because `#strsub` is a preprocessor directive at the same level, and its pipeline position (250) falls between embed (100) and macro (200)/sugar (300). The section structure mirrors the existing Embed section: definition, syntax examples, semantic rules, special cases.

**Verification:** Read the file after editing. Confirm the section appears between `## Embed` and `## Macro`. Confirm no existing content was removed or shifted incorrectly.

**If this fails:** Revert the insertion. The rest of the plan (impl changes) can still proceed independently.

---

### Step 2: Update pipeline diagram in `impl/03_preprocessor.md`

**Objective:** Add SubPreprocessor to the ASCII pipeline diagram and priority listing.
**Confidence:** High
**Depends on:** None

**Files:**
- `impl/03_preprocessor.md`

**Changes:**

**2a.** Update the Purpose description (line 5) to mention `#strsub`:

```
// Before:
The preprocessor handles text-level transformations before parsing. It processes `#embed` and `|%macro%|` directives by manipulating the raw source text.

// After:
The preprocessor handles text-level transformations before parsing. It processes `#embed`, `|%macro%|`, and `#strsub` directives by manipulating the raw source text.
```

**2b.** Insert SubPreprocessor block between MacroPreprocessor and SugarPreprocessor in the diagram (lines 33–39):

```
// Before:
│  3. #macro/#exp  │  ← MacroPreprocessor
│                  │    USER-DEFINED SYNTAX via text templates
└────────┬─────────┘
         ▼
┌──────────────────┐
│  4. Built-in     │  ← SugarPreprocessor

// After:
│  3. #macro/#exp  │  ← MacroPreprocessor
│                  │    USER-DEFINED SYNTAX via text templates
└────────┬─────────┘
         ▼
┌──────────────────┐
│  4. #strsub      │  ← SubPreprocessor
│                  │    STRING LITERAL SUBSTITUTION with optional filters
└────────┬─────────┘
         ▼
┌──────────────────┐
│  5. Built-in     │  ← SugarPreprocessor
```

**2c.** Update the priority comment block (lines 55–58):

```
// Before:
# - EmbedPreprocessor:  priority 100
# - MacroPreprocessor:  priority 200
# - SugarPreprocessor:  priority 300

// After:
# - EmbedPreprocessor:  priority 100
# - MacroPreprocessor:  priority 200
# - SubPreprocessor:    priority 250
# - SugarPreprocessor:  priority 300
```

**Rationale:** The diagram is the first thing readers see. It must reflect the full pipeline.

**Verification:** Read the diagram section. Confirm 5 stages, correct ordering, correct priority numbers.

**If this fails:** Revert. Diagram is cosmetic — later steps don't depend on it.

---

### Step 3: Add `#strsub` directive syntax to `impl/03_preprocessor.md`

**Objective:** Document the `#strsub` directive syntax in the Directive Syntax section.
**Confidence:** High
**Depends on:** None

**Files:**
- `impl/03_preprocessor.md`

**Changes:**

Insert a new `### #strsub` subsection after the Macro Expansion subsection (after line 128, before the `---` at line 130) in the Directive Syntax section:

```markdown
### #strsub

```zoh
#strsub "source string" -> "replacement string";
#strsub "source" -> "replacement" where prefix /pattern/;
#strsub "source" -> "replacement" where suffix /pattern/;
#strsub "source" -> "replacement" where prefix /p1/, suffix /p2/;
```

- Source and replacement are ZOH string literals (single quotes, double quotes, or `"""` multiline)
- `->` separator (can start on its own line for multi-line form)
- Optional `where` clause with regex prefix/suffix predicates
- Multiple predicates separated by `,` (OR semantics)
- Regex delimited by `/pattern/` with optional flags (`/i` for case-insensitive)
- Matching is on evaluated string content, quote-style agnostic
- Duplicate source keys produce a compile error
```

**Rationale:** Follows the pattern of `### #embed` and `### Macro Definition` subsections.

**Verification:** Read the Directive Syntax section. Confirm `#strsub` appears after macro syntax, before Implementation Steps.

**If this fails:** Revert. Independent of other steps.

---

### Step 4: Add SubPreprocessor implementation steps to `impl/03_preprocessor.md`

**Objective:** Add the collect/validate/replace algorithm as implementation steps, including filter support.
**Confidence:** High
**Depends on:** Step 2 (pipeline numbering)

**Files:**
- `impl/03_preprocessor.md`

**Changes:**

**4a.** Rename existing "Step 6: Syntactic Sugar Transformation" (line 325) to "Step 8: Syntactic Sugar Transformation" to make room for two new steps.

**4b.** Insert after Step 5 (Macro Expansion, ends around line 323) and before the renamed Step 8:

```markdown
### Step 6: String Substitution Collection

```
SubstitutionEntry:
  sourceContent: string           # Normalized string content (unescaped, unquoted)
  replacementContent: string      # Normalized replacement content
  replacementQuoteStyle: char     # Quote style of replacement literal
  filter: FilterClause?           # Optional where clause
  sourceLine: int                 # For diagnostics

FilterClause:
  predicates: List<FilterPredicate>   # OR-combined

FilterPredicate:
  type: enum { PREFIX, SUFFIX }
  pattern: Regex                      # Compiled regex
  flags: string                       # e.g., "i"

collectSubstitutions(source: string): (string, Map<string, SubstitutionEntry>)
    entries = Map<string, SubstitutionEntry>()
    result = StringBuilder()

    for line in source.lines():
        if matches(line, /^\s*#strsub\s+/):
            entry = parseStrsubDirective(line, lines)
            # line may span multiple lines for multi-line form

            if entry.sourceContent in entries:
                error("Duplicate #strsub key: " + quote(entry.sourceContent))

            entries[entry.sourceContent] = entry
            # Directive line(s) consumed — not appended to result
        else:
            result.append(line + '\n')

    return (result.toString(), entries)

parseStrsubDirective(line, lines): SubstitutionEntry
    # Parse: #strsub <source-literal> -> <replacement-literal> [where <predicates>] ;
    # Source and replacement: parse as ZOH string literals (handle ', ", """)
    # Normalize content: unescape, strip quotes, apply indent-stripping for """
    # Parse optional where clause: 'where' predicate (',' predicate)*
    # Each predicate: 'prefix' | 'suffix' followed by /regex/flags?
    # Compile regex patterns
    return SubstitutionEntry { ... }
```

### Step 7: String Substitution Replacement

```
applySubstitutions(source: string, entries: Map<string, SubstitutionEntry>): string
    if entries.isEmpty():
        return source

    matchedKeys = Set<string>()
    result = StringBuilder()

    # Walk source text, identifying string literal boundaries
    # Skip: backtick expression bodies, comments
    for each string literal found at position P:
        content = normalizeStringContent(literal)

        if content in entries:
            entry = entries[content]

            if entry.filter != null:
                if not matchesFilter(source, P, literal, entry.filter):
                    # Filter did not match — leave unchanged
                    result.append(literal.originalText)
                    continue

            # Replace: re-quote replacement in original quote style
            replaced = quoteAs(entry.replacementContent, literal.quoteStyle)
            result.append(replaced)
            matchedKeys.add(content)
        else:
            result.append(literal.originalText)

    # Stale-key diagnostic
    for key in entries.keys():
        if key not in matchedKeys:
            warning("Unmatched #strsub key: " + quote(key) + " (line " + entries[key].sourceLine + ")")

    return result.toString()

matchesFilter(source: string, pos: int, literal: StringLiteral, filter: FilterClause): bool
    lineStart = findLineStart(source, pos)
    lineEnd = findLineEnd(source, pos + literal.length)

    prefixText = source[lineStart..pos]
    suffixText = source[pos + literal.length..lineEnd]

    # OR semantics: return true if ANY predicate matches
    for predicate in filter.predicates:
        if predicate.type == PREFIX:
            if predicate.pattern.matchesIn(prefixText):
                return true
        else:  # SUFFIX
            if predicate.pattern.matchesIn(suffixText):
                return true

    return false
```
```

**Rationale:** Two steps mirror the two-pass algorithm from the proposal: collect-then-replace. Filter support is integrated into the replace step rather than being a separate step, keeping the algorithm cohesive.

**Verification:** Read Steps 6–8. Confirm step numbering is sequential. Confirm the algorithm covers: collection, duplicate detection, replacement, filter matching, stale-key diagnostics.

**If this fails:** Revert Steps 6–7 and the renumbering of Step 8. The spec file change (Step 1) is unaffected.

---

### Step 5: Add `#strsub` testing checklist to `impl/03_preprocessor.md`

**Objective:** Add testing checklist for the SubPreprocessor.
**Confidence:** High
**Depends on:** None

**Files:**
- `impl/03_preprocessor.md`

**Changes:**

Insert a new `### #strsub` checklist section after the existing `### #expand` checklist (line 464) and before `### Source Mapping` (line 466):

```markdown
### #strsub
- [ ] Basic substitution: single `#strsub` directive replaces matching string
- [ ] Multiple directives: all mappings applied
- [ ] Multi-line form: `->` on its own line
- [ ] Multiline strings: `"""..."""` source and replacement
- [ ] Quote-style agnostic: `'text'` matches `#strsub "text" -> ...`
- [ ] Escape normalization: `\"` in source matches literal `"`
- [ ] Duplicate source key: compile error
- [ ] Stale key diagnostic: unmatched source key produces warning
- [ ] Attribute values replaced: `[By: "text"]`
- [ ] Sugar forms replaced: `*var <- "text";`
- [ ] Nested verbs replaced: `/if *cond, /converse "text";;`
- [ ] Expression bodies skipped: strings inside `` `...` `` not replaced
- [ ] Comments skipped: `:: "text"` not replaced
- [ ] Directive strings not self-matching: `#strsub` source/replacement not replaced by other entries
- [ ] Interpolation templates: `"Hello ${*name}"` matched and replaced as whole string
- [ ] `where prefix`: replaces only when prefix regex matches
- [ ] `where suffix`: replaces only when suffix regex matches
- [ ] `where` with multiple predicates: OR semantics
- [ ] `where` with regex flags: `/pattern/i` for case-insensitive
- [ ] `where` line scope: only same-line text checked
- [ ] No `where` clause: blanket replacement (backward compatible)
- [ ] Mixed: directives with and without `where` in same file
- [ ] Companion file pattern: `#strsub` directives delivered via `#embed`
- [ ] Pipeline ordering: runs after macro expansion, before sugar
```

Also add to the existing `### Source Mapping` checklist (line 466):

```markdown
- [ ] Line numbers correct after #strsub directive removal
- [ ] Error messages point to original source after substitution
```

**Rationale:** Follows the existing checklist format. Covers base directive, matching semantics, exclusions, filter mechanism, and integration scenarios.

**Verification:** Read the Testing Checklist section. Confirm `#strsub` checklist appears after `#expand` and before `Source Mapping`.

**If this fails:** Revert. Testing checklist is documentation — independent of other steps.

---

## Verification Plan

### Manual Verification

- [ ] Read `spec/1_concepts.md` — `## String Substitution` section exists between `## Embed` and `## Macro`, covers all specified topics
- [ ] Read `impl/03_preprocessor.md` — pipeline diagram shows 5 stages with SubPreprocessor at priority 250
- [ ] Read `impl/03_preprocessor.md` — `#strsub` directive syntax documented in Directive Syntax section
- [ ] Read `impl/03_preprocessor.md` — Steps 6 and 7 cover collection and replacement with filter support
- [ ] Read `impl/03_preprocessor.md` — Step 8 is Sugar Transformation (renumbered from 6)
- [ ] Read `impl/03_preprocessor.md` — `#strsub` testing checklist exists
- [ ] Cross-check spec section against both proposals for consistency

### Acceptance Criteria Validation

| Criterion | How to Verify | Expected Result |
|-----------|---------------|-----------------|
| Spec has `## String Substitution` | Read `spec/1_concepts.md` | Section between Embed and Macro with directive syntax, matching rules, `where` clause |
| Pipeline diagram updated | Read `impl/03_preprocessor.md` lines 13–50 | SubPreprocessor at stage 4, priority 250 |
| Directive syntax documented | Read Directive Syntax section | `### #strsub` with syntax examples including `where` |
| Implementation steps added | Read Steps 6–7 | Collection and replacement algorithms with filter support |
| Testing checklist added | Read Testing Checklist section | `### #strsub` with 24 test cases |

---

## Rollback Plan

1. `git revert` the commit — both files return to prior state
2. No downstream dependencies to unwind (spec-only changes)

---

## Notes

### Risks

- **Step numbering cascade:** Renaming Step 6 to Step 8 may cause confusion if other documents reference "Step 6: Sugar Transformation" by number. Mitigated: no known external references to these step numbers.
- **Regex subset underspecified:** The exact portable regex subset is described conceptually ("no lookbehind, no backreferences") but not formally defined. Acceptable for spec draft; exact subset can be tightened in a future pass.

### Open Questions

None for this plan. Open questions in the source proposals are explicitly out of scope.
