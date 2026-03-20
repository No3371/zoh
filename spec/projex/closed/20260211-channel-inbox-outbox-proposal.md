# Channel Inbox/Outbox Architecture

> **Status:** Draft
> **Created:** 2026-02-11
> **Author:** Agent
> **Related Projex:** `20260211-channel-inbox-outbox-redteam.md`, `20260211-channel-inbox-outbox-spec-plan.md`, `20260211-channel-inbox-outbox-impl-spec-plan.md`

---

## Summary

Restructure the channel implementation spec so that message data lives in context-local **outboxes** and **inboxes** rather than a single global queue. Channels in the runtime become lightweight **hubs** — identified by `name:generation` — that route values from outboxes to inboxes. Go-style single-dispatch FIFO semantics are preserved. This is primarily an architecture change with spec-level additions: `/push` gains `wait` and `timeout` named parameters, requiring `spec.md` amendment.

---

## Problem Statement

### Current State

The spec (`spec.md`) defines channels as global FIFO pipes managed by the runtime. The impl spec (`impl/08_concurrency.md`, `impl/09_runtime.md`) specifies a `ChannelManager` at the runtime level holding a `Map<string, Channel>` where each `Channel` owns a `ConcurrentQueue`. All contexts access the same manager; `/push` enqueues to the global queue, `/pull` dequeues from it.

```
CURRENT MODEL
┌──────────────────────────────────────────────────┐
│  Runtime ChannelManager                          │
│  ┌────────────────────────────────────────────┐  │
│  │  <events> → Queue [val, val, val]          │  │
│  │  <results> → Queue [val]                   │  │
│  └────────────────────────────────────────────┘  │
│       ▲ push        │ pull ▼                     │
│  Context A      Context B      Context C         │
└──────────────────────────────────────────────────┘
```

### Gap / Need / Opportunity

The global-queue model works but has implementation-level limitations:

1. **Context isolation** — Contexts are coupled to shared mutable state. Cloning, serializing, or migrating a context requires entangling with all other contexts' channel data.
2. **Inspectability** — No per-context visibility into what a context has sent or is waiting to receive (origin tracking). Per-context outboxes/inboxes give clear sender/receiver attribution, which the global queue lacks. (Note: aggregate channel-level inspection requires scanning participating outboxes; mitigated by hub participation dictionaries.)
3. **Wake-up routing** — When a value is pushed, the runtime must scan all waiting contexts to find who to wake. With per-context storage, the push can trigger wake directly.
4. **Contention** — A single shared queue is a contention point. Per-context queues reduce lock contention.

### Why Now?

The impl spec's concurrency section (`impl/08_concurrency.md`) notes that blocking pull and waiter notification are not yet fully specified. This is the natural point to restructure before those mechanisms solidify.

---

## Proposed Change

### Overview

Each context owns per-channel **outboxes** and **inboxes**:

- **Outbox**: where `/push` deposits values. Data lives in the pushing context.
- **Inbox**: where `/pull` reads from. Data lives in the pulling context.
- **Hub**: routing metadata that connects outboxes to inboxes. Holds no data — only tracks who is connected and who is waiting.

On `/push`, the value enters the pusher's outbox. On `/pull`, the hub drains a value from an outbox into the puller's inbox (or blocks if all connected outboxes are empty).

```
PROPOSED MODEL
┌────────────────────────────────────────────────────────────────┐
│  Runtime Channel Hub Registry                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  <events>:gen3 → Hub { seq: 7 }                         │  │
│  │  <results>:gen1 → Hub { seq: 2 }                        │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                │
│  Context A                     Context B                       │
│  ┌────────────────────┐       ┌────────────────────┐          │
│  │ outbox:            │       │ inbox:             │          │
│  │  <events>[(v,4)]   │──hub──│  <events>[(v,3)]   │          │
│  │ inbox:             │       │ outbox:            │          │
│  │  <results>[(v,1)]  │◄─hub─│  <results>[(v,2)]  │          │
│  └────────────────────┘       └────────────────────┘          │
│                                                                │
│  /push <events>, x;  →  x enters Context A's outbox           │
│  /pull <events>;     →  hub finds oldest across outboxes       │
│                         → moves it to puller's inbox → return  │
└────────────────────────────────────────────────────────────────┘
```

