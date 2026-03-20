# Runtime API Surface Revision — Spec Update

> **Status:** Complete
> **Created:** 2026-03-04
> **Completed:** 2026-03-04
> **Walkthrough:** 20260304-runtime-api-surface-spec-plan-walkthrough.md
> **Author:** Agent
> **Source:** 20260304-context-runtime-coupling-eval.md
> **Related Projex:** 20260226-zohruntime-run-context-design-eval.md

---

## Summary

Revise `impl/09_runtime.md` to internalize `Context` and expose `ContextHandle` + `ExecutionResult` as the public API surface. The runtime owns context lifecycle; callers interact via opaque handles and result types.

**Scope:** `impl/09_runtime.md` only — spec text changes. No C# implementation changes.
**Estimated Changes:** 1 file, 5 sections

---

## Objective

### Problem / Gap / Need

The spec's Runtime Interface exposes `Context` as a public type. `Context` is an internal execution state machine (instruction pointer, defer stacks, continuations, delegate slots). Exposing it lets callers corrupt execution, pass contexts to the wrong runtime, and creates dual ownership between the runtime's `contexts` list and the caller's reference. The spec arrived here by documenting an implementation leak, not by designing the API.

### Success Criteria

- [ ] Runtime Interface section uses `ContextHandle` instead of `Context` in all public operations
- [ ] `createContext` replaced by `startContext → ContextHandle`
- [ ] `run(context)` and `runToCompletion(context)` removed from public surface
- [ ] `tick(deltaTimeMs: double)` is the sole execution driver
- [ ] `resume(handle, value)` replaces direct `context.resume()` for host interaction
- [ ] `ExecutionResult` type defined with lazy `VariableAccessor`
- [ ] `RuntimeConfig` includes `maxStatementsPerTick` (statement budget)
- [ ] `elapsedMs` documented as internal runtime state (no system clock dependency)
- [ ] `Context` section explicitly marked as internal implementation
- [ ] Host Resume Path examples use `runtime.resume(handle, value)` instead of `context.resume()`
- [ ] Implementation note: runtimes may provide `runToCompletion` as a convenience (not spec-mandated)
- [ ] Execution Model Compatibility section updated for both tick and async models

### Out of Scope

- C# implementation changes (separate plan, separate projex scope)
- Channel hub, signal manager, or verb driver internals (unchanged)
- Blocking Operations table (unchanged — internal mechanics stay the same)
- `Context.run()`, `Context.applyResult()`, `Context.blockOnRequest()` pseudo-code (staying, but relabeled as internal)
- `resolveWait()` pseudo-code (unchanged)

---

## Context

### Current State

The Runtime Interface (L53-91) exposes:
```
createContext(story: CompiledStory): Context
run(context: Context): void
runToCompletion(context: Context): Value
```

The Context Structure (L347-390) documents all internal fields as public. The Host Resume Path (L636-656) shows callers calling `context.resume()` directly.

### Key Files

| File | Purpose | Changes Needed |
|------|---------|----------------|
| `impl/09_runtime.md` | Runtime architecture spec | All changes below |

### Dependencies

- **Requires:** None — spec-first
- **Blocks:** Downstream C# implementation plan (separate projex in `csharp/` scope)

---

## Implementation

### Overview

Five targeted edits to `impl/09_runtime.md`:
1. Runtime Interface — new public API surface
2. New types — `ContextHandle`, `ExecutionResult`, `VariableAccessor`
3. Context Structure — mark as internal
4. Host Resume Path — use `runtime.resume(handle, value)`
5. Execution Model Compatibility — update both models

### Step 1: Revise Runtime Interface (L53–91)

**Objective:** Replace `Context`-exposing operations with `ContextHandle`-based ones.

**File:** `impl/09_runtime.md`

**Changes:**

```
# Before (L74-79):
  # Operations
  loadStory(path: string): CompiledStory
  createContext(story: CompiledStory): Context
  run(context: Context): void
  runToCompletion(context: Context): Value
  shutdown(): void

# After:
  # Operations
  loadStory(path: string): CompiledStory
  startContext(story: CompiledStory): ContextHandle
  tick(deltaTimeMs: double): void
  resume(handle: ContextHandle, value: Value): void
  shutdown(): void
```

> `runToCompletion` is removed from the spec. It is an implementation convenience that runtimes may provide for tests and simple embedders. It is not part of the public API contract.

Also update the Signals section — `subscribe`/`unsubscribe` take context IDs internally now, not `Context`:

```
# Before (L81-84):
  # Signals
  subscribe(name: string, context: Context): void
  unsubscribe(name: string, context: Context): void

# After:
  # Signals (internal — used by verb drivers, not callers)
  subscribe(name: string, contextId: string): void
  unsubscribe(name: string, contextId: string): void
```

