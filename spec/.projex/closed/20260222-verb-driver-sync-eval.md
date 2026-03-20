# Verb Driver Synchronous Design Evaluation

> **Created:** 2026-02-22
> **Author:** Antigravity
> **Subject:** C# design of `IVerbDriver.Execute` being synchronous vs. asynchronous
> **Type:** Gap Analysis / Compatibility Evaluation
> **Related Projex:** None

---

## Executive Summary

The current C# `IVerbDriver.Execute` signature is synchronous (`VerbResult Execute(...)`) — and this is **not a deviation from the spec**: the impl spec (`09_runtime.md`) explicitly defines a cooperative tick-based model where `execute()` is synchronous and blocking is simulated via cooperative state mutation. The C# implementation faithfully follows this model. The problem is that **the spec's tick-based model carries an implicit host assumption** — that the host provides a periodic `tick()` call (as in a game engine). That assumption was never stated, never validated against real C# host environments, and is fundamentally incompatible with the modern .NET async ecosystem. The synchronous design is correct per spec, but the spec model itself is the wrong model for C#. The runtime must evolve to `ValueTask<VerbResult>` — not because the spec implies async (it does not), but because the spec's tick-model assumption is wrong for the target deployment environments.

---

## Evaluation Scope

### Subject
The C# implementation of `IVerbDriver` and `ZohRuntime`, specifically:
- Whether the synchronous `Execute` signature is a faithful implementation of the spec or a deviation
- Whether the impl spec implies asynchronous execution
- What the actual root cause of the design friction is
- What the right long-term execution model is for C#

### Questions Addressed
1. **Does the impl spec assume async?** Or is the synchronous C# design spec-faithful?
2. **What is the implicit host assumption in the spec's tick-based model**, and is it valid for C#?
3. **Is the synchronous driver design itself broken**, or is it the scheduler model that is missing?
4. **How can the C# runtime retain performance while natively supporting async hosts?**

### Out of Scope
- Changes to the ZOH language spec itself (`spec/`)
- Non-C# implementations

---

## Context Analysis

### The Spec's Explicit Execution Model

The impl spec (`09_runtime.md`) is **unambiguously synchronous and tick-based**. Key evidence:

**Runtime interface (09_runtime.md:77):**
```
run(context: Context): void            # synchronous
runToCompletion(context: Context): Value  # synchronous
```

**VerbDriver interface (09_runtime.md:183–192):**
```
VerbDriver:
  execute(call: CompiledVerbCall, context: Context): ExecutionResult  # synchronous
```

**Blocking pattern (09_runtime.md:456–466):**
```
SleepDriver.execute(call, context):
    context.state = SLEEPING         # mutate state
    context.waitCondition = SleepCondition { wakeTime: now() + ... }
    return ok()                      # return synchronously
```

**Scheduler tick (09_runtime.md:469–543):**
```
Runtime.tick():
    for context in contexts:
        match context.state:
            SLEEPING: if now() >= wakeTime → context.state = RUNNING
            WAITING_CHANNEL: ...
        if context.state == RUNNING:
            context.run()
```

This is a **cooperative, tick-driven state machine** — the canonical pattern for game-engine scripting. The spec explicitly models async *operations* (sleep, channel pull) as synchronous driver calls that set cooperative blocking state, relying on an external scheduler tick to resume them.

### The C# Implementation Is Spec-Faithful

`IVerbDriver.Execute` returning `VerbResult` and `ZohRuntime.Run()` being `void` are direct, correct translations of the spec. The synchronous design is **not** a mistake relative to the spec.

What IS incomplete in the C# implementation: there is no `Tick()` method on `ZohRuntime`. The current `Run()` drives a single context to completion (or until it blocks) but there is no external loop to resume sleeping/waiting contexts or to advance concurrent forked contexts. This is a separate gap — the multi-context scheduler is missing — but it is orthogonal to the sync/async question.

