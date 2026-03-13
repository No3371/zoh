# Variable-Aware `#embed` and Optional Embed (`#embed?`)

> **Status:** Complete
> **Completed:** 2026-03-13
> **Walkthrough:** 20260313-embed-variable-interpolation-walkthrough.md
> **Created:** 2026-03-13
> **Author:** Agent
> **Source:** 20260313-embed-variable-locale-flag-proposal.md (Draft)
> **Related Projex:** 20260313-embed-variable-locale-flag-proposal.md, 20260312-script-level-localization-proposal.md, 20260313-runtime-scoped-flags-spec-plan.md (Complete)
> **Worktree:** No

---

## Summary

Add `${}` variable interpolation to `#embed` file paths and introduce `#embed?` as an optional (non-fatal) embed form. These two changes enable parameterized file inclusion — the immediate use case is locale companion file delivery, but the mechanism is general-purpose.

**Scope:** Spec (`1_concepts.md`) and implementation spec (`03_preprocessor.md`) only.
**Estimated Changes:** 2 files, 2 sections each.

---

## Objective

### Problem / Gap / Need

`#embed` accepts only static file paths. There is no way to parameterize the path based on runtime configuration, story metadata, or intrinsic file properties. Separately, the `/patch` localization proposal needs a delivery mechanism for locale companion files — currently no way to conditionally embed a file that may or may not exist.

### Success Criteria

- [ ] `spec/1_concepts.md` documents `${}` interpolation syntax in embed paths with resolution order
- [ ] `spec/1_concepts.md` documents `#embed?` optional form and its behavior when the file is missing
- [ ] `spec/1_concepts.md` documents built-in preprocessor variable `${filename}`
- [ ] `impl/03_preprocessor.md` specifies interpolation resolution algorithm (built-in vars -> runtime flags -> metadata -> empty string)
- [ ] `impl/03_preprocessor.md` specifies optional embed processing (skip instead of error)
- [ ] `impl/03_preprocessor.md` testing checklist covers interpolation and optional embed cases
- [ ] Existing `#embed` behavior is unchanged (no breaking changes)

### Out of Scope

- `spec/std_flags.md` changes — `locale` flag already defined with runtime scope and preprocessor access
- `spec/3_runtime.md` changes — preprocessor contract already includes runtime-scoped flags
- Extending `${}` interpolation to macro arguments or other directives (open question in proposal)
- Runtime auto-injection of `#embed?` (convention, not a language feature)
- Fallback path syntax (e.g., `#embed? "a.zoh" || "b.zoh";`)
- Additional built-in variables beyond `${filename}` (e.g., `${filepath}`, `${directory}`)

---

## Context

### Current State

**`spec/1_concepts.md` L243-259 — Embed section:**
- `#embed "path";` with static paths only
- Path relative to current file or absolute
- File not found = compile error
- Cycle detection (each file embedded once per pass)

**`impl/03_preprocessor.md` L70-81, L130-174 — Embed processing:**
- `FileResolver` handles path resolution (relative/absolute)
- `processEmbeds()` scans for `#embed "..."` via regex, resolves path, reads file, recurses
- No interpolation, no optional handling

**Already completed (not in scope):**
- `spec/1_concepts.md` L181-191 — Flags section documents runtime + context scopes
- `spec/std_flags.md` L20-23 — `locale` standard flag with runtime scope
- `spec/3_runtime.md` L34 — Preprocessors receive runtime-scoped flags
- `impl/03_preprocessor.md` pipeline — EmbedPreprocessor at priority 100

### Key Files

| File | Role | Change Summary |
|------|------|----------------|
| `spec/1_concepts.md` | Core language concepts | Expand Embed section with interpolation, optional form, built-in vars |
| `impl/03_preprocessor.md` | Preprocessor implementation spec | Add interpolation resolution step, optional embed handling, tests |

### Dependencies

- **Requires:** Runtime-scoped flags (20260313-runtime-scoped-flags-spec-plan.md) — **Complete**
- **Blocks:** `/patch` locale delivery mechanism (20260312-script-level-localization-proposal.md)

### Constraints

- Resolution order must match proposal: built-in -> runtime flag -> metadata -> empty string
- `#embed` without `${}` must behave identically to current behavior
- Cycle detection still applies after interpolation resolution
- `#embed?` only suppresses "file not found" — other errors (cycles, read failures) remain fatal

### Assumptions

