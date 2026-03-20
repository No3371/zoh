# Spec Channel Push Amendment Plan

> **Status:** Complete
> **Created:** 2026-02-11
> **Author:** Agent
> **Source:** `20260211-channel-inbox-outbox-proposal.md`, `20260211-channel-inbox-outbox-redteam.md`
> **Related Projex:** `20260211-channel-inbox-outbox-impl-spec-plan.md` (impl-scope sibling)

---

## Summary

Amend `spec.md` to reflect the new blocking push semantics and channel architecture: add `wait` and `timeout` named parameters to Channel.Push, and update the channel type definition to clarify hub-based routing with per-context outboxes/inboxes.

**Scope:** `spec.md` only — channel-related sections
**Estimated Changes:** 1 file, 3 sections

---

## Objective

### Problem / Gap / Need

The channel inbox/outbox proposal adds `wait` and `timeout` named parameters to `/push` and requires `/open` before any channel operation. These are language-surface changes that must be reflected in `spec.md`.

### Success Criteria
- [ ] `Channel.Push` section has `wait` and `timeout` named parameters documented
- [ ] `Channel.Push` diagnostics include push-specific error/info codes
- [ ] Channel type definition clarifies hub-based architecture with per-context outboxes/inboxes
- [ ] Channel.Open description grammar fixed

### Out of Scope
- Implementation spec changes (`impl/08_concurrency.md`, `impl/09_runtime.md`) — covered by sibling plan
- C# runtime changes — separate scope
- Any functional behavior not already resolved in the proposal

---

## Context

### Current State

`spec.md` defines Channel.Push (lines 1466-1481) as a simple push-and-return verb with no named parameters, no blocking behavior, and no diagnostics. The channel type definition (lines 340-345) describes channels as "global pipes" pointing to "one same underlying data structure."

### Key Files

| File | Purpose | Changes Needed |
|------|---------|----------------|
| `spec.md` | Language specification | Amend Channel.Push, update channel type definition, note `/open` requirement |

### Dependencies
- **Requires:** Proposal accepted with all questions resolved
- **Blocks:** `20260211-channel-inbox-outbox-impl-spec-plan.md` (impl changes should align with spec)

### Constraints
- Spec language must remain implementation-agnostic — describe behavior, not internals
- Existing channel semantics (FIFO, concurrent-safe, generation tracking) preserved

---

## Implementation

### Overview

Three targeted edits to `spec.md`:
1. Amend Channel.Push with `wait` and `timeout` parameters
2. Update channel type definition
3. Add `/open` requirement note to Channel.Open

### Step 1: Amend Channel.Push (spec.md lines 1466-1481)

**Objective:** Add `wait` and `timeout` named parameters, blocking behavior description, and diagnostics.

**Files:**
- `spec.md`

**Changes:**

```markdown
// Before (lines 1466-1481):
### Channel.Push

A push verb pushes a variable to a channel.

#### Parameters
- `channel`: The channel to push to. Accept `<channel>` or `*<channel>`.
- `var`: The variable to push. Accept `any`. If it is a reference, the runtime takes the value.

#### Returns
A nothing.

#### Examples
` ` `
/push *channel, *var;
/push <channel>, *var;
` ` `

// After:
### Channel.Push

A push verb pushes a variable to a channel. By default, the push blocks until the value is consumed by a puller (rendezvous semantics). Use `wait: false` for fire-and-forget behavior.

#### Named Parameters
- `wait`: Whether to block until the value is consumed. Accept `boolean`. Optional. Default to `true`.
- `timeout`: The timeout in seconds when `wait` is `true`. Accept `double` or `*double`. Optional. Default to `?`. Ignored when `wait` is `false`.

#### Parameters
- `channel`: The channel to push to. Accept `<channel>` or `*<channel>`.
- `var`: The variable to push. Accept `any`. If it is a reference, the runtime takes the value.

#### Returns
A nothing.

#### Diagnostics
- Error: `not_found`: The channel does not exist.
- Error: `closed`: The channel is closed, or closed while waiting.
- Info: `timeout`: The push timed out before the value was consumed (only with `wait: true` and `timeout`).

#### Examples
` ` `
:: Blocking push (default) — waits until consumed
/push <channel>, *var;

:: Blocking push with timeout
/push <channel>, *var, timeout: 5;

:: Fire-and-forget push
/push <channel>, *var, wait: false;
` ` `
```

