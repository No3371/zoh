# Core Verb Namespace Restructure

> **Status:** Ready
> **Created:** 2026-03-20
> **Author:** Claude
> **Source:** Direct request
> **Worktree:** No

---

## Summary

Move all core verbs under a unified `core` top-level namespace with logical sub-namespaces. Currently, verbs use a flat `Core.X` naming with `Store.*`, `Channel.*` as separate top-level namespaces and `Sleep`/`Signal` having no namespace at all. After this change, every core verb has a canonical three-level name like `core.store.write` or `core.var.set`.

**Scope:** `spec/2_verbs.md` only — spec headings, document structure, and group organization.
**Estimated Changes:** 1 file, ~47 heading renames + document reorganization.

---

## Objective

### Problem / Gap / Need

Core verb naming is inconsistent:
- 36 verbs use flat `Core.X`
- 4 use `Store.X` (separate top-level)
- 4 use `Channel.X` (separate top-level)
- 2 have no namespace (`Sleep`, `Signal`)
- Debug verbs lack individual headings (`### Debug Verbs` section)

There's no logical grouping — variables, flow control, persistence, and channels are interleaved throughout the document.

### Success Criteria

- [ ] Every core verb heading follows the pattern `Core.{Group}.{Name}`
- [ ] Document is organized into h3 group sections, each containing h4 verb definitions
- [ ] No orphan verbs without a namespace group
- [ ] Debug verbs (Info, Warning, Error, Fatal) have individual h4 headings
- [ ] `Core.Flag` is promoted to a full verb entry in its group

### Out of Scope

- C# implementation changes (separate plan, `csharp/projex/`)
- Standard verbs (`Std.*`) — unchanged
- Other spec files (no cross-references to formal verb names exist)
- CLAUDE.md quick reference table (updated separately)
- Script-level syntax — `/set;`, `/write;` etc. still work via suffix resolution

---

## Context

### Current State

`spec/2_verbs.md` has a single `## Core Verbs` h2 section. All 47 verbs are h3 headings (`### Core.Set`, `### Store.Write`, etc.) in a loosely ordered flat list. `Core.Flag` (line 930) is a stray h4 sub-section under `Core.Call`.

### Key Files

| File | Role | Change Summary |
|------|------|----------------|
| `spec/2_verbs.md` | Core verb specification | Rename all headings, restructure into grouped sections |

### Dependencies

- **Requires:** Nothing
- **Blocks:** C# implementation namespace alignment plan

### Constraints

- Heading renames must preserve all content within each verb definition (parameters, diagnostics, returns, attributes, examples)
- Group ordering should follow conceptual dependency (variables before flow, flow before navigation)

### Assumptions

- The existing verb body content (parameters, diagnostics, examples) contains no formal `Core.X` name references that need updating (verified — only headings use formal names)
- `Core.Flag` (line 930) is a standalone verb that belongs in `core.var`, not a sub-feature of `Core.Call`

### Impact Analysis

- **Direct:** `spec/2_verbs.md` heading renames and restructuring
- **Adjacent:** C# `IVerbDriver.Namespace` values will need a follow-up plan
- **Downstream:** CLAUDE.md quick reference should be updated to reflect new canonical names

---

## Implementation

### Overview

Single step: restructure `spec/2_verbs.md` by introducing h3 group sections under the existing `## Core Verbs` h2, moving verb definitions into their groups as h4 headings, and renaming each heading to the three-level `Core.{Group}.{Name}` pattern.

### Step 1: Restructure and Rename

**Objective:** Reorganize the entire `## Core Verbs` section with grouped sub-sections and renamed headings.

**Confidence:** High

**Depends on:** None

**Files:**
- `spec/2_verbs.md`

**Changes:**

The `## Core Verbs` section is restructured into 12 h3 groups. Each verb heading changes from h3 (`###`) to h4 (`####`) and gains its group prefix. Verb body content is preserved verbatim — only the heading line changes.

**Group order and heading renames:**

#### 1. `### Variables (core.var)`

| Current Heading (h3) | New Heading (h4) |
|---|---|
| `### Core.Set` | `#### Core.Var.Set` |
| `### Core.Get` | `#### Core.Var.Get` |
| `### Core.Drop` | `#### Core.Var.Drop` |
| `### Core.Capture` | `#### Core.Var.Capture` |
| `### Core.Type` | `#### Core.Var.Type` |
| `### Core.Count` | `#### Core.Var.Count` |
| `#### Core.Flag` | `#### Core.Var.Flag` |
| `### Core.Parse` | `#### Core.Var.Parse` |

#### 2. `### Evaluation (core.eval)`

| Current Heading (h3) | New Heading (h4) |
|---|---|
| `### Core.Evaluate` | `#### Core.Eval.Evaluate` |
| `### Core.Interpolate` | `#### Core.Eval.Interpolate` |

#### 3. `### Control Flow (core.flow)`

| Current Heading (h3) | New Heading (h4) |
|---|---|
| `### Core.Do` | `#### Core.Flow.Do` |
| `### Core.If` | `#### Core.Flow.If` |
| `### Core.Sequence` | `#### Core.Flow.Sequence` |
| `### Core.Loop` | `#### Core.Flow.Loop` |
| `### Core.While` | `#### Core.Flow.While` |
| `### Core.Foreach` | `#### Core.Flow.Foreach` |
| `### Core.Switch` | `#### Core.Flow.Switch` |
| `### Core.Exit` | `#### Core.Flow.Exit` |