### Key Design Decisions

#### Outbox = push buffer, Inbox = pull buffer

- `/push <chan>, val;` → value + sequence number go into the calling context's `outbox[chan]`
- `/pull <chan>;` → hub scans participating outboxes for `chan`, picks the value with the **lowest sequence number** (global FIFO), moves it into the puller's `inbox[chan]`, returns it
- If all outboxes are empty → puller blocks (`WAITING_CHANNEL`)
- When a new value is pushed and a puller is waiting → hub wakes the puller immediately

This preserves exact Go-style semantics:
- **FIFO**: sequence numbers across all outboxes ensure global ordering
- **Single dispatch**: each value goes to exactly one puller
- **Push-before-pull works**: values sit in the pusher's outbox until a puller arrives
- **Multiple pushers**: values from different contexts interleave by sequence number

#### Waited push (`wait` and `timeout` parameters)

`/push` gains two named parameters:

- **`wait`** (boolean, default `true`) — whether the push blocks until consumed.
  - **`wait: true`** (default) — **blocking push**. The verb does not complete until the pushed value has been consumed by a puller. The pushing context enters `WAITING_CHANNEL_PUSH` state. This is analogous to Go's unbuffered channel send.
  - **`wait: false`** — **fire-and-forget**. The value enters the outbox and the verb returns immediately. The value may sit in the outbox indefinitely until a puller retrieves it.

- **`timeout`** (double, optional) — maximum seconds to wait when `wait: true`. If the value is not consumed within the timeout, the push unblocks and returns `Info: timeout` diagnostic. Only meaningful with `wait: true`; ignored when `wait: false`.

```zoh
:: Blocking (default) — waits until someone pulls this value
/push <results>, *answer;

:: Blocking with timeout — waits up to 5 seconds
/push <results>, *answer, timeout: 5;

:: Non-blocking — fire and forget
/push <events>, *update, wait: false;
```

> [!NOTE]
> With `wait: true`, both sides can block: the puller blocks waiting for a value, and the pusher blocks waiting for consumption. This creates natural synchronization points between contexts — a true rendezvous.

> [!WARNING]
> **Deadlock risk:** Two contexts each doing waited push to each other's channels without pulling first will deadlock. Use `timeout` in bidirectional communication patterns to prevent indefinite blocking.

#### Hub is routing metadata + sequence counter + participation dictionary

```
ChannelHub:
  name: string
  generation: int
  closed: bool
  sequenceCounter: int              # Monotonic, incremented on each push
  participants: Map<contextId, {outbox: Queue, inbox: Queue}>  # Registered via /open
  waitingPullers: Queue<Context>    # Contexts blocked on /pull (FIFO)
  waitingPushers: Queue<(Context, int)>  # Contexts blocked on waited push (context, seqNum)
```

No message data in the hub. It tracks:
1. The next sequence number to assign (for FIFO ordering across outboxes)
2. Which contexts are participating (registered via `/open`) — pull scans only these
3. Which contexts are currently blocking on `/pull`
4. Which contexts are currently blocking on waited `/push` (waiting for their value to be consumed)
5. Generation for staleness detection

> [!IMPORTANT]
> **Concurrency safety:** All hub fields and outbox/inbox queues must be implemented with concurrency-safe data structures (e.g., concurrent collections, lock-guarded access). The scan-and-dequeue in the pull flow must be atomic — concurrent pullers must not race on the same outbox entry.

#### Context channel state

```
Context:
  # ... existing fields ...
  outboxes: Map<string, Queue<(Value, int)>>   # channel → queue of (value, seqNum)
  inboxes: Map<string, Queue<Value>>           # channel → queue of received values
```

