# Plan: Quest Log Processor End-to-End Scenario Spec

> **Status:** Ready
> **Created:** 2026-02-22
> **Author:** Agent
> **Source:** Direct request — cover spec verbs absent from `mud_server` scenario
> **Related Projex:** `20260222-specs-nav.md`

---

## Summary

Create `impl/scenarios/quest_log.md`: a single-context, sequential conformance test that exercises the core verb set that the MUD server scenario does not touch. The scenario drives a four-quest pipeline through `/call`, `/switch`+`/do` dispatch, `/defer`, `/try`+`/diagnose`, `/foreach`, `/flag`, `/type`, `/has`, persistent storage, typed variables, and expression evaluation — all in a deterministic, non-interactive flow with precisely assertable output.

**Scope:** Create one new file — `impl/scenarios/quest_log.md`.
**Estimated Changes:** 1 file.

---

## Objective

### Problem / Gap / Need

`mud_server` covers concurrency primitives: `/fork`, `/push`, `/pull`, `/close`, `/open`, `/loop`, `/while`, `/increase`, `/if`+`/sequence`, `/any`, `/eval` (arithmetic), `/exit`. A broad swath of the core verb spec is untouched by any existing scenario:

| Verb / Feature | Spec Section |
|----------------|-------------|
| `/call` (`<===+`) with return value | `2_verbs.md` — Core.Call |
| `/switch` + `/do` dispatch pattern | `2_verbs.md` — Core.Switch, Core.Do |
| `/defer [scope:"context"]` | `2_verbs.md` — Core.Defer |
| `/try` + `/diagnose` | `2_verbs.md` — Core.Try, Core.Diagnose |
| `/foreach` | `2_verbs.md` — Core.Foreach |
| `/flag` | `2_verbs.md` — Core.Flag |
| `/type` | `2_verbs.md` — Core.Type |
| `/has` | `2_verbs.md` — Core.Has |
| `/append`, `/count` | `2_verbs.md` — Core.Append, Core.Count |
| `/read`, `/write`, `/purge` | `2_verbs.md` — Store.Read, Store.Write, Store.Purge |
| `/get` explicit path access | `2_verbs.md` — Core.Get |
| Typed variables (`[typed:]`) | `1_concepts.md` — Variable Types |
| Checkpoint contracts with type | `1_concepts.md` — Checkpoint |
| Map literals and nested `["key"]` access | `1_concepts.md` — Variable Types |
| `$#{*var}` count interpolation | `2_verbs.md` — Core.Interpolate |

### Success Criteria

- [ ] `impl/scenarios/quest_log.md` exists with correct ZOH syntax throughout.
- [ ] The script exercises every verb/feature listed in the table above.
- [ ] All four assertions are deterministic and scheduling-independent.
- [ ] Assertion 3 (defer fires 4 times) validates that `/defer` runs even on early-exit paths.
- [ ] No verb from the MUD server scenario's coverage set is relied on as the primary mechanism for anything — channels, `/fork`, and `/loop` are not used.

### Out of Scope

- Interactive verbs (`/converse`, `/choose`, `/prompt`) — require host UI.
- Media verbs (`/show`, `/play`, etc.) — require host media layer.
- Cross-story jumps — single-story scenario only.
- `mud_server` scenario coverage — deliberately excluded to avoid overlap.

---

## Context

### Current State

`impl/scenarios/` contains one file: `mud_server.md`. No other scenarios exist yet. The three unexecuted plans (`card_battler`, `ecosystem_sim`, `metagame_tutorial`) are out of scope per user instruction.

### Key Files

| File | Purpose | Changes Needed |
|------|---------|----------------|
| `impl/scenarios/quest_log.md` | New scenario spec | Create |

### Dependencies

- **Requires:** Nothing — spec-only, standalone file.
- **Blocks:** Nothing directly.

### Constraints

- All ZOH syntax must comply strictly with `spec/0_basic.md`, `spec/1_concepts.md`, `spec/2_verbs.md`.
- The scenario must be non-interactive (no verbs requiring user input).
- All assertions must be deterministic on a single run regardless of scheduler behavior.
- Persistent storage is used intentionally to share state across called sub-contexts — the only cross-context state channel besides channels themselves.

---

## Implementation

### Overview

The scenario runs four quests through a sequential `/call`-based pipeline:

1. **fetch herb** (xp: 40) — adds "herb" to persistent inventory; exercises `/append`, `/count`, `/write`.
2. **combat bandit** (xp: 80, power: 2) — scales XP by power; exercises `/has` (optional field check), `/eval` (expression).
3. **fetch scroll** (xp: `"INVALID"`) — XP field is a string, not an integer; exercises `/type` check and early `/exit` with defer still firing.
4. **deliver herb** (xp: 60) — checks persistent inventory for "herb" (present from quest 1); exercises `/read`, `/has` (list membership).

The `@run_quest` checkpoint dispatches each quest via `/switch`+`/do`, wraps the handler call in `/try`+`/diagnose`, and registers a `/defer` that fires on every exit path. The script produces a precisely assertable log.

**Expected XP totals:** 40 + 80 + 0 (skipped) + 60 = 180. Level: `180 / 100 = 1`. XP remainder: 80.

---

### Step 1: Create `impl/scenarios/quest_log.md`

**Objective:** Write the complete scenario document.

**Files:** `impl/scenarios/quest_log.md`

**Content:**

````markdown
# Conformance Test: Quest Log Processor Scenario

This document defines the Quest Log Processor scenario: a sequential, non-interactive test that exercises the verb set absent from the MUD server scenario. A single setup context drives four quests through a `/call`-based pipeline, covering `/call` return values, `/switch`+`/do` dispatch, `/defer`, `/try`+`/diagnose`, `/foreach`, `/flag`, `/type`, `/has`, persistent storage, typed checkpoint contracts, and expression evaluation.

## Behaviors Under Test

| # | Behavior | How It Is Exercised |
|---|----------|---------------------|
| 1 | **`/call` with return value** | Setup calls `@run_quest` for each quest via `<===+`; each handler exits with an XP integer that the caller accumulates |
| 2 | **`/switch` + `/do` dispatch** | `@run_quest` switches on quest type to store the matching handler verb, then executes it with `/do` |
| 3 | **`/defer [scope:"context"]`** | `@run_quest` defers a closing log; it fires on clean exits AND on the early-exit path of the invalid quest |
| 4 | **`/try` + `/diagnose`** | `@run_quest` wraps the handler `/do` in `/try`; if a handler raises a fatal, `/diagnose` captures the downgraded error |
| 5 | **`/foreach`** | Setup iterates the quest list and the final inventory list |
| 6 | **`/flag`** | Setup sets `#interactive: false` before any quest runs |
| 7 | **`/type` field validation** | `@run_quest` checks the XP field type before dispatch; the invalid quest's `"INVALID"` string is caught here |
| 8 | **`/has` — map key and list membership** | `@quest_combat` checks for optional `"power"` key; `@quest_deliver` checks inventory list for the item |
| 9 | **`/read`, `/write`, `/purge`** | `@quest_fetch` reads, appends to, and writes back the persistent inventory; setup purges at start and reads at end |
| 10 | **Typed variables + checkpoint contracts** | `*xp` is declared `[typed:"integer"]`; all handler checkpoints declare `*q:map` |
| 11 | **Map literals + nested `["key"]` access** | Quest board is a list of map literals; all handlers access fields via `/get *q["field"]` |
| 12 | **`/eval` expression** | `@quest_combat` computes scaled XP: `` `*base_xp * *power / 2` `` |
| 13 | **`/append` + `/count`** | `@quest_fetch` appends item to inventory list and counts the result |
| 14 | **`$#{*var}` count interpolation** | Inventory size displayed in setup and fetch handler messages |

## Test Harness Requirements

1. **Headless execution**: the runtime must execute the script without interactive UI or user input.
2. **Log capture**: all `/info`, `/warning`, and `/error` output must be captured to a structured log for assertion analysis.
3. **Persistent storage**: the runtime must provide a working storage backend (in-memory is sufficient). Storage must be cleared between test runs, or `/purge` at script start must be sufficient to guarantee a clean state.
4. **Global timeout**: enforce a wall-clock timeout of at least 30 seconds.

## Scenario Script: `quest_log.zoh`

