# Proposal: Align Channel Semantics + `/try` Suspension Behavior

> **Status:** Draft
> **Created:** 2026-03-07
> **Author:** Agent
> **Reviewed:** 2026-03-15 - `20260315-channel-semantics-try-suspension-proposal-review.md`
> **Review Outcome:** Valid - Recommended Option A. Proceeding to create actionable Plan.
> **Source:** `20260304-runtime-spec-gaps-memo.md` (Issues **3–7**)
> **Related Projex:** `20260211-channel-inbox-outbox-proposal.md`, `20260211-channel-inbox-outbox-spec-plan.md`, `20260211-channel-inbox-outbox-impl-spec-plan.md`, `20260301-flow-driver-continuation-fix-eval.md`, `20260315-channel-semantics-try-suspension-proposal-review.md`

---

## Summary

Clarify and align the **spec** and **implementation docs** for channel lifecycle/semantics (blocking `/push`, “unbounded” wording, FIFO guarantees, and `/open` requirements), and fully specify how **`/try` behaves when the inner verb suspends** (e.g., `/try /pull …`, `/try /prompt …`). This is primarily a docs/spec alignment change, plus a concrete continuation-wrapping requirement for `/try` implementations.

---

## Problem Statement

### Current State

Five moderate inconsistencies/gaps exist between the spec and the implementation documentation:

**3. Push blocking table is misleading.**  
`impl/09_runtime.md`’s blocking table describes `/push` as blocking on “`wait: true` and no puller”, which reads like “if a puller exists anywhere, push won’t block”. But the actual waited-push contract is per-value: **block until the pushed value is consumed**, which is only *immediately* true when there is a **waiting puller** at the time of the push.

**4. Channel “unbounded” vs rendezvous tension.**  
`spec/1_concepts.md` says a channel is an “unbounded … global pipe”, while `spec/2_verbs.md` defines default `/push` as rendezvous (`wait: true`) that blocks until consumption. These are reconcilable (dual-mode channel), but the spec does not make the reconciliation explicit.

**5. `0_basic.md` example uses channels without `/open`.**  
The showcase story uses `/push` and `/pull` on `<stop_idle>` and `<ending>` but never calls `/open`. Yet the spec says `/push`/`/pull` can return `Error: not_found` when the channel does not exist, and the implementation docs define `/open` as the creation/re-create verb.

**6. `/try` with suspending inner verbs is unspecified.**  
The spec and `impl/07_control_flow.md` describe `/try` only in terms of fatal→error downgrading, without defining what happens when the target verb returns `Suspend`. Without a rule, implementations can (and currently do) diverge: if `/try` returns the inner continuation unmodified, the eventual post-resume result bypasses `/try`’s downgrade/catch/suppress logic.

**7. “Global FIFO pipe” vs per-context outbox/inbox mapping is under-explained.**  
The spec uses “global pipe” language while also mentioning outbox/inbox + hub coordination. This is correct but easy to misread as “FIFO per-context” unless we explicitly state the **observable queue semantics** (global FIFO, each value delivered once, ordering guarantees under concurrency).

### Gap / Need / Opportunity

- **Implementer clarity:** runtime implementers need a crisp contract (especially for `/try` + suspension) to avoid host-facing correctness bugs.
- **Spec coherence:** readers should not have to infer how “unbounded global pipe” interacts with rendezvous waits and explicit channel opening.
- **Conformance:** scenario scripts and tests (e.g., MUD/Quest scenarios) should not depend on undefined behavior.

### Why Now?

Channel hubs + outbox/inbox routing and the two-phase continuation model are already documented elsewhere; these gaps are now “paper cuts” that repeatedly resurface as design drift. Fixing them now reduces future rework and makes downstream implementation plans unambiguous.

---

## Proposed Change

### Overview

Make the channel and `/try` contracts explicit and self-consistent across:

- `spec/1_concepts.md` (channel definition and guarantees)
- `spec/0_basic.md` (example correctness)
- `spec/2_verbs.md` (Channel.* and Core.Try semantics)
- `impl/09_runtime.md` (blocking table wording)
- `impl/07_control_flow.md` (TryDriver pseudocode updated for `DriverResult` + `Continuation`)

### Approach Options

#### Option A: Keep explicit `/open`, fix docs/examples, and define continuation-safe `/try`

