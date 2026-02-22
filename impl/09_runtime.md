# 09: Runtime Architecture

## Purpose

The runtime is the top-level system that manages contexts, coordinates execution, and provides the extension points for handlers.

---

## Architecture Overview

```
┌────────────────────────────────────────────────────────────────────────────┐
│                                RUNTIME                                     │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        HANDLER REGISTRY                              │   │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐    │   │
│  │  │Preprocessors│ │  Compilers  │ │ Validators  │ │VerbDrivers  │    │   │
│  │  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         STORY CACHE                                  │   │
│  │  story1.zoh → CompiledStory, story2.zoh → CompiledStory, ...        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                       CONTEXT MANAGER                                │   │
│  │  Context[0] (main), Context[1] (forked), Context[2] (forked), ...   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    CHANNEL HUB REGISTRY                              │   │
│  │  <channel1>:gen → Hub, <channel2>:gen → Hub, ...                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    PERSISTENT STORAGE                                │   │
│  │  store1 → {vars}, store2 → {vars}, ...                               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    SIGNAL MANAGER                                    │   │
│  │  /wait listeners, /signal broadcasters                               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Runtime Interface

```
Runtime:
  # Configuration
  config: RuntimeConfig
  
  # Handler registries (ordered by priority)
  preprocessors: List<Preprocessor>
  compilers: List<Compiler>
  storyValidators: List<StoryValidator>
  verbValidators: Map<string, VerbValidator>
  verbDrivers: Map<string, VerbDriver>
  
  # State
  stories: Map<string, CompiledStory>
  contexts: List<Context>
  channelHubs: ChannelHubRegistry
  storage: PersistentStorage
  signals: SignalManager
  
  # Operations
  loadStory(path: string): CompiledStory
  createContext(story: CompiledStory): Context
  run(context: Context): void
  runToCompletion(context: Context): Value
  shutdown(): void
  
  # Signals
  subscribe(name: string, context: Context): void
  unsubscribe(name: string, context: Context): void
  broadcastSignal(name: string, payload: Value): void

RuntimeConfig:
  assetResolver: (address: string) -> bytes
  maxContexts: int
  executionTimeoutMs: int
  enableDiagnostics: bool
```

---

## Handler Types

### Preprocessor

Operates on raw source text before parsing.

```
Preprocessor:
  priority: int
  
  process(source: string, metadata: StoryMetadata): PreprocessorResult

PreprocessorResult:
  source: string              # Transformed source
  diagnostics: List<Diagnostic>
```

### Compiler

Converts parsed AST to runtime data structures.

```
Compiler:
  priority: int
  
  # Called to check interest in current AST node
  canCompile(node: ASTNode, position: int): bool
  
  # Compile the node, may read ahead in AST
  compile(nodes: List<ASTNode>, position: int, compiled: CompiledStory): CompileResult

CompileResult:
  consumed: int               # Number of nodes consumed
  diagnostics: List<Diagnostic>
```

### Story Validator

Validates compiled story before execution.

```
StoryValidator:
  priority: int
  
  validate(story: CompiledStory): List<Diagnostic>
```

### Namespace Validator
Validates that verb calls and attributes are not ambiguous and resolve to existing symbols.

```
NamespaceValidator:
  priority: int
  registry: HandlerRegistry

  validate(story: CompiledStory): List<Diagnostic>
    # 1. Iterate all verb calls
    # 2. Suffix Match each call.name against verbRegistry
    # 3. If >1 match: FATAL "namespace_ambiguity"
    # 4. If 0 match: FATAL "unknown_verb"
    
    # 5. Iterate all attributes in calls
    # 6. Suffix Match each attribute.name against AttributeRegistry
    # 7. If >1 match: FATAL "namespace_ambiguity"
    # 8. If 0 match: Warning? or FATAL "unknown_attribute" (strict)

    # Optimization: 
    # The validation result for specific suffixes can be cached globally 
    # until the verb/attribute registry changes.
```

### Verb Validator (Specific)

Validates compiled verb calls.

```
VerbValidator:
  verbName: string            # Verb this validator handles
  priority: int
  
  validate(call: CompiledVerbCall, story: CompiledStory): List<Diagnostic>
```

### Verb Driver

Executes verb calls at runtime.

```
VerbDriver:
  namespace: string?
  name: string
  priority: int

  execute(call: CompiledVerbCall, context: Context): ExecutionResult

ExecutionResult:
  value: Value                # Return value
  diagnostics: List<Diagnostic>
  continuation: Continuation? # null = completed; non-null = context must yield

