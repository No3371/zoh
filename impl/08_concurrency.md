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

Channels are named FIFO queues for inter-context communication.

### Channel Properties

| Property | Description |
|----------|-------------|
| Name | Unique identifier (case-insensitive) |
| Type | Unbounded FIFO queue |
| Scope | Global across all contexts |
| Safety | Thread-safe, concurrent access |

### Runtime Channel Storage

```
ChannelManager:
  channels: Map<string, Channel>
  
  get(name: string): Channel
    name = name.toLowerCase()
    if name not in channels:
      channels[name] = Channel.new()
    return channels[name]
  
  exists(name: string): bool
    return name.toLowerCase() in channels
  
  remove(name: string)
    channels.remove(name.toLowerCase())
  
  close(name: string)
    name = name.toLowerCase()
    if name in channels:
      channels[name].close()
      channels.remove(name)

Channel:
  queue: ConcurrentQueue<Value>
  closed: bool
  generation: int          # Incremented on each close to detect stale references
  waiters: List<Waiter>    # Blocked pullers waiting for values
  
  push(value: Value)
    if closed: error("Channel closed")
    queue.enqueue(value)
    notifyWaiters()        # Wake one waiter if any
  
  # Returns a PullResult: { status: "success"|"timeout"|"closed", value: Value? }
  pull(timeout: double?, expectedGeneration: int): PullResult
    if closed or generation != expectedGeneration:
      return { status: "closed" }
    
    if timeout == null:
      result = queue.dequeueBlocking()
      if result is CloseSignal: return { status: "closed" }
      return { status: "success", value: result }
    else:
      result = queue.dequeueWithTimeout(timeout)
      if result is TimeoutSignal: return { status: "timeout" }
      if result is CloseSignal: return { status: "closed" }
      return { status: "success", value: result }
  
  close()
    closed = true
    generation++
    # Instantly notify ALL blocked pullers
    for waiter in waiters:
      waiter.notifyClose()  # Wake with close signal, not a value
    waiters.clear()
  
  count(): int
    return queue.size

  # Non-blocking check
  hasValue(): bool
    return !queue.isEmpty()
```

> [!IMPORTANT]
> **Close Notification**: Closing a channel MUST instantly wake all blocked pullers.
> Pullers must receive an error, not a nothing, to distinguish from timeout.
> 
> **Generation IDs**: Each channel has a generation number. Pullers should check
> generation before/after blocking to ensure they don't accidentally pull from
> a NEW channel with the same name that was created after the old one closed.

---

## Push

**Purpose**: Add a value to a channel.

### Signature
```
/push channel, value;
```

### Behavior

- **Creates a new channel** if it doesn't exist or is closed

### Implementation

```
PushDriver.execute(call, context):
    channelRef = resolve(call.params[0], context)
    value = resolve(call.params[1], context)
    
    if channelRef is not ChannelValue:
        return fatal("invalid_type", "Expected channel, got: " + channelRef.getType())
    
    channelName = channelRef.name
    
    # get() auto-creates channel if it doesn't exist
    # If channel exists but is closed, create a new one
    if context.runtime.channels.exists(channelName):
        channel = context.runtime.channels.get(channelName)
        if channel.closed:
            context.runtime.channels.remove(channelName)
            channel = context.runtime.channels.get(channelName)  # Creates new
    else:
        channel = context.runtime.channels.get(channelName)  # Creates new
    
    channel.push(value)
    
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

- Blocks until value is available
- With timeout: returns a nothing if timeout expires
- **Throws error if channel doesn't exist or is closed**

### Implementation

```
PullDriver.execute(call, context):
    channelRef = resolve(call.params[0], context)
    timeout = getNamedParam(call, "timeout")
    
    if channelRef is not ChannelValue:
        return fatal("invalid_type", "Expected channel, got: " + channelRef.getType())
    
    channelName = channelRef.name
    
    # Check if channel exists
    if not context.runtime.channels.exists(channelName):
        return error("not_found", "Channel does not exist: " + channelName)
    
    channel = context.runtime.channels.get(channelName)
    
    # Check if channel is closed
    if channel.closed:
        return error("closed", "Cannot pull from closed channel: " + channelName)
    
    # Set blocking state
    context.state = WAITING_CHANNEL
    context.waitCondition = ChannelWaitCondition {
        channelName: channelName,
        timeout: timeout != null ? resolve(timeout, context).toDouble() : null,
        generation: channel.generation,
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
    
    # Check if channel exists
    if not context.runtime.channels.exists(channelName):
        return error("not_found", "Channel does not exist: " + channelName)
    
    channel = context.runtime.channels.get(channelName)
    
    # Check if already closed
    if channel.closed:
        return error("closed", "Channel already closed: " + channelName)
    
    context.runtime.channels.close(channelName)
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
