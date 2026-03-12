# Runtime-Scoped Flags — Impl Spec Changes

> **Status:** Ready
> **Created:** 2026-03-13
> **Author:** Agent
> **Source:** 20260313-runtime-scoped-flags-proposal.md
> **Related Projex:** 20260313-runtime-scoped-flags-proposal.md, 20260313-runtime-scoped-flags-spec-plan.md

---

## Summary

Add runtime-level flag storage and preprocessor flag access to the implementation spec. The Runtime interface gains a `flags` map and flag operations. The story compilation pipeline passes runtime flags to preprocessors. Context flag reading falls back to runtime flags.

**Scope:** Impl spec files only (`impl/09_runtime.md`)
**Estimated Changes:** 1 file, 4 locations

---

## Objective

### Problem / Gap / Need

The Runtime interface has no flag storage. Context has `flags: Map<string, Value>` but there is no runtime-level equivalent. The preprocessor phase receives only `(source, metadata)` — no access to runtime state. The spec plan introduces runtime-scoped flags; this plan updates the implementation spec to match.

### Success Criteria

- [ ] Runtime interface includes `flags: Map<string, Value>` in its state section
- [ ] Runtime interface includes `setFlag` and `getFlag` operations
- [ ] Preprocessor phase in the pipeline passes `runtimeFlags` to preprocessors
- [ ] Context section documents flag reading fallback (context → runtime)

### Out of Scope

- Language spec changes — covered by 20260313-runtime-scoped-flags-spec-plan.md
- `${}` interpolation in embed paths — covered by embed proposal's impl plan
- Preprocessor implementation changes (`impl/03_preprocessor.md`) for `${}` syntax — covered by embed proposal
- C# runtime implementation

---

## Context

### Current State

**Runtime interface** (`impl/09_runtime.md:56-93`): State section includes `stories`, `contexts`, `channelHubs`, `storage`, `signals`, `elapsedMs`. No flag storage.

**Context structure** (`impl/09_runtime.md:376-426`): Has `flags: Map<string, Value>` for context-scoped flags. No fallback to runtime-level flags.

**Preprocessor phase** (`impl/09_runtime.md:274-281`): Pipeline step calls `process(source, metadata)` per preprocessor. No runtime flags passed.

### Key Files

| File | Role | Change Summary |
|------|------|----------------|
| `impl/09_runtime.md` | Runtime architecture and data structures | Add runtime flags, flag ops, pipeline update, context fallback |

### Dependencies

- **Requires:** 20260313-runtime-scoped-flags-spec-plan.md (spec must define semantics first)
- **Blocks:** Embed variable interpolation impl plan (preprocessor consumes runtime flags)

### Assumptions

- Runtime interface state section at lines 67-73 unchanged since last read
- Runtime operations section at lines 75-80 unchanged
- Preprocessor phase diagram at lines 274-281 unchanged
- Context structure at lines 376-426 unchanged

### Impact Analysis

- **Direct:** `impl/09_runtime.md` — 4 locations modified
- **Adjacent:** `impl/03_preprocessor.md` — will need updates for the embed proposal (not this plan)
- **Downstream:** C# runtime implementation must implement the new flag storage and pipeline change

---

## Implementation

### Overview

Four targeted edits to `impl/09_runtime.md`: add flag storage to Runtime, add flag operations, update pipeline diagram, and document context fallback.

### Step 1: Add Flag Storage to Runtime Interface

**Objective:** Add runtime-level flag map to Runtime state
**Confidence:** High
**Depends on:** None

**Files:**
- `impl/09_runtime.md`

**Changes:**

In the Runtime interface (line 56-93), within the `# State` section, add `flags` after `signals`:

```
// Before (lines 71-73):
  signals: SignalManager
  elapsedMs: double                      # internal — accumulated from tick(deltaTimeMs) calls

// After:
  signals: SignalManager
  flags: Map<string, Value>             # Runtime-scoped flags — visible to all contexts and preprocessors
  elapsedMs: double                      # internal — accumulated from tick(deltaTimeMs) calls
```

**Rationale:** Placed with other runtime state. Parallels `Context.flags` (line 391).