# Describes WHAT the driver is waiting for.
# The runtime decides HOW to fulfill it (tick-loop or async task).
# Drivers MUST NOT set context.state or context.waitCondition directly.
Continuation:
  | Sleep       { durationMs: double }
  | ChannelPull { channelName: string, generation: int, timeoutMs: double? }
  | ChannelPush { channelName: string, seqNum: int, generation: int, timeoutMs: double? }
  | Message     { messageName: string, timeoutMs: double? }
  | Context     { contextId: string, inlineVars: List<VarRef> }
```

---

## Handler Priority

| Priority Range | Purpose |
|----------------|---------|
| -2^31 to 0 | Core handlers (built-in) |
| 1 to 1000 | High-priority extensions |
| 1001 to 10000 | Standard extensions |
| 10001 to 2^31-1 | Low-priority extensions |

**Rule**: Only the highest-priority verb driver for a given verb name executes.

---

## Story Compilation Pipeline

```
┌─────────────┐
│ Source File │
└──────┬──────┘
       │
       ▼
┌──────────────────────────────────────────────┐
│             PREPROCESSOR PHASE               │
│  1. Load source text                         │
│  2. For each preprocessor (priority order):  │
│     - process(source, metadata)              │
│  3. Collect diagnostics                      │
│  4. If fatal: abort                          │
└──────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────┐
│               LEXER PHASE                    │
│  - Tokenize preprocessed source              │
│  - Generate token stream                     │
└──────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────┐
│               PARSER PHASE                   │
│  - Parse token stream to AST                 │
│  - Extract story name, metadata, checkpoints │
└──────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────┐
│              COMPILER PHASE                  │
│  For each AST node:                          │
│    For each compiler (priority order):       │
│      if canCompile(node):                    │
│        compile(nodes, position, story)       │
│  Result: CompiledStory                       │
│  If fatal diagnostics: abort                 │
└──────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────┐
│            VALIDATION PHASE                  │
│  1. For each story validator:                │
│     - validate(story)                        │
│  2. Run NamespaceValidator (Ambiguity Check) │
│  3. For each compiled verb call:             │
│     - Find verb validator                    │
│     - validate(call, story)                  │
│  4. If fatal diagnostics: abort              │
└──────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────┐
│             COMPILED STORY                   │
│  Ready for execution                         │
└──────────────────────────────────────────────┘
```

---

## Compiled Story Structure

```
CompiledStory:
  name: string
  metadata: Map<string, Value>
  checkpoints: Map<string, CompiledCheckpoint>  # checkpoint name → compiled node
  statements: List<CompiledStatement>
  sourceMap: SourceMap            # For error reporting

CompiledStatement:
  | CompiledVerbCall
  | CompiledCheckpoint

CompiledVerbCall:
  namespace: string?
  name: string
  attributes: List<CompiledAttribute>
  namedParams: Map<string, CompiledValue>
  unnamedParams: List<CompiledValue>
  sourceLine: int                 # Original source line
  storyLine: int                  # Line in compiled story body

ContractParam:
  name: string                    # Variable name
  type: string?                   # Type constraint (null = any non-nothing)

CompiledCheckpoint:
  name: string
  contract: List<ContractParam>   # Required variables with optional types
  statementIndex: int             # Index in statements list

CompiledValue:
  # Pre-processed for efficient execution
  | LiteralValue   { value: any, type: string }
  | ReferenceValue { name: string, index: CompiledValue? }
  | ChannelValue   { name: string }
  | ExprValue      { ast: ExprAST }
  | VerbValue      { call: CompiledVerbCall }
```

---

## Context Structure

```
Context:
  id: string                      # Unique context ID
  runtime: Runtime                # Parent runtime
  
  # Execution state
  currentStory: CompiledStory
  instructionPointer: int         # Current statement index
  state: ContextState             # RUNNING, WAITING, SLEEPING, TERMINATED
  
  # Variable storage
  storyVars: Map<string, Variable>
  contextVars: Map<string, Variable>
  flags: Map<string, Value>
  
  # Channel storage
  outboxes: Map<string, Queue<(Value, int)>>   # channel → queue of (value, seqNum)
  inboxes: Map<string, Queue<Value>>           # channel → queue of received values
  
  # Execution state
  lastReturnValue: Value
  lastDiagnostics: List<Diagnostic>
  
  # Deferred verbs
  storyDefers: Stack<CompiledVerbCall>
  contextDefers: Stack<CompiledVerbCall>
  
  # Waiting state
  waitCondition: WaitCondition?   # For /pull, /push, /wait, /sleep

