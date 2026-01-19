# Standard Verbs

Standard verbs are expected to play most stories without issues.

## Converse

A converse verb indicates the runtime to present a text at presentation layer.

When multiple contents are provided, the runtime should represent them sequentially in individual presentations. That is, each content is handled as if it was a separate `/converse` verb. This enables clean syntax for multiple dialog in succession.

### Parameters
- Repeating of the following:
    - `content`: content to present. Accept `any`. In case of references, the value is used. In case of `` `expr` ``, the expression is evaluated. In case of `"string"`, if it contains any `*`, it performs `/interpolate` once.

### Returns
A Nothing.

### Attributes
- **Append**: specify if the content should be appended to the existing content. Accept `boolean` or `*boolean`. Defaults to `false`.
- **By**: specify the speaker. Accept `string` or `*string`.
- **Wait**: specify if the playback should block until the user responds. Accept `boolean` or `*boolean`. Defaults to the value of flag `#interactive`.
- **Portrait**: specify an avatar image for the speaker. Accept `string` or `*string`.
- **Style**: specify the style of the text. Accept `string` or `*string`. For example: "dialog", "notification", "toast", "modal".

### Examples
```
/converse "Hello, world!";
/converse [By: "John"] "Hello, world!";
/converse [Wait: false] "Hello";
/converse [Wait: true] `*name + "!"`;
/converse "Hello, ${*name}!";

/converse
    "Well, you see...",
    "Maybe we can leave early...",
    "What do you think?"
;
```

## Choose

A choose verb indicates the runtime to present a choice interaction.

### Named Parameters
- `prompt`: the prompt text. Accept `"string"`/`*"string` or `` `expr` ``/`` *`expr` ``. In case of references, the value is used. In case of `` `expr` ``, the expression is evaluated. In case of `"string"`, it performs `/interpolate` once. Optional.

### Parameters
- Repeating of the following:
    - `visible`: specify if the choice is visible. Accept `boolean`, boolean-returning `/verb`, boolean-evaluated `` `expr` ``, or references to these types.  Default to `true`.
    - `text`: the choice text. Accept `any`. In case of references, the value is used. In case of `` `expr` ``, the expression is evaluated. In case of `"string"`, it performs `/interpolate` once.
    - `value`: the value to return if selected. Accept `any`.

### Returns
The value of the selected choice.

### Attributes
- **By**: specify the speaker. Accept `string` or `*string`.
- **Portrait**: specify an avatar image for the speaker. Accept `string` or `*string`.
- **Style**: specify the style of the text. Accept `string` or `*string`. For example: "normal", "focus".

### Examples
```
/choose prompt: "prompt", true, "opt1", "value1", true, "opt2", "value2";
/choose true, "opt1", "value1", true, "opt2", "value2"; :: No prompt
/choose prompt: "prompt",
	true, "What would you like to eat?", "food",
	true, "How's your progress?", "work",
	*bool, "Hidden option", "hidden"
;
```

## ChooseFrom

A chooseFrom verb indicates the runtime to present a choice interaction from a list.

### Named Parameters
- `prompt`: the prompt text. Accept `"string"`/`*"string` or `` `expr` ``/`` *`expr` ``. In case of references, the value is used. In case of `` `expr` ``, the expression is evaluated. In case of `"string"`, it performs `/interpolate` once. Optional.

### Parameters
- `choices`: the choices. Accept `[list]`/`*[list]` of single entry string-any `{map}`.

### Returns
The value of the selected choice.

### Attributes
- **By**: specify the speaker. Accept `string` or `*string`.
- **Portrait**: specify an avatar image for the speaker. Accept `string` or `*string`.
- **Style**: specify the style of the text. Accept `string` or `*string`. For example: "normal", "focus".

### Examples
```
*choices <- [];
/append *choices, {"food":   value};
/append *choices, {"work":   value};
/append *choices, {"hidden": value};
/chooseFrom *choices;
/chooseFrom *choices, prompt: "prompt";
```

## Prompt

A prompt verb indicates the runtime to present a text input.

### Parameters
- `prompt`: the prompt text. Accept `"string"`/`*"string` or `` `expr` ``/`` *`expr` ``. In case of references, the value is used. In case of `` `expr` ``, the expression is evaluated. In case of `"string"`, it performs `/interpolate` once. Optional.

### Returns
The string value of the input.

