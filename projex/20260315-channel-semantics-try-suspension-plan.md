# Plan: Align Channel Semantics + `/try` Suspension Behavior (Spec + Impl Docs)

> **Status:** Ready
> **Created:** 2026-03-15
> **Author:** Antigravity Agent
> **Source:** `20260307-channel-semantics-try-suspension-proposal.md`
> **Related Projex:** `20260315-channel-semantics-try-suspension-proposal-review.md`
> **Worktree:** Yes

---

## Summary

This plan updates the language specification and runtime implementation documents to clarify channel rendezvous semantics and formally define how `/try` must wrap continuations when an inner verb suspends. 

**Scope:** Zoh language specification (`spec/`) and runtime architecture docs (`impl/`).
**Estimated Changes:** 3 files, ~4 sections.

---

## Objective

### Problem / Gap / Need
The channel semantics in the implementation docs (`impl/09_runtime.md`) use misleading wording about blocking. More critically, the `/try` verb's specification (`spec/2_verbs.md`) and implementation pseudo-code (`impl/07_control_flow.md`) do not account for the `Suspend` result, meaning fatal diagnostics returned asynchronously after a suspension might bypass the `/try` boundary.

### Success Criteria
- [ ] `/push` blocking is accurately described without implying global non-blocking behavior if any puller exists.
- [ ] `/try` specification explicitly mandates that execution must propagate suspension and resume from the same point, applying downgrade rules to the final outcome.
- [ ] `TryDriver` pseudocode demonstrates properly wrapping a `Suspend` continuation.

### Out of Scope
- Actually implementing these changes in the C# runtime (`csharp/`). That will be handled in a separate, downstream plan (`csharp/projex/20260315-channel-try-suspension-impl-plan.md`) after this spec plan is approved.

---

## Context

### Current State
1. `spec/2_verbs.md` defines `/try` but does not address `Suspend` behavior.
2. `impl/07_control_flow.md` shows a `TryDriver` that only checks for fatals on the immediate `result`, ignoring `DriverResult.Suspend`.
3. `impl/09_runtime.md` states `/push` blocks on "`wait: true` and no puller", which fails to describe the rendezvous "wait until value is consumed" contract.

### Key Files

| File | Role | Change Summary |
|------|------|----------------|
| `impl/09_runtime.md` | Core runtime mechanics | Update `/push` blocking table wording to "Value not yet consumed / No waiting puller" |
| `impl/07_control_flow.md` | Flow driver pseudo-code | Rewrite `TryDriver.execute` to intercept `Suspend` and return a wrapped continuation |
| `spec/2_verbs.md` | Verb specifications | Add rule defining `/try` behavior when target verb suspends |

### Dependencies
- **Requires:** `20260307-channel-semantics-try-suspension-proposal.md` (Option A accepted)
- **Blocks:** C# runtime implementation plan for these semantics.

### Assumptions
- The two-phase continuation model (`Complete` | `Suspend`) remains the standard for verb drivers.

### Impact Analysis
- **Direct:** Documentation and specifications.
- **Downstream:** Standardizes the required behavior for all runtime implementations, ensuring cross-platform consistency for flow control.

---

## Implementation

### Overview
We will update the 3 documents with targeted documentation and pseudo-code updates.

### Step 1: Clarify `/push` blocking semantics

**Objective:** Fix wording in the blocking table.
**Confidence:** High
**Depends on:** None

**Files:**
- `impl/09_runtime.md`

**Changes:**

```markdown
// Before (around line 631):
| `/push` | `wait: true` and no puller | Value consumed, timeout, or channel closed |

// After:
| `/push` | `wait: true` and value not yet consumed | Value consumed, timeout, or channel closed |
```

**Rationale:** The previous wording confusingly implied that if *any* puller existed, the push wouldn't block, missing the point that it blocks until the *specific pushed value* is consumed.

**Verification:** Manual visual check of markdown table rendering.

---

### Step 2: Define `/try` suspension rules in Spec

**Objective:** Make `/try` suspension safety an explicit spec guarantee.
**Confidence:** High
**Depends on:** None

**Files:**
- `spec/2_verbs.md`

**Changes:**
Add a new rule under the `/try` verb documentation (around line 236):

```markdown
// After (appending to the list of /try rules):
7. **Suspension Transparency**: If the inner verb suspends (e.g., waiting for a channel or host interaction), `/try` must propagate the suspension and resume from the same point. The downgrade, catch, and suppress logic is then applied to the eventual post-resume result.
```

