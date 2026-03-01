# Continuation Resume: Instruction Pointer Advancement Gap

> **Created:** 2026-03-01
> **Author:** agent
> **Subject:** Whether the runtime spec correctly handles resumption after a blocking verb yields via continuation
> **Type:** Gap Analysis
> **Related Projex:** `20260301-two-phase-continuation-model-proposal.md` (supersedes this eval's fix recommendation)

---

## Executive Summary

The runtime spec (`impl/09_runtime.md`) has a gap: when a blocking verb yields via `Continuation`, the instruction pointer (IP) is never advanced past the yielded statement. On resume, `context.run()` re-enters at the same IP, causing the blocking verb to **re-execute**. The C# reference implementation shares this latent bug. The fix is a one-line IP advancement, but its placement has design implications.

---

## Evaluation Scope

### Subject
The interaction between `ExecutionResult.continuation`, `Context.block()`, `Runtime.tick()`, and `Context.run()` — specifically, instruction pointer management across yield/resume boundaries.

### Questions Addressed
1. Does the current spec cause re-execution of yielded verbs?
2. Is the C# implementation affected?
3. Where should the IP advancement go?

### Evaluation Criteria
| Criterion | Weight | Description |
|-----------|--------|-------------|
| Correctness | High | Blocking verbs must not re-execute on resume |
| Consistency | High | Both tick-loop and async models must behave identically |
| Simplicity | Med | Fix should be minimal and obvious |

### Out of Scope
- Whether the continuation type system itself is well-designed
- Async task model implementation details beyond IP handling

---

## Analysis

### The Execution Loop Gap

**`Context.run()`** (spec lines 374–416):

```
while state == RUNNING:
    statement = currentStory.statements[instructionPointer]
    ...
    result = driver.execute(call, this)

    if result.continuation != null:
        block(result.continuation)
        return  ← exits WITHOUT advancing IP

    instructionPointer++  ← only reached for non-blocking verbs
```

When a driver returns a continuation, `block()` sets the context state to `SLEEPING`/`WAITING_*` and the method returns. The `instructionPointer++` on the last line is **never reached**.

**`Runtime.tick()`** (spec lines 519–594):

```
for context in contexts:
    match context.state:
        SLEEPING:
            if now() >= context.waitCondition.wakeTime:
                context.state = RUNNING
                context.waitCondition = null
        ...
    if context.state == RUNNING:
        context.run()  ← re-enters at unchanged IP
```

After unblocking, tick calls `context.run()`. Since IP still points at the blocking verb, that verb's driver executes **again** — producing another continuation, blocking again, in an infinite yield loop.

**Evidence:** There are exactly 6 places in the spec where `instructionPointer` is modified. None are in the resume/unblock path:

| Location | File | Purpose |
|----------|------|---------|
| L384 | `impl/09_runtime.md` | Skip checkpoint markers |
| L398 | `impl/09_runtime.md` | Skip unknown verbs |
| L416 | `impl/09_runtime.md` | Advance after non-blocking execution |
| L134 | `impl/08_concurrency.md` | `/jump` sets IP to label index |
| L202 | `impl/08_concurrency.md` | `/fork` sets new context IP |
| L267 | `impl/08_concurrency.md` | `/call` sets new context IP |

### C# Implementation Status

The C# `Context.Run()` (`csharp/src/Zoh.Runtime/Execution/Context.cs:37-97`) has the same structure:

```csharp
if (result.Continuation != null)
{
    Block(result.Continuation);
    break;  // exits loop, IP not advanced
}
...
// IP advancement guard — only fires if State == Running
if (State == ContextState.Running && InstructionPointer == entryIp && ...)
    InstructionPointer++;
```

After `Block()` changes state away from `Running`, the IP guard at L91-95 does not fire. The `Resume()` method (L173-180) restores `Running` state but does not touch IP.

**Saving grace:** The C# runtime has no `Tick()` loop yet — only `RunToCompletion`. The resume-after-block path is **not exercised**, so this is a latent bug, not a live one.

---

## Fix Options

### Option A: Advance IP in `block()`

```
Context.block(continuation: Continuation):
    instructionPointer++           ← advance past blocking verb
    match continuation:
        Sleep { durationMs }:
            state = SLEEPING
            ...
```

**Pro:** Centralizes the fix. Every blocking path goes through `block()`.
**Con:** Mixes IP management (an execution concern) into state management (a blocking concern).

### Option B: Advance IP before calling `block()` in the execution loop

```
if result.continuation != null:
    instructionPointer++
    block(result.continuation)
    return
```

**Pro:** Keeps IP advancement next to the existing `instructionPointer++` for non-blocking verbs. The execution loop owns all IP logic.
**Con:** If `block()` is ever called from elsewhere, the caller must remember to advance IP.

### Option C: Advance IP in the tick loop on unblock

```
if context.state == RUNNING:
    if context.justUnblocked:      ← requires new flag
        context.instructionPointer++
    context.run()
```

**Pro:** Defers advancement to resume time.
**Con:** Requires a new flag or sentinel. Fragile — tick loop must track whether each transition was from a blocked state.

### Recommendation: Option B

Option B is the cleanest. It mirrors the existing non-blocking path structure:

```
# Non-blocking: execute, then advance
result = driver.execute(call, this)
...
instructionPointer++

# Blocking: execute, advance, then yield
result = driver.execute(call, this)
...
instructionPointer++
block(result.continuation)
return
```

Both paths advance IP after successful execution. The only difference is whether the method continues or yields. This is conceptually correct: the verb **has been executed** (it returned a result with a value and continuation). Advancing IP acknowledges that execution is complete; the context is now waiting for an external condition, not waiting to re-run the verb.

---

## Challenges and Risks

### Edge Case: Navigation After Block

If a blocking verb's driver also modifies IP (e.g., a hypothetical blocking `/jump`), advancing IP in the execution loop would double-advance. However, no current verb combines continuation with IP modification, and such a verb would be semantically incoherent (you can't both "go somewhere else" and "wait here"). **Risk: negligible.**

