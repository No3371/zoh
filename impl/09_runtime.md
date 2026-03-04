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
  contexts: List<Context>               # internal
  channelHubs: ChannelHubRegistry
  storage: PersistentStorage
  signals: SignalManager
  elapsedMs: double                      # internal — accumulated from tick(deltaTimeMs) calls
  
  # Operations
  loadStory(path: string): CompiledStory
  startContext(story: CompiledStory): ContextHandle
  tick(deltaTimeMs: double): void
  resume(handle: ContextHandle, value: Value): void
  shutdown(): void
  
  # Signals (internal — used by verb drivers, not callers)
  subscribe(name: string, contextId: string): void
  unsubscribe(name: string, contextId: string): void
  broadcastSignal(name: string, payload: Value): void

RuntimeConfig:
  assetResolver: (address: string) -> bytes
  maxContexts: int
  maxStatementsPerTick: int        # Statement budget per context.run() invocation
  executionTimeoutMs: int
  enableDiagnostics: bool
```

### Public Types

```
ContextHandle:
  id: string                    # Opaque identifier
  state: ContextState           # Read-only current state

ExecutionResult:
  value: Value                  # Last return value
  diagnostics: List<Diagnostic> # All diagnostics from the run
  variables: VariableAccessor   # Lazy accessor into finished context state

VariableAccessor:
  get(name: string): Value      # Read a variable by name
  has(name: string): bool       # Check if a variable exists
  keys(): List<string>          # List all variable names
```

> `ContextHandle` is the only representation of a context visible to callers. It exposes read-only state sufficient for host code to identify and track contexts. Internal fields (IP, continuations, defers, delegates) are not accessible.
>
> `VariableAccessor` reads from the internal context on demand — no variable data is copied until accessed. Only valid on terminated contexts (from `ExecutionResult`).

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

  execute(call: CompiledVerbCall, context: Context): DriverResult

# Discriminated union: a verb either completes immediately or suspends.
# Replaces the old ExecutionResult.
DriverResult:
  | Complete {
      value: Value
      diagnostics: List<Diagnostic>
    }
  | Suspend {
      continuation: Continuation
      diagnostics: List<Diagnostic>
    }

# Pairs a wait request with a resume handler.
# The runtime fulfills the request; the driver handles the outcome.
Continuation:
  request: WaitRequest
  onFulfilled: (WaitOutcome) -> DriverResult

# Describes the external condition the driver is waiting for.
# Drivers MUST NOT set context.state or context.waitCondition directly.
WaitRequest:
  | Sleep       { durationMs: double }
  | ChannelPull { channelName: string, generation: int, timeoutMs: double? }
  | ChannelPush { channelName: string, seqNum: int, generation: int, timeoutMs: double? }
  | Signal      { messageName: string, timeoutMs: double? }
  | JoinContext { contextId: string }
  | Host        { timeoutMs: double? }

# What actually happened when the wait condition was fulfilled.
# Produced by the runtime, consumed by onFulfilled.
WaitOutcome:
  | Completed { value: Value }
  | TimedOut
  | Cancelled { code: string, message: string }
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

## Context Structure (Internal)

> **Implementation detail.** The `Context` type is internal to the runtime. Callers interact via `ContextHandle` (see Runtime Interface). This section documents the internal execution state for implementers.

```
Context:
  id: string                      # Unique context ID
  # Access to runtime services (verb registry, story cache, channel hubs,
  # signal manager) is implementation-specific — back-reference, delegates,
  # or dependency injection. Not part of the public contract.
  
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
  pendingContinuation: Continuation?   # Stored while blocked; cleared on resume
  waitCondition: WaitCondition?        # For scheduler condition checks
  resumeToken: int                     # Incremented on each Suspend; callers must match

ContextState:
  RUNNING                # Actively executing
  WAITING_CHANNEL        # Blocked on /pull
  WAITING_CHANNEL_PUSH   # Blocked on waited /push
  WAITING_MESSAGE        # Blocked on /wait
  WAITING_CONTEXT        # Blocked on /call
  WAITING_HOST           # Blocked on host interaction (/converse, /choose, etc.)
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
            validateContract(statement)
            instructionPointer++
            continue

        call = statement as CompiledVerbCall

        # Find verb driver (Suffix Matching)
        suffix = call.namespace ? call.namespace + "." + call.name : call.name
        driver = runtime.verbDrivers.resolveBySuffix(suffix)

        if driver == null:
            # Unknown verb - treat as no-op
            runtime.log(WARN, "Unknown verb: " + suffix)
            instructionPointer++
            continue

        # Capture state before execution to detect jumps
        entryIp = instructionPointer
        entryStory = currentStory

        try:
            result = driver.execute(call, this)
        catch FatalError as e:
            lastDiagnostics = [Diagnostic(FATAL, e.message)]
            terminate()
            return

        applyResult(result, entryIp, entryStory)

