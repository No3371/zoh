# Standard Verbs

Standard verbs are expected to play most stories without issues.

## Std.Converse

A converse verb indicates the runtime to present a text at presentation layer.

When multiple contents are provided, the runtime should represent them sequentially in individual presentations. That is, each content is handled as if it was a separate `/converse` verb. This enables clean syntax for multiple dialog in succession.

### Named Parameters
- `timeout`: the duration in seconds to wait before timing out. Accept `double`/`*double` or `?`. Optional.

### Parameters
- Repeating of the following:
    - `content`: content to present. Accept `"string"`/`*"string"` or `` `expr` ``/`` *`expr` ``. In case of references, the value is used. In case of `` `expr` ``, the expression is evaluated. In case of `"string"`, it performs `/interpolate` once.

### Returns
A nothing.

### Diagnostics
- Info: `timeout`: The timeout was reached.

### Attributes
- **Append**: specify the content should be appended to existing presentation.
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

## Std.Choose

A choose verb indicates the runtime to present a choice interaction.

### Named Parameters
- `prompt`: the prompt text. Accept `"string"`/`*"string` or `` `expr` ``/`` *`expr` ``. In case of references, the value is used. In case of `` `expr` ``, the expression is evaluated. In case of `"string"`, it performs `/interpolate` once. Optional.
- `timeout`: the duration in seconds to wait before timing out. Accept `double`/`*double` or `?`. Optional.

### Parameters
- Repeating of the following:
    - `visible`: specify if the choice is visible. Accept `boolean`, boolean-returning `/verb`, boolean-evaluated `` `expr` ``, or references to these types.  Default to `true`.
    - `text`: the choice text. Accept `"string"`/`*"string"` or `` `expr` ``/`` *`expr` ``. In case of references, the value is used. In case of `` `expr` ``, the expression is evaluated. In case of `"string"`, it performs `/interpolate` once.
    - `value`: the value to return if selected. Accept `any`.

### Returns
The value of the selected choice.

### Diagnostics
- Info: `timeout`: The timeout was reached.

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

## Std.ChooseFrom

A chooseFrom verb indicates the runtime to present a choice interaction from a list.

### Named Parameters
- `prompt`: the prompt text. Accept `"string"`/`*"string` or `` `expr` ``/`` *`expr` ``. In case of references, the value is used. In case of `` `expr` ``, the expression is evaluated. In case of `"string"`, it performs `/interpolate` once. Optional.
- `timeout`: the duration in seconds to wait before timing out. Accept `double`/`*double` or `?`. Optional.

### Parameters
- `choices`: the choices. Accept `[list]`/`*[list]` of single entry string-any `{map}`.

### Returns
The value of the selected choice.

### Diagnostics
- Info: `timeout`: The timeout was reached.

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

## Std.Prompt

A prompt verb indicates the runtime to present a text input.

### Parameters
- `prompt`: the prompt text. Accept `"string"`/`*"string` or `` `expr` ``/`` *`expr` ``. In case of references, the value is used. In case of `` `expr` ``, the expression is evaluated. In case of `"string"`, it performs `/interpolate` once. Optional.
- `timeout`: the duration in seconds to wait before timing out. Accept `double`/`*double` or `?`. Optional.

### Returns
The string value of the input.

### Diagnostics
- Info: `timeout`: The timeout was reached.

### Attributes
- **Style**: specify the style of the text. Accept `string` or `*string`. For example: "normal", "focus".

### Examples
```
/prompt "What is your name?";
```

## Std.Show

A show verb indicates the runtime to present a visual media resource at presentation layer.

### Parameters
- `resource`: the resource to show (e.g. image path). Accept `string` or `*string`.

### Returns
The id of the presentation.

