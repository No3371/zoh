# Review: Align Channel Semantics + `/try` Suspension Behavior

> **Review Date:** 2026-03-15
> **Reviewer:** Antigravity Agent
> **Reviewed Projex:** `20260307-channel-semantics-try-suspension-proposal.md`
> **Original Date:** 2026-03-07
> **Time Since Creation:** 8 days

---

## Review Summary

**Verdict:** Valid

This proposal remains highly relevant and accurate. The observed gaps and inconsistencies in the spec and implementation documentation (`spec/0_basic.md`, `spec/1_concepts.md`, `spec/2_verbs.md`, `impl/09_runtime.md`, and `impl/07_control_flow.md`) are still present as of today. The problem statement cleanly captures the friction between explicit channel lifecycles and "unbounded" terminology, as well as the ambiguous `/try` behavior during suspension.

---

## Timeline Analysis

### When Authored
- Created: 2026-03-07
- Last modified: 2026-03-07
- Status at authoring: Draft proposal addressing 5 specific documentation/spec gaps regarding channel semantics and `/try` behavior.

### What Changed Since
| Area | Then | Now | Impact |
|------|------|-----|--------|
| `spec/0_basic.md` | Channel usage lacked `/open` | Unchanged | None. Issue still exists. |
| `impl/09_runtime.md` | Blocking row for `/push` states `wait: true` and no puller | Unchanged | None. Phrasing still misleadingly suggests push won't block if a puller exists somewhere. |
| `impl/07_control_flow.md` | `/try` pseudocode lacks `Suspend` continuation wrapping | Unchanged | None. Implementation spec still incomplete. |
| Flow Verbs Impl | Partial implementation | Improved tests & implementation for flow verbs exist now | Highlighting need to formalize `/try` behavior to ensure runtime correctness. |

### Related Events
- 2026-03-15: "Executing Flow Verb Tests" projex heavily expanded flow verb test coverage. This makes it an ideal time to formalize the `/try` driver behavior and corresponding tests.

---

## Status Quo Assessment

### Current State
The language and runtime specifications still contain the precise gaps outlined in the proposal. 
1. `spec/0_basic.md` still performs `/push` and `/pull` operations without explicitly calling `/open`.
2. `impl/09_runtime.md` line 631 states that `/push` blocks on "`wait: true` and no puller", which remains ambiguous regarding rendezvous consumption mechanics.
3. Code changes addressing control flow verbs (`IfDriver`, `WhileDriver`) have progressed, making the `/try` suspension wrapper gap more pronounced.

### Drift from Projex Assumptions
| Assumption | Original | Current Reality | Drift Level |
|------------|----------|-----------------|-------------|
| Gaps exist in docs | The 5 gaps exist as stated | Gaps still exist identically | None |
| Implementation needed | `/try` needs to wrap continuations | Runtimes still need this formally specified to standardize behavior across hosts | None |

---

## Validity Assessment

### Problems Stated
| Problem | Still Valid? | Notes |
|---------|--------------|-------|
| Push blocking table misleading | Yes | Wording remains unchanged in `impl/09_runtime.md`. |
| Unbounded vs rendezvous tension | Yes | Semantic tension remains unaddressed. |
| Showcase missing `/open` | Yes | `spec/0_basic.md` untouched. |
| Try wrapper unspecified | Yes | Pseudo-code in `impl/07_control_flow.md` unchanged. |
| Global FIFO vs contextual routing | Yes | Wording remains the same. |

### Approach Proposed
| Aspect | Still Valid? | Notes |
|--------|--------------|-------|
| Option A (Keep explicit /open, fix docs, wrap /try) | Yes | Fits best with the language's explicit safety guarantees. Option B (implicit creation) is riskier for typos. |

### Prerequisites/Dependencies
| Dependency | Status | Impact |
|------------|--------|--------|
| Outbox/inbox + hub model | Met | Already described and established conceptually in implementation docs. |

---

## Completeness Assessment

### Coverage Gaps
- None identified. The proposal accurately scopes the 5 exact touchpoints needing clarification.

### Scope Expansion Candidates
- Since recent efforts (2026-03-15) focused on testing flow control drivers (`IfDriver`, `WhileDriver`), expanding the scope to explicitly mandate unit tests for `/try` with suspending inner verbs (like `/pull` or `/prompt`) in the official C# implementation test suite is highly recommended.

---

## Accuracy Assessment

### Technical Content
| Content | Status | Issue |
|---------|--------|-------|
| File references | Accurate | All paths and file names from the original proposal still exist. |

---

## Challenge Questions

### Challenge 1: Should we just adopt Option B (implicit channel creation)?
**Evidence for Option B:**
- Eliminates boilerplate. `0_basic.md` would just work without edits.
- Easier for new authors to jump in.

**Evidence against Option B:**
- A typo in a channel name (`/push <endgin>`) would implicitly create a black-hole channel, causing silent hangs on matching `/pull <ending>`.
- The language currently strictly favors explicitness (e.g., forbidding implicit variable declarations without `/set` when strict rules apply).
- Explicit `/open` guarantees controlled lifecycle events and unambiguous creation of hubs.

**Assessment:** Option A (recommended by the proposal) remains superior. The explicit requirement aligns better with Zoh's overall typing and explicit state management design.

---

## Value Assessment

| Aspect | Original Value | Current Reality | Change |
|--------|----------------|-----------------|--------|
| Problem significance| Medium | Medium/High | Delta: Increased due to recent active development on flow control verbs. |
| Solution benefit | Clarity & Safety | Clarity & Safety | Delta: None |
| Implementation cost | Low/Medium | Low/Medium | Delta: None |

**Value Verdict:** Still highly valuable. Clear semantics are foundational for creating a robust virtual machine and valid tests.

---

## Recommendations

### Required Changes
1. Accept the proposal and move forward with **Option A**.
2. Transition this proposal into an actionable Plan projex (`20260315-channel-semantics-try-suspension-plan.md`) describing specific lines to alter in `spec/` and `impl/`.

### Suggested Improvements
1. Incorporate a specific test objective into the resulting Plan to verify `/try` wrapping of continuation results under multi-yield scenarios within `csharp/tests/Zoh.Tests/Verbs/Flow/`.

### Action Items
- [x] Review completed.
- [ ] Add Review Note and Outcome to the original proposal (`20260307-channel-semantics-try-suspension-proposal.md`).
- [ ] Draft the accompanying `<date>-channel-semantics-try-suspension-plan.md` to execute the changes.

### Next Review
- Recommended: N/A (Proposal should be accepted and planned).
