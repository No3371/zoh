# Timeout Verb Consistency Fixes

> **Status:** Ready
> **Created:** 2026-03-20
> **Author:** agent
> **Source:** Direct request
> **Related Projex:** 2603201600-timeout-verb-behavior-explore.md
> **Worktree:** No

---

## Summary

Closes spec gaps across all seven timeout verbs: `Core.Wait` lacks a default and Diagnostics section; the three core verbs are missing `?` acceptance and the `<= 0` immediate-poll rule; the four std verbs already accept `?` but are missing the default and `<= 0` rule. Two files change, seven verb sections touched.

**Scope:** `spec/2_verbs.md` and `spec/std_verbs.md` — all verbs with a `timeout` parameter.
**Estimated Changes:** 2 files, 7 sections.

---

## Objective

### Problem / Gap / Need

Seven verbs across two files accept a `timeout` parameter but their specs are inconsistent and incomplete:

**`spec/2_verbs.md`** — `Core.Wait`, `Channel.Push`, `Channel.Pull`:
1. **`Core.Wait`** — `timeout` has no stated default, no `Optional` marker, and no Diagnostics section.
2. **All three** — `timeout` type declaration excludes `?` even though `?` is the intended default (no timeout).
3. **All three** — no rule exists for `timeout <= 0`; implementors have no guidance on whether zero means "immediate poll" or is an error.

**`spec/std_verbs.md`** — `Std.Converse`, `Std.Choose`, `Std.ChooseFrom`, `Std.Prompt`:
4. **All four** — already accept `?` in type declaration, but are missing `Default to ?` and the `<= 0` rule.

### Success Criteria
- [ ] All seven verbs state `Optional. Default to ?` with the meaning "no timeout"
- [ ] All seven state that `timeout <= 0` triggers an immediate timeout (poll)
- [ ] The three core verbs explicitly accept `double`, `*double`, or `?` for `timeout`
- [ ] `Core.Wait` has a Diagnostics section listing `Info: timeout`
- [ ] No other verb behavior is changed

### Out of Scope
- Runtime/C# implementation
- `Sleep` (no timeout parameter)
- `1_concepts.md` — the global `Info: timeout` diagnostic definition is already correct
- Returns section wording for `Channel.Pull` (diagnostics are the canonical signal)

---

## Context

### Current State

The three core verbs in `spec/2_verbs.md` use `Accept \`double\` or \`*double\`` for `timeout` with no `?` acceptance. `Core.Wait` additionally has no `Optional`/`Default` annotation and no Diagnostics section. The four std verbs in `spec/std_verbs.md` already accept `?` but all four are missing `Default to ?` and the `<= 0` rule.

### Key Files

| File | Role | Change Summary |
|------|------|----------------|
| `spec/2_verbs.md` | Core verb spec | 3 sections updated (`Core.Wait`, `Channel.Push`, `Channel.Pull`) |
| `spec/std_verbs.md` | Std verb spec | 4 sections updated (`Std.Converse`, `Std.Choose`, `Std.ChooseFrom`, `Std.Prompt`) |

### Dependencies
- **Requires:** Nothing
- **Blocks:** Nothing

### Constraints
- Wording must stay consistent across all seven verbs — use identical phrasing for the shared rules.
- `std_verbs.md` uses `double`/`*double` slash notation (not `double` or `*double`) — preserve that style when adding the new sentences.

### Assumptions
- `?` passed explicitly for `timeout` is semantically equivalent to omitting it (no timeout).
- `timeout <= 0` means "check now, if nothing available, timeout immediately" — poll semantics, not an error.

---

## Implementation

### Overview

Steps 1–3 edit `spec/2_verbs.md` (one per verb); Step 4 edits `spec/std_verbs.md` (all four std verbs share identical wording so one before/after covers all four lines). `Core.Wait` requires an additional block insertion (Diagnostics). All other content in each section is untouched.

---

### Step 1: `Core.Wait` — align timeout parameter and add Diagnostics

**Objective:** Add `?` to accepted types, add Optional/Default/behavior annotation, insert Diagnostics section.
**Confidence:** High
**Depends on:** None

**Files:**
- `spec/2_verbs.md`

**Changes:**

Line 977 — timeout parameter declaration:

```markdown
// Before:
- `timeout`: The timeout in seconds. Accept `double` or `*double`.

// After:
- `timeout`: The timeout in seconds. Accept `double`, `*double`, or `?`. Optional. Default to `?`. A value of `?` means no timeout (blocks indefinitely). A value of `0` or less triggers an immediate timeout.
```

Lines 984–985 — insert Diagnostics block between Returns and Examples:

```markdown
// Before:
#### Returns
The message received. Could be `integer`, `double`, `boolean`, `string`, `list`, `map`. If the timeout is reached, returns a nothing.

#### Examples

// After:
#### Returns
The message received. Could be `integer`, `double`, `boolean`, `string`, `list`, `map`. If the timeout is reached, returns a nothing.

#### Diagnostics
- Info: `timeout`: The timeout was reached.

#### Examples
```

**Rationale:** Aligns `Core.Wait` with `Channel.Pull` and `Channel.Push`, which already have Diagnostics sections. The `Info: timeout` entry is the standard diagnostic defined in `1_concepts.md:240`.

