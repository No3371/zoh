# Runtime-Scoped Flags — Spec Changes

> **Status:** In Progress
> **Created:** 2026-03-13
> **Author:** Agent
> **Source:** 20260313-runtime-scoped-flags-proposal.md
> **Related Projex:** 20260313-runtime-scoped-flags-proposal.md, 20260313-embed-variable-locale-flag-proposal.md, 20260313-runtime-scoped-flags-impl-plan.md

---

## Summary

Add runtime-scoped flags to the ZOH language spec. Flags gain a second scope (runtime) alongside the existing context scope, with a reading fallback chain and `[scope]` attribute on `/flag`. The `locale` standard flag is added. Preprocessors gain access to runtime flags.

**Scope:** Spec files only (`spec/1_concepts.md`, `spec/2_verbs.md`, `spec/std_flags.md`, `spec/3_runtime.md`)
**Estimated Changes:** 4 files

---

## Objective

### Problem / Gap / Need

Flags exist only at context scope. The preprocessor runs before any context exists, so it cannot access flag values. This creates a phase gap: values like `locale` need to be available at both preprocess time (for `#embed` path resolution) and runtime (for verb drivers). Without runtime-scoped flags, a separate "preprocessor variable" concept would be needed.

More broadly, some flags describe the environment (locale, platform, debug), not any individual context.

### Success Criteria

- [ ] `spec/1_concepts.md` documents flag scoping (runtime and context) with fallback semantics
- [ ] `spec/2_verbs.md` `/flag` verb accepts `[scope: "runtime"/"context"]` attribute, defaults to `"context"`
- [ ] `spec/std_flags.md` includes `locale` standard flag
- [ ] `spec/3_runtime.md` documents runtime flag storage, preprocessor access, and flag resolution
- [ ] All changes are backwards compatible — existing `/flag` calls unchanged

### Out of Scope

- Implementation spec changes (`impl/` files) — covered by 20260313-runtime-scoped-flags-impl-plan.md
- `${}` interpolation syntax in `#embed` paths — covered by the embed proposal
- `#embed?` optional embed form — covered by the embed proposal
- C# runtime implementation

---

## Context

### Current State

**Flags today:** Context-scoped only. `/flag "name", value;` sets a named value visible to all verb drivers in the current context. Flags are copied to forked contexts. No `[scope]` attribute exists.

**Variable scoping (model to follow):** Variables have two scopes — story (default) and context — selected via `[scope]` attribute on `/set`. Story variables shadow context variables. Same pattern applies to `/drop` and `/defer`.

**Preprocessor contract:** Preprocessors receive story name, metadata entries, and story body text. No access to runtime state or flags.

### Key Files

| File | Role | Change Summary |
|------|------|----------------|
| `spec/1_concepts.md` | Core language concepts | Add `# Flags` section with scoping semantics |
| `spec/2_verbs.md` | Verb definitions | Update Core.Flag with `[scope]` attribute and fallback |
| `spec/std_flags.md` | Standard flag definitions | Add `locale` flag |
| `spec/3_runtime.md` | Runtime design guidance | Add flag resolution, update preprocessor contract, update state management |

### Dependencies

- **Requires:** Nothing — additive spec changes
- **Blocks:** 20260313-runtime-scoped-flags-impl-plan.md (impl spec changes)
- **Blocks:** Embed variable interpolation plan (consumes runtime flags as preprocessor variables)

### Constraints

- Backwards compatible — existing `/flag` calls must not change behavior
- Mirror `/set`'s `[scope]` pattern for consistency
- Flag scoping should parallel variable scoping conceptually

### Assumptions

- The `## Attributes` section at `spec/1_concepts.md:162` is the last content before `# Verb` at line 181
- The Core.Flag verb definition at `spec/2_verbs.md:929-945` has not been modified since last read
- `spec/std_flags.md` ends at line 18 with the Pace flag
- `spec/3_runtime.md` has 70 lines total

### Impact Analysis

