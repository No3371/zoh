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
          content1, content2, ...;
```

#### Implementation

```
ConverseDriver.execute(call, context):
    speaker = getAttribute(call, "By")?.value
    portrait = getAttribute(call, "Portrait")?.value
    append = hasAttribute(call, "Append")
    style = getAttribute(call, "Style")?.value ?? "dialog"
    
    # Check wait behavior: attribute > flag > default
    waitAttr = getAttribute(call, "Wait")
    if waitAttr != null:
        shouldWait = resolve(waitAttr.value, context).toBool()
    else:
        shouldWait = context.getFlag("interactive") ?? true
    
    # Process each content parameter as separate presentation
    for param in call.unnamedParams:
        content = resolve(param, context)
        
        # Validate type: only string or expression allowed per spec
        if content is not StringValue and content is not ExpressionValue:
            return fatal("type_mismatch", 
                "Content must be string or expression, got: " + content.getType())
        
        # Expression: evaluate
        if content is ExpressionValue:
            content = evaluate(content, context)
        
        # String: always interpolate (per spec: performs /interpolate once)
        if content is StringValue:
            content = interpolate(content.value, context)
        
        # Send to presentation layer
        presentation = PresentationRequest {
            type: "converse",
            content: content.toString(),
            speaker: speaker?.toString(),
            portrait: portrait?.toString(),
            append: append,
            style: style
        }
        
        context.runtime.present(presentation)
        
        # Wait for user input if interactive
        if shouldWait:
            context.runtime.waitForContinue(context)
    
    return ok()
```

---

### Std.Choose

**Purpose**: Present choices to user.

#### Signature
```
/choose [By:speaker] [Portrait:image] [Style:string]
        prompt:text?,
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
    
    # Process prompt
    if prompt != null:
        prompt = resolveAndInterpolate(prompt, context)
    
    # Parse choices (triplets: visible, text, value)
    choices = []
    params = call.unnamedParams
    i = 0
    while i + 2 < params.length:
        visible = params[i]
        text = params[i + 1]
        value = params[i + 2]
        
        # Evaluate visibility
        visibleResolved = resolve(visible, context)
        if visibleResolved is ExpressionValue:
            visibleResolved = evaluate(visibleResolved, context)
        if visibleResolved is VerbValue:
            visibleResolved = executeVerb(visibleResolved, context)
        
        isVisible = visibleResolved.toBool()
        
        if isVisible:
            textResolved = resolveAndInterpolate(text, context)
            valueResolved = resolve(value, context)
            
            choices.add({
                text: textResolved.toString(),
                value: valueResolved
            })
        
        i += 3
    
    # Present choices
    presentation = PresentationRequest {
        type: "choose",
        prompt: prompt?.toString(),
        choices: choices,
        speaker: speaker?.toString(),
        portrait: portrait?.toString(),
        style: style
    }
    
    selectedIndex = context.runtime.presentChoice(presentation, context)
    return ok(choices[selectedIndex].value)

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
/chooseFrom [By:speaker] choices, prompt:text?;
```

#### Implementation

```
ChooseFromDriver.execute(call, context):
    choicesList = resolve(call.params[0], context)
    prompt = getNamedParam(call, "prompt")
    
    # choicesList is List of single-entry Maps: [{"text": value}, ...]
    choices = []
    
    if choicesList is not ListValue:
        return fatal("invalid_type", "Examples expected list of maps, got: " + choicesList.getType())
        
    for item in choicesList.elements:
        if item is not MapValue or item.entries.size != 1:
            return fatal("invalid_type", "Each choice must be a single-entry map")
        
        for (text, value) in item.entries:
            choices.add({ text, value })
    
    presentation = PresentationRequest {
        type: "choose",
        prompt: prompt?.toString(),
        choices: choices
    }
    
    selectedIndex = context.runtime.presentChoice(presentation, context)
    return ok(choices[selectedIndex].value)
```

---

### Std.Prompt

**Purpose**: Get text input from user.

#### Signature
```
/prompt [Style:string] text?;
```

#### Implementation

```
PromptDriver.execute(call, context):
    promptText = null
    if call.params.length > 0:
        promptText = resolveAndInterpolate(call.params[0], context)
    
    style = getAttribute(call, "Style")?.value ?? "normal"
    
    presentation = PresentationRequest {
        type: "prompt",
        prompt: promptText?.toString(),
        style: style.toString()
    }
    
    userInput = context.runtime.presentPrompt(presentation, context)
    return ok(StringValue(userInput))
```

---

## Media Verbs

### Std.Show

**Purpose**: Display visual media.

#### Signature
```
/show [id:string] [rw:double] [rh:double] [w:double] [h:double]
      [anchor:string] [x:double] [y:double] [z:double]
      [fade:double] [opacity:double] [easing:string]
      resource;
```

#### Implementation

```
ShowDriver.execute(call, context):
    resource = resolve(call.params[0], context).toString()
    
    # Build media request from attributes
    request = MediaRequest {
        type: "show",
        resource: resource,
        id: getAttribute(call, "id")?.value.toString() ?? resource,
        relativeWidth: getAttribute(call, "rw")?.value.toDouble(),
        relativeHeight: getAttribute(call, "rh")?.value.toDouble(),
        width: getAttribute(call, "w")?.value.toDouble(),
        height: getAttribute(call, "h")?.value.toDouble(),
        anchor: getAttribute(call, "anchor")?.value.toString() ?? "center",
        x: getAttribute(call, "x")?.value.toDouble() ?? 0.5,
        y: getAttribute(call, "y")?.value.toDouble() ?? 0.5,
        z: getAttribute(call, "z")?.value.toDouble() ?? 0,
        fadeDuration: getAttribute(call, "fade")?.value.toDouble() ?? 0,
        opacity: getAttribute(call, "opacity")?.value.toDouble() ?? 1.0,
        easing: getAttribute(call, "easing")?.value.toString() ?? "linear"
    }
    
    context.runtime.showMedia(request)
    return ok(StringValue(request.id))
