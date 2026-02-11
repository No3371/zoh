# 08: Concurrency Implementation

## Purpose

ZOH supports concurrent execution through contexts, channels, and navigation verbs. This document covers parallel execution, inter-context communication, and synchronization.

---

## Concurrency Model

```
┌────────────────────────────────────────────────────────────────┐
│                          RUNTIME                               │
│                                                                │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐   │
│  │  Context A   │     │  Context B   │     │  Context C   │   │
│  │  (main)      │     │  (forked)    │     │  (forked)    │   │
│  │              │     │              │     │              │   │
│  │  ┌────────┐  │     │  ┌────────┐  │     │  ┌────────┐  │   │
│  │  │ Story  │  │     │  │ Story  │  │     │  │ Story  │  │   │
│  │  │  vars  │  │     │  │  vars  │  │     │  │  vars  │  │   │
│  │  └────────┘  │     │  └────────┘  │     │  └────────┘  │   │
│  │  ctx vars    │     │  ctx vars    │     │  ctx vars    │   │
│  └──────┬───────┘     └──────┬───────┘     └──────┬───────┘   │
│         │                    │                    │            │
│         ▼                    ▼                    ▼            │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    CHANNELS                              │   │
│  │  <events>   <signals>   <results>   ...                  │   │
│  │    FIFO        FIFO        FIFO                          │   │
│  └─────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────┘
```

### Key Concepts

| Concept | Description |
|---------|-------------|
| **Context** | Isolated execution thread with own variables |
| **Fork** | Create new context running in parallel |
| **Call** | Fork + wait for completion |
| **Jump** | Move current context to new location |
| **Channel** | FIFO queue for inter-context communication |

---

## Labels

Labels mark navigation targets in stories.

### Definition

```zoh
@label_name
/verb;
```

### Resolution

```
LabelTable:
  entries: Map<string, int>  # label name → statement index
  
resolveLabel(name: string, story: Story): int
    normalizedName = name.toLowerCase()
    if normalizedName not in story.labels:
        fatal("Label not found: " + name)
    return story.labels[normalizedName]
```

---

## Jump

**Purpose**: Move current context to a label, optionally in a different story.

### Signature
```
/jump story?, label, var1, var2, ...;
```

### Behavior

1. If jumping to another story, clear story-scoped variables
2. `/set` transferred variables in new scope
3. Move instruction pointer to label
4. Continue execution from label

### Implementation

```
JumpDriver.execute(call, context):
    storyPath = resolve(call.params[0], context)
    labelName = resolve(call.params[1], context).toString()
    transferVars = call.unnamedParams.slice(2)
    
    # Resolve target story
    if storyPath.isNothing() or storyPath.toString() == "?":
        targetStory = context.currentStory
    else:
        targetStory = context.runtime.loadStory(storyPath.toString())
        if targetStory == null:
            return fatal("invalid_story", "Story not found: " + storyPath.toString())
    
    # Resolve label position
    labelIndex = targetStory.labels.get(labelName.toLowerCase())
    if labelIndex == null:
        return fatal("invalid_label", "Label not found: " + labelName)
    
    # If changing stories, handle scope transition
    if targetStory != context.currentStory:
        # Execute story defers before leaving
        context.executeStoryDefers()
        
        # Collect transfer values BEFORE clearing story scope
        transfers = []
        for ref in transferVars:
            name = ref.name
            value = context.get(name)
            scope = context.getVariableScope(name)
            transfers.add({ name, value, scope })
        
        # Clear story scope
        context.clearStoryScope()
        
        # Set new story
        context.currentStory = targetStory
        
        # Apply transfers
        for t in transfers:
            context.set(t.name, t.value, scope: t.scope)
    
    # Jump to label
    context.instructionPointer = labelIndex
    
    return ok()
```

### Syntactic Sugar

```zoh
====> @label;                    # Same story
====> @story:label;              # Different story
====> @label *var1 *var2;        # With transfers
```

---

## Fork

**Purpose**: Create new context running in parallel.

### Signature
```
/fork [clone] story?, label, var1, var2, ...;
```

### Behavior

1. Create new context
2. If `[clone]`, copy all variables and flags from parent
3. `/set` specified variables in new context
4. Start execution from label in parallel
5. Parent continues immediately

### Implementation

