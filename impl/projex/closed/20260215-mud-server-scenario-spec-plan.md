# Plan: MUD Server End-to-End Validation Scenario Spec

> **Status:** Complete
> **Completed:** 2026-02-15
> **Walkthrough:** [20260215-mud-server-scenario-spec-walkthrough.md](../closed/20260215-mud-server-scenario-spec-walkthrough.md)
> **Created:** 2026-02-15
> **Author:** Antigravity
> **Source:** Direct request (via `explore-projex` findings)
> **Related Projex:** `projex/20260215-project-context-explore.md`

---

## Summary

Create a **Conformance Test Specification** for the "MUD Server" scenario. This is not just a concept, but a rigorous, executable test definition designed to verify a runtime's compliance with ZOH's concurrency model. It includes the full stress-test script, required test harness hooks, and deterministic assertions for message integrity and ordering.

**Scope:** Create `impl/scenarios/mud_server.md` containing `mud_stress_test.zoh`.
**Estimated Changes:** 1 file.

---

## Objective

### Problem / Gap / Need
ZOH lacks end-to-end validation scenarios that test its advanced concurrency features (forking, channels, context isolation) in concert. The "MUD Server" architecture is the canonical stress test for these systems.

### Success Criteria
- [ ] `impl/scenarios/mud_server.md` defines a **Reproducible Test Case** (inputs, seed, script).
- [ ] Spec includes the **Full Source Code** of `mud_stress_test.zoh` (no "TODOs" or "pseudo-code").
- [ ] Spec defines **Deterministic Assertions** (e.g., "Log file line 500 must match Regex X").
- [ ] Spec covers specific interaction bugs: Message loss, Message reordering, **Transaction Atomicity**, **Channel Destruction Handling**.

### Out of Scope
- Implementing the runtime changes to fix bugs found by this scenario (that's a separate execution).
- Actual execution of the scenario (this plan *defines* the scenario).

---

## Context

### Current State
`impl/` contains feature-specific specs (`08_concurrency.md`) but no integrated scenarios. `tests/` has unit tests but no complex end-to-end scripts.

### Key Files
| File | Purpose | Changes Needed |
|------|---------|----------------|
| `impl/scenarios/mud_server.md` | New scenario spec | Create new file |

### Dependencies
- **Requires:** None.
- **Blocks:** Future runtime validation and stress testing.

---

## Implementation

### Overview
Create `impl/scenarios/mud_server.md` as a **Conformance Test Specification**. This document will contain the full source code for `mud_stress_test.zoh` and define the required Test Harness behavior to verify it.

### Step 1: Define Test Harness Requirements
**Objective:** Specify what the runtime needs to support to run this test.
**Files:** `impl/scenarios/mud_server.md`
**Content:**
- **Headless Execution:** Ability to run without UI.
- **Log Capture:** Ability to capture `/log` or `/converse` output to a structured file.
- **Deterministic RNG:** Requirement to support `/seed` (or external seeding) for reproducibility.

### Step 2: Write `mud_stress_test.zoh`
**Objective:** Provide the complete, executable script.
**Files:** `impl/scenarios/mud_server.md` (Code Block)
**Content:**
- **Phase 1 (Setup):** Server initializes `<lobby>`, `<chat>`, and 50 "Player" contexts.
- **Phase 2 (Stress - Chat):** 
    - Players loop 50 times:
        - `/push` a unique message (ID+Seq) to `<chat>`.
        - `/pull` from `<chat>` (verify receipt).
        - `/fork` a short-lived "minion" task (stress context creation).
- **Phase 3 (Stress - Transactions):**
    - "Trade" stress test: Player A gives Item to Player B.
    - Requires locking or atomic hand-off to ensure Item isn't duplicated or lost if both act simultaneously.
- **Phase 4 (Chaos - Room Nuke):**
    - Admin context force-closes `<chat>` channel while 50 players are writing to it.
    - Assert: Players receive "Channel Closed" signal/error, NOT runtime crash.

### Step 3: Define Expected Trace & Assertions
**Objective:** Define exactly what "Pass" looks like.
**Files:** `impl/scenarios/mud_server.md`
**Content:**
- **Assertion 1 (Message Integrity):** All 50 players must log "Received: [All messages]" (checking central registry).
- **Assertion 2 (Atomic Trade):** Total Item count in system must remain constant. No duplication, no voiding.
- **Assertion 3 (Graceful Chaos):** "Room Nuke" results in 50 handled errors, 0 crashes.
- **Assertion 4 (Deadlock):** Test must complete within X virtual/real seconds.

---

## Verification Plan

### Automated Checks
- [ ] `git` checks to confirm file creation.

### Manual Verification
- [ ] Review the ZOH script in the spec for syntax correctness (mentally or via `c#` parser if possible, but this is a spec doc).

---

## Rollback Plan
1. Delete `impl/scenarios/mud_server.md`.