- **Channel lifecycle stays explicit:** channels must be created (or re-created) via `/open`; `/push`/`/pull` on unknown channels returns `not_found`.
- **Clarify dual-mode semantics:** channels are “logically unbounded” *when* using fire-and-forget (`wait: false`); rendezvous (`wait: true`) adds synchronization by blocking push until consumption.
- **Fix the showcase:** add `/open <stop_idle>;` and `/open <ending>;` to `spec/0_basic.md` before first use.
- **Define `/try` + suspension:** `/try` must **wrap** any inner continuation so that downgrade/catch/suppress logic applies to the eventual post-resume result.

**Pros**
- Aligns with existing impl docs (`impl/08_concurrency.md` uses `not_found` when hub missing).
- Avoids silent channel-typo bugs (implicit channel creation can hide misspellings).
- Makes `/try` behavior deterministic across hosts and runtimes.

**Cons**
- Adds a small amount of boilerplate (`/open`) to examples and scripts.

**Effort**
- Low-to-medium: doc/spec edits plus targeted runtime driver fixes (in each implementation language).

#### Option B: Implicitly create channels on first `/push` or `/pull`

- Change `/push`/`/pull` to `getOrCreate` the hub when missing (no `not_found` for unopened channels).
- Keep `/open` primarily as “re-open closed channel (new generation)” and/or “explicit predeclare”.

**Pros**
- Keeps scripts terse; makes `0_basic.md` valid without edits.

**Cons**
- Typos create new channels silently (hard-to-debug hangs).
- Requires spec changes (diagnostics) and implementation changes across runtimes.

**Effort**
- Medium-to-high: semantics + diagnostics changes ripple into spec, impl docs, and all runtimes/tests.

### Recommended Approach

Recommend **Option A**:

1. Preserve the safety/intent signal of explicit `/open`.
2. Clarify the dual personality (rendezvous vs buffered) through wording, not semantics changes.
3. Make `/try` continuation-safe by contract (wrap inner continuation), matching the two-phase continuation model.

---

## Impact Analysis

### Affected Areas

- **Spec**
  - `spec/1_concepts.md`: clarify “unbounded” wording + explicitly state *observable* FIFO semantics (global queue behavior even if implemented via outboxes/inboxes).
  - `spec/0_basic.md`: add `/open` calls for channels used in the example.
  - `spec/2_verbs.md`: add an explicit rule for `/try` when the inner verb suspends (and, optionally, a general rule for flow verbs that execute inner verbs).
- **Implementation docs**
  - `impl/09_runtime.md`: fix `/push` blocking row to describe “value not yet consumed / no waiting puller” rather than “no puller”.
  - `impl/07_control_flow.md`: rewrite TryDriver pseudocode to handle `Suspend` by wrapping the continuation.
- **Runtime implementations**
  - Any runtime implementing `DriverResult.Suspend(Continuation)` needs to ensure `/try` wraps continuations and can handle multi-yield (Suspend → Suspend → Complete).

### Dependencies

- Depends conceptually on the outbox/inbox + hub model and two-phase continuation model already being accepted elsewhere.

### Risks

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Example edits introduce new mismatches | Low | Low | Keep example minimal: open the two channels used, nothing else |
| `/try` wrapper logic subtly wrong for multi-yield | Medium | Medium | Specify and test: wrapper must re-wrap on repeated `Suspend` and only apply downgrade/catch on `Complete` |

### Breaking Changes

- **None intended at the language level** (Option A is wording + example correctness + making implicit expectations explicit).
- Runtime implementations may change behavior where `/try` previously failed to apply downgrade/catch across suspension (bug fix).

---

## Open Questions

- Should the spec explicitly guarantee **global FIFO** across concurrent pushers, or only “FIFO in observed enqueue order” (concurrency makes the exact interleaving runtime-defined)?
- Do we want a single normative rule like: “Any verb that evaluates an inner verb must propagate suspension and resume from the same point,” covering `/try` + other flow verbs uniformly?

---

## Next Steps

If accepted, derive a **Plan** with explicit file edits:

1. Spec patch plan: `spec/0_basic.md`, `spec/1_concepts.md`, `spec/2_verbs.md`
2. Impl-doc patch plan: `impl/09_runtime.md`, `impl/07_control_flow.md`
3. Runtime plan(s): update `/try` driver(s) to wrap continuations; add tests demonstrating `/try /prompt` and `/try /wait` preserve try semantics across resume