```zoh
Quest Log Processor
===

@setup
:: Clear persistent storage for a deterministic test run
/purge;

:: Context flags and typed state
/flag "interactive", false;
/set [typed:"integer"] *xp, 0;
/set [typed:"list"] *inventory, [];

/info "Session start.";

:: Quest board — list of maps, one with a deliberately corrupt XP field
*quests <- [
    {"type": "fetch",   "item": "herb",    "xp": 40},
    {"type": "combat",  "enemy": "bandit", "xp": 80, "power": 2},
    {"type": "fetch",   "item": "scroll",  "xp": "INVALID"},
    {"type": "deliver", "item": "herb",    "xp": 60}
];

:: Process each quest sequentially — /call blocks until each handler exits
/foreach *quests, *q, /sequence/
    <===+ @run_quest *q; -> *earned;
    /if /any *earned;, /sequence/
        /increase *xp, *earned;
        /info "Quest done. +${*earned} XP. Running total: ${*xp}.";
    /;;
/;;

:: Level calculation (100 XP per level, integer division)
/eval `*xp / 100`; -> *level;
/eval `*xp - *level * 100`; -> *xp_rem;
/info "Level: ${*level}, XP: ${*xp_rem} into next level.";

:: Read final inventory from persistent storage (written by fetch handlers)
/read default:[], *inventory;
/info "Inventory ($#{*inventory} item(s)):";
/foreach *inventory, *item, /sequence/
    /info "  - ${*item}";
/;;

:: Persist XP for next session
/write *xp;
/info "Session complete.";
/exit;


@run_quest *q:map

:: Defer a closing log — fires on exit regardless of which path was taken
/defer [scope:"context"] /info "run_quest: context closing.";;

:: Validate XP field type — skip quest if it is not an integer
/get *q["xp"]; -> *raw_xp;
/type *raw_xp; -> *xp_type;
/if *xp_type, is: "integer", /sequence/
    *xp_val <- *raw_xp;
/;, else: /sequence/
    /info "Quest has non-integer XP (type: ${*xp_type}) — skipping.";
    /exit ?;
/;;

:: Switch on quest type to obtain the handler verb, then execute it safely
/get *q["type"]; -> *type;
/switch/ *type
    "fetch"   /call ?, "quest_fetch", *q;
    "combat"  /call ?, "quest_combat", *q;
    "deliver" /call ?, "quest_deliver", *q;
/; -> *handler;

/if /any *handler;, /sequence/
    /try/
        /do *handler;
        catch: /sequence/
            /diagnose; -> *diag;
            /info "Handler fatal — skipping. Errors: ${*diag}.";
            /exit ?;
        /;
    /; -> *result;
    /exit *result;
/;, else: /sequence/
    /info "Unknown quest type '${*type}' — skipping.";
    /exit ?;
/;;


@quest_fetch *q:map
/get *q["item"]; -> *item;
/get *q["xp"]; -> *reward;

:: Read current inventory from persistent storage, append item, write back
/read default:[], *inventory;
/append *inventory, *item;
/write *inventory;

/count *inventory; -> *inv_size;
/info "Fetched '${*item}'. Inventory: ${*inv_size} item(s).";
/exit *reward;


@quest_combat *q:map
/get *q["enemy"]; -> *enemy;
/get *q["xp"]; -> *base_xp;

:: Power is optional in quest data — default to 1 if absent
/has *q, "power"; -> *has_power;
/if *has_power, /sequence/
    /get *q["power"]; -> *power;
/;, else: /sequence/
    *power <- 1;
/;;

/eval `*base_xp * *power / 2`; -> *reward;
/info "Defeated '${*enemy}' (power ${*power}). XP: ${*base_xp} x ${*power} / 2 = ${*reward}.";
/exit *reward;


@quest_deliver *q:map
/get *q["item"]; -> *item;
/get *q["xp"]; -> *reward;

:: Check persistent inventory for the required delivery item
/read default:[], *inventory;
/has *inventory, *item; -> *found;

/if *found, /sequence/
    /info "Delivered '${*item}'.";
    /exit *reward;
/;, else: /sequence/
    /info "Cannot deliver '${*item}': not in inventory.";
    /exit ?;
/;;
```

## Execution Trace

Deterministic walk-through of the expected execution (first run, empty persistent storage):

| Step | Quest | Key Events | XP After |
|------|-------|-----------|----------|
| 1 | fetch herb (xp: 40) | inventory: [] → ["herb"]; persisted | 40 |
| 2 | combat bandit (xp: 80, power: 2) | `80 × 2 / 2 = 80`; no storage change | 120 |
| 3 | fetch scroll (xp: "INVALID") | `/type` returns "string" ≠ "integer" → early exit; defer fires | 120 |
| 4 | deliver herb (xp: 60) | reads ["herb"] from storage; `/has` → true | 180 |
| Final | — | level: `180/100=1`, remainder: 80; inventory read: ["herb"] | — |

