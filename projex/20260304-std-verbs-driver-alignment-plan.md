# Std Verbs: Remove Presentation Subsystem, Align with Verb Driver Model

> **Status:** Ready
> **Created:** 2026-03-04
> **Reviewed:** 2026-03-04 — 20260304-std-verbs-driver-alignment-plan-review.md
> **Review Outcome:** Valid — Ready to Execute
> **Author:** agent
> **Source:** Direct request — post-consistency audit conversation
> **Related Projex:** 20260304-runtime-api-surface-spec-plan.md, 20260304-runtime-api-surface-spec-plan-log.md

---

## Summary

`impl/10_std_verbs.md` currently describes `/converse`, `/choose`, `/chooseFrom`, and `/prompt` as delegating to a made-up Runtime Presentation Interface (`runtime.present()`, `runtime.waitForContinue()`, `runtime.presentChoice()`, `runtime.presentPrompt()`), and `/show`, `/play`, `/hide`, etc. as calling runtime media methods. None of these methods belong in the runtime interface — they are rendering concerns entirely up to the host. The standard verbs are just verb drivers, like any other. Blocking verbs should resolve their parameters and return `Suspend { Host { timeoutMs } }`. Non-blocking media verbs are also just verb drivers; if the host wants them, it registers its own driver. The `impl` spec should not prescribe rendering details.

**Scope:** `impl/10_std_verbs.md` only. No changes to `09_runtime.md` or any other file.
**Estimated Changes:** 1 file — rewrite of presentation and media driver implementation sections + removal of "Runtime Presentation Interface" section.

---

## Objective

### Problem / Gap / Need

`10_std_verbs.md` treats the runtime as a presentation subsystem owner, adding methods (`present`, `waitForContinue`, `presentChoice`, `presentPrompt`, `showMedia`, `hideMedia`, etc.) that have no place in the `Runtime` interface defined in `09_runtime.md`. This:

1. Contradicts the established verb driver model — verb drivers are self-contained.
2. Leaks `Context` (or `ContextHandle`) as parameters to runtime methods, re-inventing host integration instead of using the existing `Suspend { Host }` mechanism.
3. Invents `PresentationRequest` and `MediaRequest` data types that have no canonical definition.
4. Describes non-blocking media verbs (`/show`, `/play`, `/hide`, `/stop`, etc.) as calling `runtime.showMedia()` etc. — these should simply be implementation-defined driver registrations by the host, not prescribed by the spec at all.

### Success Criteria

- [ ] `PresentationRequest`, `MediaRequest` types removed from the spec
- [ ] "Runtime Presentation Interface" section removed
- [ ] `/converse`, `/choose`, `/chooseFrom`, `/prompt` driver bodies return `Suspend { Host { timeoutMs } }` — no direct runtime method calls
- [ ] Value returned by `/choose`/`/chooseFrom`/`/prompt` is delivered via `onFulfilled: (Completed { value }) -> Complete { value, [] }`
- [ ] Media verb driver bodies (Show, Hide, Play, PlayOne, Stop, Pause, Resume, SetVolume, Focus, Unfocus) are replaced with a prose note that these are implementation-defined — host registers its own drivers
- [ ] No `context.runtime.X()` calls remain for presentation or media operations in the driver pseudocode
- [ ] Shorthand aliases (`ok`, `fatal`, etc.) consistent with the convention documented in `06_core_verbs.md`

### Out of Scope

- Changes to `09_runtime.md` — `WaitRequest.Host` already has `timeoutMs` only; no payload needed
- Changes to `06_core_verbs.md` or `08_concurrency.md`
- C# runtime implementation (separate projex scope under `csharp/`)

---

## Context

### Current State

`10_std_verbs.md` (565 lines) has two major areas needing change:

**Presentation verbs (L26–272):** `/converse`, `/choose`, `/chooseFrom`, `/prompt` all end by calling `context.runtime.waitForContinue(handle, ...)` / `context.runtime.presentChoice(handle, ...)` / `context.runtime.presentPrompt(handle, ...)` instead of returning `Suspend`. They also call `context.runtime.present(presentation)` for fire-and-forget display.

**Media verbs (L290–484):** `/show`, `/hide`, `/play`, `/playOne`, `/stop`, `/pause`, `/resume`, `/setVolume`, `/focus`, `/unfocus` — all call `context.runtime.showMedia(request)` etc.

**Runtime Presentation Interface (L488–512):** A `Runtime:` block defining ~12 methods that don't exist in `09_runtime.md`'s Runtime interface.

### Key Files

| File | Purpose | Changes Needed |
|------|---------|----------------|
| `impl/10_std_verbs.md` | Standard verb impl spec | Rewrite presentation driver bodies; replace media driver bodies with impl-defined note; remove Runtime Presentation Interface section |

