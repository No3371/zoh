# 10: Standard Verbs Implementation

## Purpose

Standard verbs handle presentation and media, expected to play most stories. They're not strictly required but provide the common storytelling interface.

---

## Presentation Verbs

### Std.Converse

**Purpose**: Display dialogue or narration.

#### Signature
```
/converse [By:speaker] [Portrait:image] [Append] [Wait:bool] [Style:string]
/converse [By:speaker] [Portrait:image] [Append] [Wait:bool] [Style:string]
          timeout:seconds?
          content1, content2, ...;
```

#### Implementation

```
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

---

### Std.Choose

**Purpose**: Present choices to user.

#### Signature
```
/choose [By:speaker] [Portrait:image] [Style:string]
        prompt:text?, timeout:seconds?,
        visible1, text1, value1,
        visible2, text2, value2,
        ...;
```

#### Implementation

```
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

resolveAndInterpolate(param, context): Value
    value = resolve(param, context)
    
    # Validate type: only string or expression allowed per spec
    if value is not StringValue and value is not ExpressionValue:
        return fatal("type_mismatch", 
            "Text must be string or expression, got: " + value.getType())
    
    if value is ExpressionValue:
        value = evaluate(value, context)
    # String: always interpolate (per spec: performs /interpolate once)
    if value is StringValue:
        value = StringValue(interpolate(value.value, context))
    return value
```

---

### Std.ChooseFrom

**Purpose**: Present choices from a list.

#### Signature
```
/chooseFrom [By:speaker] choices, prompt:text?, timeout:seconds?;
```

#### Implementation

```
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

---

### Std.Prompt

**Purpose**: Get text input from user.

#### Signature
```
/prompt [Style:string] text?, timeout:seconds?;
```

#### Implementation

```
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

---

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

---

## Testing Checklist

> These are embedder-level behavioral tests — verifying that a host's driver
> implementation meets the verb semantics defined in `spec/std_verbs.md`.
> They are not unit tests of the spec pseudocode.

### Converse
- [ ] Single content
- [ ] Multiple contents (sequential)
- [ ] With speaker
- [ ] With interpolation
- [ ] Wait behavior (attribute, flag)
- [ ] Append mode

### Choose
- [ ] Simple choices
- [ ] With prompt
- [ ] Conditional visibility
- [ ] Expression visibility
- [ ] Returns selected value

### ChooseFrom
- [ ] From list of maps
- [ ] Invalid format error

### Prompt
- [ ] Basic input
- [ ] With prompt text
- [ ] Returns string

### Show/Hide
- [ ] Basic show
- [ ] With positioning
- [ ] With fade
- [ ] Hide by id
- [ ] Returns id

### Play/Stop
- [ ] Play audio
- [ ] With loops
- [ ] With volume
- [ ] Stop by id
- [ ] Stop all
- [ ] Fade out

### Volume Control
- [ ] Set volume
- [ ] With fade transition