- Outbox entries are `(value, sequenceNumber)` pairs for cross-outbox FIFO ordering
- Inbox holds values already dispatched to this context, ready for consumption

#### Push flow

```
PushDriver.execute(call, context):
    channelName = resolve(call.params[0])
    value = resolve(call.params[1])
    wait = resolve(call.namedParams["wait"]) ?? true    # default: blocking
    timeout = resolve(call.namedParams["timeout"])       # optional, double or null
    
    hub = runtime.channelHubs.get(channelName)
    if hub == null or hub.closed:
        return error("not_found" / "closed")
    
    seq = hub.sequenceCounter++
    
    # If a puller is waiting, deliver directly (skip outbox entirely)
    if hub.waitingPullers.hasNext():
        target = hub.waitingPullers.dequeue()
        target.inbox[channelName].enqueue(value)
        target.state = RUNNING
        target.waitCondition = null
        return ok()    # Value consumed immediately — no blocking needed even for waited push
    
    # No waiter — park value in pusher's outbox
    context.outbox[channelName].enqueue((value, seq))
    
    if wait:
        # Block until this specific value is consumed by a puller
        hub.waitingPushers.enqueue((context, seq))
        context.state = WAITING_CHANNEL_PUSH
        context.waitCondition = ChannelPushWaitCondition {
            channelName, seq, generation,
            timeout: timeout,
            startTime: now()
        }
    
    return ok()
```

#### Pull flow

```
PullDriver.execute(call, context):
    channelName = resolve(call.params[0])
    
    hub = runtime.channelHubs.get(channelName)
    if hub == null or hub.closed:
        return error("not_found" / "closed")
    
    # Check own inbox first (previously dispatched values)
    if context.inbox[channelName].hasValue():
        return ok(context.inbox[channelName].dequeue())
    
    # Scan participating outboxes for this channel, find lowest sequence number
    # NOTE: This scan-and-dequeue must be atomic (hub-level lock)
    bestSource = null
    bestSeq = MAX_INT
    for (ctxId, state) in hub.participants:
        if state.outbox.hasValue():
            (_, seq) = state.outbox.peek()
            if seq < bestSeq:
                bestSeq = seq
                bestSource = state
    
    if bestSource != null:
        (value, consumedSeq) = bestSource.outbox.dequeue()
        
        # Wake the pusher if they're blocked on a waited push for this value
        if hub.waitingPushers.contains(bestSource.context, consumedSeq):
            hub.waitingPushers.remove(bestSource.context, consumedSeq)
            bestSource.context.state = RUNNING
            bestSource.context.waitCondition = null
        
        return ok(value)
    
    # Nothing available — block
    hub.waitingPullers.enqueue(context)
    context.state = WAITING_CHANNEL
    context.waitCondition = ChannelWaitCondition {
        channelName, timeout, generation, startTime
    }
    return ok()
```

#### Open flow

```
OpenDriver.execute(call, context):
    channelName = resolve(call.params[0])
    
    hub = runtime.channelHubs.getOrCreate(channelName)
    if hub.closed:
        hub = runtime.channelHubs.recreate(channelName)
    
    # Initialize context's outbox and inbox for this channel (idempotent)
    if channelName not in context.outbox:
        context.outbox[channelName] = Queue.new()
    if channelName not in context.inbox:
        context.inbox[channelName] = Queue.new()
    
    # Register context with hub participation dictionary (idempotent)
    if context.id not in hub.participants:
        hub.participants[context.id] = {
            context: context,
            outbox: context.outbox[channelName],
            inbox: context.inbox[channelName]
        }
    
    return ok()
```

#### Close flow