#### 4. `### Navigation (core.nav)`

| Current Heading (h3) | New Heading (h4) |
|---|---|
| `### Core.Jump` | `#### Core.Nav.Jump` |
| `### Core.Fork` | `#### Core.Nav.Fork` |
| `### Core.Call` | `#### Core.Nav.Call` |

#### 5. `### Collections (core.collection)`

| Current Heading (h3) | New Heading (h4) |
|---|---|
| `### Core.Append` | `#### Core.Collection.Append` |
| `### Core.Insert` | `#### Core.Collection.Insert` |
| `### Core.Remove` | `#### Core.Collection.Remove` |
| `### Core.Clear` | `#### Core.Collection.Clear` |
| `### Core.Has` | `#### Core.Collection.Has` |
| `### Core.Any` | `#### Core.Collection.Any` |
| `### Core.First` | `#### Core.Collection.First` |

#### 6. `### Math (core.math)`

| Current Heading (h3) | New Heading (h4) |
|---|---|
| `### Core.Increase` | `#### Core.Math.Increase` |
| `### Core.Decrease` | `#### Core.Math.Decrease` |
| `### Core.Rand` | `#### Core.Math.Rand` |
| `### Core.Roll` | `#### Core.Math.Roll` |
| `### Core.WRoll` | `#### Core.Math.WRoll` |

#### 7. `### Persistence (core.store)`

| Current Heading (h3) | New Heading (h4) |
|---|---|
| `### Store.Write` | `#### Core.Store.Write` |
| `### Store.Read` | `#### Core.Store.Read` |
| `### Store.Erase` | `#### Core.Store.Erase` |
| `### Store.Purge` | `#### Core.Store.Purge` |

#### 8. `### Channels (core.channel)`

| Current Heading (h3) | New Heading (h4) |
|---|---|
| `### Channel.Open` | `#### Core.Channel.Open` |
| `### Channel.Push` | `#### Core.Channel.Push` |
| `### Channel.Pull` | `#### Core.Channel.Pull` |
| `### Channel.Close` | `#### Core.Channel.Close` |

#### 9. `### Signals (core.signal)`

| Current Heading (h3) | New Heading (h4) |
|---|---|
| `### Core.Wait` | `#### Core.Signal.Wait` |
| `### Signal` | `#### Core.Signal.Signal` |
| `### Sleep` | `#### Core.Signal.Sleep` |

#### 10. `### Error Handling (core.error)`

| Current Heading (h3) | New Heading (h4) |
|---|---|
| `### Core.Try` | `#### Core.Error.Try` |
| `### Core.Defer` | `#### Core.Error.Defer` |

#### 11. `### Debug (core.debug)`

| Current Heading (h3) | New Heading (h4) |
|---|---|
| `### Core.Diagnose` | `#### Core.Debug.Diagnose` |
| `### Debug Verbs` (section) | Split into 4 individual headings: |
| — Info | `#### Core.Debug.Info` |
| — Warning | `#### Core.Debug.Warning` |
| — Error | `#### Core.Debug.Error` |
| — Fatal | `#### Core.Debug.Fatal` |
| `### Core.Assert` | `#### Core.Debug.Assert` |

**Rationale:** Three-level naming (`Core.Group.Verb`) provides logical grouping while the existing suffix-based namespace resolution ensures scripts still work with short names (`/set;`, `/write;`). Document reorganization by group makes the spec navigable.

**Verification:**
- Count h4 headings matching `#### Core\.\w+\.\w+` — should be 47
- Every verb body (Parameters, Diagnostics, Returns, Attributes, Examples subsections) preserved
- No orphan content between groups

**If this fails:** Revert the single file via `git checkout -- spec/2_verbs.md`

---

## Verification Plan

### Automated Checks

- [ ] Every h4 heading under `## Core Verbs` matches `#### Core.{Group}.{Name}` pattern
- [ ] Total verb count is 47 (46 current + Debug split into 4 individual - 1 Debug section + Flag promoted = 47... verify exact count during execution)
- [ ] No h3 headings remain that aren't group headers
- [ ] Markdown renders correctly (no broken heading hierarchy)

### Manual Verification

- [ ] Read through each group section — verbs are logically placed
- [ ] Verb body content unchanged (spot-check 5-6 verbs across different groups)
- [ ] Debug Verbs section correctly decomposed into individual verb entries

### Acceptance Criteria Validation

| Criterion | How to Verify | Expected Result |
|-----------|---------------|-----------------|
| Three-level naming | `grep -c '#### Core\.' spec/2_verbs.md` | 47 |
| Group sections | `grep -c '### ' spec/2_verbs.md` under Core Verbs | 12 |
| No orphans | Diff old vs new — every verb block accounted for | Zero missing |
| Debug individuals | Search for `Core.Debug.Info`, `.Warning`, `.Error`, `.Fatal` | 4 matches |

---

## Rollback Plan

1. `git checkout -- spec/2_verbs.md`

---

## Notes

### Risks

- **Heading count mismatch**: The Debug Verbs section (line 1252) is currently a single section with all 4 levels described together. Splitting into 4 individual entries requires understanding the current structure — verify during execution.
- **Core.Flag placement**: Currently nested as h4 under Core.Call area (line 930). Confirm it's a standalone verb definition before moving to `core.var`.

### Open Questions

- (none)