**Rationale:** The proposal resolves `wait: true` as the default (ZOH is a new language, no backward compat concern). `timeout` mirrors `/pull`'s existing `timeout` parameter. Diagnostics mirror `/pull`'s pattern.

**Verification:** Read the amended section — parameters follow the same format as Channel.Pull, diagnostics are consistent.

---

### Step 2: Update Channel Type Definition (spec.md lines 340-345)

**Objective:** Clarify that the "underlying data structure" includes a routing hub, and that `/open` is required.

**Files:**
- `spec.md`

**Changes:**

```markdown
// Before (lines 340-345):
- \<channel\>
	- A channel is a FIFO, concurrent safe, unbounded, global pipe managed by the runtime.
	- Denoted as `<channel_name>`, which servers as a pointer to the underlying data structure uniquely identified by "channel_name" in the channel-dedicated storage in the runtime.
	- No white space is allowed between `<` and `>`.
	- `<channel>` points to one same underlying data structure for any executing contexts at the same time.
	- A channel can be closed. New channel can be created with the same name, but does not point to the old channel. Internally, channels have `generation` to distinguish channels with same names.

// After:
- \<channel\>
	- A channel is a FIFO, concurrent safe, unbounded, global pipe managed by the runtime.
	- Denoted as `<channel_name>`, which serves as a pointer to the underlying channel hub uniquely identified by "channel_name" in the channel-dedicated storage in the runtime.
	- No white space is allowed between `<` and `>`.
	- `<channel>` points to one same logical channel for any executing contexts at the same time. Each context maintains its own outbox (push buffer) and inbox (pull buffer), coordinated by the channel hub. Contexts are auto-registered with the hub on first `/push` or `/pull`.
	- A channel can be closed. New channel can be created with the same name, but does not point to the old channel. Internally, channels have `generation` to distinguish channels with same names.
```

**Rationale:** Preserves "global pipe" semantics while clarifying the hub/outbox/inbox architecture. Adds `/open` requirement.

**Verification:** Read the amended definition — still describes channels as global FIFO pipes, adds architectural detail without being implementation-prescriptive.

---

### Step 3: Fix Channel.Open Description (spec.md lines 1450-1458)

**Objective:** Fix grammar in Channel.Open description.

**Files:**
- `spec.md`

**Changes:**

```markdown
// Before (lines 1450-1458):
### Channel.Open

A open verb creates a new channel or re-create a closed channel.

#### Parameters
- `channel`: The channel to push to. Accept `<channel>` or `*<channel>`.

#### Returns
A nothing.

// After:
### Channel.Open

An open verb creates a new channel or re-creates a closed channel.

#### Parameters
- `channel`: The channel to open. Accept `<channel>` or `*<channel>`.

#### Returns
A nothing.
```

**Rationale:** Grammar fix ("A open" → "An open") and parameter description fix ("push to" → "open").

**Verification:** Read the section — grammar correct, no registration language added.

---

## Verification Plan

### Automated Checks
- [ ] No automated tests for spec.md — it's a markdown document

### Manual Verification
- [ ] Read amended Channel.Push section — confirm `wait` and `timeout` parameters follow the same format as Channel.Pull
- [ ] Read amended channel type definition — confirm it still describes channels as global FIFO pipes
- [ ] Read Channel.Open — confirm grammar fixed
- [ ] Cross-reference with proposal's resolved questions — all decisions reflected
- [ ] Grep for "Channel.Push" or "/push" in spec.md — ensure no other sections contradict the new semantics

### Acceptance Criteria Validation

| Criterion | How to Verify | Expected Result |
|-----------|---------------|-----------------|
| Push has `wait` and `timeout` params | Read Channel.Push section | Named Parameters section with both params |
| Push has diagnostics | Read Channel.Push Diagnostics | `not_found`, `closed`, `timeout` codes |
| Channel type updated | Read channel type definition | Mentions hub, outbox/inbox, auto-registration |
| `/open` grammar fixed | Read Channel.Open section | Grammar corrected, no registration language |

---

## Rollback Plan

If spec changes cause confusion or are rejected:
1. Revert `spec.md` to previous content (git revert)
2. Re-open proposal for further discussion

---

## Notes

### Assumptions
- The proposal's resolved questions are final
- `/open` is for channel creation only, not context registration
- Registration is automatic on first push/pull

### Risks
- Spec language too implementation-specific: mitigated by keeping "hub" language abstract