ContextState:
  RUNNING                # Actively executing
  WAITING_CHANNEL        # Blocked on /pull
  WAITING_CHANNEL_PUSH   # Blocked on waited /push
  WAITING_MESSAGE        # Blocked on /wait
  WAITING_CONTEXT        # Blocked on /call
  SLEEPING               # Blocked on /sleep
  TERMINATED             # Finished execution
```

---

## Execution Loop

```
Context.run():
    while state == RUNNING:
        if instructionPointer >= currentStory.statements.length:
            terminate()
            return
        
        statement = currentStory.statements[instructionPointer]
        
        if statement is CompiledCheckpoint:
            # Checkpoints are markers, skip
            instructionPointer++
            continue
        
        call = statement as CompiledVerbCall
        
        # Find verb driver (Suffix Matching)
        # Note: Validator guarantees uniqueness or existence roughly, 
        # but runtime must still resolve the specific driver.
        suffix = call.namespace ? call.namespace + "." + call.name : call.name
        driver = runtime.verbDrivers.resolveBySuffix(suffix)
        
        if driver == null:
            # Unknown verb - treat as no-op
            runtime.log(WARN, "Unknown verb: " + driverKey)
            instructionPointer++
            continue
        
        # Execute
        try:
            result = driver.execute(call, this)
            lastReturnValue = result.value
            lastDiagnostics = result.diagnostics
        catch FatalError as e:
            lastDiagnostics.add(Diagnostic(FATAL, e.message))
            terminate()
            return

        # If driver returned a continuation, block and suspend
        if result.continuation != null:
            block(result.continuation)
            return  # Will resume when continuation is fulfilled

        instructionPointer++
    
Context.terminate():
    # Execute defers
    executeStoryDefers()
    executeContextDefers()

    # Channel cleanup
    cleanupChannels()

    state = TERMINATED
    runtime.removeContext(this)

# Processes a Continuation returned by a driver.
# Sets internal state/waitCondition and handles registrations (subscribe, enqueue).
# This is the ONLY place context.state and context.waitCondition are set for blocking.
Context.block(continuation: Continuation):
    match continuation:
        Sleep { durationMs }:
            state = SLEEPING
            waitCondition = SleepCondition { wakeTime: now() + durationMs }

        ChannelPull { channelName, generation, timeoutMs }:
            hub = runtime.channelHubs.get(channelName)
            hub.waitingPullers.enqueue(this)
            state = WAITING_CHANNEL
            waitCondition = ChannelWaitCondition {
                channelName, generation, startTime: now(),
                timeout: timeoutMs
            }

        ChannelPush { channelName, seqNum, generation, timeoutMs }:
            hub = runtime.channelHubs.get(channelName)
            hub.waitingPushers.enqueue((this, seqNum))
            state = WAITING_CHANNEL_PUSH
            waitCondition = ChannelPushWaitCondition {
                channelName, seq: seqNum, generation, startTime: now(),
                timeout: timeoutMs
            }

        Message { messageName, timeoutMs }:
            runtime.signals.subscribe(messageName, this)
            state = WAITING_MESSAGE
            waitCondition = MessageWaitCondition {
                messageName, startTime: now(), timeout: timeoutMs
            }

        Context { contextId, inlineVars }:
            state = WAITING_CONTEXT
            waitCondition = ContextWaitCondition {
                targetContextId: contextId, inlineVars
            }

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
    
    # Prune any waited-pusher entries for this context
    for channelName in outboxes.keys():
        hub = runtime.channelHubs.get(channelName)
        if hub != null:
            hub.waitingPushers.removeByContext(this.id)
```

---

## Blocking Operations

Some verbs block context execution:

| Verb | Block Condition | Unblock Condition |
|------|-----------------|-------------------|
| `/sleep` | Always | Timer expires |
| `/push` | `wait: true` and no puller | Value consumed, timeout, or channel closed |
| `/pull` | Channel empty | Value available, timeout, or channel closed |
| `/wait` | No message | Message received or timeout |
| `/call` | Forked context | Forked context terminates |

### Implementation Pattern

```
SleepDriver.execute(call, context):
    seconds = resolve(call.params[0], context).toDouble()

    # Declare blocking intent — runtime fulfills it via Context.block()
    return ok(continuation: Sleep { durationMs: seconds * 1000 })