```
ForkDriver.execute(call, context):
    storyPath = resolve(call.params[0], context)
    labelName = resolve(call.params[1], context).toString()
    initVars = call.unnamedParams.slice(2)
    shouldClone = hasAttribute(call, "clone")
    
    # Resolve target
    if storyPath.isNothing():
        targetStory = context.currentStory
    else:
        targetStory = context.runtime.loadStory(storyPath.toString())
        if targetStory == null:
            return fatal("invalid_story", "Story not found: " + storyPath.toString())
    
    # Resolve label
    labelIndex = targetStory.labels.get(labelName.toLowerCase())
    if labelIndex == null:
        return fatal("invalid_label", "Label not found: " + labelName)
    
    # Create new context
    if shouldClone:
        newContext = context.clone()
    else:
        newContext = Context.new(context.runtime)
    
    # Set initial variables
    for ref in initVars:
        value = context.get(ref.name)
        scope = context.getVariableScope(ref.name)
        newContext.set(ref.name, value, scope: scope)
    
    # Initialize context state
    newContext.currentStory = targetStory
    newContext.instructionPointer = labelIndex
    
    # Register and start context
    context.runtime.addContext(newContext)
    context.runtime.scheduleContext(newContext)
    
    return ok()
```

### Syntactic Sugar

```zoh
====+ @label;                     # Fork to label
====+ [clone] @label;             # Fork with clone
====+ @story:label *var1 *var2;   # Fork with variable init
```

---

## Call

**Purpose**: Fork and wait for completion, returning the result.

### Signature
```
/call [inline] [clone] story?, label, var1, var2, ...;
```

### Behavior

1. Fork new context (same as `/fork`)
2. Block parent context until forked context terminates
3. Return last return value from forked context
4. If `[inline]`, copy specified variables back to parent

### Implementation

```
CallDriver.execute(call, context):
    storyPath = resolve(call.params[0], context)
    labelName = resolve(call.params[1], context).toString()
    initVars = call.unnamedParams.slice(2)
    shouldClone = hasAttribute(call, "clone")
    shouldInline = hasAttribute(call, "inline")
    
    # Resolve target story
    if storyPath.isNothing():
        targetStory = context.currentStory
    else:
        targetStory = context.runtime.loadStory(storyPath.toString())
        if targetStory == null:
            return fatal("invalid_story", "Story not found: " + storyPath.toString())
    
    # Resolve label
    labelIndex = targetStory.labels.get(labelName.toLowerCase())
    if labelIndex == null:
        return fatal("invalid_label", "Label not found: " + labelName)
    
    newContext = shouldClone ? context.clone() : Context.new(context.runtime)
    
    for ref in initVars:
        value = context.get(ref.name)
        newContext.set(ref.name, value)
    
    newContext.currentStory = targetStory
    newContext.instructionPointer = labelIndex
    
    # Schedule new context
    context.runtime.addContext(newContext)
    context.runtime.scheduleContext(newContext)
    
    # Block parent until child done
    context.state = WAITING_CONTEXT
    context.waitCondition = ContextWaitCondition {
        targetContextId: newContext.id,
        inlineVars: shouldInline ? initVars : []
    }
    
    return ok()
```

### Syntactic Sugar

```zoh
<===+ @label;                       # Call and wait
<===+ [inline] @label *var;         # Call and merge back
```

---

## Channels

Channels are named FIFO pipes for inter-context communication, routed through hubs.

### Channel Properties

| Property | Description |
|----------|-------------|
| Name | Unique identifier (case-insensitive) |
| Type | Unbounded FIFO with per-context outboxes |
| Scope | Global across all contexts (via hub) |
| Safety | Thread-safe, concurrent access. All hub and outbox/inbox operations must use concurrency-safe data structures. |

### Channel Hub Registry

```
ChannelHubRegistry:
  hubs: Map<string, ChannelHub>
  
  get(name: string): ChannelHub?
    return hubs.get(name.toLowerCase())
  
  getOrCreate(name: string): ChannelHub
    name = name.toLowerCase()
    if name not in hubs:
      hubs[name] = ChannelHub.new(name)
    return hubs[name]
  
  recreate(name: string): ChannelHub
    name = name.toLowerCase()
    old = hubs.get(name)
    newHub = ChannelHub.new(name)
    if old != null:
      newHub.generation = old.generation + 1
    hubs[name] = newHub
    return newHub
  
  remove(name: string)
    hubs.remove(name.toLowerCase())
```