## Expected Trace & Assertions

A test runner parses the captured log and verifies the following. All counts are deterministic and scheduling-independent.

### Assertion 1 — Invalid quest skipped cleanly

**Pattern:** `"Quest has non-integer XP (type: string) — skipping."` — **expected count: exactly 1**.
**Rationale:** quest 3 has `"xp": "INVALID"` (a string). `/type` returns `"string"`, which is not `"integer"`, so `@run_quest` exits early without dispatching to any handler.

### Assertion 2 — Defer fires on every path including early exit

**Pattern:** `"run_quest: context closing."` — **expected count: exactly 4**.
**Rationale:** `/defer [scope:"context"]` is registered at the top of `@run_quest` for all four quests. It fires when the context is terminated, regardless of whether exit was via normal handler completion or the early `type` check path. Quest 3 (early exit) must produce this log line too.

### Assertion 3 — Correct XP accumulation

**Patterns (must all appear, exactly once each):**
- `"Quest done. +40 XP. Running total: 40."`
- `"Quest done. +80 XP. Running total: 120."`
- `"Quest done. +60 XP. Running total: 180."`

**Anti-pattern (must NOT appear):** any `"Quest done."` line for quest 3 (the invalid one).
**Rationale:** quests 1, 2, 4 each return XP via `/exit`; quest 3 returns nothing, so the setup's `/any *earned;` guard prevents accumulation.

### Assertion 4 — Correct final state

**Patterns (all exactly once):**
- `"Level: 1, XP: 80 into next level."`
- `"Inventory (1 item(s)):"`
- `"  - herb"`
- `"Session complete."`

**Rationale:** 180 XP / 100 = level 1, remainder 80. Only "herb" is in inventory: quest 1 fetched it, quest 3 was skipped (no "scroll"), and deliver does not remove items. Persistent storage correctly carries "herb" from `@quest_fetch` to setup's final read.

### Assertion 5 — No fatal errors

**Constraint:** the log must NOT contain any fatal-level diagnostic.
**Rationale:** the corrupt XP field is caught by a `/type` check before any potentially fatal operation. No handler is given data that would produce a fatal.
````

---

## Verification Plan

### Automated Checks

- [ ] `git` diff confirms only `impl/scenarios/quest_log.md` is added.

### Manual Verification

- [ ] Every verb in the "Behaviors Under Test" table appears in the script.
- [ ] No channel, `/fork`, or `/loop` verb appears in the script (these belong to the MUD server scenario; their absence confirms no overlap).
- [ ] All checkpoint contract types match the data passed at each `/call` site.
- [ ] Execution trace math is correct: 40 + 80 + 0 + 60 = 180; 180/100 = 1; 180 − 100 = 80.
- [ ] Defer count assertion (4) accounts for all four `/call` invocations of `@run_quest`.

---

## Notes

### Assumptions

- `/switch` stores the matching verb as a verb-type value (not executing it); `/do` executes the stored verb. This matches the "At a Glance" `/switch`→`/do` pattern in `spec/0_basic.md`.
- `/call` stores the verb parameters (including `*q` as a reference) at verb-object creation time; `*q` resolves in `@run_quest`'s context when `/do` executes the stored call.
- `/defer [scope:"context"]` fires when the context is terminated via `/exit`, including early-exit paths. If the runtime interprets "context" scope differently, the defer assertion count will be wrong and the spec should be clarified.
- Integer division: `` `*base_xp * *power / 2` `` with values 80, 2 gives 80 (80×2=160, 160/2=80). `` `*xp / 100` `` with 180 gives 1. Both assume integer division semantics, which `expr.md` should confirm.
- Persistent storage is single-tenant for this test. `/purge;` at start clears the default store, making the test idempotent across runs.

### Risks

- **`/switch` value semantics:** If a runtime executes the value verb immediately (rather than storing it), the `/do` step would fail with `invalid_type`. Risk: Low — the spec example in `0_basic.md` clearly stores the verb. Mitigation: the manual verification step checks for this explicitly.
- **`/defer` scope ambiguity:** `spec/3_runtime.md` does not define the default scope for `/defer`. The plan uses explicit `[scope:"context"]` to remove ambiguity. If the runtime ignores this attribute, the defer may not fire on `/exit`. Mitigation: assertion 2 will catch this immediately.