### The Implicit Host Assumption

The spec's tick model only works correctly if the host calls `tick()` repeatedly on a timer, exactly like a game engine `Update()` loop. This assumption is never stated in the spec. It is the hidden architectural decision that determines whether the sync model is viable at all.

**Where this assumption holds:**
- Unity, Godot, custom game engines
- Desktop applications with a render/game loop
- Any host with a periodic polling loop

**Where this assumption breaks down:**
- ASP.NET / web servers (event-driven, thread-pool based)
- Discord bots, Telegram bots (event callbacks)
- Azure Functions / cloud functions (single-shot execution)
- MAUI / WinUI applications (async UI events)

In these environments, there is no `Update()` loop to call `tick()`. Contexts sleeping or waiting for channels simply never resume, unless the host manually wires up a background timer to call `tick()` — which is non-obvious, error-prone, and still synchronously blocks a thread while running.

---

## Analysis

### Finding 1: The "Spec Implies Async" Framing Is Inaccurate

The spec does **not** imply async. It explicitly and deliberately defines cooperative synchronous blocking. The framing "the spec assumes async without saying so" is incorrect; the spec assumes a **tick loop** without saying so. These are different problems with different solutions.

Correctly naming this distinction matters because:
- "Spec implies async" suggests the fix is only in C# (add `async/await`)
- "Spec assumes tick host" means the spec model itself may need an annotation or an alternative model for non-game hosts

**Implication:** The spec should be annotated with its host assumption, or a `Task`-based execution model should be added as a standard alternative.

### Finding 2: The Tick-Loop Model Has an Incomplete C# Implementation Regardless

Even ignoring async, the current C# implementation is incomplete for the spec's cooperative model. `ZohRuntime` has no `Tick()` method. Without it:
- `/sleep` contexts never wake up
- Forked contexts (`/fork`) run but there is no way to advance all contexts concurrently
- `/call` blocks the parent context but there is no scheduler to also advance the child

This means even the synchronous tick model is not fully realized. Any move to async would need to address this at the same time.

### Finding 3: The Tick-Loop Is a Leaky Abstraction in C#

The cooperative blocking pattern — where drivers reach into the engine's context and mutate `ctx.SetState(ContextState.Sleeping)` while returning `VerbResult.Ok()` — is a leaky abstraction regardless of the async question:

- A driver returning `Ok()` when the operation is semantically incomplete (still "sleeping") violates the principle that a result should describe the outcome of the call
- The engine must infer the actual state from `ctx.State` rather than from the return value
- Any new driver author must know to mutate context state AND return Ok simultaneously — this is an invisible contract, not an enforced interface

A proper design would have the return value *be* the full outcome, including "I am yielding for X milliseconds."

### Finding 4: The C# Ecosystem Is Async-First, Making Tick-Loop Integration Painful

Under the current synchronous `IVerbDriver.Execute`, a developer implementing `/prompt` (ask a user a question) in an ASP.NET or Discord host must either:

1. **Block the thread:** `task.GetAwaiter().GetResult()`. This causes ThreadPool starvation and breaks ASP.NET and UI threads.
2. **Reinvent continuations:** Manually set `ctx.SetState(WaitingExternal)`, attach a callback to the async task that re-calls `ZohRuntime.Run(ctx)` from a ThreadPool thread — requiring manual threading/locking and defeating the purpose of C#'s async model.

Both options are a terrible developer experience.

### Finding 5: `ValueTask<VerbResult>` Solves Both Problems Without Regression

`ValueTask<T>` is the C# type designed for exactly this situation — methods that are *usually* synchronous but *sometimes* need to `await` I/O. Key properties:
- **Zero allocation on the fast synchronous path** — `new ValueTask<VerbResult>(result)` wraps an existing result with no heap allocation
- **Native async support** — when a driver returns an incomplete `ValueTask`, the `await` in the runtime loop suspends naturally without blocking a thread
- **Eliminates the tick-loop need for I/O verbs** — `SleepDriver` becomes `await Task.Delay(ms)` and the .NET scheduler handles resumption
- **Eliminates cooperative state mutation** — drivers don't need to reach into `ctx.SetState()` for their own blocking logic

