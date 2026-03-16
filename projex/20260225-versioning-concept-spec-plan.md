# Versioning Concept — Spec Plan

> **Status:** Ready
> **Created:** 2026-02-25
> **Author:** agent
> **Source:** Direct request
> **Related Projex:** 20260225-compiled-ir-eval.md

---

## Summary

Introduce a formal versioning concept to the zoh language spec. `zoh_version` is already a recognised metadata key in `spec/std_metadata.md` but is undefined beyond a one-line description. This plan defines the versioning scheme, what each version level means for the language, compatibility rules between story and runtime, and the accompanying diagnostics. Changes are spec-only; runtime enforcement is a downstream plan.

**Scope:** `spec/` files only — no impl or C# changes.
**Estimated Changes:** 1 new file (`spec/versioning.md`), 1 edited file (`spec/std_metadata.md`).

---

## Objective

### Problem / Gap / Need

`spec/std_metadata.md` defines `zoh_version` as:

> "State the zoh spec version the story is written for. Optional. Defaults to the latest version."

There is no:
- Defined versioning scheme (what format is a version?)
- Definition of the current version (what is "latest"?)
- Specification of what constitutes a breaking vs additive change
- Compatibility policy (what must a runtime accept/reject?)
- Diagnostic codes for version mismatch

Without these, `zoh_version` is meaningless — runtimes cannot act on it and story authors cannot use it correctly.

### Success Criteria

- [ ] `spec/versioning.md` exists and defines: versioning scheme, version level semantics (MAJOR/MINOR/PATCH), the current spec version, story-runtime compatibility rules, and version mismatch diagnostics
- [ ] `spec/std_metadata.md` `zoh_version` entry references `spec/versioning.md`, specifies the accepted format, and states the default
- [ ] The current spec version is declared (in `spec/versioning.md` and/or `spec/std_metadata.md`)
- [ ] Compatibility rules are unambiguous: given story version S and runtime version R, the rule produces exactly one outcome (run / warn / fatal)

### Out of Scope

- Impl changes to validators, preprocessors, or the runtime (downstream)
- A language changelog (separate document)
- Version-specific feature flags
- Tooling (linters, IDE plugins)
- IR versioning (handled by the IR proposal)

---

## Context

### Current State

`spec/std_metadata.md` (3-line entry):

```markdown
## `zoh_version`
State the zoh spec version the story is written for. Optional. Defaults to the latest version.
```

No other spec file mentions versioning. No format, no current version, no runtime behaviour is defined.

### Key Files

| File | Purpose | Changes Needed |
|------|---------|----------------|
| `spec/std_metadata.md` | Standard metadata schema | Expand `zoh_version` section to add format, default value, compatibility reference |
| `spec/versioning.md` | *(new)* | Create: versioning scheme, current version, level semantics, compatibility rules, diagnostics |

### Dependencies

- **Requires:** Nothing (self-contained spec addition)
- **Blocks:** Downstream runtime impl plan that enforces version checking

### Constraints

- Must not break existing valid stories: `zoh_version` is currently optional, so the absence of it must remain valid
- Must be simple enough for story authors to use: no complex version range syntax
- Must leave runtime enforcement to the impl layer (spec defines the rules, impl enforces them)

---

## Implementation

### Overview

Create `spec/versioning.md` as the canonical versioning policy document, then expand the `zoh_version` entry in `spec/std_metadata.md` to reference it and define the field format.

---

### Step 1: Create `spec/versioning.md`

**Objective:** Define the complete versioning policy for the zoh language spec.

**Files:**
- `spec/versioning.md` *(new)*

**Content:**

```markdown
# Versioning

Zoh uses [Semantic Versioning](https://semver.org): `MAJOR.MINOR.PATCH`.

## Current Version

The current zoh spec version is **1.0.0**.

---

## Version Levels

### MAJOR — Breaking changes

Increment when a change is incompatible with existing stories:

- Syntax changes that invalidate previously valid source
- Removal or renaming of core verbs or their required parameters
- Semantic changes to existing core verb behaviour
- Changes to the type system that break existing type contracts
- Changes to variable scoping rules

### MINOR — Backward-compatible additions

Increment when new capabilities are added without breaking existing stories:

- New standard verbs or attributes
- New metadata fields
- New syntactic sugar forms
- New expression special forms
- New diagnostic codes (additions only)
- Spec clarifications that do not alter behaviour for existing valid scripts

### PATCH — Corrections

Increment for non-normative spec changes:

- Corrections to examples or wording
- Clarifications that align the spec text with always-intended behaviour
- Fixes to spec inconsistencies that did not affect valid stories

---

## Story Version Declaration

Stories declare their target spec version via the `zoh_version` metadata field (see `std_metadata.md`).

```
My Story
zoh_version: "1.0.0";
===
```

---

## Runtime Version

Each runtime declares the spec version it implements. This is the runtime's *supported version* and is used to check compatibility when loading a story.

---

## Compatibility Rules

Given a story with version **S** and a runtime with supported version **R**:

| Condition | Behaviour | Diagnostic |
|-----------|-----------|------------|
| `S.MAJOR != R.MAJOR` | Fatal — story cannot run | `version_major_mismatch` |
| `S.MINOR > R.MINOR` | Warning — story may use features the runtime lacks | `version_minor_ahead` |
| `S.MINOR <= R.MINOR` | Compatible — proceed normally | *(none)* |
| PATCH difference (any) | Ignored — no compatibility concern | *(none)* |
| `zoh_version` absent | Treat story as compatible with any same-MAJOR runtime | *(none)* |

> PATCH differences are never a compatibility concern. A runtime at 1.2.3 and a story at 1.2.0 are fully compatible.

---

## Diagnostics

| Code | Level | Condition |
|------|-------|-----------|
| `version_major_mismatch` | Fatal | Story MAJOR ≠ Runtime MAJOR |
| `version_minor_ahead` | Warning | Story MINOR > Runtime MINOR |
| `version_invalid_format` | Error | `zoh_version` value does not match `MAJOR.MINOR.PATCH` format |

---

## Versioning Policy for This Document

Changes to this document increment the spec version according to the level semantics above. The version declared in the **Current Version** section is normative and must be updated in the same commit as any spec change.
```