- The preprocessor receives runtime-scoped flags as a `Map<string, value>` (per `spec/3_runtime.md` L34)
- The preprocessor receives story metadata entries (per `spec/3_runtime.md` L34)
- The preprocessor knows the source file path (already required for relative path resolution)

### Impact Analysis

- **Direct:** `spec/1_concepts.md` Embed section, `impl/03_preprocessor.md` embed processing
- **Adjacent:** Source mapping — interpolated embeds still produce source map entries (no change to mechanism, just different resolved paths)
- **Downstream:** Locale companion file delivery pattern enabled; `/patch` proposal can reference this mechanism

---

## Implementation

### Overview

Two steps, each touching one file. Step 1 defines the language-level behavior (spec). Step 2 specifies the implementation algorithm (impl spec). Both are additive — existing text is expanded, not replaced.

### Step 1: Expand Embed Section in `spec/1_concepts.md`

**Objective:** Document `${}` interpolation, `#embed?`, and built-in preprocessor variables.
**Confidence:** High
**Depends on:** None

**Files:**
- `spec/1_concepts.md` (L243-259)

**Changes:**

Replace the current Embed section (L243-259):

```markdown
## Embed

The language should support embedding of other script files.

Denoted as `#embed` in their own lines, embeds are one of the rare exceptions in the language that are non-verbs that are first-class citizens as verbs.

The implementation should simply replace the embed syntax with the content of the designated file with a standard pre-processor.

The path must be absoulute or relative to the current file.

If the file is not found, the runtime should emit a compile error.

During each embed resolution, each file can only be embedded once.

```
#embed "relative/path/to/file.zoh";
```
```

With:

```markdown
## Embed

The language should support embedding of other script files.

Denoted as `#embed` in their own lines, embeds are one of the rare exceptions in the language that are non-verbs that are first-class citizens as verbs.

The implementation should simply replace the embed syntax with the content of the designated file with a standard pre-processor.

The path must be absolute or relative to the current file.

If the file is not found, the runtime should emit a compile error.

During each embed resolution, each file can only be embedded once.

```
#embed "relative/path/to/file.zoh";
```

### Variable Interpolation

Embed paths support `${}` interpolation:

```
#embed "${filename}.${locale}.zoh";
```

Resolution order for `${name}`:
1. **Built-in preprocessor variables** — intrinsic values about the current file:
   - `filename` — base name of the current file without extension (e.g., `"cafe"` for `cafe.zoh`)
2. **Runtime-scoped flags** — flags set by the runtime API (e.g., `locale`, `platform`)
3. **Story metadata** — metadata entries from the current story header
4. **Empty string** — if no source has the name, resolves to `""`

Variable names in `${}` follow identifier rules (alphanumeric and underscores). Unknown names resolve to empty string — they are not errors.

### Optional Embed

`#embed?` is the optional form — if the resolved file does not exist, the directive is silently removed (no error). Without `?`, a missing file remains a compile error.

```
#embed? "${filename}.${locale}.zoh";
```

`#embed?` suppresses only "file not found". Other errors (circular embed, read failures) are still fatal.

### Example

Given `cafe.zoh` with runtime flag `locale` set to `"fr"`:

```
#embed? "${filename}.${locale}.zoh";
```

`${filename}` resolves to `"cafe"`, `${locale}` resolves to `"fr"`. The directive becomes `#embed? "cafe.fr.zoh";`. If `cafe.fr.zoh` exists, its content is embedded. If not, the line is silently removed.
```

**Rationale:** The spec section is the authoritative language definition. Changes are additive subsections under Embed — existing description is preserved (with the typo "absoulute" fixed to "absolute"). The resolution order, optional form semantics, and example are drawn directly from the accepted proposal.

**Verification:** Read the updated section and confirm:
- Static `#embed "path";` behavior is unchanged
- `${}` resolution order is explicit (4 levels)
- `#embed?` behavior is specified for both found and not-found cases
- Built-in variable `filename` is defined

**If this fails:** Revert the Embed section to its original content (L243-259).

---

### Step 2: Update Embed Processing in `impl/03_preprocessor.md`

**Objective:** Specify the interpolation resolution algorithm, optional embed handling, and update test checklist.
**Confidence:** High
**Depends on:** Step 1 (spec defines the behavior this step implements)

**Files:**
- `impl/03_preprocessor.md` (L70-81, L130-174, L386-394)