### Channel Hub

```
ChannelHub:
  name: string
  generation: int
  closed: bool
  sequenceCounter: int                                     # Monotonic, incremented on each push
  participants: Map<contextId, ParticipantState>            # Auto-registered on first push/pull (or /open)
  waitingPullers: Queue<Context>                            # Contexts blocked on /pull (FIFO)
  waitingPushers: Queue<(Context, int)>                     # Contexts blocked on waited push (context, seqNum)

ParticipantState:
  context: Context
  outbox: Queue<(Value, int)>                              # (value, sequenceNumber) pairs
  inbox: Queue<Value>                                      # Values dispatched to this context
```

### Helper: ensureParticipant

```
ensureParticipant(hub, context, channelName):
    if context.id not in hub.participants:
        if channelName not in context.outboxes:
            context.outboxes[channelName] = Queue.new()
        if channelName not in context.inboxes:
            context.inboxes[channelName] = Queue.new()
        hub.participants[context.id] = ParticipantState {
            context: context,
            outbox: context.outboxes[channelName],
            inbox: context.inboxes[channelName]
        }
```

> [!IMPORTANT]
> **Concurrency safety**: All hub fields and outbox/inbox queues must be implemented with concurrency-safe data structures (e.g., concurrent collections, lock-guarded access). The scan-and-dequeue in the pull flow must be atomic — concurrent pullers must not race on the same outbox entry.

> [!IMPORTANT]
> **Close Notification**: Closing a channel MUST instantly wake all blocked pullers AND waited pushers.
> Pullers receive an error, not a nothing, to distinguish from timeout.
> Waited pushers receive an error to unblock.

> [!NOTE]
> **Generation IDs**: Each hub has a generation number. Pullers and waited pushers should check
> generation before/after blocking to ensure they don't accidentally operate on
> a NEW channel with the same name created after the old one closed.
>
> When `/open` re-creates a closed channel, the new hub has `generation = oldGeneration + 1`.

> [!NOTE]
> **Clone behavior**: When forking with `[clone]`, outboxes and inboxes are NOT cloned.
> A forked context starts with empty channel state. It will be auto-registered with hubs on first push/pull.

---

## Open

**Purpose**: Create a new channel or re-create a closed channel. Also registers the calling context with the hub as a side effect.

### Signature
```
/open channel;
```

### Behavior

- Creates a new hub if it doesn't exist
- If hub exists and is closed, creates a new hub with incremented generation
- If hub exists and is open, no-op for the hub
- Initializes context-local outbox and inbox for this channel (idempotent)
- Registers context with hub participation dictionary (idempotent)

### Implementation

```
OpenDriver.execute(call, context):
    channelRef = resolve(call.params[0], context)
    
    if channelRef is not ChannelValue:
        return fatal("invalid_type", "Expected channel, got: " + channelRef.getType())
    
    channelName = channelRef.name
    
    hub = context.runtime.channelHubs.getOrCreate(channelName)
    if hub.closed:
        hub = context.runtime.channelHubs.recreate(channelName)
    
    # Initialize context's outbox and inbox for this channel (idempotent)
    if channelName not in context.outboxes:
        context.outboxes[channelName] = Queue.new()
    if channelName not in context.inboxes:
        context.inboxes[channelName] = Queue.new()
    
    # Register context with hub participation dictionary (idempotent)
    if context.id not in hub.participants:
        hub.participants[context.id] = ParticipantState {
            context: context,
            outbox: context.outboxes[channelName],
            inbox: context.inboxes[channelName]
        }
    
    return ok()
```

---

## Push

**Purpose**: Add a value to a channel. Optionally block until consumed.

### Signature
```
/push channel, value, wait:bool?, timeout:seconds?;
```

### Behavior

- Auto-registers context with hub if not already a participant (lazy outbox/inbox creation)
- Pushes value to the calling context's outbox for the channel
- If a puller is waiting, delivers directly to puller's inbox (fast path — no blocking regardless of `wait`)
- If `wait: true` (default), blocks until the value is consumed by a puller
- If `wait: false`, returns immediately after enqueuing
- `timeout` controls max wait time when `wait: true`; ignored when `wait: false`

