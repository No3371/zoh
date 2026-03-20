# Update Impl Docs to Match Spec Channel Changes (Finding 1 - Part A)

> **Status:** Complete
> **Completed:** 2026-02-07
> **Walkthrough:** [Link](./20260207-channel-racecond-walkthrough.md)
> **Author:** Agent
> **Source:** [20260207-spec-impl-redteam.md](./20260207-spec-impl-redteam.md) - Finding 1
> **Related Projex:** [20260207-channel-racecond-csharp-plan.md](../../csharp/projex/closed/20260207-channel-racecond-csharp-plan.md) (Part B)

---

## Summary

Update `impl/08_concurrency.md` to match the spec.md changes that address channel race conditions. The spec now has `/open` verb, `/push` no longer auto-creates channels, and channel generations are documented.

**Scope:** Documentation updates to `impl/08_concurrency.md` only
**Estimated Changes:** 1 file, ~4 sections

---

## Objective

### Problem / Gap / Need

Spec.md has been updated with:
1. New `/open` verb to explicitly create/recreate channels
2. `/push` no longer auto-creates channels
3. Channel generations documented explicitly

But `impl/08_concurrency.md` still shows the old behavior where `/push` auto-creates channels.

### Success Criteria
- [ ] `/open` verb section added to impl/08
- [ ] `/push` behavior updated to NOT auto-create
- [ ] `/push` returns error when channel doesn't exist or is closed
- [ ] Generation ID documentation aligns with spec language

### Out of Scope
- C# runtime implementation changes (separate plan)
- Other concurrency changes
- Changes to other impl files

---

## Context

### Current State

In `impl/08_concurrency.md` lines 381-419, `/push` behavior says:
```
### Behavior
- **Creates a new channel** if it doesn't exist or is closed
```

And the pseudocode shows auto-creation logic.

### Key Files
| File | Purpose | Changes Needed |
|------|---------|----------------|
| `impl/08_concurrency.md` | Concurrency implementation guide | Add `/open`, update `/push`, align generations |

### Dependencies
- **Requires:** spec.md changes (already done by user)
- **Blocks:** C# implementation plan

### Constraints
- Must match spec.md exactly
- Follow existing impl documentation style

---

## Implementation

### Overview

1. Add new `/open` verb section before `/push`
2. Update `/push` to error on missing/closed channel
3. Verify generation ID documentation is complete

### Step 1: Add Channel.Open Verb Section

**Objective:** Add `/open` verb documentation following the pattern of other verbs

**Files:**
- `impl/08_concurrency.md`

**Changes:**

Insert new section at line ~380 (before Push section):

```markdown
## Open

**Purpose**: Create a new channel or re-create a closed channel.

### Signature
```
/open channel;
```

### Behavior

- Creates a new channel if it doesn't exist
- If channel exists and is closed, creates a new channel with incremented generation
- If channel exists and is open, no-op (returns success)

### Implementation

```
OpenDriver.execute(call, context):
    channelRef = resolve(call.params[0], context)
    
    if channelRef is not ChannelValue:
        return fatal("invalid_type", "Expected channel, got: " + channelRef.getType())
    
    channelName = channelRef.name
    
    # Check if channel exists
    if context.runtime.channels.exists(channelName):
        channel = context.runtime.channels.get(channelName)
        if channel.closed:
            # Create new channel with incremented generation
            context.runtime.channels.remove(channelName)
            context.runtime.channels.create(channelName)
    else:
        # Create new channel
        context.runtime.channels.create(channelName)
    
    return ok()
```

---
```

**Rationale:** Follows the same documentation pattern as other verb sections in the file.

**Verification:** Visually inspect that the section follows the same structure as Push/Pull/Close.

---

### Step 2: Update Push Behavior

**Objective:** Change `/push` to error on missing/closed channels instead of auto-creating

**Files:**
- `impl/08_concurrency.md`

**Changes:**

Update lines 390-419 from:

```markdown
### Behavior

- **Creates a new channel** if it doesn't exist or is closed

### Implementation

```
PushDriver.execute(call, context):
    channelRef = resolve(call.params[0], context)
    value = resolve(call.params[1], context)
    
    if channelRef is not ChannelValue:
        return fatal("invalid_type", "Expected channel, got: " + channelRef.getType())
    
    channelName = channelRef.name
    
    # get() auto-creates channel if it doesn't exist
    # If channel exists but is closed, create a new one
    if context.runtime.channels.exists(channelName):
        channel = context.runtime.channels.get(channelName)
        if channel.closed:
            context.runtime.channels.remove(channelName)
            channel = context.runtime.channels.get(channelName)  # Creates new
    else:
        channel = context.runtime.channels.get(channelName)  # Creates new
    
    channel.push(value)
    
    return ok()
```
```

To:

```markdown
### Behavior

- Pushes value to existing open channel
- **Errors if channel doesn't exist or is closed**

### Diagnostics

- Fatal: `invalid_type` — Parameter is not a channel
- Error: `not_found` — Channel does not exist
- Error: `closed` — Channel is closed

### Implementation

```
PushDriver.execute(call, context):
    channelRef = resolve(call.params[0], context)
    value = resolve(call.params[1], context)
    
    if channelRef is not ChannelValue:
        return fatal("invalid_type", "Expected channel, got: " + channelRef.getType())
    
    channelName = channelRef.name
    
    # Check if channel exists
    if not context.runtime.channels.exists(channelName):
        return error("not_found", "Channel does not exist: " + channelName)
    
    channel = context.runtime.channels.get(channelName)
    
    # Check if channel is closed
    if channel.closed:
        return error("closed", "Cannot push to closed channel: " + channelName)
    
    channel.push(value)
    
    return ok()
```
```

**Rationale:** Aligns with spec.md change that removed auto-creation from `/push`.

**Verification:** Compare with updated spec.md to ensure consistency.

---

### Step 3: Verify Generation ID Documentation

**Objective:** Ensure generation IDs are documented consistently

**Files:**
- `impl/08_concurrency.md`

**Changes:**

The current documentation at lines 371-377 and in the Channel struct (line 332) already mentions generation IDs, but add a clarifying note:

At line ~377, add:

```markdown
> [!NOTE]
> When `/open` re-creates a closed channel, the new channel has `generation = oldGeneration + 1`.
> This ensures that any `<channel>` references captured before the close will fail on the
> generation check during `/pull`.
```

**Rationale:** Makes the generation increment behavior explicit.

**Verification:** Read the entire Channel section to ensure consistency.

---

## Verification Plan

### Manual Verification
- [ ] Open `impl/08_concurrency.md` and verify new `/open` section exists
- [ ] Verify `/push` no longer mentions auto-creation
- [ ] Verify generation increment behavior is documented
- [ ] Compare structure to spec.md Channel sections for consistency

### Acceptance Criteria Validation
| Criterion | How to Verify | Expected Result |
|-----------|---------------|-----------------|
| `/open` section added | View impl/08 | Section exists with correct pseudocode |
| `/push` updated | View impl/08 | No auto-creation, errors on missing/closed |
| Generations documented | View impl/08 | Clear explanation of generation increment |

---

## Rollback Plan

If changes cause confusion:

1. `git checkout -- impl/08_concurrency.md`
2. Review spec.md changes with user

---

## Notes

### Assumptions
- spec.md changes are authoritative
- impl guides should mirror spec semantics

### Risks
- None significant (documentation only)

### Open Questions
- (none)