```
CloseDriver.execute(call, context):
    channelName = resolve(call.params[0])
    
    hub = runtime.channelHubs.get(channelName)
    if hub == null or hub.closed:
        return error("not_found" / "closed")
    
    hub.closed = true
    hub.generation++
    
    # Wake all blocked pullers with close error
    while hub.waitingPullers.hasNext():
        waiter = hub.waitingPullers.dequeue()
        waiter.state = RUNNING
        waiter.lastDiagnostics = [Diagnostic(ERROR, "closed", "Channel closed")]
        waiter.waitCondition = null
    
    # Wake all blocked waited pushers with close error
    while hub.waitingPushers.hasNext():
        (pusher, seq) = hub.waitingPushers.dequeue()
        pusher.state = RUNNING
        pusher.lastDiagnostics = [Diagnostic(ERROR, "closed", "Channel closed")]
        pusher.waitCondition = null
    
    return ok()
```

#### Context termination cleanup

When a context terminates (end of statements, `/exit`, or fatal), channel cleanup runs alongside defers:

```
Context.cleanupChannels():
    # Deregister from all hub participation dictionaries
    for channelName in outboxes.keys():
        hub = runtime.channelHubs.get(channelName)
        if hub != null:
            hub.participants.remove(this.id)
    
    # Discard remaining outbox values (data belongs to the context)
    for channelName in outboxes.keys():
        outboxes[channelName].clear()
    
    # Discard unread inbox values
    for channelName in inboxes.keys():
        inboxes[channelName].clear()
    
    # If this context was a waited pusher, it's already being woken by
    # the termination flow — the hub's waitingPushers entry becomes stale
    # and should be pruned (or pre-removed here)
    for channelName in outboxes.keys():
        hub = runtime.channelHubs.get(channelName)
        if hub != null:
            hub.waitingPushers.removeByContext(this.id)
```

### Verb Behavior Changes

| Verb | Current | Proposed |
|------|---------|----------|
| `/open` | Creates global queue | Creates hub, initializes context outbox + inbox, **registers context with hub** |
| `/push` | Enqueues to global queue | Enqueues to pusher's outbox. If `wait: true` (default), blocks until consumed. Supports `timeout`. |
| `/push wait: false` | N/A (new) | Fire-and-forget into outbox, returns immediately. |
| `/pull` | Dequeues from global queue | Scans **participating** outboxes for oldest value, dequeues into puller's inbox. Wakes blocked pusher if waited. |
| `/close` | Closes global queue, wakes waiters | Closes hub, wakes all blocked pullers **and pushers**. |
| Context exit | N/A | Deregisters from all hubs, discards outbox/inbox, prunes waited-pusher entries. |

### Where data lives at each stage

```
  WAITED PUSH (wait: true, default):
  
  /push → outbox → pusher BLOCKS → puller /pull → value consumed → pusher WAKES
  
  NON-WAITED PUSH (wait: false):
  
  /push → outbox → return immediately    ...later...    puller /pull → value consumed
  
  FAST PATH (puller already waiting):
  
  /push → direct to puller.inbox → wake puller → return (no blocking regardless of wait)
```

---

## Impact Analysis

### Affected Areas

| Area | Impact |
|------|--------|
| `spec.md` Channel.Push section | **Required amendment**: add `wait` and `timeout` named parameters, document blocking behavior and diagnostics. |
| `spec.md` channel type definition | Minor clarification: "underlying data structure" includes the routing hub. External semantics unchanged — channels are still "FIFO pipes for inter-context communication." |
| `impl/08_concurrency.md` | Major: rewrite `ChannelManager` → `ChannelHubRegistry`, add outbox/inbox to Context, rewrite all verb driver pseudocode, add context cleanup flow. |
| `impl/09_runtime.md` | Moderate: Context adds outbox/inbox fields, `cleanupChannels()` in termination. Runtime `channels` becomes hub registry. Scheduler adds `WAITING_CHANNEL_PUSH` handling. |

### Dependencies

- **Waiter notification** — Replaced by direct delivery on push when a puller is waiting. Simpler than current incomplete mechanism.
- **Generation tracking** — Unchanged. Hub generation increments on close/reopen.
- **Sequence counter** — New. Monotonic per hub, ensures cross-outbox FIFO ordering.