**Changes:**

**2a. Update `#embed` directive syntax section (L70-81):**

Replace:

```markdown
### #embed

```zoh
#embed "relative/path/to/file.zoh";
#embed "absolute/path/to/file.zoh";
```

- Path is relative to current file or absolute
- File content replaces the `#embed` line
- Each file can only be embedded once per resolution pass (cycle detection)
- Throws compile error if file not found
```

With:

```markdown
### #embed

```zoh
#embed "relative/path/to/file.zoh";
#embed "absolute/path/to/file.zoh";
#embed "${filename}.${locale}.zoh";
#embed? "${filename}.${locale}.zoh";
```

- Path is relative to current file or absolute
- File content replaces the `#embed` / `#embed?` line
- Each file can only be embedded once per resolution pass (cycle detection)
- `#embed`: throws compile error if file not found
- `#embed?`: silently removes the directive if file not found (other errors remain fatal)
- Paths may contain `${name}` interpolation — resolved before file lookup (see Step 2 below)
```

**2b. Add interpolation resolution step between Step 1 and Step 2 (after L147, before L149):**

Insert new section:

```markdown
### Step 2: Embed Path Interpolation

Before file resolution, `${name}` placeholders in embed paths are replaced with values from the following sources, checked in order:

```
BuiltInVars:
  filename: string    # Base name of current file without extension

InterpolationContext:
  builtIns: BuiltInVars
  runtimeFlags: Map<string, Value>    # From runtime-scoped flags
  metadata: Map<string, string>       # From story header

interpolatePath(path: string, ctx: InterpolationContext): string
    return path.replaceAll(/\$\{(\w+)\}/, (match, name) =>
        if name in ctx.builtIns:
            return ctx.builtIns[name]
        if name in ctx.runtimeFlags:
            return toString(ctx.runtimeFlags[name])
        if name in ctx.metadata:
            return ctx.metadata[name]
        return ""   # Unknown name resolves to empty string
    )
```

Resolution order: built-in variables > runtime-scoped flags > story metadata > empty string.

Built-in variables cannot be shadowed by flags or metadata of the same name.
```

**2c. Update the existing embed processing algorithm (current Step 2, L149-174) — renumber to Step 3 and add interpolation + optional embed:**

Replace:

```
### Step 2: Embed Processing

```
processEmbeds(source: string, sourceFile: string, embedded: Set<string>): string
    result = StringBuilder()
    lines = source.split('\n')

    for line in lines:
        if matches(line, /^#embed\s+"(.+)"\s*;/):
            path = extractPath(line)
            absPath = resolve(path, sourceFile)

            if absPath in embedded:
                error("Circular embed: " + absPath)

            embedded.add(absPath)
            content = readFile(absPath)

            # Recursively process embeds in included file
            content = processEmbeds(content, absPath, embedded)
            result.append(content)
        else:
            result.append(line + '\n')

    return result.toString()
```
```

With:

```
### Step 3: Embed Processing

```
processEmbeds(source: string, sourceFile: string, embedded: Set<string>, ctx: InterpolationContext): string
    result = StringBuilder()
    lines = source.split('\n')

    # Update built-in for current file
    ctx.builtIns.filename = stripExtension(basename(sourceFile))

    for line in lines:
        if matches(line, /^#embed(\??)\s+"(.+)"\s*;/):
            optional = match[1] == "?"
            rawPath = match[2]

            # Interpolate variables in path
            path = interpolatePath(rawPath, ctx)
            absPath = resolve(path, sourceFile)

            if absPath in embedded:
                error("Circular embed: " + absPath)

            if not fileExists(absPath):
                if optional:
                    continue    # #embed? — silently skip
                else:
                    error("File not found: " + absPath)

            embedded.add(absPath)
            content = readFile(absPath)

            # Recursively process embeds in included file
            content = processEmbeds(content, absPath, embedded, ctx)
            result.append(content)
        else:
            result.append(line + '\n')

    return result.toString()
```
```

**2d. Renumber subsequent steps:** Current "Step 3: Macro Collection" becomes "Step 4", "Step 4: Macro Expansion" becomes "Step 5", "Step 5: Syntactic Sugar" becomes "Step 6".

**2e. Update the `#embed` testing checklist (L386-394):**

Replace:

```markdown
### #embed
- [ ] Basic embed: single file
- [ ] Nested embed: file A embeds B embeds C
- [ ] Circular embed detection: A embeds A
- [ ] Indirect circular: A embeds B embeds A
- [ ] File not found error
- [ ] Relative path resolution
- [ ] Absolute path resolution
```

With:

```markdown
### #embed
- [ ] Basic embed: single file (static path)
- [ ] Nested embed: file A embeds B embeds C
- [ ] Circular embed detection: A embeds A
- [ ] Indirect circular: A embeds B embeds A
- [ ] File not found error (`#embed`)
- [ ] Relative path resolution
- [ ] Absolute path resolution
- [ ] Interpolation: `${filename}` resolves to current file base name
- [ ] Interpolation: `${flag}` resolves from runtime-scoped flags
- [ ] Interpolation: `${meta}` resolves from story metadata
- [ ] Interpolation: unknown `${name}` resolves to empty string
- [ ] Interpolation: resolution order (built-in > flag > metadata)
- [ ] Optional embed: `#embed?` skips silently when file not found
- [ ] Optional embed: `#embed?` still errors on circular embed
- [ ] Optional embed with interpolation: `#embed? "${filename}.${locale}.zoh";`
- [ ] Mixed: static `#embed` and interpolated `#embed?` in same file
```

**Rationale:** The impl spec mirrors the language spec's behavior definition with concrete algorithms. The interpolation step is inserted before file resolution (as a new Step 2) because variables must be resolved before the path can be looked up. The `InterpolationContext` aggregates the three sources the preprocessor already receives. Step renumbering keeps the pipeline in logical order.

**Verification:** Read the updated sections and confirm:
- Regex matches both `#embed` and `#embed?`
- `interpolatePath` checks sources in correct order
- `filename` built-in is set per-file (updates on recursive embed)
- Optional embed skips on missing file but errors on cycles
- Test checklist covers all new cases

**If this fails:** Revert each sub-change independently — the directive syntax, interpolation step, embed processing, step numbers, and test checklist are independent sections.

---

## Verification Plan

### Manual Verification

- [ ] Read `spec/1_concepts.md` Embed section end-to-end — no contradictions with rest of spec
- [ ] Read `impl/03_preprocessor.md` — pipeline diagram still accurate, step numbering consistent
- [ ] Confirm `#embed "static/path.zoh";` behavior unchanged in both files
- [ ] Confirm resolution order documented identically in spec and impl
- [ ] Cross-reference with `spec/3_runtime.md` L34 — preprocessor contract still aligns

### Acceptance Criteria Validation

| Criterion | How to Verify | Expected Result |
|-----------|---------------|-----------------|
| `${}` interpolation documented in spec | Read Embed section | Subsection with syntax, resolution order, identifier rules |
| `#embed?` documented in spec | Read Embed section | Subsection explaining silent removal on missing file |
| `${filename}` built-in documented | Read Embed section | Listed under built-in preprocessor variables |
| Interpolation algorithm in impl | Read new Step 2 | `interpolatePath` with 4-level resolution |
| Optional embed in impl | Read updated Step 3 | `optional` flag check, `continue` on missing |
| Test checklist updated | Read #embed checklist | 16 items covering static, interpolation, optional, and mixed |
| No breaking changes | Compare static `#embed` path in both files | Identical behavior to current spec |

---

## Rollback Plan

1. Revert `spec/1_concepts.md` Embed section to original L243-259 content
2. Revert `impl/03_preprocessor.md` to original content (directive syntax L70-81, file resolution L130-147, embed processing L149-174, step numbering, test checklist L386-394)

Both files are under version control — `git checkout` of the two files is sufficient.

---

## Notes

### Risks

- **Proposal is Draft, not Accepted**: This plan is built from the proposal's design. If the proposal is rejected or substantially revised, this plan must be updated. Mitigation: changes are additive and isolated to two sections.
- **Step renumbering in impl spec**: Other projex or documents may reference "Step 3: Macro Collection" by name or number. Mitigation: references are by section title (which is preserved), not number.

### Open Questions

(These are inherited from the proposal and explicitly out of scope for this plan — they do not block execution.)

- Should `${}` interpolation extend beyond `#embed` to macro arguments and other directives?
- Should runtime auto-inject `#embed?` for locale, or must authors always write it?
- Should `#embed?` support fallback paths (`|| "fallback.zoh"`)?
- Should additional built-in variables be provided (`${filepath}`, `${directory}`)?
