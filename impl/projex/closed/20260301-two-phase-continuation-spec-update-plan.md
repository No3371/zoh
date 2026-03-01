# Two-Phase Continuation Model — Spec Update

> **Status:** Complete
> **Created:** 2026-03-01
> **Completed:** 2026-03-01
> **Author:** agent
> **Source:** `20260301-two-phase-continuation-model-proposal.md`
> **Related Projex:** `20260301-continuation-resume-ip-gap-eval.md`, `20260301-two-phase-continuation-csharp-impl-plan.md`
> **Walkthrough:** `20260301-two-phase-continuation-spec-update-walkthrough.md`

---

## Summary

Update `impl/09_runtime.md` and `impl/08_concurrency.md` to replace the one-phase `ExecutionResult`/`Continuation` model with the two-phase `DriverResult`/`WaitRequest`/`WaitOutcome` model from the accepted proposal.

**Scope:** Spec files only — `impl/09_runtime.md` and `impl/08_concurrency.md`
**Estimated Changes:** 2 files, ~8 sections

---

## Objective

### Problem / Gap / Need
The current spec has: (1) an IP advancement gap causing infinite re-execution of blocking verbs, (2) post-resume verb logic baked into the scheduler, and (3) no mechanism for extension drivers to define blocking behavior with custom resume logic.

### Success Criteria
- [ ] `ExecutionResult` and flat `Continuation` union replaced with `DriverResult`, `Continuation` (with `onFulfilled`), `WaitRequest`, `WaitOutcome`
- [ ] `Context` structure gains `pendingContinuation`, `resumeToken`, `resume()`, `applyResult()`, `blockOnRequest()`
- [ ] `ContextState` gains `WAITING_HOST`
- [ ] Execution loop uses `applyResult()` for both blocking and non-blocking paths
- [ ] `resume()` uses token-guarded double-call protection
- [ ] Tick loop is a pure condition resolver — no verb-specific logic
- [ ] All driver pseudocode in `08_concurrency.md` returns `DriverResult` with `onFulfilled` closures
- [ ] Async model section updated symmetrically

### Out of Scope
- C# implementation changes (separate plan: `20260301-two-phase-continuation-csharp-impl-plan.md`)
- Language spec files (`spec/`)
- New verbs or features

---

## Context

### Current State
`impl/09_runtime.md` defines `ExecutionResult { value, diagnostics, continuation? }` where `Continuation` is a closed union of data-only wait types. The tick loop contains verb-specific resume logic (inline var copying, timeout diagnostics, signal unsubscription). The execution loop has no IP advancement on the blocking path.

### Key Files
| File | Purpose | Changes Needed |
|------|---------|----------------|
| `impl/09_runtime.md` | Runtime architecture spec | Replace types, execution loop, context structure, tick loop, async model |
| `impl/08_concurrency.md` | Concurrency/driver specs | Rewrite all driver pseudocode to use `DriverResult` |

### Dependencies
- **Requires:** Accepted proposal (`20260301-two-phase-continuation-model-proposal.md`)
- **Blocks:** C# implementation plan

---

## Implementation

### Overview
Replace spec sections in dependency order: types first, then context structure, then execution loop, then scheduler, then drivers.

### Step 1: Replace Type Definitions in 09_runtime.md

**Objective:** Replace `ExecutionResult`/`Continuation` with the new type system.

**File:** `impl/09_runtime.md`

**Changes:** Replace the "Verb Driver" section (L178–204). The `VerbDriver` interface stays the same except `execute` returns `DriverResult` instead of `ExecutionResult`.

```
// Before (L182–204):
VerbDriver:
  ...
  execute(call: CompiledVerbCall, context: Context): ExecutionResult

ExecutionResult:
  value: Value
  diagnostics: List<Diagnostic>
  continuation: Continuation?

Continuation:
  | Sleep       { durationMs: double }
  | ChannelPull { channelName: string, generation: int, timeoutMs: double? }
  | ChannelPush { channelName: string, seqNum: int, generation: int, timeoutMs: double? }
  | Message     { messageName: string, timeoutMs: double? }
  | Context     { contextId: string, inlineVars: List<VarRef> }

// After:
VerbDriver:
  ...
  execute(call: CompiledVerbCall, context: Context): DriverResult

DriverResult:
  | Complete {
      value: Value
      diagnostics: List<Diagnostic>
    }
  | Suspend {
      continuation: Continuation
      diagnostics: List<Diagnostic>
    }

Continuation:
  request: WaitRequest
  onFulfilled: (WaitOutcome) -> DriverResult

WaitRequest:
  | Sleep       { durationMs: double }
  | ChannelPull { channelName: string, generation: int, timeoutMs: double? }
  | ChannelPush { channelName: string, seqNum: int, generation: int, timeoutMs: double? }
  | Signal      { messageName: string, timeoutMs: double? }
  | JoinContext { contextId: string }
  | Host        { timeoutMs: double? }

WaitOutcome:
  | Completed { value: Value }
  | TimedOut
  | Cancelled { code: string, message: string }
```