- **Direct:** 4 spec files modified
- **Adjacent:** `/flag` verb drivers must honor the new `[scope]` attribute and reading fallback
- **Downstream:** Embed proposal depends on preprocessor access to runtime flags; impl plan depends on this spec

---

## Implementation

### Overview

Four spec files updated in sequence. Each step is independent (no ordering dependency between files), but listed in conceptual order: concept definition → verb syntax → standard flags → runtime design.

### Step 1: Add Flags Section to `spec/1_concepts.md`

**Objective:** Define the flag scoping concept parallel to variable scoping
**Confidence:** High
**Depends on:** None

**Files:**
- `spec/1_concepts.md`

**Changes:**

Insert a new `# Flags` top-level section between the end of `## Attributes` (line 180) and `# Verb` (line 181):

```markdown
# Flags

Flags are named values visible to all verb drivers. Unlike variables, flags are not referenced in expressions or passed as parameters — they configure verb behavior.

Flags are stored in either of 2 scopes, runtime and context:
- Runtime: flags set by the runtime API or `/flag [scope: "runtime"]`. Persist for the runtime's lifetime. Shared across all contexts. Available to preprocessors.
- Context: flags set by `/flag` (default). Persist for the context's lifetime. Copied to forked contexts.

The runtime should lookup flags in context scope first, then runtime scope. Context flags shadow runtime flags of the same name.

Flag names are case-insensitive.
```

**Rationale:** Mirrors the structure of the Variables section (lines 58-72) which similarly defines two scopes with shadowing. Placed at the same `#` heading level as Variables, immediately before Verb, since flags are a parallel storage concept.

**Verification:** Read the file and confirm the section flows naturally between Attributes and Verb.

**If this fails:** Remove the inserted section; no other changes depend on its exact wording.

---

### Step 2: Update Core.Flag Verb in `spec/2_verbs.md`

**Objective:** Add `[scope]` attribute and document reading fallback
**Confidence:** High
**Depends on:** None

**Files:**
- `spec/2_verbs.md`

**Changes:**

Replace lines 929-945:

```
#### Core.Flag
A flag verb set named parameters for the context that is visible to all verb drivers.

Context flags are copied to forks.

#### Parameters
- `name`: the name of the flag. Accept `"string"` or `*"string"`.
- `value`: the value of the flag. Accept `any`. In case of reference, it takes the value of the reference.

#### Returns
A nothing.

#### Examples
```
/flag "flag_name", value;
/flag [attr] "flag_name", *value;
```
```

With:

```
#### Core.Flag
Set a named flag visible to all verb drivers.

Flags exist at two scopes:
- **Context** (default): Visible within the current context. Copied to forks.
- **Runtime**: Visible to all contexts and the preprocessor. Persists for the runtime's lifetime.

When reading a flag, the runtime checks context scope first, then runtime scope. Context flags shadow runtime flags of the same name.

#### Parameters
- `name`: the name of the flag. Accept `"string"` or `*"string"`.
- `value`: the value of the flag. Accept `any`. In case of reference, it takes the value of the reference.

#### Attributes
- **scope** (string: `context`/`runtime`): the scope to set the flag to. Defaults to `context` if not specified. Accept `"string"`.

#### Returns
A nothing.

#### Examples
```
/flag "interactive", false;
/flag [scope: "runtime"] "locale", "fr";
/flag [scope: "context"] "instant", true;
```
```

**Rationale:** Mirrors `/set`'s `[scope]` attribute pattern (line 24). Default is `context` (backwards compatible — existing `/flag` calls set context scope).

**Verification:** Confirm the `[scope]` attribute definition matches the pattern used by `/set` at line 24.

**If this fails:** Revert to original text at lines 929-945.

---

### Step 3: Add `locale` Standard Flag to `spec/std_flags.md`

**Objective:** Define `locale` as a standard flag
**Confidence:** High
**Depends on:** None

**Files:**
- `spec/std_flags.md`

**Changes:**

Append after the Pace section (after line 18):