This doesn't eliminate the scheduler for multi-context parallelism (`/fork`), but it eliminates the need for the scheduler to handle I/O blocking.

---

## Comparative Analysis

| Aspect | Current Sync (Tick-Loop) | `async Task<VerbResult>` | `async ValueTask<VerbResult>` |
|--------|-------------------------|--------------------------|-------------------------------|
| **Sync Performance** | Excellent (0 alloc) | Poor (Task per call) | **Excellent** (0 alloc sync) |
| **I/O Driver Integration** | Very High Friction | Trivial (`await`) | **Trivial** (`await`) |
| **Host Requirements** | Requires external tick loop | Native .NET scheduler | Native .NET scheduler |
| **Threading Model** | Manual locking/resumes | Handled by .NET | Handled by .NET |
| **Spec Fidelity** | Faithful | Departure | Departure (but warranted) |
| **SleepDriver complexity** | ctx.SetState + scheduler tick | `await Task.Delay` | `await Task.Delay` |
| **Multi-context parallelism** | Still needs scheduler | Still needs scheduler | Still needs scheduler |

---

## Evaluation Against Criteria

| Criterion | Current Sync | `ValueTask` | Rationale |
|-----------|-------------|-------------|-----------|
| Spec fidelity | Strong | Weak | Spec is tick-based; ValueTask departs from spec |
| C# ecosystem fit | Weak | Strong | Async-first .NET ecosystem |
| Developer experience (host) | Weak | Strong | Native `await` vs. callback soup |
| Performance | Strong | Strong | ValueTask = zero alloc on fast path |
| Multi-context concurrency | Partial | Partial | Scheduler still needed for `/fork` either way |
| Implementation completeness | Incomplete | Incomplete | Tick scheduler not yet implemented |

**Overall Assessment:** The synchronous design is spec-faithful but the spec model carries an unsound implicit assumption for the C# target. Migration to `ValueTask<VerbResult>` is warranted and practically necessary for real-world host integrations.

---

## Challenges and Risks

### Identified Challenges
1. **Multi-context scheduling remains unsolved:** Migrating `IVerbDriver` to `async ValueTask` eliminates I/O blocking friction but does not solve multi-context scheduling for `/fork`/`/call`. These verbs still require a runtime-level scheduler or a `Task`-per-context model.
2. **Migration scope:** Every existing driver (`SleepDriver`, `CallDriver`, `SetDriver`, `PushDriver`, `PullDriver`, etc.) must be updated. While mostly mechanical, it is broad.
3. **Spec divergence:** The impl spec will be out of sync with the C# implementation. The spec should be updated with a C#-specific annotation or an async execution model addendum.

### Risk Assessment
| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| Breaking changes to existing driver authors | High | Medium | Provide `IVerbDriver` → `IAsyncVerbDriver` shim |
| Concurrency bugs in Task-per-context model | Medium | High | Use `Task.Run` carefully; prefer sequential context scheduling |
| Spec divergence creates confusion | High | Low | Document async model in impl notes |

---

## Findings

### Key Findings
1. **The spec is explicitly synchronous tick-based, not implicitly async.** The C# impl being synchronous is correct per spec. The misconception that the spec "implies async" conflates the spec's tick model (synchronous cooperative) with the C# target's natural model (async/await).
2. **The spec carries an implicit host assumption — a periodic `tick()` caller — that is not valid for modern .NET hosts.** This is the actual root cause of the friction, not a missing `async` keyword.
3. **The C# implementation is incomplete even for the tick model.** `ZohRuntime` has no `Tick()` method, meaning blocked contexts (sleep, channel wait) never resume and concurrent forked contexts cannot advance.
4. **The cooperative state-mutation pattern is a leaky abstraction.** Drivers returning `Ok()` while encoding blocking state in `ctx.SetState()` violates clear result semantics regardless of sync/async.
5. **`ValueTask<VerbResult>` is the correct C# evolution.** It resolves the I/O integration friction with zero performance regression on the fast path, and removes the need for the tick scheduler for all I/O-bound blocking verbs.