**Rationale:** Semantic versioning is the de-facto standard. The three-level rule (MAJOR blocks, MINOR warns, PATCH ignores) is simple enough for both story authors and runtime implementors to act on without ambiguity.

**Verification:** Read the file; verify it contains a `Current Version` declaration, a table of level semantics, the compatibility rule table, and all three diagnostic codes.

---

### Step 2: Expand `spec/std_metadata.md` — `zoh_version` entry

**Objective:** Replace the one-line description with a full field spec referencing the new versioning document.

**Files:**
- `spec/std_metadata.md`

**Changes:**

```markdown
// Before:
## `zoh_version`
State the zoh spec version the story is written for. Optional. Defaults to the latest version.

// After:
## `zoh_version`

Declares the zoh spec version this story was written for.

- **Type:** `string`
- **Format:** `"MAJOR.MINOR.PATCH"` (e.g. `"1.0.0"`) or `"*"` (any version)
- **Optional.** When absent, the runtime treats the story as compatible.
- **Default:** treated as `"*"` (compatible with any runtime)

See `spec/versioning.md` for the versioning scheme, level semantics, and the current spec version.

#### Runtime behaviour

The runtime checks `zoh_version` against its supported version using the compatibility rules in `spec/versioning.md`. A `version_major_mismatch` is fatal; a `version_minor_ahead` is a warning.

#### Example

```
My Story
zoh_version: "1.0.0";
===
```
```

**Rationale:** The field entry in `std_metadata.md` should be self-contained enough to use without reading the full `versioning.md`, while delegating policy detail to the dedicated document.

**Verification:** Read `spec/std_metadata.md`; confirm the `zoh_version` entry includes type, format, default behaviour, a reference to `spec/versioning.md`, and a runtime behaviour note.

---

## Verification Plan

### Manual Verification

- [ ] `spec/versioning.md` exists and is well-formed markdown
- [ ] `spec/versioning.md` declares the current version as `1.0.0`
- [ ] The compatibility table covers all cases (MAJOR mismatch, MINOR ahead, MINOR same/behind, PATCH, absent)
- [ ] `spec/std_metadata.md` `zoh_version` entry references `spec/versioning.md`
- [ ] `spec/std_metadata.md` specifies `"*"` as the wildcard/absent default behaviour
- [ ] The three diagnostic codes (`version_major_mismatch`, `version_minor_ahead`, `version_invalid_format`) are defined in `spec/versioning.md`

### Acceptance Criteria Validation

| Criterion | How to Verify | Expected Result |
|-----------|---------------|-----------------|
| Versioning scheme defined | Read `spec/versioning.md` header | Semver MAJOR.MINOR.PATCH stated |
| Current version declared | Read `spec/versioning.md` — Current Version section | `1.0.0` |
| Compatibility rules unambiguous | Apply table to edge cases (1.0.0 story, 1.0.0 runtime; 2.0.0 story, 1.0.0 runtime; absent version) | Each maps to exactly one outcome |
| `std_metadata.md` updated | Diff `spec/std_metadata.md` | `zoh_version` section expanded, format and defaults present |

---

## Rollback Plan

1. Delete `spec/versioning.md`
2. Revert `spec/std_metadata.md` `zoh_version` to the original one-line entry
3. No other files are affected

---

## Notes

### Assumptions

- The spec is being declared as version 1.0.0 retroactively (first named version). If the project uses a different numbering convention (e.g., 0.x.x for pre-stable), update the Current Version in Step 1 accordingly.
- `"*"` as a wildcard is borrowed from `required_verbs` behaviour in `std_metadata.md` for consistency.

### Risks

- **Version inflation risk**: Defining MAJOR semantics strictly may make it hard to evolve the language. Mitigation: keep `1.0.0` as the current version; avoid premature version bumps.
- **Downstream impl dependency**: spec defines rules; enforcement requires a downstream runtime plan. Stories using `zoh_version` will have no enforcement until the impl plan executes.

### Open Questions

*(none — this plan is ready for execution)*