### Diagnostics

- Fatal: `invalid_type` — Parameter is not a channel
- Error: `not_found` — Channel does not exist
- Error: `closed` — Channel is closed, or closed while waiting
- Info: `timeout` — Push timed out before value was consumed (waited push only)

### Implementation

```
PushDriver.execute(call, context):
    channelRef = resolve(call.params[0], context)
    value = resolve(call.params[1], context)
    wait = getNamedParam(call, "wait") ?? true         # default: blocking
    timeout = getNamedParam(call, "timeout")            # optional, double or null
    
    if channelRef is not ChannelValue:
        return fatal("invalid_type", "Expected channel, got: " + channelRef.getType())
    
    channelName = channelRef.name
    
    hub = context.runtime.channelHubs.get(channelName)
    if hub == null:
        return error("not_found", "Channel not found: " + channelName)
    if hub.closed:
        return error("closed", "Channel closed: " + channelName)
    
    # Auto-register context if not yet a participant
    ensureParticipant(hub, context, channelName)
    
    seq = hub.sequenceCounter++
    
    # Fast path: if a puller is waiting, deliver directly (skip outbox)
    if hub.waitingPullers.hasNext():
        target = hub.waitingPullers.dequeue()
        targetState = hub.participants[target.id]
        targetState.inbox.enqueue(value)
        target.state = RUNNING
        target.waitCondition = null
        return ok()    # Value consumed immediately — no blocking needed
    
    # No puller waiting — park value in pusher's outbox
    participantState = hub.participants[context.id]
    participantState.outbox.enqueue((value, seq))
    
    if wait:
        # Block until this specific value is consumed
        hub.waitingPushers.enqueue((context, seq))
        context.state = WAITING_CHANNEL_PUSH
        context.waitCondition = ChannelPushWaitCondition {
            channelName, seq, generation: hub.generation,
            timeout: timeout != null ? resolve(timeout, context).toDouble() : null,
            startTime: now()
        }
    
    return ok()
```

---

## Pull

**Purpose**: Remove and return first value from channel.

### Signature
```
/pull channel, timeout:seconds?;
```

### Behavior

- Auto-registers context with hub if not already a participant (lazy outbox/inbox creation)
- Checks own inbox first (previously dispatched values)
- If inbox empty, scans all participating outboxes for the value with lowest sequence number
- Wakes blocked waited pusher if the consumed value was from a waited push
- Blocks if no values available anywhere

### Implementation

```
PullDriver.execute(call, context):
    channelRef = resolve(call.params[0], context)
    timeout = getNamedParam(call, "timeout")
    
    if channelRef is not ChannelValue:
        return fatal("invalid_type", "Expected channel, got: " + channelRef.getType())
    
    channelName = channelRef.name
    
    hub = context.runtime.channelHubs.get(channelName)
    if hub == null:
        return error("not_found", "Channel not found: " + channelName)
    if hub.closed:
        return error("closed", "Channel closed: " + channelName)
    
    # Auto-register context if not yet a participant
    ensureParticipant(hub, context, channelName)
    
    # Check own inbox first (previously dispatched values)
    myState = hub.participants[context.id]
    if myState.inbox.hasValue():
        return ok(myState.inbox.dequeue())
    
    # Scan participating outboxes — find lowest sequence number
    # NOTE: This scan-and-dequeue MUST be atomic (hub-level lock)
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
        channelName: channelName,
        timeout: timeout != null ? resolve(timeout, context).toDouble() : null,
        generation: hub.generation,
        startTime: now()
    }
    return ok()
```

---

## Close

**Purpose**: Close a channel, preventing further operations.

### Signature
```
/close channel;
```

### Implementation

```
CloseDriver.execute(call, context):
    channelRef = resolve(call.params[0], context)
    
    if channelRef is not ChannelValue:
        return fatal("invalid_type", "Expected channel")
    
    channelName = channelRef.name
    
    hub = context.runtime.channelHubs.get(channelName)
    if hub == null:
        return error("not_found", "Channel does not exist: " + channelName)
    if hub.closed:
        return error("closed", "Channel already closed: " + channelName)
    
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
    
    # Hub is NOT removed — kept with closed=true so subsequent
    # push/pull returns "closed" (not "not_found"), and generation
    # is preserved for /open re-creation.
    return ok()
```