### Risks

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Pull scan O(K) in participating contexts | Low | Low | Hub participation dictionary bounds scan to K (contexts that `/open`'d). |
| Concurrent puller races | Low | Medium | Scan-and-dequeue must be atomic (hub-level lock). Concurrency-safe data structures required. |
| Context lifecycle (cleanup outbox/inbox on exit) | Low | Medium | `cleanupChannels()` runs in context termination flow (alongside defers). Deregisters from hubs, discards values, prunes waited-pusher entries. |
| Ordering edge cases with many pushers | Low | Low | Sequence counter guarantees global FIFO. |
| Deadlock from waited push | Medium | Medium | Two contexts each doing waited push to each other's channels without pulling first = deadlock. Mitigated by `timeout` parameter on `/push`. |

### Breaking Changes

> [!WARNING]
> **Behavioral change to `/push`:** With `wait: true` as default, `/push` now blocks until the value is consumed. Previously `/push` was always non-blocking (fire-and-forget into the global queue). Existing scripts that push without a corresponding puller will hang. To preserve current behavior, use `wait: false`.

> [!NOTE]
> The outbox/inbox restructuring itself is transparent — push-before-pull patterns still work because values buffer in the pusher's outbox.

---

## Resolved Questions

- [x] **Outbox cleanup on context exit**: Outbox values are discarded on context termination ("data belongs to the context"). Hub deregistration and waited-pusher pruning included in `cleanupChannels()` flow.
- [x] **Spec wording**: `spec.md` **requires amendment** — adding `wait` and `timeout` named parameters to Channel.Push. The channel type definition needs minor clarification that "underlying data structure" includes the routing hub.
- [x] **Timeout on waited push**: Yes — `timeout` parameter added to `/push`, usable alongside `wait: true`.
- [x] **Hub participation tracking**: Hub maintains a `participants` dictionary. Contexts are auto-registered on first `/push` or `/pull` (outbox/inbox created lazily). Pull scans only participants.
- [x] **Close flow**: Wakes both blocked pullers **and** waited pushers with close error.
- [x] **Interaction with `[clone]`**: Outboxes/inboxes are **not** cloned. A forked context starts with empty channel state. It will be auto-registered with hubs on first push/pull. Cloning channel buffers mid-flight is unlikely to be expected behavior.
- [x] **Default wait value**: `wait: true` (blocking) is the default. ZOH is a brand new language with no existing scripts to break — backward compatibility is not a concern.
- [x] **Must `/open` before `/push` or `/pull`?** No — `/open` only creates/re-creates channels. Contexts are auto-registered with the hub (outbox/inbox created) on first `/push` or `/pull`.

---

## Next Steps

If accepted:
1. Resolve remaining open questions (auto-register on push/pull, clone behavior, wait default)
2. Amend `spec.md` Channel.Push section: add `wait` and `timeout` named parameters
3. Update `impl/08_concurrency.md` channel section (hub registry, participation dict, cleanup flow)
4. Update `impl/09_runtime.md` context structure (outbox/inbox fields, `cleanupChannels()`, `WAITING_CHANNEL_PUSH`)

---

## Appendix

### Research / References

- Channel type spec: `spec.md` lines 340-345
- Channel impl architecture: `impl/08_concurrency.md` lines 292-378
- Runtime channel integration: `impl/09_runtime.md` lines 33-36, 446-455
- Go channel semantics: multiple senders, multiple receivers, each value dispatched to exactly one receiver, FIFO, blocking send/receive

### Alternatives Considered

**Keep global queues, add metadata** — Add sender context ID to queued values. Addresses inspectability but doesn't solve isolation or contention.

**Inbox-only (no outbox)** — Hub holds pending values until a puller arrives. Rejected: defeats the goal of moving data out of global space. The outbox keeps data in the pushing context until consumption.

**Rendezvous-only (no buffer)** — Push drops value if no puller waiting. Rejected: breaks push-before-pull patterns used in the spec example and common in storytelling flows.

**Cursor-based views** — Global queue with per-context read cursors. Rejected: doesn't decouple contexts from shared mutable state.
