# Per-Statement Temp State and Driver Stage Numbers

> **Created:** 2026-03-04
> **Author:** agent
> **Subject:** Per-statement temp state on Context — verb drivers save stage number for flat suspend/resume
> **Type:** Compatibility Evaluation
> **Related Projex:** 20260304-std-verbs-driver-alignment-plan.md

---

## Executive Summary

A per-statement temporary state slot on `Context` would allow verb drivers to persist data (including a stage number) across suspend/resume cycles for the **same statement**. The runtime clears it when the instruction pointer advances. On resume, instead of relying on closure-captured state, the runtime re-invokes the driver's execute method — the driver reads its stage number from the temp state and dispatches accordingly. This replaces recursive closure chains with flat, switch-based driver code, making driver state inspectable, serializable, and structurally transparent to the host.

The mechanism is simple, backward-compatible (drivers that don't use it keep working via closures), and fills a real gap: closures are opaque, unserializable, and force recursive patterns for multi-step verbs.

**Verdict:** Sound and worthwhile. The temp state is a thin addition to `Context`; the stage pattern is opt-in for drivers. Should be spec'd and adopted for `/converse` (and any future multi-step verb). Closures remain available as fallback for one-shot suspensions.

---

## Evaluation Scope

### Questions Addressed

1. How does a per-statement temp state fit into the existing `Context` / `DriverResult` / `Continuation` architecture?
2. What does the re-invocation flow look like concretely?
3. What changes to the runtime spec and C# implementation are implied?
4. What are the trade-offs vs the current closure model?

### Out of Scope

- Specific C# implementation plan (that's a separate plan-projex)
- Media verbs (implementation-defined, single-step)
- Changes to `WaitRequest` or `WaitOutcome`

---

## The Mechanism

### Context Addition

```
Context:
  # ... existing fields ...
  
  # Per-statement temp state — driver-private scratch space.
  # Persists across suspend/resume cycles for the SAME statement.
  # Cleared when instructionPointer advances (i.e., on Complete or jump).
  statementState: Map<string, Value>?
```

Drivers read and write this freely. The runtime never interprets it — it only clears it on IP advance.

### Stage Number Convention

By convention, drivers store a stage number under a well-known key (e.g., `"_stage"` or just `"stage"`). The runtime doesn't enforce this — it's a driver pattern, not a runtime concept. A driver with N stages writes `0`, `1`, ..., `N-1`.

### Resume Flow: Re-Invocation

When the runtime resumes a suspended context and the continuation signals "re-invoke driver" (rather than providing an `onFulfilled` callback), the execution loop calls `driver.execute(call, context)` again. The driver reads its stage from `context.statementState` and dispatches:

```
ConverseDriver.execute(call, context):
    stage = context.statementState?.get("stage") ?? 0

    match stage:
        0:  # Initial: resolve and cache all content
            ... resolve contents, cache in statementState ...
            context.statementState["stage"] = 1
            context.statementState["index"] = 0
            context.statementState["contents"] = contents
            return Suspend { request: Host { timeoutMs } }

        1:  # Per-content loop
            contents = context.statementState["contents"]
            index = context.statementState["index"]
            if index >= contents.length:
                return Complete { Nothing, [] }
            context.statementState["index"] = index + 1
            return Suspend { request: Host { timeoutMs } }
```

### Continuation Variant

The `Continuation` type gains an alternative that signals re-invocation instead of a callback:

```
Continuation:
  request: WaitRequest
  onFulfilled: (WaitOutcome) -> DriverResult    # Existing — callback-based
  
# OR: replace with a discriminated union:
Continuation:
  | Callback { request: WaitRequest, onFulfilled: (WaitOutcome) -> DriverResult }
  | Reinvoke { request: WaitRequest }
```

When the runtime sees `Reinvoke`, it stores the `WaitRequest`, blocks, and on resume:
1. Sets `state = RUNNING`
2. Does **not** advance IP
3. Re-enters the execution loop, which calls `driver.execute(call, context)` again
4. The driver reads `statementState` and picks up where it left off

When the runtime sees `Callback`, behavior is unchanged — `onFulfilled` is called as today.

### What Happens on Resume with WaitOutcome?

The `Reinvoke` variant doesn't have an `onFulfilled` to receive the `WaitOutcome`. Two options:

**Option A — Outcome in statementState:** The runtime writes the `WaitOutcome` into `context.statementState["_outcome"]` before re-invoking. The driver reads it.

**Option B — Outcome parameter on execute:** `execute(call, context, outcome?)` gains an optional outcome parameter. Null on first call, populated on resume. This changes the driver interface.

**Option A** is cleaner — it keeps the `execute` signature stable and uses the same statementState bag for everything.

---

## Concrete Example: `/converse` With Staging

```
ConverseDriver.execute(call, context):
    stage = context.statementState?.get("stage") ?? 0

    if stage == 0:
        # First call — resolve everything, fail fast
        speaker = getAttribute(call, "By")?.value
        style = getAttribute(call, "Style")?.value ?? "dialog"
        
        waitAttr = getAttribute(call, "Wait")
        shouldWait = waitAttr != null ? resolve(waitAttr, context).toBool()
                                     : context.getFlag("interactive") ?? true
        
        timeout = getNamedParam(call, "timeout")
        timeoutMs = timeout != null ? resolve(timeout, context).toDouble() * 1000 : null
        if timeout != null and timeoutMs <= 0:
            return info("timeout", "Immediate timeout")

        contents = []
        for param in call.unnamedParams:
            content = resolve(param, context)
            if content is not StringValue and content is not ExpressionValue:
                return fatal("type_mismatch", ...)
            if content is ExpressionValue: content = evaluate(content, context)
            if content is StringValue: content = interpolate(content.value, context)
            contents.add(content)

        if not shouldWait or contents.isEmpty():
            return Complete { Nothing, [] }

        # Cache resolved state, advance to stage 1
        context.statementState = {
            "stage": 1, "index": 0,
            "contents": contents, "timeoutMs": timeoutMs
        }
        return Suspend { continuation: Reinvoke { Host { timeoutMs } } }

    elif stage == 1:
        # Per-content loop — check previous outcome, then advance
        outcome = context.statementState["_outcome"]
        if outcome is TimedOut:
            return Complete { Nothing, [Diagnostic(INFO, "timeout", "Converse timed out")] }
        if outcome is Cancelled { code, msg }:
            return Complete { Nothing, [Diagnostic(ERROR, code, msg)] }

        contents = context.statementState["contents"]
        index = context.statementState["index"]
        timeoutMs = context.statementState["timeoutMs"]

        if index >= contents.length:
            return Complete { Nothing, [] }

        context.statementState["index"] = index + 1
        return Suspend { continuation: Reinvoke { Host { timeoutMs } } }
```

Compare with the closure model:

| Aspect | Closure (`converseNext`) | Staging |
|--------|--------------------------|---------|
| Control flow | Recursive closures | Flat switch on stage |
| State storage | Lambda captures | `statementState` map |
| Host visibility | Opaque | `statementState` is readable |
| Serializable | No | Yes (plain data) |
| Lines of code | ~20 | ~30 (slightly more explicit) |
| New runtime concept | None | `statementState` field, `Reinvoke` variant |

---

## Integration With Existing Architecture

### What Changes

| Component | Change |
|-----------|--------|
| `Context` (spec + C#) | Add `statementState: Map<string, Value>?` field |
| `Context.applyResult` | Clear `statementState` when IP advances (on `Complete`) |
| `Continuation` (spec + C#) | Add `Reinvoke` variant (or: make `onFulfilled` nullable, treat null as "re-invoke") |
| `Context.resume()` | If continuation is `Reinvoke`: write outcome to `statementState["_outcome"]`, set state to RUNNING, don't call a handler — let the run loop re-invoke the driver |
| Verb drivers | Opt-in: `/converse` uses staging; single-step verbs unchanged |

### What Doesn't Change

- `WaitRequest` / `WaitOutcome` — untouched
- `DriverResult` — `Complete` and `Suspend` stay the same
- `execute(call, context)` signature — unchanged
- Single-step verbs (`/choose`, `/prompt`, `/sleep`, etc.) — use `Callback` as today
- IP management, jump guard — unchanged

### Backward Compatibility

Drivers that return `Suspend { Callback { request, onFulfilled } }` work exactly as before. `Reinvoke` is a new opt-in path. No existing code breaks.

---

## Evaluation Against Criteria

| Criterion | Assessment |
|-----------|------------|
| **Fits existing architecture** | Yes — additive. One field on Context, one variant on Continuation. |
| **Driver simplicity** | Better for multi-step verbs (flat switch vs recursive closures). Neutral for single-step. |
| **Host transparency** | Significantly better — host can read `statementState` to see driver progress. |
| **Serializability** | Possible — `statementState` is plain data. Closures are not. |
| **Runtime complexity** | Minimal — clear statementState on IP advance, write outcome on reinvoke resume. |
| **Backward compatible** | Fully — closures still work via `Callback` variant. |

---

## Challenges

1. **Type safety of `statementState`:** It's a `Map<string, Value>`, so drivers must agree on keys. Typos in key names are runtime errors, not compile errors. Mitigated by well-named constants or typed wrappers per driver.

2. **Two resume paths:** `Callback` vs `Reinvoke` means the runtime's `resume()` / execution loop has two code paths. Acceptable complexity — it's a clean branch, not entangled logic.

3. **`statementState` lifetime:** Must be cleared on IP advance, AND on jump (since a jump skips the natural IP++ path). The existing jump guard in `applyResult` needs to also clear `statementState`.

---

## Recommendations

1. **Adopt this mechanism.** Add `statementState` to Context in the spec and C# implementation.
2. **Use `Reinvoke` for `/converse`** (the only current multi-step verb). Keep `Callback` for single-step verbs.
3. **Make `onFulfilled` nullable** as the simplest implementation path — if null, the runtime treats it as "reinvoke driver." This avoids adding a new Continuation variant and keeps the type simpler.
4. **Spec the clearing semantics:** `statementState` is cleared whenever:
   - `applyResult` receives `Complete` and advances IP
   - A jump changes IP or story
   - Context terminates

### Suggested Next Steps

1. Amend `09_runtime.md` to add `statementState` to `Context` and document the `Reinvoke` path
2. Update the alignment plan (`20260304-std-verbs-driver-alignment-plan.md`) Step 1 to use staging instead of `converseNext` closures
3. Implement in C# as a follow-up projex

---

## Open Questions

- [x] ~~Should `statementState` use `Value` (Zoh type system) or `object` (host-native)?~~ **Resolved:** Host-native. `statementState` is a host entity — drivers store arbitrary host objects, not Zoh `Value`s. This gives drivers full flexibility and avoids unnecessary boxing into the Zoh type system.
- [x] ~~On reinvoke, how does the runtime deliver the `WaitOutcome` to the driver?~~ **Resolved:** N/A. Outcome delivered via normal `onFulfilled(outcome)` callback. The `Reinvoke` variant, nullable `onFulfilled`, `reinvoke` boolean, and `reinvoke()` method were all explored and rejected. `Continuation` stays unchanged. Staging is a driver-level convention using existing closures + `statementState`.

---

## Appendix

### Sources

- `impl/09_runtime.md` — `Context`, `DriverResult`, `Continuation`, execution loop, `resume()`
- `impl/10_std_verbs.md` — Current verb driver implementations
- `spec/std_verbs.md` — `/converse` multi-content semantics (L9)
- `20260304-std-verbs-driver-alignment-plan.md` — `converseNext` closure pattern (L85–152)
- `Context.cs`, `Continuation.cs`, `DriverResult.cs` — C# implementation