### Attributes
- **Id** (string): the id of the presentation. Defaults to the resource path.
- **RW** (double): relative width to the viewport.
- **RH** (double): relative height to the viewport.
- **Width** (double): absolute width.
- **Height** (double): absolute height.
- **Anchor** (string): the anchor point ("center", "top", "bottom", "left", "right", "top-left", "top-right", "bottom-left", "bottom-right").
- **PosX** (double): anchor x position (relative to parent/screen).
- **PosY** (double): anchor y position (relative to parent/screen).
- **PosZ** (double): z-index.
- **Fade**: the fade duration in seconds. Defaults to 0.0.
- **Opacity**: the opacity. Defaults to 1.0.
- **Easing**: the easing function. Defaults to "linear".

### Examples
```
/show "image.png";
/show "image.png" [anchor: "center"];
/show "image.png" [anchor: "center", x: 0.5, y: 0.5];
```

## Std.Hide

A hide verb indicates the runtime to hide a media resource at presentation layer.

### Parameters
- `id`: the id of the presentation to hide. Accept `string` or `*string`.

### Returns
A nothing.

### Attributes
- **Fade**: the fade duration in seconds. Defaults to 0.0.
- **Easing**: the easing function. Defaults to "linear".

### Examples
```
/hide "image.png";
```

## Std.Focus

A focus verb indicates the runtime to help the user focus on something at presentation layer, if applicable.

### Parameters
- `target`: the target to focus on. Accept `string` or `*string`.

### Returns
A nothing.

### Examples
```
/focus "npc";
```

## Std.Unfocus

An unfocus verb indicates the runtime to help the user unfocus from something at presentation layer, if applicable.

### Parameters
None.

### Returns
A nothing.

### Examples
```
/unfocus;
```

## Std.Play

A Play verb indicates the runtime to play something playable. Should be compatible with Show and Hide, as those controls visiblity, not playback.

### Parameters
- `resource`: the resource to play. Accept `string` or `*string`.

### Returns
The id of the playback.

### Attributes
- **Id** (string): specify the playback id. Defaults to the resource path. Cut and replace existing playback with the same id.
- **Volume** (double): specify the volume. Defaults to 1.
- **Loops** (integer): specify the number of times to loop. Defaults to 1. -1 means infinite.
- **Easing**: the easing function. Defaults to "linear".
- Attributes from Std.Show, if applicable.

### Examples
```
/play "sound.mp3";
```

### Std.PlayOne

A PlayOne verb indicates the runtime to play something playable without keeping track of it.

### Parameters
- `resource`: the resource to play. Accept `string` or `*string`.

### Returns
The id of the playback.

### Attributes
- **Volume** (double): specify the volume. Defaults to 1.
- **Loops** (integer): specify the number of times to loop. Defaults to 1. -1 means infinite.
- **Easing**: the easing function. Defaults to "linear".
- Attributes from Std.Show, if applicable.

### Examples
```
/playOne "sound.mp3";
```

## Std.Stop

A Stop verb indicates the runtime to stop something playable.

### Parameters
- `id`: the playback id to stop. Accept `string` or `*string`.

### Attributes
- **Fade**: the fade duration in seconds. Defaults to 0.0.
- **Easing**: the easing function. Defaults to "linear".

### Returns
A nothing.

### Examples
```
/stop [AudioChannel: "channel_name"]; :: Stops the channel
/stop "sound.mp3"; :: Stops the specific sound instance
/stop; :: Stops all sounds
```

## Std.Pause

A Pause verb indicates the runtime to pause something playable.

### Parameters
- `id`: the playback id to pause. Accept `string` or `*string`.

### Returns
A nothing.

### Examples
```
/pause "sound.mp3";
```

## Std.Resume

A Resume verb indicates the runtime to resume something playable.

### Parameters
- `id`: the playback id to resume. Accept `string` or `*string`.

### Returns
A nothing.

### Examples
```
/resume "sound.mp3";
```

## Std.SetVolume

A SetVolume verb indicates the runtime to set the volume of a ongoing playback.

### Parameters
- `id`: the playback id to set the volume for. Accept `string` or `*string`.
- `volume`: the volume level. Accept `double` or `*double`.

### Returns
A nothing.

### Attributes
- **Fade**: the fade duration in seconds. Defaults to 0.0.
- **Easing**: the easing function. Defaults to "linear".

### Examples
```
/setVolume "sound.mp3", 0.5;
```