**Rationale:** Types must be defined before they can be referenced in subsequent sections.

**Verification:** All six `WaitRequest` variants present. `DriverResult` is a discriminated union. `Continuation` pairs request with callback.

---

### Step 2: Update Context Structure in 09_runtime.md

**Objective:** Add new fields, add `WAITING_HOST` state.

**File:** `impl/09_runtime.md`

**Changes:** Update the "Context Structure" section (L327–367).

```
// Before (relevant fields at L335–357):
  state: ContextState
  ...
  waitCondition: WaitCondition?

ContextState:
  RUNNING
  WAITING_CHANNEL
  WAITING_CHANNEL_PUSH
  WAITING_MESSAGE
  WAITING_CONTEXT
  SLEEPING
  TERMINATED

// After — add three fields and one state:
  state: ContextState
  pendingContinuation: Continuation?   # Stored while blocked
  waitCondition: WaitCondition?
  resumeToken: int                     # Incremented on each Suspend

ContextState:
  RUNNING
  WAITING_CHANNEL
  WAITING_CHANNEL_PUSH
  WAITING_MESSAGE
  WAITING_CONTEXT
  WAITING_HOST              # NEW — blocked on host interaction
  SLEEPING
  TERMINATED
```

**Verification:** `pendingContinuation`, `resumeToken` present. `WAITING_HOST` in enum.

---

### Step 3: Replace Execution Loop in 09_runtime.md

**Objective:** Replace `Context.run()` with the version that uses `applyResult()`. Add `resume()` and `blockOnRequest()`.

**File:** `impl/09_runtime.md`

**Changes:** Replace the entire "Execution Loop" section (L371–489). Use the pseudocode from the proposal with one correction — `applyResult` must guard against jump-modified IP:

- `Context.run()` — captures `entryIp`/`entryStory` before driver execution, passes them to `applyResult(result, entryIp, entryStory)`
- `Context.applyResult(result, entryIp, entryStory)` — `Complete` → set value + advance IP **only if IP/story unchanged** (jump guard); `Suspend` → increment `resumeToken`, store continuation, call `blockOnRequest()`

**Note on jump guard:** The proposal's `applyResult` does unconditional `instructionPointer++` on `Complete`. This would break jump verbs (e.g., `/jump`) that modify `instructionPointer` directly before returning `Complete` — the first statement at the target would be skipped. The fix: `applyResult` receives `entryIp`/`entryStory` and only increments IP when both match current values (same pattern as the C# implementation). This is a pre-existing spec bug (current L416 has the same issue) that we fix while rewriting the loop.
- `Context.resume(outcome, token)` — guard on token match + `pendingContinuation != null`, invoke `onFulfilled`, call `applyResult()`
- `Context.blockOnRequest(request)` — match `WaitRequest` to set state + waitCondition (including `Host`)
- `Context.terminate()` — unchanged
- `Context.cleanupChannels()` — unchanged

**Rationale:** The execution loop is the core of the model change.

**Verification:** `applyResult` is used by both `run()` and `resume()`. IP advances only on `Complete`. `resume()` has double-call guard with token.

---

### Step 4: Replace Blocking Operations / Tick Loop in 09_runtime.md

**Objective:** Replace the blocking operations table, implementation pattern, and tick loop with the condition-only resolver.

**File:** `impl/09_runtime.md`

**Changes:** Replace the "Blocking Operations" section (L493–594) and the "Execution Model Compatibility" section (L598–631).

The blocking operations table stays similar but add `/converse`, `/choose`, `/prompt` under `Host` wait type.

Replace `Runtime.tick()` with the proposal's `resolveWait()` function that returns `WaitOutcome?` and calls `context.resume(outcome, token)`. No verb-specific logic in the scheduler.

Add host resume path section explaining that `WAITING_HOST` is resolved by the host calling `context.resume()` directly.

Replace the async model section to use `fulfillAsync(request)` → `WaitOutcome` pattern.

**Verification:** Tick loop contains zero verb-specific logic (no inline var copying, no diagnostic construction). Each `ContextState` match arm returns `WaitOutcome` or null.

---

### Step 5: Update Checkpoint Contract Validation in 09_runtime.md

**Objective:** No changes needed — checkpoint validation is independent of the continuation model.

**Verification:** Confirm the section (L636–663) does not reference `ExecutionResult` or `Continuation`.

---

### Step 6: Rewrite CallDriver in 08_concurrency.md

**Objective:** Replace `CallDriver` pseudocode to use `DriverResult` with `onFulfilled` closure for inline var copying.