### Dependencies

- **Requires:** `09_runtime.md` already defines `WaitRequest.Host { timeoutMs: double? }` and the host resume pattern — no changes needed there.
- **Blocks:** Nothing currently depends on the removed presentation interface.

---

## Implementation

### Overview

Rewrite the three affected areas in `10_std_verbs.md`:
1. Presentation verb driver bodies → `Suspend { Host { timeoutMs } }`
2. Media verb section → prose note (implementation-defined)
3. Remove the Runtime Presentation Interface section

### Step 1: Rewrite `/converse` Driver Body

**Objective:** Per spec (`spec/std_verbs.md` L9): each content item must block independently, as if it were a separate `/converse` call. The driver must issue one `Suspend { Host }` per content item, chained via `onFulfilled`. Remove all `context.runtime.*` calls.

**Files:** `impl/10_std_verbs.md`

**Changes:**

```
// Before (L26–82): calls context.runtime.present(), context.runtime.waitForContinue()

// After:
ConverseDriver.execute(call, context):
    speaker = getAttribute(call, "By")?.value
    portrait = getAttribute(call, "Portrait")?.value
    append = hasAttribute(call, "Append")
    style = getAttribute(call, "Style")?.value ?? "dialog"

    waitAttr = getAttribute(call, "Wait")
    if waitAttr != null:
        shouldWait = resolve(waitAttr.value, context).toBool()
    else:
        shouldWait = context.getFlag("interactive") ?? true

    timeout = getNamedParam(call, "timeout")
    timeoutMs = timeout != null ? resolve(timeout, context).toDouble() * 1000 : null
    if timeout != null and timeoutMs <= 0:
        return info("timeout", "Immediate timeout")

    # Pre-resolve all content items; fail fast on type errors before any suspension
    contents = []
    for param in call.unnamedParams:
        content = resolve(param, context)
        if content is not StringValue and content is not ExpressionValue:
            return fatal("type_mismatch",
                "Content must be string or expression, got: " + content.getType())
        if content is ExpressionValue:
            content = evaluate(content, context)
        if content is StringValue:
            content = interpolate(content.value, context)
        contents.add(content)

    if not shouldWait or contents.isEmpty():
        return Complete { Nothing, [] }

    # Per spec: each content blocks independently — chain suspensions via onFulfilled
    return converseNext(contents, 0, timeoutMs)

# Inner helper: drives the per-content suspension loop.
# The host driver has access to `contents[index]` via the resolved list
# captured in this closure — it reads the current item before resuming.
converseNext(contents, index, timeoutMs):
    if index >= contents.length:
        return Complete { Nothing, [] }
    return Suspend {
        continuation: Continuation {
            request: Host { timeoutMs },
            onFulfilled: (outcome) -> match outcome:
                Completed { _ }:          converseNext(contents, index + 1, timeoutMs)
                TimedOut:                 Complete { Nothing, [Diagnostic(INFO, "timeout", "Converse timed out")] }
                Cancelled { code, msg }:  Complete { Nothing, [Diagnostic(ERROR, code, msg)] }
        }
    }
```

**Rationale:** The spec requires per-content blocking. Chaining suspensions through `onFulfilled` is the correct pattern — each fulfillment drives the next item. The host driver override has access to both the `call` verb attributes (speaker, style, etc.) and the resolved `contents` list via the closure. When `shouldWait` is false, content delivery is fire-and-forget and implementation-defined: the host's driver override handles display without suspension.

**Verification:** Section contains no `context.runtime.*` calls; driver produces one `Suspend { Host }` per content item for the blocking path; non-blocking path returns `Complete { Nothing, [] }` immediately.

---

### Step 2: Rewrite `/choose` Driver Body

**Objective:** Build and resolve choices, return `Suspend { Host { timeoutMs } }`. The host delivers the selected value via `runtime.resume(handle, value)`, arriving as `Completed { value }` in `onFulfilled`.

**Files:** `impl/10_std_verbs.md`

**Changes:**