# Unified result handler — used by both run() and resume().
# entryIp/entryStory: IP and story captured before driver execution.
# IP only advances when the driver has not already modified it (jump guard).
Context.applyResult(result: DriverResult, entryIp: int, entryStory: CompiledStory):
    match result:
        Complete { value, diagnostics }:
            lastReturnValue = value
            lastDiagnostics = diagnostics
            if hasFatal(diagnostics):
                terminate()
            elif instructionPointer == entryIp and currentStory == entryStory:
                instructionPointer++
                # run() loop continues to next statement

        Suspend { continuation, diagnostics }:
            lastDiagnostics = diagnostics
            resumeToken++
            pendingContinuation = continuation
            blockOnRequest(continuation.request)
            # state is now SLEEPING/WAITING_* — run() loop exits (state != RUNNING)

# Called by the scheduler OR the host when the wait condition is met.
# token must match resumeToken — rejects stale resumes (e.g., host responds
# after timeout already advanced the context).
Context.resume(outcome: WaitOutcome, token: int):
    if token != resumeToken:
        return    # Stale token — context has moved on
    if pendingContinuation == null:
        return    # Already resumed (double-call race) — no-op

    handler = pendingContinuation.onFulfilled
    pendingContinuation = null
    waitCondition = null

    result = handler(outcome)
    state = RUNNING
    applyResult(result, instructionPointer, currentStory)
    # If Complete → IP advances, run() can continue
    # If Suspend  → re-blocks, IP stays, new continuation stored

Context.terminate():
    # Execute defers
    executeStoryDefers()
    executeContextDefers()

    # Channel cleanup
    cleanupChannels()

    state = TERMINATED
    runtime.removeContext(this)

# Maps a WaitRequest to internal scheduler state.
# This is the ONLY place context.state and context.waitCondition are set for blocking.
# Thinner than the old block() — no verb-specific logic.
Context.blockOnRequest(request: WaitRequest):
    match request:
        Sleep { durationMs }:
            state = SLEEPING
            waitCondition = SleepCondition { wakeTime: runtime.elapsedMs + durationMs }

        ChannelPull { channelName, generation, timeoutMs }:
            hub = runtime.channelHubs.get(channelName)
            hub.waitingPullers.enqueue(this)
            state = WAITING_CHANNEL
            waitCondition = ChannelWaitCondition {
                channelName, generation, startTime: runtime.elapsedMs, timeout: timeoutMs
            }

        ChannelPush { channelName, seqNum, generation, timeoutMs }:
            hub = runtime.channelHubs.get(channelName)
            hub.waitingPushers.enqueue((this, seqNum))
            state = WAITING_CHANNEL_PUSH
            waitCondition = ChannelPushWaitCondition {
                channelName, seq: seqNum, generation, startTime: runtime.elapsedMs, timeout: timeoutMs
            }

        Signal { messageName, timeoutMs }:
            runtime.signals.subscribe(messageName, this)
            state = WAITING_MESSAGE
            waitCondition = MessageWaitCondition {
                messageName, startTime: runtime.elapsedMs, timeout: timeoutMs
            }

        JoinContext { contextId }:
            state = WAITING_CONTEXT
            waitCondition = ContextWaitCondition { targetContextId: contextId }

        Host { timeoutMs }:
            state = WAITING_HOST
            waitCondition = HostWaitCondition { startTime: runtime.elapsedMs, timeout: timeoutMs }

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
| `/converse`, `/choose`, `/prompt` | Host interaction | Host calls `runtime.resume(handle, value)` |

### Tick-Loop Scheduler

The scheduler is a pure condition resolver. It checks each blocked context, determines if the wait condition is met, and if so produces a `WaitOutcome` and calls `context.resume()`. No verb-specific logic lives here.