**Verification:** Read the `Core.Wait` section — it should now have the same structure as `Channel.Pull` (Named Parameters → Parameters → Returns → Diagnostics → Examples), with identical `timeout` wording.

**If this fails:** Revert lines 977 and 984–986 to original text.

---

### Step 2: `Channel.Push` — align timeout parameter

**Objective:** Add `?` to accepted types and `<= 0` rule.
**Confidence:** High
**Depends on:** None (can run in any order)

**Files:**
- `spec/2_verbs.md`

**Changes:**

Line 1027 — timeout parameter declaration:

```markdown
// Before:
- `timeout`: The timeout in seconds when `wait` is `true`. Accept `double` or `*double`. Optional. Default to `?`. Ignored when `wait` is `false`.

// After:
- `timeout`: The timeout in seconds when `wait` is `true`. Accept `double`, `*double`, or `?`. Optional. Default to `?`. A value of `?` means no timeout (blocks indefinitely). A value of `0` or less triggers an immediate timeout. Ignored when `wait` is `false`.
```

**Rationale:** Adds the two missing rules. The rest of the Push section (rendezvous semantics, Diagnostics, examples) is correct and unchanged.

**Verification:** `timeout` line for Push contains `or \`?\`` and the two behavior sentences.

**If this fails:** Revert line 1027 to original text.

---

### Step 3: `Channel.Pull` — align timeout parameter

**Objective:** Add `?` to accepted types and `<= 0` rule.
**Confidence:** High
**Depends on:** None (can run in any order)

**Files:**
- `spec/2_verbs.md`

**Changes:**

Line 1058 — timeout parameter declaration:

```markdown
// Before:
- `timeout`: The timeout in seconds. Accept `double` or `*double`. Optional. Default to `?`.

// After:
- `timeout`: The timeout in seconds. Accept `double`, `*double`, or `?`. Optional. Default to `?`. A value of `?` means no timeout (blocks indefinitely). A value of `0` or less triggers an immediate timeout.
```

**Rationale:** Matches the wording of `Core.Wait` (Step 1) exactly — identical phrasing enforces a single consistent rule across all timeout verbs.

**Verification:** `timeout` line for Pull is identical to the `Core.Wait` version (excluding the "when `wait` is `true`" and "Ignored when `wait` is `false`" clauses that are Push-specific).

**If this fails:** Revert line 1058 to original text.

---

### Step 4: `Std.Converse`, `Std.Choose`, `Std.ChooseFrom`, `Std.Prompt` — add default and `<= 0` rule

**Objective:** Add `Default to ?` and `<= 0` behavior to all four std verb timeout lines.
**Confidence:** High
**Depends on:** None (can run in any order)

**Files:**
- `spec/std_verbs.md`

**Changes:**

Lines 12, 52, 88, 120 — all four use identical text; apply the same change to each:

```markdown
// Before:
- `timeout`: the duration in seconds to wait before timing out. Accept `double`/`*double` or `?`. Optional.

// After:
- `timeout`: the duration in seconds to wait before timing out. Accept `double`/`*double` or `?`. Optional. Default to `?`. A value of `?` means no timeout. A value of `0` or less triggers an immediate timeout.
```

**Rationale:** These verbs already accept `?` (ahead of the core verbs) but are missing the two behavioral rules. The slash notation (`double`/`*double`) is preserved to stay consistent with the rest of `std_verbs.md`. "No timeout" is used instead of "blocks indefinitely" because whether a std verb blocks depends on its own `Wait` attribute, not the timeout alone.

**Verification:** Each of the four `timeout` lines contains `Default to \`?\`` and `0\` or less triggers an immediate timeout`.

**If this fails:** Revert lines 12, 52, 88, 120 to original text.

---

## Verification Plan

### Manual Verification
- [ ] `Core.Wait` section has: Named Parameters → Parameters → Returns → Diagnostics → Examples (matches Pull structure)
- [ ] `timeout` line in the three core verbs contains `double`, `*double`, or `?`
- [ ] `timeout` line in all seven verbs contains `Default to \`?\``
- [ ] `timeout` line in all seven verbs contains `0\` or less triggers an immediate timeout`
- [ ] No other lines in any of the seven sections changed

### Acceptance Criteria Validation

| Criterion | How to Verify | Expected Result |
|-----------|---------------|-----------------|
| `?` accepted (core) | Read `timeout` param line in `2_verbs.md` | `or \`?\`` present in all three core verbs |
| Default documented (all) | Read `timeout` param line for each of 7 verbs | `Default to \`?\`` present in all seven |
| `<= 0` rule (all) | Read `timeout` param line for each of 7 verbs | `0\` or less triggers an immediate timeout` in all seven |
| Core.Wait Diagnostics | Read Core.Wait section | Diagnostics block with `Info: \`timeout\`` before Examples |

---

## Rollback Plan

All changes are single-line edits or a small block insertion in one file. Revert by restoring the original text for each modified line as shown in the before/after above. No cascading changes.

---

## Notes

### Risks
- **Wording drift:** If the three verbs use slightly different phrasing for the same rules, implementors may read in subtle distinctions. Mitigated by copying the identical sentence for the shared rules.

### Open Questions
None.