```
// After:
ChooseDriver.execute(call, context):
    speaker = getAttribute(call, "By")?.value
    portrait = getAttribute(call, "Portrait")?.value
    style = getAttribute(call, "Style")?.value ?? "normal"
    prompt = getNamedParam(call, "prompt")
    if prompt != null:
        prompt = resolveAndInterpolate(prompt, context)

    timeout = getNamedParam(call, "timeout")
    timeoutMs = timeout != null ? resolve(timeout, context).toDouble() * 1000 : null
    if timeout != null and timeoutMs <= 0:
        return info("timeout", "Immediate timeout")

    choices = []
    params = call.unnamedParams
    i = 0
    while i + 2 < params.length:
        visible = resolve(params[i], context)
        if visible is ExpressionValue: visible = evaluate(visible, context)
        if visible is VerbValue: visible = executeVerb(visible, context)

        if visible.toBool():
            text = resolveAndInterpolate(params[i + 1], context)
            value = resolve(params[i + 2], context)
            choices.add({ text: text.toString(), value })
        i += 3

    if choices.isEmpty():
        return Complete { Nothing, [Diagnostic(WARNING, "no_choices", "No visible choices")] }

    # choices resolved — host driver renders UI and resumes with selected value
    return Suspend {
        continuation: Continuation {
            request: Host { timeoutMs },
            onFulfilled: (outcome) -> match outcome:
                Completed { value }:       Complete { value, [] }
                TimedOut:                  Complete { Nothing, [Diagnostic(INFO, "timeout", "Choose timed out")] }
                Cancelled { code, msg }:   Complete { Nothing, [Diagnostic(ERROR, code, msg)] }
        }
    }
```

**Rationale:** The host's driver implementation has full access to the verb call (choices, prompt, speaker) from the `call` argument and the resolved `choices` list in its own closure. It resumes with the chosen value.

**Verification:** No `context.runtime.*` calls; `onFulfilled` returns the host-provided value.

---

### Step 3: Rewrite `/chooseFrom` Driver Body

**Objective:** Same as `/choose` but takes a list parameter. Prompt must be resolved with `resolveAndInterpolate` before suspension.

**Files:** `impl/10_std_verbs.md`

**Changes:**

```
// After:
ChooseFromDriver.execute(call, context):
    choicesList = resolve(call.params[0], context)
    prompt = getNamedParam(call, "prompt")
    if prompt != null:
        prompt = resolveAndInterpolate(prompt, context)
    timeout = getNamedParam(call, "timeout")
    timeoutMs = timeout != null ? resolve(timeout, context).toDouble() * 1000 : null
    if timeout != null and timeoutMs <= 0:
        return info("timeout", "Immediate timeout")

    if choicesList is not ListValue:
        return fatal("invalid_type", "Expected list of maps, got: " + choicesList.getType())

    choices = []
    for item in choicesList.elements:
        if item is not MapValue or item.entries.size != 1:
            return fatal("invalid_type", "Each choice must be a single-entry map")
        for (text, value) in item.entries:
            choices.add({ text, value })

    return Suspend {
        continuation: Continuation {
            request: Host { timeoutMs },
            onFulfilled: (outcome) -> match outcome:
                Completed { value }:      Complete { value, [] }
                TimedOut:                 Complete { Nothing, [Diagnostic(INFO, "timeout", "ChooseFrom timed out")] }
                Cancelled { code, msg }:  Complete { Nothing, [Diagnostic(ERROR, code, msg)] }
        }
    }
```

**Verification:** No `context.runtime.*` calls; `prompt` resolved with `resolveAndInterpolate` matching convention in `/choose`; same `Suspend { Host }` pattern.

---

### Step 4: Rewrite `/prompt` Driver Body

**Objective:** Suspend waiting for text input; host resumes with the entered string as value.

**Files:** `impl/10_std_verbs.md`

**Changes:**

```
// After:
PromptDriver.execute(call, context):
    promptText = null
    if call.params.length > 0:
        promptText = resolveAndInterpolate(call.params[0], context)

    style = getAttribute(call, "Style")?.value ?? "normal"
    timeout = getNamedParam(call, "timeout")
    timeoutMs = timeout != null ? resolve(timeout, context).toDouble() * 1000 : null
    if timeout != null and timeoutMs <= 0:
        return info("timeout", "Immediate timeout")

    return Suspend {
        continuation: Continuation {
            request: Host { timeoutMs },
            onFulfilled: (outcome) -> match outcome:
                Completed { value }:      Complete { StringValue(value.toString()), [] }
                TimedOut:                 Complete { Nothing, [Diagnostic(INFO, "timeout", "Prompt timed out")] }
                Cancelled { code, msg }:  Complete { Nothing, [Diagnostic(ERROR, code, msg)] }
        }
    }
```

**Verification:** No `context.runtime.*` calls.

---

### Step 5: Replace Media Verb Section with Impl-Defined Note

**Objective:** Remove the 10 media driver bodies (`ShowDriver`, `HideDriver`, `PlayDriver`, etc.) and replace with a short prose section explaining these verbs are host-defined.

**Files:** `impl/10_std_verbs.md`

**Changes:** Replace the entire "## Media Verbs" section (L276–484) with:

```markdown
## Media Verbs

Media verbs (`/show`, `/hide`, `/play`, `/playOne`, `/stop`, `/pause`, `/resume`,
`/setVolume`, `/focus`, `/unfocus`) are **implementation-defined**. The spec defines
their signatures and attributes; the host registers its own verb drivers.

Media verbs are typically non-blocking — they fire a rendering command and return
`Complete { Nothing, [] }` (or `Complete { StringValue(id), [] }` for verbs that
return a handle). Blocking variants (e.g., waiting for an animation to finish) can
be implemented with `Suspend { Host { timeoutMs } }` if the host chooses.

No default driver is provided. If no driver is registered, the verb is an unknown
verb and becomes a no-op (a warning is logged per the execution loop).
```