---

## Wait / Signal

For broadcast communication across all contexts.

### Wait

```
/wait name, timeout:seconds?;
```

Blocks until a message with matching name is received.

```
WaitDriver.execute(call, context):
    name = resolve(call.params[0], context).toString()
    timeout = getNamedParam(call, "timeout")?.toDouble()

    context.state = WAITING_MESSAGE
    context.waitCondition = MessageWaitCondition {
        messageName: name,
        timeout: timeout != null ? resolve(timeout, context).toDouble() : null,
        startTime: now()
    }
    
    context.runtime.signals.subscribe(name, context)
    
    return ok()
```

### Signal

```
/signal name, message;
```

Broadcasts a message to all waiting contexts.

```
SignalDriver.execute(call, context):
    name = resolve(call.params[0], context).toString()
    message = resolve(call.params[1], context)
    
    context.runtime.broadcastMessage(name, message)
    return ok()
```

---

## Sleep

**Purpose**: Block context for specified duration.

### Signature
```
/sleep seconds;
```

### Implementation

```
SleepDriver.execute(call, context):
    seconds = resolve(call.params[0], context)

    if seconds is VerbValue:
        seconds = executeVerb(seconds, context)
    if seconds is ExpressionValue:
        seconds = evaluate(seconds, context)

    duration = seconds.toDouble()
    
    context.state = SLEEPING
    context.waitCondition = SleepCondition {
        wakeTime: now() + duration * 1000
    }
    
    return ok()
```

---

## Flag

**Purpose**: Set context-wide flags visible to all verb drivers.

### Signature
```
/flag name, value;
```

### Implementation

```
FlagDriver.execute(call, context):
    name = resolve(call.params[0], context).toString()
    value = resolve(call.params[1], context)
    
    context.setFlag(name, value)
    return ok()

# Flags are copied to forked contexts
Context.clone():
    newContext = Context.new(runtime)
    newContext.flags = this.flags.clone()
    newContext.contextVars = this.contextVars.clone()
    newContext.storyVars = this.storyVars.clone()
    return newContext
```

---

## Context Lifecycle

```
┌─────────────────────────────────────────────────────┐
│                  Context Lifecycle                  │
└─────────────────────────────────────────────────────┘

    ┌──────────┐
    │ Created  │  (by fork/call or initial main)
    └────┬─────┘
         │
         ▼
    ┌──────────┐
    │ Running  │  (executing statements)
    └────┬─────┘
         │
         ├──── /sleep ───► Sleeping ───► Running
         │
         ├──── /pull (blocking) ───► Waiting ───► Running
         │
         ├──── /push (waited) ───► Waiting ───► Running
         │
         ├──── /wait ───► Waiting ───► Running
         │
         │
         ▼
    ┌──────────┐
    │ Exiting  │  (no more statements or /exit)
    │          │  1. Execute story defers (LIFO)
    │          │  2. Execute context defers (LIFO)
    └────┬─────┘
         │
         ▼
    ┌──────────┐
    │Terminated│  (removed from runtime)
    └──────────┘
```

---

## Testing Checklist

### Jump
- [ ] Jump within same story
- [ ] Jump to different story
- [ ] Transfer variables
- [ ] Story defers execute on jump out
- [ ] Invalid label error

### Fork
- [ ] Basic fork creates new context
- [ ] Fork with clone copies state
- [ ] Fork with variable init
- [ ] Parent continues after fork
- [ ] Forked context runs independently

### Call
- [ ] Call blocks until completion
- [ ] Return value from called context
- [ ] Inline merge variables back
- [ ] Error in called context propagates

### Channels
- [ ] Push/pull basic
- [ ] Pull blocks until available
- [ ] Pull with timeout
- [ ] Channel FIFO order
- [ ] Channel cross-context
- [ ] Close channel

### Wait/Signal
- [ ] Signal wakes waiting contexts
- [ ] Wait timeout returns a nothing
- [ ] Multiple contexts waiting

### Sleep
- [ ] Sleep blocks for duration
- [ ] Sleep with verb (e.g., /rand)
- [ ] Sleep with expression

### Flags
- [ ] Set and read flags
- [ ] Flags visible to verbs
- [ ] Flags copied on fork/clone