```
Runtime.tick(deltaTimeMs: double):
    for context in contexts:
        if context.state != RUNNING and context.state != TERMINATED:
            token = context.resumeToken
            outcome = resolveWait(context)
            if outcome != null:
                context.resume(outcome, token)

        if context.state == RUNNING:
            context.run()

# Returns WaitOutcome if the condition is met, null if still waiting.
resolveWait(context: Context): WaitOutcome?
    match context.state:
        SLEEPING:
            if runtime.elapsedMs >= context.waitCondition.wakeTime:
                return Completed { Nothing }
            return null

        WAITING_CHANNEL:
            hub = runtime.channelHubs.get(context.waitCondition.channelName)
            if hub == null:
                return Cancelled { "not_found", "Channel not found" }
            if hub.closed:
                return Cancelled { "closed", "Channel closed" }
            if context.waitCondition.isTimedOut():
                hub.waitingPullers.remove(context)
                return TimedOut
            # Value delivery handled by PushDriver fast path
            return null

        WAITING_CHANNEL_PUSH:
            hub = runtime.channelHubs.get(context.waitCondition.channelName)
            if hub == null:
                return Cancelled { "not_found", "Channel not found" }
            if hub.closed:
                return Cancelled { "closed", "Channel closed" }
            if context.waitCondition.isTimedOut():
                hub.waitingPushers.remove(context, context.waitCondition.seq)
                return TimedOut
            return null

        WAITING_CONTEXT:
            target = contexts.find(c => c.id == context.waitCondition.targetContextId)
            if target == null or target.state == TERMINATED:
                return Completed { target?.lastReturnValue ?? Nothing }
            return null

        WAITING_MESSAGE:
            if context.waitCondition.isFulfilled():
                runtime.signals.unsubscribe(context.waitCondition.messageName, context)
                return Completed { context.waitCondition.payload }
            if context.waitCondition.isTimedOut():
                runtime.signals.unsubscribe(context.waitCondition.messageName, context)
                return TimedOut
            return null

        WAITING_HOST:
            # Host-driven: the scheduler does NOT resolve this.
            # The host calls context.resume(outcome, token) directly.
            # Scheduler only handles timeout if configured.
            if context.waitCondition.isTimedOut():
                return TimedOut
            return null

    return null
```

### Host Resume Path

For `WAITING_HOST`, fulfillment is the host application's responsibility — the scheduler only checks timeout. The verb driver calls the host handler *before* returning `Suspend`, passing a `ContextHandle`. The host feeds the response back via `runtime.resume(handle, value)`.

```
# Host-side example (game engine / UI framework):
# The verb driver passes a ContextHandle to the host handler before suspending.
# The host calls runtime.resume(handle, value) when the interaction completes.
# The runtime resolves the handle, validates the resume token internally,
# and delegates to the internal context.

onChoose(runtime, handle, request):
    displayChoiceUI(request, onSelected: (value) ->
        runtime.resume(handle, value)
    )

onConverse(runtime, handle, request):
    displayDialog(request, onDismissed: () ->
        runtime.resume(handle, Nothing)
    )
```

---

## Execution Model Compatibility

The `WaitRequest` type decouples **blocking intent** (declared by drivers via `Suspend.continuation.request`) from **scheduling strategy** (chosen by the implementer). Drivers are written once and work under either model.

### Cooperative Tick Model

The host calls `runtime.tick(deltaTimeMs)` each frame. The runtime accumulates `elapsedMs += deltaTimeMs`, then checks each blocked context via `resolveWait()` and resumes those whose condition is met. Host interactions (`WAITING_HOST`) are resumed via `runtime.resume(handle, value)`. The runtime never reads a system clock — all time is host-supplied.

```
# Host loop (game engine example):
while running:
    dt = timeSinceLastFrame()
    runtime.tick(dt)
    renderFrame()
    sleepUntilNextFrame()
```

`Context.blockOnRequest()` populates `context.state` and `context.waitCondition`; `resolveWait()` reads them unchanged. No tick-loop logic changes are needed to support new blocking verbs.

### Async Task Model

Each context runs as an async task. When a `Suspend` is returned, the runtime maps the `WaitRequest` to an awaitable and calls `context.resume()` when fulfilled. No external tick loop required. Suitable for servers, bots, and cloud hosts.

```
# Internal to the runtime — not a public API.
async runContextAsync(context: Context):
    while context.state != TERMINATED:
        context.run()
        if context.pendingContinuation != null:
            token = context.resumeToken
            outcome = await fulfillAsync(context.pendingContinuation.request)
            context.resume(outcome, token)

# WaitRequest → awaitable mapping:
async fulfillAsync(request: WaitRequest): WaitOutcome
    match request:
        Sleep { durationMs }      → await asyncSleep(durationMs); return Completed { Nothing }
        ChannelPull { ... }       → return await channel.pullAsync(timeout)
        ChannelPush { ... }       → return await channel.awaitConsumedAsync(seq, timeout)
        Signal { messageName }    → return await signals.waitAsync(messageName, timeout)
        JoinContext { contextId } → return Completed { await contexts.awaitTerminationAsync(contextId) }
        Host { ... }              → return await host.awaitInteractionAsync(timeout)
```

> **Note:** Multi-context parallelism (`/fork`) requires a scheduler under both models — either `tick()` advancing all contexts, or spawning a new async task per forked context. `WaitRequest` does not change this requirement.

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