```markdown

## Locale
The active locale identifier (BCP 47: `"fr"`, `"ja"`, `"pt-BR"`). Accept `string` or `*string`. Default to `""` (empty — no locale).

Typically set at runtime scope. Available to preprocessors for path interpolation and to all contexts for locale-aware verb behavior.
```

**Rationale:** Follows the existing format (heading, one-line purpose, type, default, usage note). Runtime scope recommendation aligns with `locale` being an environment property.

**Verification:** Confirm the entry follows the same format as Interactive, Instant, and Pace.

**If this fails:** Remove the appended section.

---

### Step 4: Update `spec/3_runtime.md`

**Objective:** Document runtime flag storage, flag resolution, and preprocessor access to runtime flags
**Confidence:** High
**Depends on:** None

**Files:**
- `spec/3_runtime.md`

**Changes:**

**4a. Add Flag Resolution section after Variable Resolution (after line 13):**

```markdown

## Flag Resolution

Flags are named values visible to all verb drivers. They exist at two scopes:

- **Runtime scope**: Set by the runtime API before story loading, or by `/flag [scope: "runtime"]`. Persists for the runtime's lifetime. Shared across all contexts. Available to preprocessors.
- **Context scope**: Set by `/flag` (default). Persists for the context's lifetime. Copied to forked contexts.

The runtime should lookup flags in context scope first, then runtime scope. Context flags shadow runtime flags of the same name.
```

**4b. Update preprocessor description (line 25):**

Replace:
```
They are provided with the story name, the metadata entries and the story body text, and can temper them in anyway.
```

With:
```
They are provided with the story name, the metadata entries, the story body text, and runtime-scoped flags, and can temper them in anyway.
```

**4c. Update State Management (line 64):**

Replace:
```
Handlers should be stateless and concurrent safe. State should only exists in runtime and context.
```

With:
```
Handlers should be stateless and concurrent safe. State should only exist in runtime and context. Runtime state includes compiled stories, contexts, channels, signals, and runtime-scoped flags.
```

**Rationale:**
- 4a parallels Variable Resolution (lines 7-13) at the same heading level
- 4b makes preprocessor access to runtime flags explicit in the contract
- 4c clarifies what "runtime state" includes, fixing a typo ("exists" → "exist") in passing

**Verification:** Read the updated file and confirm Flag Resolution flows naturally after Variable Resolution, and that the preprocessor description now mentions runtime flags.

**If this fails:** Revert each sub-change independently.

---

## Verification Plan

### Manual Verification

- [ ] Read each modified file end-to-end to confirm changes are coherent
- [ ] Confirm `/flag` attribute definition mirrors `/set`'s `[scope]` pattern
- [ ] Confirm `locale` entry in `std_flags.md` follows existing format
- [ ] Confirm no existing spec text contradicts the new scoping model
- [ ] Search for other references to "flag" across spec files to ensure consistency

### Acceptance Criteria Validation

| Criterion | How to Verify | Expected Result |
|-----------|---------------|-----------------|
| Flag scoping in concepts | Read `1_concepts.md` Flags section | Two scopes documented with fallback |
| `/flag` `[scope]` attribute | Read `2_verbs.md` Core.Flag | Attribute defined, defaults to `context` |
| `locale` standard flag | Read `std_flags.md` | Entry present with BCP 47 format |
| Runtime flag storage | Read `3_runtime.md` Flag Resolution | Section present after Variable Resolution |
| Preprocessor access | Read `3_runtime.md` Pre-processor | Mentions runtime-scoped flags |
| Backwards compatibility | Grep for existing `/flag` examples | No syntax changes to existing usage |

---

## Rollback Plan

Each step modifies a different file. Revert any file independently via `git checkout -- spec/<file>`.

---

## Notes

### Risks
- **Conceptual overlap between Flags section in `1_concepts.md` and description in `2_verbs.md`**: Mitigated by keeping `1_concepts.md` focused on the scoping model and `2_verbs.md` focused on verb syntax/behavior.

### Open Questions
- (none — all resolved in the proposal)
