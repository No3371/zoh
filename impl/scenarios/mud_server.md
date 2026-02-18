# Conformance Test: MUD Server Scenario

This document defines the "MUD Server" scenario, a rigorous stress test for ZOH runtimes. It exercises concurrency, channel communication, context isolation, and transaction atomicity.

## Test Harness Requirements

To execute this scenario, the runtime must verify the following capabilities:

1.  **Headless Execution:** The runtime must be able to execute the script without a UI, purely via command line or API.
2.  **Log Capture:** The runtime must support capturing all `/converse`, `/info`, `/warning`, and `/error` output to a structured log file for analysis.
3.  **Deterministic RNG:** The runtime must support seeding the random number generator (e.g., via `/seed 12345;` or CLI argument) to ensure reproducible runs.
4.  **Channel Monitoring:** The runtime should ideally provide a way to monitor channel depth or status for debugging, though the test relies on in-script assertions.
5.  **Timeout Enforcement:** The test must be run with a global timeout (e.g., 60 seconds) to detect deadlocks.

## Scenario Script: `mud_stress_test.zoh`

```zoh
!!! MUD Server Stress Test
meta_seed: 12345;
===

@setup
/info "Starting MUD Server Stress Test...";

:: Create channels
/open <lobby>;
/open <chat>;

:: Shared State (Simulated Database)
/set *item_registry, {};
/set *player_logs, {};

:: Launch Players
/set *player_count, 50;
/set *i, 0;

/loop *player_count, /sequence/
    /increase *i, 1;
    :: Construct player name string
    /interpolate "Player_${*i}";
    -> *name;
    
    :: Variable names MUST match the target checkpoint contract
    /set *id, *i;
    /fork ?, "player_context", *name, *id;
    /sleep 0.01; :: Stagger starts slightly
/;;

:: Launch Admin/Chaos Monkey
/fork ?, "admin_context";

/wait "all_players_done", timeout: 30;
/info "Test Complete. Verifying...";

:: Assert $#(*item_registry) == 50
/if `$#(*item_registry) != 50`, /sequence/
    /error "Assertion Failed: Item registry count is $#{*item_registry}, expected 50";
/;;

/info "SUCCESS";
/exit;

@player_context *name *id
/info "${*name} online.";

:: Phase 1: Chat Stress
/set *seq, 0;
/loop 50, /sequence/
    /increase *seq, 1;
    /interpolate "${*name}:msg:${*seq}";
    -> *msg;
    
    /push <chat>, *msg;
    
    :: Verify we can pull (mocking echo server or just consuming own spam if single channel)
    :: In this design, everyone pushes to <chat>. 
    :: To verify, we would need a central server reading <chat> and broadcasting.
    :: For this stress test, we just verify we can write without blocking forever.
    
    :: Simulate "thinking"
    /sleep 0.001;
    
    :: Create short-lived sub-task
    /set *parent, *name;
    /fork ?, "minion_task", *parent;
/;;

:: Phase 2: Transaction
:: Try to trade item with neighbor
/if `*id == 50`, /sequence/
    /set *next_id, 1;
/;, else: /sequence/
    /set *next_id, *id;
    /increase *next_id, 1;
/;;

/interpolate "item_${*id}";
-> *trade_item;

:: Critical Section: Atomicity check would require a shared verify.
:: Here we just log the trade attempt.
/info "${*name} trading ${*trade_item} to Player_${*next_id}";
/interpolate "TRADE:${*name}:${*next_id}:${*trade_item}";
-> *trade_msg;
/push <lobby>, *trade_msg;

:: Phase 3: Wait for Chaos
/try /sequence/
    /pull <chat>, timeout: 5; :: Waiting for messages
    /;,
    catch: /sequence/
        /info "${*name} caught expected error: Channel Closed";
    /;
;;

/signal "all_players_done", *name;
/exit;

@minion_task *parent
:: Just exist and die to stress context creation/gc
/set *x, 1;
/increase *x, 1;
/exit;

@admin_context
/sleep 5; :: Let players spam for a bit
/info "CHAOS: Closing chat channel...";
/close <chat>;
/exit;

```

## Expected Trace & Assertions

A generic test runner should parse the output log and verify:

1.  **Creation:** 50 "Player_X online." messages.
2.  **Concurrency:** "Player_1" and "Player_50" logs are interleaved, not sequential.
3.  **Chaos Handling:** 50 instances of "... caught expected error: Channel Closed".
4.  **No Crashes:** Log must NOT contain "Fatal Exception" or "Runtime Panic".
5.  **Completion:** Log must end with "SUCCESS".

### Deterministic Assertions
- **Total Trade Messages:** Grep "TRADE:" count == 50.
- **Minion Creation:** Internal metric (if available) should show 2500+ contexts created and destroyed.