Add `maxStatementsPerTick` to `RuntimeConfig`:

```
# Before (L86-91):
RuntimeConfig:
  assetResolver: (address: string) -> bytes
  maxContexts: int
  executionTimeoutMs: int
  enableDiagnostics: bool

# After:
RuntimeConfig:
  assetResolver: (address: string) -> bytes
  maxContexts: int
  maxStatementsPerTick: int        # Statement budget per context.run() invocation
  executionTimeoutMs: int
  enableDiagnostics: bool
```

Also add `elapsedMs` to the Runtime internal state:

```
# Before (L67-72):
  # State
  stories: Map<string, CompiledStory>
  contexts: List<Context>
  channelHubs: ChannelHubRegistry
  storage: PersistentStorage
  signals: SignalManager

# After:
  # State
  stories: Map<string, CompiledStory>
  contexts: List<Context>               # internal
  channelHubs: ChannelHubRegistry
  storage: PersistentStorage
  signals: SignalManager
  elapsedMs: double                      # internal — accumulated from tick(deltaTimeMs) calls
```

**Rationale:** `startContext` replaces the create-then-run pattern. `tick(deltaTimeMs)` is the sole execution driver — the runtime owns the clock, advancing `elapsedMs` on each tick. Sleep/timeout conditions compare against `elapsedMs`, not a system clock. `run(context)` and `runToCompletion` are removed from the spec — the runtime advances all contexts through `tick`.

---

### Step 2: Add New Types Section (after Runtime Interface, before Handler Types)

**Objective:** Define `ContextHandle`, `ExecutionResult`, and `VariableAccessor`.

**File:** `impl/09_runtime.md`

**Changes:** Insert a new section between the Runtime Interface code block (L91) and the `---` before Handler Types (L93):

```markdown
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
```

**Rationale:** These types formalize the API boundary. `ContextHandle` provides minimal necessary exposure (ID + read-only state). `VariableAccessor` is lazy per the eval's resolved question.

---

### Step 3: Mark Context Structure as Internal (L347–390)

**Objective:** Relabel the Context Structure section to make clear it is an implementation detail.

**File:** `impl/09_runtime.md`

**Changes:**

```
# Before (L347):
## Context Structure

# After:
## Context Structure (Internal)

> **Implementation detail.** The `Context` type is internal to the runtime. Callers interact via `ContextHandle` (see Runtime Interface). This section documents the internal execution state for implementers.
```

Remove the `runtime: Runtime` field from line 352 — replace with a note that the context accesses runtime services directly (implementation-specific):

```
# Before (L351-352):
Context:
  id: string                      # Unique context ID
  runtime: Runtime                # Parent runtime

# After:
Context:
  id: string                      # Unique context ID
  # Access to runtime services (verb registry, story cache, channel hubs,
  # signal manager) is implementation-specific — back-reference, delegates,
  # or dependency injection. Not part of the public contract.
```

**Rationale:** The context's runtime linkage is a coupling mechanism, not a public field. Implementations choose their own pattern.

---

### Step 4: Update Host Resume Path (L636–656)

**Objective:** Host examples use `runtime.resume(handle, value)` instead of `context.resume()`.

**File:** `impl/09_runtime.md`

**Changes:**

```
# Before (L638):
For `WAITING_HOST`, fulfillment is the host application's responsibility — the scheduler only checks timeout. The driver calls the host handler *before* yielding (to trigger UI); the host feeds the response back via `context.resume()`.

# After:
For `WAITING_HOST`, fulfillment is the host application's responsibility — the scheduler only checks timeout. The verb driver calls the host handler *before* returning `Suspend`, passing a `ContextHandle`. The host feeds the response back via `runtime.resume(handle, value)`.
```

```
# Before (L640-656):
# Host-side example (game engine / UI framework):
# The host receives the resumeToken alongside the request.
# It must pass it back when resuming — stale tokens are rejected.

onChoose(context, request):
    token = context.resumeToken
    displayChoiceUI(request, onSelected: (value) ->
        context.resume(Completed { value }, token)
    )

onConverse(context, request):
    token = context.resumeToken
    displayDialog(request, onDismissed: () ->
        context.resume(Completed { Nothing }, token)
    )

# After:
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

**Rationale:** The host no longer holds a `Context` — only a handle. Resume tokens are an internal concern (the runtime matches them); the host doesn't need to manage them.

---

### Step 5: Update Execution Model Compatibility (L660–702)

**Objective:** Reflect that both tick and async models use `runtime.resume()` and handles, not direct context access.

**File:** `impl/09_runtime.md`

**Changes to Cooperative Tick Model (L664-676):**

Update the tick loop example and prose to use `tick(deltaTimeMs)` and handles:

```
# Before (L666):
The host calls `runtime.tick()` periodically. On each tick, the runtime checks each blocked context via `resolveWait()` and calls `context.resume(outcome, token)` for those whose condition is met.