**Rationale:** Enforces deterministic flow control behavior across runtimes when dealing with async operations.

**Verification:** Manual visual check of `2_verbs.md`.

---

### Step 3: Rewrite TryDriver pseudo-code

**Objective:** Demonstrate how a runtime should properly wrap a continuation.
**Confidence:** High
**Depends on:** None

**Files:**
- `impl/07_control_flow.md`

**Changes:**
Update `TryDriver.execute` to handle `Suspend`:

```javascript
// Before (around line 393):
TryDriver.execute(call, context):
    verb = call.params[0]
    catchHandler = getNamedParam(call, "catch")
    suppressDiagnostics = hasAttribute(call, "suppress")

    # Execute verb and capture result
    result = executeVerb(verb, context)

    # Check if fatal occurred
    hasFatal = context.diagnostics.fatal.length > 0

    if hasFatal:
// ... (rest of downgrade logic) ...
    return result

// After:
TryDriver.execute(call, context):
    verb = call.params[0]
    catchHandler = getNamedParam(call, "catch")
    suppressDiagnostics = hasAttribute(call, "suppress")

    # Execute verb and capture result
    result = executeVerb(verb, context)

    return handleTryResult(result, catchHandler, suppressDiagnostics, context)

handleTryResult(result, catchHandler, suppressDiagnostics, context):
    if result is Suspend:
        # Wrap the continuation to intercept the future Complete result
        originalContinuation = result.continuation
        wrappedContinuation = Continuation {
            request: originalContinuation.request,
            onFulfilled: (outcome) -> 
                nextResult = originalContinuation.onFulfilled(outcome)
                return handleTryResult(nextResult, catchHandler, suppressDiagnostics, context)
        }
        return Suspend {
            continuation: wrappedContinuation,
            diagnostics: result.diagnostics
        }

    # Handle Complete phase
    hasFatal = context.diagnostics.fatal.length > 0
    if hasFatal:
        # Downgrade fatal to error (preserve original codes)
        for diagnostic in context.diagnostics.fatal:
            context.diagnostics.error.append(diagnostic)
        context.diagnostics.fatal.clear()

        # Execute catch handler if provided
        if catchHandler != null:
            finalResult = executeVerb(catchHandler, context)
            # If catch handler itself suspends, it bypasses the *outer* try's catch, 
            # which is correct (catch handlers aren't self-catching).
            result = finalResult
        else:
            result = Complete { value: Nothing, diagnostics: [] }

    # Apply suppress attribute
    if suppressDiagnostics:
        context.diagnostics.clear()

    return result
```

**Rationale:** This standardizes the two-phase execution wrapper pattern needed for all flow verbs invoking other verbs. By extracting `handleTryResult` and calling it recursively in the wrapper, it seamlessly handles multi-yield situations (Suspend -> Suspend -> Complete).

**Verification:** The pseudo-code logically covers both `Complete` and `Suspend` variations without compiling.

---

## Verification Plan

### Manual Verification
- [ ] Review `impl/09_runtime.md` diff to ensure the blocking table is clear.
- [ ] Review `spec/2_verbs.md` to ensure the new rule integrates naturally with the existing 6 rules.
- [ ] Review `impl/07_control_flow.md` pseudo-code to ensure logic is robust (especially the recursive wrapper handling multiple suspends).

### Acceptance Criteria Validation
| Criterion | How to Verify | Expected Result |
|-----------|---------------|-----------------|
| `/push` blocking is accurately described | Read diff | Table row states "value not yet consumed" |
| `/try` suspension specified | Read diff | Rule 7 added explaining Suspension Transparency |
| `TryDriver` pseudocode updated | Read diff | `handleTryResult` properly wraps and unpacks `Suspend` continuations recursively. |

---

## Rollback Plan

If the implementations prove logically flawed:
1. Revert modifications in `impl/07_control_flow.md` via Git.
2. Revert modifications in `spec/2_verbs.md` via Git.
3. Revert modifications in `impl/09_runtime.md` via Git.

---

## Notes

### Risks
- Pseudocode wrapper logic relies on `executeVerb` in the catch handler not recursively re-entering the failing state.
- **Mitigation:** Document mentions catch handlers aren't self-catching.

### Open Questions
- [ ] None
