# Conformance Test: MUD Server Scenario

This document defines the MUD Server scenario: a concurrent load test that exercises parallel context lifecycle, rendezvous channel coordination, fire-and-forget broadcast, channel-close notification under concurrent load, and completion tracking through a channel.

## Behaviors Under Test

| # | Behavior | How It Is Exercised |
|---|----------|---------------------|
| 1 | **Parallel context lifecycle** | 10 player contexts and 1 broker run concurrently; setup waits for all to exit cleanly |
| 2 | **Rendezvous fan-in** | Each player makes a blocking push to `<register>`; broker pulls each in turn, serializing registrations |
| 3 | **Fire-and-forget broadcast** | Broker pushes N−2 = 8 messages to `<announce>` with `wait: false` while all 10 players are already blocking on pull |
| 4 | **Channel-close under concurrent load** | After the 8 messages, broker closes `<announce>`; the 2 remaining blocked pullers receive a close notification (non-fatal error) and continue gracefully |
| 5 | **Completion tracking via channel** | Each player does a fire-and-forget push to `<completions>` on exit; setup pulls N times to confirm all contexts finished |

## Test Harness Requirements

1. **Headless execution**: the runtime must execute the script without any interactive UI or user input.
2. **Log capture**: all `/info`, `/warning`, and `/error` output must be captured to a structured log for assertion analysis.
3. **Concurrent context support**: the runtime must support at least 13 simultaneous live contexts (10 players + 1 broker + 1 setup).
4. **Global timeout**: enforce a wall-clock timeout of at least 60 seconds to detect deadlocks.

## Scenario Script: `mud_server.zoh`

```zoh
MUD Server Stress Test
===

@setup
/open <register>;
/open <announce>;
/open <completions>;

*player_count <- 10;
*i <- 0;

:: Fork all player contexts first — each immediately blocks at its registration push
/loop *player_count, /sequence/
    /increase *i;
    ====+ @player *i;
/;;

:: Fork broker after all players are running so all pushes are already blocking
====+ @broker *player_count;

:: Collect completion confirmations — each player fire-and-forgets to <completions> on exit
*done <- 0;
/while `*done < *player_count`, /sequence/
    /pull <completions>, timeout: 30;
    /increase *done;
/;;

/info "SUCCESS: all ${*player_count} players done.";
/exit;


@player *i:integer

:: Phase 1: register — rendezvous push blocks until broker pulls
/push <register>, *i;

:: Phase 2: pull announcement — receives a welcome message or a close-error nothing
/pull <announce>, timeout: 5; -> *welcome;
/if /any *welcome;, /sequence/
    /info "Player ${*i}: connected.";
/;, else: /sequence/
    /info "Player ${*i}: missed announcement.";
/;;

:: Phase 3: confirm completion
/push [wait: false] <completions>, *i;
/exit;


@broker *player_count:integer

:: Pull all N registrations — each pull unblocks one player's rendezvous push in turn
*n <- 0;
/while `*n < *player_count`, /sequence/
    /pull <register>; -> *player_id;
    /increase *n;
    /info "Broker: player ${*player_id} registered (${*n} of ${*player_count}).";
/;;

/info "Broker: all ${*player_count} players registered.";

:: Push N-2 fire-and-forget announcements
:: At this point all 10 players are blocking on /pull <announce>
:: The 2 that do not receive a message will be woken by the channel close
/eval `*player_count - 2`; -> *announced;
/loop *announced, /sequence/
    /push [wait: false] <announce>, "Server is live!";
/;;
/close <announce>;

/info "Broker: announced to ${*announced} players, channel closed.";
/exit;
```

## Expected Trace & Assertions

A test runner parses the captured log and verifies the following. All counts are deterministic and scheduling-independent.

### Assertion 1 — All registrations received

**Pattern:** `"Broker: player N registered (M of 10)."` where M = 1, 2, …, 10 and N ∈ 1–10.
**Expected count:** exactly 10.
**Rationale:** broker's while loop runs exactly `player_count` times, one pull per player.

### Assertion 2 — Deterministic connected / missed split

**Pattern (connected):** `"Player N: connected."` — **expected count: 8** (= `player_count − 2`).
**Pattern (missed):** `"Player N: missed announcement."` — **expected count: 2**.
**Rationale:** all 10 players are blocking on `/pull <announce>` before the broker pushes anything. The broker pushes 8 messages (`wait: false`) then closes. The channel hub delivers each message to one waiting puller; the remaining 2 are woken by the close event. The split is 8/2 regardless of scheduling order.

### Assertion 3 — All players complete

**Pattern:** `"SUCCESS: all 10 players done."` — **expected count: exactly 1**.
**Rationale:** setup's collection loop pulls from `<completions>` exactly 10 times. If any player context deadlocks or crashes, the count stalls and the global timeout fires instead.

### Assertion 4 — No fatal errors

**Constraint:** the log must NOT contain any fatal-level diagnostic.
**Rationale:** channel-close errors are non-fatal (Error level); all 10 players must reach their exit whether or not they received an announcement.