### Edge Case: Error on Resume

If the resumed context needs to know which verb it was blocked on (for error reporting), the IP has already moved past it. The `waitCondition` or continuation could carry the source location. **Risk: low, mitigated by existing `sourceLine` in diagnostics.**

### C# Implementation Sync

The C# fix is equivalent — add `InstructionPointer++` before `Block()` at L72 in `Context.cs`. The existing IP guard (L91-95) would still not fire for blocking verbs since `State != Running` after `Block()`, so there is no double-advance.

---

## Findings

### Key Findings
1. **The spec has a confirmed gap**: no IP advancement on the yield/resume path, causing infinite re-execution of blocking verbs
2. **The C# implementation has the same latent bug**, currently unexercised because no tick loop exists
3. **The fix is a single line** (`instructionPointer++` before `block()`) with no cascading effects
4. **Both execution models (tick-loop and async) are affected** — the async model would similarly re-enter `run()` at the stale IP

### Gaps Identified
- The spec never defines "resume" as a concept — `block()` is specified but its inverse is implicit in the tick loop's state transitions
- No spec prose explains the IP lifecycle across yield/resume boundaries

---

## Recommendations

### Primary Recommendation
Add `instructionPointer++` immediately before the `block()` call in the execution loop (Option B). Update both the spec pseudocode and the C# implementation.

### Suggested Next Steps
1. **Spec patch**: Add IP advancement to `Context.run()` in `impl/09_runtime.md` L411-414
2. **C# patch**: Add `InstructionPointer++` before `Block()` in `Context.cs` L70-73
3. **Consider**: Add a brief "Resume Semantics" prose section to the spec explaining that a blocked verb is considered "executed" and IP advancement happens before yield

---

## Open Questions

- [ ] Should the spec formally define a `Context.resume()` method rather than leaving resume as implicit tick-loop state transitions?
- [ ] Should the async model section show explicit IP handling for symmetry with the tick model?

---

## Appendix

### Methodology
- Full-text search of all `spec/` and `impl/` files for IP-related terms
- Trace of execution flow through spec pseudocode (yield path and resume path)
- Cross-reference with C# implementation (`csharp/src/Zoh.Runtime/Execution/Context.cs`)

### Sources
- `impl/09_runtime.md` — Execution loop (L371-493), Blocking operations (L493-607), Tick loop (L519-594)
- `impl/08_concurrency.md` — Navigation verbs and IP modification
- `csharp/src/Zoh.Runtime/Execution/Context.cs` — `Run()` (L37-97), `Block()` (L130-168), `Resume()` (L173-180)
