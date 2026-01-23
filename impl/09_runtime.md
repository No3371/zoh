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
│  │                      CHANNEL MANAGER                                 │   │
│  │  <channel1> → Queue, <channel2> → Queue, ...                         │   │
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
  channels: ChannelManager
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

CompiledCheckpoint:
  name: string
  contract: List<string>          # Required variable names
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
  
  # Execution state
  lastReturnValue: Value
  lastDiagnostics: List<Diagnostic>
  
  # Deferred verbs
  storyDefers: Stack<CompiledVerbCall>
  contextDefers: Stack<CompiledVerbCall>
  
  # Waiting state
  waitCondition: WaitCondition?   # For /pull, /wait, /sleep

ContextState:
  RUNNING           # Actively executing
  WAITING_CHANNEL   # Blocked on /pull
  WAITING_MESSAGE   # Blocked on /wait
  WAITING_CONTEXT   # Blocked on /call
  SLEEPING          # Blocked on /sleep
  TERMINATED        # Finished execution
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
        
        # Check for state changes (blocking verbs set state)
        if state != RUNNING:
            return  # Will resume when unblocked
        
        instructionPointer++
    
Context.terminate():
    # Execute defers
    executeStoryDefers()
    executeContextDefers()
    
    state = TERMINATED
    runtime.removeContext(this)
```

---

## Blocking Operations

Some verbs block context execution:

| Verb | Block Condition | Unblock Condition |
|------|-----------------|-------------------|
| `/sleep` | Always | Timer expires |
| `/pull` | Channel empty | Value available or timeout |
| `/wait` | No message | Message received or timeout |
| `/call` | Forked context | Forked context terminates |

### Implementation Pattern

```
SleepDriver.execute(call, context):
    seconds = resolve(call.params[0], context).toDouble()
    
    # Set blocking state
    context.state = SLEEPING
    context.waitCondition = SleepCondition { 
        wakeTime: now() + seconds * 1000 
    }
    
    # Runtime scheduler will resume when condition met
    return ok()

# In runtime scheduler:
Runtime.tick():
    for context in contexts:
        match context.state:
            SLEEPING:
                if now() >= context.waitCondition.wakeTime:
                    context.state = RUNNING
                    context.waitCondition = null
            
            WAITING_CHANNEL:
                channel = channels.get(context.waitCondition.channelName)
                if channel.hasValue() or context.waitCondition.isTimedOut():
                    # Handle result with diagnostics even on resumption
                    result = channel.pull(context.waitCondition.timeout, context.waitCondition.generation)
                    context.state = RUNNING
                    context.lastReturnValue = result.value
                    context.lastDiagnostics = result.diagnostics
                    context.waitCondition = null

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