```

---

### Std.Hide

**Purpose**: Hide visual media.

#### Signature
```
/hide [fade:double] [easing:string] id;
```

#### Implementation

```
HideDriver.execute(call, context):
    id = resolve(call.params[0], context).toString()
    fadeDuration = getAttribute(call, "fade")?.value.toDouble() ?? 0
    easing = getAttribute(call, "easing")?.value.toString() ?? "linear"
    
    request = MediaRequest {
        type: "hide",
        id: id,
        fadeDuration: fadeDuration,
        easing: easing
    }
    
    context.runtime.hideMedia(request)
    return ok()
```

---

### Std.Play

**Purpose**: Play audio/video.

#### Signature
```
/play [id:string] [volume:double] [loops:int] [easing:string] resource;
```

#### Implementation

```
PlayDriver.execute(call, context):
    resource = resolve(call.params[0], context).toString()
    
    request = MediaRequest {
        type: "play",
        resource: resource,
        id: getAttribute(call, "id")?.value.toString() ?? resource,
        volume: getAttribute(call, "volume")?.value.toDouble() ?? 1.0,
        loops: getAttribute(call, "loops")?.value.toInt() ?? 1,
        easing: getAttribute(call, "easing")?.value.toString() ?? "linear"
    }
    
    context.runtime.playMedia(request)
    return ok(StringValue(request.id))
```

---

### Std.PlayOne

**Purpose**: Play audio without tracking (fire-and-forget).

#### Implementation

```
PlayOneDriver.execute(call, context):
    resource = resolve(call.params[0], context).toString()
    
    request = MediaRequest {
        type: "playOne",
        resource: resource,
        volume: getAttribute(call, "volume")?.value.toDouble() ?? 1.0,
        loops: getAttribute(call, "loops")?.value.toInt() ?? 1
    }
    
    context.runtime.playOneShot(request)
    return ok()
```

---

### Std.Stop

**Purpose**: Stop playing media.

#### Signature
```
/stop [fade:double] [easing:string] id?;
```

#### Implementation

```
StopDriver.execute(call, context):
    fadeDuration = getAttribute(call, "fade")?.value.toDouble() ?? 0
    easing = getAttribute(call, "easing")?.value.toString() ?? "linear"
    
    if call.params.length == 0:
        # Stop all
        context.runtime.stopAllMedia(fadeDuration, easing)
    else:
        id = resolve(call.params[0], context).toString()
        context.runtime.stopMedia(id, fadeDuration, easing)
    
    return ok()
```

---

### Std.Pause / Std.Resume

```
PauseDriver.execute(call, context):
    id = resolve(call.params[0], context).toString()
    context.runtime.pauseMedia(id)
    return ok()

ResumeDriver.execute(call, context):
    id = resolve(call.params[0], context).toString()
    context.runtime.resumeMedia(id)
    return ok()
```

---

### Std.SetVolume

**Purpose**: Adjust volume of playing media.

#### Signature
```
/setVolume [fade:double] [easing:string] id, volume;
```

#### Implementation

```
SetVolumeDriver.execute(call, context):
    id = resolve(call.params[0], context).toString()
    volume = resolve(call.params[1], context).toDouble()
    fadeDuration = getAttribute(call, "fade")?.value.toDouble() ?? 0
    easing = getAttribute(call, "easing")?.value.toString() ?? "linear"
    
    context.runtime.setMediaVolume(id, volume, fadeDuration, easing)
    return ok()
```

---

### Std.Focus / Std.Unfocus

**Purpose**: Direct user attention.

```
FocusDriver.execute(call, context):
    target = resolve(call.params[0], context).toString()
    context.runtime.focus(target)
    return ok()

UnfocusDriver.execute(call, context):
    context.runtime.unfocus()
    return ok()
```

---

## Runtime Presentation Interface

Standard verbs delegate to runtime presentation layer:

```
Runtime:
  # Presentation
  present(request: PresentationRequest): void
  waitForContinue(context: Context): void
  presentChoice(request: PresentationRequest, context: Context): int
  presentPrompt(request: PresentationRequest, context: Context): string
  
  # Media
  showMedia(request: MediaRequest): void
  hideMedia(request: MediaRequest): void
  playMedia(request: MediaRequest): void
  playOneShot(request: MediaRequest): void
  stopMedia(id: string, fade: double, easing: string): void
  stopAllMedia(fade: double, easing: string): void
  pauseMedia(id: string): void
  resumeMedia(id: string): void
  setMediaVolume(id: string, volume: double, fade: double, easing: string): void
  focus(target: string): void
  unfocus(): void
```

This interface allows different runtime implementations:
- **Text Console**: Print text, read input
- **Visual Novel**: GUI dialogs, sprite system, audio
- **3D Engine**: 3D actors, camera control, spatial audio

---

## Testing Checklist

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