# After:
The host calls `runtime.tick(deltaTimeMs)` each frame. The runtime accumulates `elapsedMs += deltaTimeMs`, then checks each blocked context via `resolveWait()` and resumes those whose condition is met. Host interactions (`WAITING_HOST`) are resumed via `runtime.resume(handle, value)`. The runtime never reads a system clock — all time is host-supplied.
```

```
# Before (L668-674):
# Host loop (game engine example):
while running:
    runtime.tick()
    renderFrame()
    sleepUntilNextFrame()

# After:
# Host loop (game engine example):
while running:
    dt = timeSinceLastFrame()
    runtime.tick(dt)
    renderFrame()
    sleepUntilNextFrame()
```

**Changes to Async Task Model (L678-700):**

```
# Before (L683-689):
async runContextAsync(context: Context):
    while context.state != TERMINATED:
        context.run()
        if context.pendingContinuation != null:
            token = context.resumeToken
            outcome = await fulfillAsync(context.pendingContinuation.request)
            context.resume(outcome, token)

# After:
# Internal to the runtime — not a public API.
async runContextAsync(context: Context):
    while context.state != TERMINATED:
        context.run()
        if context.pendingContinuation != null:
            token = context.resumeToken
            outcome = await fulfillAsync(context.pendingContinuation.request)
            context.resume(outcome, token)
```

Add a comment above the async model clarifying this is internal runtime implementation, not host-facing code.

---

## Verification Plan

### Manual Verification

- [ ] Read final `impl/09_runtime.md` end-to-end and verify no remaining public references to `Context` type in the Runtime Interface or host-facing sections
- [ ] Confirm all host-facing examples use `ContextHandle` and `runtime.resume()`, not `context.resume()`
- [ ] Confirm `Context Structure` section is marked internal
- [ ] Grep for `createContext`, `run(context`, `runToCompletion`, and direct `context.resume` in the spec — should only appear in internal sections or implementation notes
- [ ] Confirm `ExecutionResult`, `ContextHandle`, `VariableAccessor` types are fully defined
- [ ] Confirm `maxStatementsPerTick` appears in `RuntimeConfig`
- [ ] Confirm `elapsedMs` appears in Runtime state
- [ ] Confirm `tick(deltaTimeMs: double)` is the signature, not `tick()`
- [ ] Confirm no `now()` calls remain in resolveWait — all comparisons use `elapsedMs`

### Acceptance Criteria Validation

| Criterion | How to Verify | Expected Result |
|-----------|---------------|-----------------|
| `ContextHandle` replaces `Context` in public ops | Check Runtime Interface section | No `Context` in Operations block |
| `startContext` replaces `createContext` | Check Runtime Interface | `startContext(story): ContextHandle` |
| `run(context)` and `runToCompletion` removed from spec | Check Runtime Interface | Not present; implementation note only |
| `tick(deltaTimeMs: double)` is the execution driver | Check Runtime Interface | Present under Operations |
| `resume(handle, value)` exists | Check Runtime Interface | Present under Operations |
| `ExecutionResult` defined | Check new Public Types section | Type block with `value`, `diagnostics`, `variables` fields |
| `maxStatementsPerTick` in config | Check RuntimeConfig | Field present |
| `elapsedMs` in runtime state | Check Runtime state block | Field present, marked internal |
| No `now()` in resolveWait | Check Tick-Loop Scheduler | Uses `elapsedMs` comparisons |
| Context section marked internal | Check section header | `## Context Structure (Internal)` |
| Host examples use handles | Check Host Resume Path | `runtime.resume(handle, value)` in examples |

---

## Rollback Plan

Single file changed — `git checkout` the previous version of `impl/09_runtime.md`.

---

## Notes

### Assumptions

- The Tick-Loop Scheduler pseudo-code (L567-633) uses internal `Context` access — this is correct and stays, since it documents runtime internals
- `resolveWait()` remains unchanged — it operates on internal context state
- `Context.run()`, `Context.applyResult()`, `Context.blockOnRequest()`, `Context.resume()` pseudo-code remains — relabeled as internal implementation

### Risks

- **Downstream impact:** C# implementation will need a separate plan to align with the revised spec. This plan does not touch C# code.
- **Incomplete grep:** Some references to `Context` in the spec may be in internal sections and should stay — the edit must be surgical, not a blanket rename.