### Gaps Identified
- No `Tick()` / multi-context scheduler in C# impl
- Spec does not document its implicit host assumption
- No `async` variant of `IVerbDriver` defined
- `ZohRuntime.Run()` is not awaitable from an async host

### Opportunities
- Migrating to `async ValueTask` may allow significant simplification of `SleepDriver`, `WaitDriver`, `PushDriver`, `PullDriver` — all of which currently require cooperative state mutation + external scheduler
- A `Task`-per-context model for `/fork` could replace the scheduler entirely in async hosts

---

## Recommendations

### Primary Recommendation
**Migrate `ZohRuntime` and `IVerbDriver` to an asynchronous design using `ValueTask`.**

The departure from the impl spec is justified: the spec's tick-loop model is grounded in a game-engine host assumption that doesn't hold for the C# target environments. Async-native design is the correct architectural direction for C#.

*Proposed Signatures:*
```csharp
public interface IVerbDriver
{
    string Namespace { get; }
    string Name { get; }
    int Priority => 0;

    ValueTask<VerbResult> ExecuteAsync(IExecutionContext context, VerbCallAst verbCall);
}

// In ZohRuntime:
public async Task RunAsync(Context ctx, CompiledStory story)
```

### Conditional Recommendations
- **If full async migration is deferred:** At minimum, add a `Tick()` method to `ZohRuntime` and a background timer-based scheduler, so the current cooperative model actually functions. Currently blocked contexts never resume.
- **If maintaining sync API for compatibility:** Introduce a parallel `IAsyncVerbDriver` interface; allow the registry to prefer async drivers and fall back to sync via `ValueTask.FromResult`.

### Suggested Next Steps
1. **Plan Projex:** Create a plan to migrate `IVerbDriver` and all built-in drivers to `ValueTask<VerbResult>`.
2. **Spec annotation:** Add a note to `impl/09_runtime.md` documenting the tick-loop model's implicit host assumption, and flag that the C# reference implementation will use async-native execution instead.
3. **Evaluate** whether `/fork` and `/call` concurrency should use `Task`-per-context or an explicit async scheduler, as this decision affects the overall threading model.

---

## Open Questions

- [ ] Should `/fork` create a `Task` per context (true .NET parallelism) or use a cooperative async scheduler (single-task, `await`-interleaved)?
- [ ] Should the spec be updated to define an async execution model as an implementation option, or remain tick-based and leave async as a C#-specific departure?
- [ ] Can `context.SetState()` side effects be removed entirely once all blocking verbs are async (returning incomplete `ValueTask`)?

---

## Appendix

### Methodology
Code review of `IVerbDriver.cs`, `ZohRuntime.cs`, `IExecutionContext.cs`, `VerbResult.cs`, cross-referenced against `impl/09_runtime.md` and `impl/08_concurrency.md`.

### Sources
- `csharp/src/Zoh.Runtime/Verbs/IVerbDriver.cs` — current synchronous interface
- `csharp/src/Zoh.Runtime/Execution/ZohRuntime.cs` — synchronous `Run()` and `ExecuteVerb()`
- `csharp/src/Zoh.Runtime/Execution/IExecutionContext.cs` — `SetState()` as driver side-effect mechanism
- `impl/09_runtime.md` — spec's tick-loop model, `Runtime.tick()`, `VerbDriver.execute()` signatures
- `impl/08_concurrency.md` — spec's cooperative blocking pattern for Sleep, Push, Pull, Wait