**Rationale:** The spec's job is to define verb signatures and language semantics, not to describe how a game engine renders sprites. The host is free to implement these however it wants.

**Verification:** Section has no pseudocode driver bodies; states they are implementation-defined.

---

### Step 6: Remove Runtime Presentation Interface Section

**Objective:** Delete the "## Runtime Presentation Interface" section (L488–518 approximately, after Step 5 renumbering).

**Files:** `impl/10_std_verbs.md`

**Changes:** Remove the section entirely:

```markdown
// REMOVE:
## Runtime Presentation Interface

Standard verbs delegate to runtime presentation layer:

```
Runtime:
  # Presentation — ...
  present(request: PresentationRequest): void
  waitForContinue(handle: ContextHandle): void
  ...
```
```

**Verification:** No `Runtime:` block with presentation/media methods in the file.

---

### Step 7: Verify `resolveAndInterpolate` Helper

`resolveAndInterpolate` is a helper used by `/choose`, `/chooseFrom`, `/converse`. It is defined inline after the `ChooseDriver` body (currently L165–178). Verify it remains in place and is still referenced correctly by the rewritten drivers.

**Verification:** `resolveAndInterpolate` definition still present; used by `/converse`, `/choose`, `/chooseFrom` (prompt param), and `/prompt` where needed.

---

### Step 8: Review Testing Checklist

**Objective:** The Testing Checklist (L524–568) documents verb behavior tests. After Steps 5 and 6, media verb driver bodies are removed. The checklist entries for Show/Hide, Play/Stop, and Volume Control test verb semantics (not runtime internals) and remain valid as embedder-level testing guidance.

**Files:** `impl/10_std_verbs.md`

**Changes:** No deletions. Add a prose header before the Testing Checklist clarifying its purpose:

```markdown
## Testing Checklist

> These are embedder-level behavioral tests — verifying that a host's driver
> implementation meets the verb semantics defined in `spec/std_verbs.md`.
> They are not unit tests of the spec pseudocode.
```

**Verification:** Header present; no checklist items removed.

---

## Verification Plan

### Automated Checks

- [ ] No `context\.runtime\.` calls remain in `10_std_verbs.md` driver pseudocode
- [ ] No `PresentationRequest` or `MediaRequest` type names remain in `10_std_verbs.md`
- [ ] No `Runtime:` block with presentation methods remains

```powershell
# From repo root:
Select-String -Path "impl/10_std_verbs.md" -Pattern "context\.runtime\.|PresentationRequest|MediaRequest|waitForContinue|presentChoice|presentPrompt|showMedia|hideMedia|playMedia"
# Expected: 0 matches
```

### Manual Verification

- [ ] Read the file top-to-bottom and confirm: every blocking presentation verb driver ends with `return Suspend { ... }`, every non-blocking path ends with `return Complete { ... }` or a `return ok()/fatal()` alias
- [ ] Confirm the media verbs section is prose-only (no pseudocode driver implementations)
- [ ] Confirm no `Runtime:` block appears in the file

### Acceptance Criteria Validation

| Criterion | How to Verify | Expected Result |
|-----------|---------------|-----------------|
| `PresentationRequest/MediaRequest` removed | `Select-String` grep | 0 matches |
| `Runtime Presentation Interface` removed | `Select-String "Runtime:"` | 0 matches |
| Presentation drivers use `Suspend { Host }` | Read driver implementations | All blocking paths return `Suspend { Host { timeoutMs } }` |
| Media verbs are impl-defined | Read Media Verbs section | Prose only, no driver pseudocode |
| No `context.runtime.*` in drivers | `Select-String` grep | 0 matches |

---

## Rollback Plan

All changes are in `impl/10_std_verbs.md` on the ephemeral branch. Rollback = `git checkout main -- impl/10_std_verbs.md`.

---

## Notes

### Assumptions

- `WaitRequest.Host { timeoutMs: double? }` in `09_runtime.md` requires no changes — drivers embed their resolved data in their own closures, not in the WaitRequest.
- The host's driver implementation has access to the full `call: CompiledVerbCall` argument and can read attributes/params directly — it doesn't need the spec to pass a pre-packaged struct.
- `resolveAndInterpolate` helper stays as-is.

### Risks

- **Coverage gap for media verbs:** Removing the pseudocode leaves no reference implementation for media verbs. Mitigated by the prose note directing hosts to write their own drivers, and by the existence of the verb signature sections (which are unchanged).