**File:** `impl/08_concurrency.md`

**Changes:** Replace the CallDriver implementation (L240–277).

```
// Before (L273-277):
    return ok(continuation: Context {
        contextId: newContext.id,
        inlineVars: shouldInline ? initVars : []
    })

// After:
    inlineVars = shouldInline ? initVars : []
    childId = newContext.id

    return Suspend {
      continuation: Continuation {
        request: JoinContext { contextId: childId },
        onFulfilled: (outcome) ->
          match outcome:
            Completed { value }:
              for ref in inlineVars:
                childCtx = context.runtime.findContext(childId)
                val = childCtx?.get(ref.name) ?? Nothing
                context.set(ref.name, val)
              Complete { value, [] }
            Cancelled { code, message }:
              Complete { Nothing, [Diagnostic(ERROR, code, message)] }
      }
    }
```

Also update the helper return conventions: `ok()` → `Complete { ... }`, `fatal()` → `Complete { Nothing, [Diagnostic(FATAL, ...)] }`.

**Verification:** Inline var copying is in `onFulfilled`, not in the tick loop.

---

### Step 7: Rewrite Channel Drivers in 08_concurrency.md

**Objective:** Replace PushDriver and PullDriver pseudocode to use `DriverResult`.

**File:** `impl/08_concurrency.md`

**Changes:** Rewrite PushDriver (around L470–506) and PullDriver (around L530–577) to use `Suspend`/`Complete` with `onFulfilled` closures. Non-blocking paths return `Complete` directly. Blocking paths return `Suspend` with `WaitRequest` + closure that maps `WaitOutcome` to `Complete`.

**Verification:** Fast paths return `Complete`. Slow paths return `Suspend` with `onFulfilled` handling `Completed`/`TimedOut`/`Cancelled`.

---

### Step 8: Rewrite Wait/Sleep Drivers in 08_concurrency.md

**Objective:** Replace WaitDriver and SleepDriver pseudocode.

**File:** `impl/08_concurrency.md`

**Changes:** Rewrite WaitDriver (around L640–660) and SleepDriver (around L690–705).

SleepDriver: `return Suspend { request: Sleep { durationMs }, onFulfilled: (_) -> Complete { Nothing, [] } }`

WaitDriver: `return Suspend { request: Signal { messageName, timeoutMs }, onFulfilled: (outcome) -> match Completed/TimedOut/Cancelled }`

**Verification:** Both use `Suspend`/`Complete` pattern.

---

### Step 9: Update ForkDriver and JumpDriver in 08_concurrency.md

**Objective:** ForkDriver and JumpDriver are non-blocking — just update return convention from `ok()` to `Complete`.

**File:** `impl/08_concurrency.md`

**Changes:**
- ForkDriver (L133–208): `return ok()` → `return Complete { Nothing, [] }`; `return fatal(...)` → `return Complete { Nothing, [Diagnostic(FATAL, ...)] }`
- JumpDriver (L85–136): same pattern

**Verification:** No `Suspend` — these verbs don't block.

---

## Verification Plan

### Manual Verification
- [ ] No references to `ExecutionResult` remain in either file
- [ ] No references to old `Continuation` union (flat data-only) remain
- [ ] Every `return ok(...)` in 08_concurrency.md replaced with `Complete`/`Suspend`
- [ ] Tick loop in 09_runtime.md contains zero verb-specific logic
- [ ] `resume()` has token guard
- [ ] `WAITING_HOST` is in ContextState and handled in `blockOnRequest()` and `resolveWait()`

### Acceptance Criteria Validation
| Criterion | How to Verify | Expected Result |
|-----------|---------------|-----------------|
| IP advancement fixed | Read `applyResult()` — IP++ only in `Complete` branch | No IP advancement in `Suspend` branch |
| Post-resume logic in drivers | Read `resolveWait()` in tick loop | Returns only `WaitOutcome`, no var copying or diagnostic construction |
| Extension point exists | `Host` variant in `WaitRequest` | Present, handled by `blockOnRequest()` and `resolveWait()` |
| Race protection | Read `resume()` | Token check + null check before invoking handler |

---

## Rollback Plan

Git revert the spec update commit. No cascading effects — the C# implementation plan depends on this but isn't executed simultaneously.

---

## Notes

### Assumptions
- The proposal pseudocode is the canonical reference — this plan transcribes it into the spec files
- `ForkDriver` and `JumpDriver` do not block (confirmed by reading 08_concurrency.md)
- The `ok()`/`fatal()` helper convention in 08_concurrency.md is informal and can be replaced with `Complete`/`Suspend` constructors

### Risks
- Large diff in 09_runtime.md — risk of accidentally dropping unrelated sections (e.g., Signal Manager, Asset Management). Mitigation: section-by-section replacement, verify table of contents after.