**Verification:** Read the Runtime interface block and confirm `flags` appears in the State section.

**If this fails:** Remove the added line.

---

### Step 2: Add Flag Operations to Runtime Interface

**Objective:** Expose flag read/write operations on Runtime
**Confidence:** High
**Depends on:** Step 1

**Files:**
- `impl/09_runtime.md`

**Changes:**

In the Runtime interface Operations section, add after `shutdown(): void` (line 80):

```
// Before (lines 79-80):
  resume(handle: ContextHandle, value: Value): void
  shutdown(): void

// After:
  resume(handle: ContextHandle, value: Value): void
  shutdown(): void

  # Runtime flag operations
  setFlag(name: string, value: Value): void    # Set a runtime-scoped flag
  getFlag(name: string): Value?                # Read a runtime-scoped flag (null if not set)
```

**Rationale:** These are the host-facing API for setting runtime flags before story loading (e.g., `runtime.setFlag("locale", "fr")`). `getFlag` returns nullable since the flag may not exist.

**Verification:** Read the Operations section and confirm the new methods appear after `shutdown()`.

**If this fails:** Remove the added lines.

---

### Step 3: Update Preprocessor Phase in Pipeline

**Objective:** Pass runtime flags to preprocessors during story compilation
**Confidence:** High
**Depends on:** Step 1

**Files:**
- `impl/09_runtime.md`

**Changes:**

In the Story Compilation Pipeline diagram (lines 274-281), update the preprocessor call:

```
// Before (line 278):
│     - process(source, metadata)              │

// After:
│     - process(source, metadata, runtimeFlags)│
```

**Rationale:** The preprocessor contract expands to include runtime flags. This is how the embed preprocessor (in a future plan) will access `locale` and other runtime flags for `${}` interpolation.

**Verification:** Read the pipeline diagram and confirm `runtimeFlags` appears in the process call.

**If this fails:** Revert the line to `process(source, metadata)`.

---

### Step 4: Document Context Flag Reading Fallback

**Objective:** Document that context flag reads fall back to runtime flags
**Confidence:** High
**Depends on:** Step 1

**Files:**
- `impl/09_runtime.md`

**Changes:**

After the Context Structure block (after line 426, after the closing ` ``` `), add:

```markdown

### Flag Resolution in Context

When a verb driver reads a flag by name:

1. Check `context.flags` — if found, return it
2. Check `runtime.flags` — if found, return it
3. Return null (flag not set)

Context flags shadow runtime flags of the same name. This mirrors the variable resolution pattern (story scope → context scope).

The `/flag` verb writes to context scope by default. With `[scope: "runtime"]`, it writes to `runtime.flags` instead.
```

**Rationale:** Verb drivers need clear guidance on flag lookup order. The fallback chain parallels variable resolution (story → context) documented at the top of the file.

**Verification:** Read the section after Context Structure and confirm the fallback chain is documented.

**If this fails:** Remove the added section.

---

## Verification Plan

### Manual Verification

- [ ] Read `impl/09_runtime.md` Runtime interface — confirm `flags` in state and `setFlag`/`getFlag` in operations
- [ ] Read pipeline diagram — confirm `runtimeFlags` parameter
- [ ] Read Context section — confirm flag resolution fallback documented
- [ ] Confirm existing `Context.flags` field (line 391) is unchanged
- [ ] Confirm consistency with spec plan's semantics

### Acceptance Criteria Validation

| Criterion | How to Verify | Expected Result |
|-----------|---------------|-----------------|
| Runtime flag storage | Read Runtime interface | `flags: Map<string, Value>` in State section |
| Runtime flag operations | Read Runtime interface | `setFlag` and `getFlag` in Operations |
| Preprocessor flag access | Read pipeline diagram | `process(source, metadata, runtimeFlags)` |
| Context fallback | Read after Context Structure | Fallback chain documented |

---

## Rollback Plan

Single file — revert via `git checkout -- impl/09_runtime.md`.

---

## Notes

### Risks
- **Pipeline diagram alignment**: The ASCII box art has fixed-width formatting. Adding `runtimeFlags` to the process call may misalign the box. Manual adjustment needed.