### Attributes
- **Style**: specify the style of the text. Accept `string` or `*string`. For example: "normal", "focus".

### Examples
```
/prompt "What is your name?";
```

## Show

A show verb indicates the runtime to present a visual media resource at presentation layer.

### Parameters
- `resource`: the resource to show (e.g. image path). Accept `string` or `*string`.

### Returns
The id of the presentation.

### Attributes
- **id** (string): the id of the presentation. Defaults to the resource path.
- **rw** (double): relative width to the viewport.
- **rh** (double): relative height to the viewport.
- **w** (double): absolute width.
- **h** (double): absolute height.
- **anchor** (string): the anchor point ("center", "top", "bottom", "left", "right", "top-left", "top-right", "bottom-left", "bottom-right").
- **x** (double): anchor x position (relative to parent/screen).
- **y** (double): anchor y position (relative to parent/screen).
- **z** (double): z-index.
- **fade**: the fade duration in seconds. Defaults to 0.0.
- **opacity**: the opacity. Defaults to 1.0.
- **easing**: the easing function. Defaults to "linear".

### Examples
```
/show "image.png";
/show "image.png" [anchor: "center"];
/show "image.png" [anchor: "center", x: 0.5, y: 0.5];
```

## Hide

A hide verb indicates the runtime to hide a media resource at presentation layer.

### Parameters
- `id`: the id of the presentation to hide. Accept `string` or `*string`.

### Returns
A Nothing.

### Attributes
- **fade**: the fade duration in seconds. Defaults to 0.0.
- **easing**: the easing function. Defaults to "linear".

### Examples
```
/hide "image.png";
```

## Focus

A focus verb indicates the runtime to help the user focus on something at presentation layer, if applicable.

### Parameters
- `target`: the target to focus on. Accept `string` or `*string`.

### Returns
A Nothing.

### Examples
```
/focus "npc";
```

## Unfocus

An unfocus verb indicates the runtime to help the user unfocus from something at presentation layer, if applicable.

### Parameters
None.

### Returns
A Nothing.

### Examples
```
/unfocus;
```

## Play

A Play verb indicates the runtime to play a sound at presentation layer.

### Parameters
- `resource`: the audio resource to play. Accept `string` or `*string`.

### Returns
The id of the playback.

### Attributes
- **Id** (string): specify the audio id. Defaults to the resource path.
- **Volume** (double): specify the volume. Defaults to 1.
- **Loops** (integer): specify the number of times to loop. Defaults to 1. -1 means infinite.
- **AudioChannel** (string): specify the audio channel. The runtime should create or use anonymous channels to play audios that never conflict with each other.
- **fade**: the fade duration in seconds. Defaults to 0.0.
- **easing**: the easing function. Defaults to "linear".

### Examples
```
/Play "sound.mp3";
```

## Stop

A Stop verb indicates the runtime to stop a sound at presentation layer.

### Parameters
- `id`: the audio id to stop. Accept `string` or `*string`.

### Attributes
- **fade**: the fade duration in seconds. Defaults to 0.0.
- **easing**: the easing function. Defaults to "linear".

### Returns
A Nothing.

### Examples
```
/Stop [AudioChannel: "channel_name"]; :: Stops the channel
/Stop "sound.mp3"; :: Stops the specific sound instance
/Stop; :: Stops all sounds
```

## Pause

A PauseAudio verb indicates the runtime to pause a sound at presentation layer.

### Parameters
- `id`: the audio id to pause. Accept `string` or `*string`.

### Returns
A Nothing.

### Examples
```
/PauseAudio "sound.mp3";
```

## ResumeAudio

A ResumeAudio verb indicates the runtime to resume a sound at presentation layer.

### Parameters
- `id`: the audio id to resume. Accept `string` or `*string`.

### Returns
A Nothing.

### Examples
```
/ResumeAudio "sound.mp3";
```

## SetAudioChannelVolume

A SetAudioChannelVolume verb indicates the runtime to set the volume of the audio channel.

### Parameters
- `channel`: the name of the audio channel. Accept `string` or `*string`.
- `volume`: the volume level. Accept `double` or `*double`.

### Returns
A Nothing.

### Attributes
- **fade**: the fade duration in seconds. Defaults to 0.0.
- **easing**: the easing function. Defaults to "linear".

### Examples
```
/SetAudioChannelVolume "channel_name", 0.5;
```