# Context.block() translates the Continuation to internal state:
#   state = SLEEPING
#   waitCondition = SleepCondition { wakeTime: now() + durationMs }

# In runtime scheduler (tick-loop model):
Runtime.tick():
    for context in contexts:
        match context.state:
            SLEEPING:
                if now() >= context.waitCondition.wakeTime:
                    context.state = RUNNING
                    context.waitCondition = null
            
            WAITING_CHANNEL:
                hub = channelHubs.get(context.waitCondition.channelName)
                if hub == null:
                    context.state = RUNNING
                    context.lastDiagnostics = [Diagnostic(ERROR, "not_found", "Channel not found")]
                    context.waitCondition = null
                elif hub.closed:
                    context.state = RUNNING
                    context.lastDiagnostics = [Diagnostic(ERROR, "closed", "Channel closed")]
                    context.waitCondition = null
                elif context.waitCondition.isTimedOut():
                    hub.waitingPullers.remove(context)
                    context.state = RUNNING
                    context.lastReturnValue = Nothing
                    context.lastDiagnostics = [Diagnostic(INFO, "timeout", "Pull timed out")]
                    context.waitCondition = null
                # Otherwise: still waiting — pusher will wake us via PushDriver fast path

            WAITING_CHANNEL_PUSH:
                hub = channelHubs.get(context.waitCondition.channelName)
                if hub == null:
                    context.state = RUNNING
                    context.lastDiagnostics = [Diagnostic(ERROR, "not_found", "Channel not found")]
                    context.waitCondition = null
                elif hub.closed:
                    context.state = RUNNING
                    context.lastDiagnostics = [Diagnostic(ERROR, "closed", "Channel closed")]
                    context.waitCondition = null
                elif context.waitCondition.isTimedOut():
                    # Timeout — remove from waiting pushers, value stays in outbox
                    hub.waitingPushers.remove(context, context.waitCondition.seq)
                    context.state = RUNNING
                    context.lastDiagnostics = [Diagnostic(INFO, "timeout", "Push timed out")]
                    context.waitCondition = null
                # Otherwise: still waiting — puller will wake us via PullDriver

            WAITING_CONTEXT:
                targetCtx = contexts.find(c => c.id == context.waitCondition.targetContextId)
                if targetCtx == null or targetCtx.state == TERMINATED:
                    # Context finished
                    returnVal = targetCtx?.lastReturnValue ?? Nothing
                    
                    # Handle inline
                    if context.waitCondition.inlineVars.length > 0:
                         for ref in context.waitCondition.inlineVars:
                             val = targetCtx?.get(ref.name) ?? Nothing
                             context.set(ref.name, val)
                             
                    context.state = RUNNING
                    context.lastReturnValue = returnVal
                    context.waitCondition = null

            WAITING_MESSAGE:
                # SignalManager directly updates the waitCondition when signal arrives
                if context.waitCondition.isFulfilled():
                    signals.unsubscribe(context.waitCondition.messageName, context)
                    context.state = RUNNING
                    context.lastReturnValue = context.waitCondition.payload
                    context.waitCondition = null
                elif context.waitCondition.isTimedOut():
                    signals.unsubscribe(context.waitCondition.messageName, context)
                    context.state = RUNNING
                    context.lastReturnValue = Nothing
                    context.waitCondition = null
            
        # Execute if running (including if just unblocked)
        if context.state == RUNNING:
            context.run()

---

## Execution Model Compatibility

The `Continuation` type decouples **blocking intent** (declared by drivers via `ExecutionResult.continuation`) from **scheduling strategy** (chosen by the implementer). Drivers are written once and work under either model.

### Cooperative Tick Model

The host calls `runtime.tick()` periodically. On each tick, the runtime checks each blocked context's `waitCondition` and resumes those whose condition is met. Suitable for game engines and any host with a natural update loop.

```
# Host loop (game engine example):
while running:
    runtime.tick()
    renderFrame()
    sleepUntilNextFrame()
```

`Context.block()` populates `context.state` and `context.waitCondition`; `Runtime.tick()` reads them unchanged. No tick-loop logic changes are needed to adopt `Continuation`.

### Async Task Model

Each context runs as an async task. When a `Continuation` is returned, the runtime converts it to an awaitable and suspends the context task until fulfilled. No external tick loop required. Suitable for servers, bots, and cloud hosts.

```
# Continuation → awaitable mapping (illustrative):
async fulfillContinuation(c: Continuation, context: Context): Value
    match c:
        Sleep { durationMs }      → await asyncSleep(durationMs); return nothing
        ChannelPull { ... }       → return await channel.pullAsync(timeout)
        ChannelPush { ... }       → await channel.awaitConsumedAsync(seq, timeout); return nothing
        Message { messageName }   → return await signals.waitAsync(messageName, timeout)
        Context { contextId }     → return await contexts.awaitTerminationAsync(contextId)
```

> **Note:** Multi-context parallelism (`/fork`) requires a scheduler under both models — either `tick()` advancing all contexts, or spawning a new async task per forked context. `Continuation` does not change this requirement.

---

## Checkpoint Contract Validation

Navigation verbs (`/jump`, `/call`, `/fork`) AND the runtime execution loop (when falling through to a checkpoint) must validate checkpoint contracts before transferring control.

```
validateContract(checkpoint: CompiledCheckpoint, context: Context):
    for param in checkpoint.contract:
        val = context.get(param.name)
        
        if val is Nothing:
            FATAL "checkpoint_violation": "Required variable '*{param.name}' is nothing"
        
        if param.type != null:
            if not matchesType(val, param.type):
                FATAL "checkpoint_violation": "Variable '*{param.name}' expected {param.type}, got {typeOf(val)}"

matchesType(val: Value, expectedType: string): bool:
    match expectedType.toLowerCase():
        "integer" -> val is ZohInteger
        "double"  -> val is ZohDouble
        "string"  -> val is ZohString
        "boolean" -> val is ZohBool
        "list"    -> val is ZohList
        "map"     -> val is ZohMap
        "channel" -> val is ZohChannel
        "verb"    -> val is ZohVerb
        "expression" -> val is ZohExpression
        _ -> false
```
```

---

## Error Handling

### Diagnostic Levels

| Level | Behavior |
|-------|----------|
| INFO | Log, continue |
| WARNING | Log, continue |
| ERROR | Log, continue (may affect results) |
| FATAL | Log, terminate context |

### Diagnostic Structure

```
Diagnostic:
  level: DiagnosticLevel
  code: string            # Error code (e.g. "type_mismatch")
  message: string
  sourceLine: int?
  storyLine: int?
  storyName: string?
  verbName: string?
```

---

## Signal Manager

The Signal Manager implements a transient Event/Subscriber model. Signals are "fire-and-forget" unless there is an active subscriber.

```
SignalManager:
  listeners: Map<string, List<Context>>

  subscribe(name: string, context: Context):
    listeners.get(name).add(context)

  unsubscribe(name: string, context: Context):
    listeners.get(name).remove(context)

  broadcast(name: string, payload: Value):
    if listeners.has(name):
      for context in listeners.get(name):
        if context.state == WAITING_MESSAGE and context.waitCondition.messageName == name:
          context.waitCondition.fulfill(payload)
```

### Wait Condition

```
MessageWaitCondition:
  messageName: string
  payload: Value?
  fulfilled: bool
  
  fulfill(value: Value):
    payload = value
    fulfilled = true
```

---

## Asset Management

Assets (images, audio, etc.) are resolved by address, not file path:

```
AssetResolver: (address: string) -> bytes

# Built-in resolvers

VirtualAssetResolver:
  manifest: Map<string, string>   # resourceId -> uri (or alias)
  delegate: AssetResolver         # Chain of responsibility

  resolve(address: string):
    # 1. Check manifest for alias/mapping
    uri = manifest.get(address) ?? address
    
    # 2. Delegate to concrete resolver (e.g. File)
    return delegate.resolve(uri)

FileAssetResolver:
  basePath: string

  resolve(address: string):
    return readFile(basePath + "/" + address)

```

---

## Testing Checklist

### Pipeline
- [ ] Source → Preprocessor → Lexer → Parser → Compiler → Validator
- [ ] Handler priority ordering
- [ ] Error propagation through pipeline
- [ ] Source map accuracy

### Context
- [ ] Context creation and initialization
- [ ] Context execution and termination
- [ ] Context cloning
- [ ] Context state transitions

### Blocking
- [ ] Sleep blocks and resumes
- [ ] Pull blocks and resumes on value
- [ ] Pull timeout
- [ ] Wait/signal coordination

### Handler Registration
- [ ] Register custom preprocessor
- [ ] Register custom verb driver
- [ ] Priority override

### Error Handling
- [ ] Fatal error terminates context
- [ ] Non-fatal errors logged
- [ ] Diagnostics accessible via /diagnose
