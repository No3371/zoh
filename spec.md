# ZOH

ZOH is an embedded scripting language built for storytelling but authored and run like a general programming language.

## Everything is a Verb

Everything in ZOH is a *verb*. Characters `/converse`. Scenes `/show`. Music `/play`. You're not writing a story—you're directing a cast.

Even the core mechanics of ZOH are built on verbs. Want to keep track of a variable? `/set` it. Want to control the flow? `/if` it. Want to loop? `/while` it.

Extensibility is inherent to the language. Define new verbs to fit your needs.

Everything is a *verb*. This uniform structure removes the need for special syntax per feature and makes scripts easier to reason about.

## Ownership to Story Playback

Compared to options like Ink or Twine, ZOH moves story text back into a "content parameter" instead of the "script body". For example, instead of writing just `The quick brown fox jumps over the lazy dog`, you write `/converse "The quick brown fox jumps over the lazy dog"`.

ZOH inverts the relationship between the story script and the player implementation so it can do more than show text line by line. Instead of requiring developers to add effects or mechanisms around the story, ZOH scripts act as stage directions that drive the player to present everything in the scene.

So while ZOH sacrifices some of the conciseness of Ink or Twine, it lets authors interact with the player implementation more directly in the same language.

## Do this, if you can

The same script can power a text console, a visual novel, or a 3D cinematic. ZOH doesn't care what your game looks like; that's the runtime's job. Write the story once, present it however you want.

One core principle of ZOH is Progressive Enhancement. As an embedded scripting language, ZOH is expected to be implemented by runtimes in various languages, on various platforms, minimalistic or full-fledged.

The idea is that any ZOH script should run in any runtime that implements the core protocol, while runtimes extended with additional verbs can provide richer experiences and represent the script better to the user.

This is because verbs are intentions executed by the runtime. On paper, no verb is required; the actual behavior of any verb is up to the runtime.

In practice, the language itself is built on verbs (including fundamental features like variable management and flow control), so a runtime must implement the core verbs and comply with their behavior definitions to run a ZOH script.

## At a Glance

```zoh
The Last Coffee Shop
===

@opening
:: Set the stage
/show [id: "bg"] [fade: 1.5] "cafe_night.png";
/play [id: "ambience"] [loops: -1] "rain_and_jazz.ogg";

*player_name <- "stranger";
*trust <- 0;

:: Fork parallel contexts—the world keeps moving while you talk
====+ @cafe_atmosphere;
====+ @barista_idle;

/converse [By: "Narrator"] "11:47 PM. The cafe closes at midnight. Rain scribbles on the glass.";
/converse "A woman in a red coat sits alone by the window, a sealed envelope on the table.";

====> @first_approach;

@first_approach
/choose/
    prompt: "The rain picks up outside."
    true  "Sit across from her"         "sit"
    true  "Order something first"       "order"
    true  "Leave—this isn't your story" "leave"
/; -> *choice;

/switch/ *choice
    "sit"   /jump ?, "sit_down";
    "order" /jump ?, "at_counter";
    "leave" /jump ?, "walk_away";
/; -> *dest;
/do *dest;

@at_counter
/converse [By: "Barista"] "Last call. What'll it be?";
/prompt "Your order:"; -> *order;
/converse [By: "Barista"] "Coming right up.";
/sleep 1.5;

*trust [resolve] <- `*trust + 1`;    :: Small gesture of patience
====> @sit_down;

@sit_down
/push <stop_idle>, true;    :: Signal the barista to stop her idle loop

/converse [By: "Woman"] "You're not from around here.";

/choose/
    prompt: "She's studying your face like it belongs to a name she's trying to place."
    true                "How can you tell?"   "deflect"
    `*trust > 0`        "Neither are you."    "mirror"
    true                "Say nothing"         "silence"
/; -> *response;

/if *response, is: "mirror", /sequence/
    *trust [resolve] <- `*trust + 2`;
    /converse [By: "Woman"] "...No. I'm not.";
    /converse "For the first time, she smiles.";
/;;

/if *response, is: "silence", /sequence/
    /converse "The silence stretches. Outside, a car glides past with no headlights.";
    /play [id: "car"] "car_passing.ogg";
/;;

/push <ending>, *trust;
====> @closing_time;

@walk_away  
/converse "Some doors are better left unopened. You leave with the rain.";
/push <ending>, -1;
====> @closing_time;

@closing_time
/pull <ending>; -> *final_trust;

/if `*final_trust < 0`, /sequence/
    /converse [By: "Narrator"] "The cafe light flickers off behind you. Whatever she needed will wait for someone else.";
    /stop [fade: 3] "ambience";
    /exit;
/;;

/if `*final_trust >= 3`, /sequence/
    /converse [By: "Woman"] "Wait—";
    /converse "She slides a folded napkin across the table. A map, a time, and a single word: \"basement\".";
    /show [fade: 0.5] "napkin_note.png";
/;;

/converse [By: "Narrator"] "The barista dims the lights. Midnight, and the street goes quiet.";
/stop [fade: 2] "ambience";
/sleep 2;

:: ═══════════════════════════════════════════════════════════
:: PARALLEL CONTEXTS — These run independently alongside the main context above
:: ═══════════════════════════════════════════════════════════

@cafe_atmosphere
:: Ambient events that fire periodically
/loop 999, /sequence/
    /sleep /rand 5.0, 8.0;;
    /roll "thunder.ogg", "dishes.ogg", "door_chime.ogg"; -> *sound;
    /play *sound;
/;;

@barista_idle
:: The barista has her own life until you interrupt it
/while `true`, /sequence/
    /pull <stop_idle>, timeout: 8; -> *stop;
    /if *stop, /exit;;
    /roll 
        "She wipes down the counter.",
        "She counts the tips, then looks toward the door.",
        "She hums along to the music."
    ; -> *action;
    /converse [By: "—"] *action;
/;;
```

This example shows:
- **Parallel contexts** (`====+`) running ambient cafe life alongside the main dialogue
- **Channel synchronization** (`/push`, `/pull`) to coordinate between contexts  
- **Conditional choices** with expressions (`` `*trust > 0` ``)
- **Presentation verbs** (`/show`, `/play`, `/stop`) directing visuals and audio
- **Block form** (`/verb/ ... /;`) for readable multi-line structures
- **Randomization** (`/rand`, `/roll`) for dynamic ambient events
- **Variable state** tracking player decisions across the scene

---

# Anatomy

```
Story Name
meta_key: meta_value;
===
#flag flag_name on
    
:: Inline
::comments
:::
Multi-line
comments
:::

/verb;

:: valid one-liners
/verb;
/verb [attribute];
/verb [attribute_with_value:attr_value] [even_more_attr];
/verb p_name:p_v;                         :: Named parameters
/verb p1, p2, p3;                         :: Unnamed parameters
/verb p_name:p_v, p1, p2, p3;             :: Named before unnamed
/verb p_name1:p_v1, p2, p_name3:p_v3, p4; :: Named mixed with unnamed
/capture "return_into_var";
-> *return_into_var;                      :: Syntactic sugar of /capture "return_into_var";
/verb;                                    :: Return value is not required.
   /verb;                                 :: Verbs don't have to start at the beginning of a line
/verb; -> *return_into_var;               :: Therefore this is valid one liner combining /capture after /verb to declutter scripts

/verb [attr1] [attr2];                    :: Spaces around and between attributes are required
/verb p1,p2,p3;                           :: Spaces between parameters are optional

:: Valid multi-line verb calls
:: Newlines do not break verb calls
/verb
;
/verb p1, p2,
p3;

/verb [attr1]
[attr2]
[attr3]
p1,p2,p3;
-> *return_into_var
; 

/verb [attribute1] [attribute_with_value:attr_value] 
[even_more_attr] param_name1:param_value1,
param_name2:param_value2,
unnamed_param1, unnamed_param2...;
-> *return_into_var;

:: Alternative block form verb call allows delimiting parameters with spaces/newlines
/verb/
	p1
	p2
/;

#flag flag_name off
```

# Runtime

A runtime is a system that implements the spec.

A runtime can manage multiple contexts.

# Context

A context is an isolated thread of story execution in the runtime. Each context manage its own states.

Contexts are created by the runtime with a story. Contexts are terminated when no more statements are to be executed, or when fatal errors occur.

# Story
A story refers to both a ZOH script file and a compiled ZOH story data structure at runtime.

Each story should be contained in a single `.zoh` file.

```
Story Name
===
```

Beginning the file is the story header which always starts with the name of the story.

Story name can contain whitespaces but not reserved characters.

# Metadata
Stories can have k-v metadata entries before `===`.
Each entry starts with a key, then a colon, then a value, and must be terminated with a semicolon.
Metadata, along with the story name, is available to the runtime since pre-processor phase, before the story body is read or parsed.
Metadata values can be of `boolean`, `integer`, `double`, `string`, `list`, `map` types.

```
Story Name
meta_key1: meta_value1;
meta_list: [meta_value2, meta_value3];
meta_map: {key1: value1, key2: value2};
===
```

# Namespace

Namespaces are used to group verbs, attributes. Each verb and attribute should be under a namespace. Namespace can be nested. Namespace MUST be explicitly prefixed if the symbol (\[namespace.\]name) is ambiguous, which results in `namespace_ambiguity` fatal diagnostic at runtime.

```
/std.converse "Hello";
/custom.converse "Hello";
/converse "Hello"; // INVALID

/c.d.c;
/a.b.c;
/d.c; // VALID
/b.c; // VALID
/c; // INVALID
```


# Variables

Variables are values of a certain type.
	
Verbs may require variables passed-in to be of certain types. Attributes may require values to be of certain types. 

Variable stroage is located in the runtime or individual context and variables are always accessible to verb drivers, regardless if references are passed in the verb call. Passing references only serves as a "pointer" to the variable.

Variables are stored in either of the 2 scopes, context and story:
- Context: variables shared through all stories in the current context and removed when the context is destroyed.
- Story: variables local to the story and removed when the context leaves the story.

That is, cross-context variable sharing can only be achieved through channels.

Variable names are case-insensitive, can NOT start with digits and can NOT contain whitespaces or reserved characters. Context variables are shadowed by story variables in case of name conflicts.

#### Variable Types
- `nothing`: The default type. Denoted as `?`.
- `"string"` or `'string'`
	- Strings can be multi-lined between quotes.
	- `'` can be used to enclose strings that contain `"`.
	- `"` can be used to enclose strings that contain `'`.
	- `"` or `'` can be escaped with `\"` or `\'`.
	- `\n` can be used to represent a newline.
	- `\` can be escaped with `\\`.
- ```
  """ or '''
  Multiline string literal
	Indented.
  Not Indented.
  """ or '''
  ```
	- Surrounding quote pairs must be matching in styles and positions.
	- Inspired by .NET, the same amount of spaces before the outer """ should be automatically removed from inner string lines.
- integer
	- Signed numbers
	- Always 64bits
	- Can be assigned to variables of type `double`.
- double
	- Signed numbers with a dot
	- Always 64bits
	- Can be assigned to variables of type `integer`, with the value rounded toward zero to the nearest integer.
- boolean
	- `true` or `false`.
	- Case insensitive.
- /verb
	- A verb call is an objectified verb invocation command, holding verb name, attributes, parameters.
- {map}
	- A map is a collection of string-keyed entries. Denoted as `{"key": value, *string: value...}`.
	- Keys must be strings; using a non-string value as a map index produces a fatal error.
	- Accept `"string"` or `*string_var` for keys in literals. In case of a reference, the resolved string value is used.
- \[list\]
	- A list is a collection of variables. Denoted as `[value1, value2, value3...]`.
- \<channel\>
	- A channel is a FIFO, concurrent safe, unbounded, global pipe managed by the runtime.
	- Denoted as `<channel_name>`, which servers as a pointer to the underlying data structure uniquely identified by "channel_name" in the channel-dedicated storage in the runtime.
	- `<channel>` points to one same underlying data structure for any executing contexts at the same time.
	- A channel can be closed. New channel can be created with the same name, but does not point to the old channel.
	- No white space is allowed between `<` and `>`.
- [\`expression\`](./expr.md)
	- An expression is a special construct that can be evaluated by `/evalulate` at runtime.
	- Denoted as `` `expr` ``.
- *reference
	- Denoted as `*variable_name` or `*variable_name[index1][index2]...` for nested access.
	- A reference value carries a path of zero or more indexes for nested collection navigation.
	- Each index navigates into a collection: integers for lists (0-based), strings for map keys.
	- **Implicit Resolution**: If an index evaluates to an `expression`, it is automatically evaluated (recursively) until a non-expression value is matched.
	- **Index Type Validation**: After resolution, indices must match collection type:
		- Lists: integer required (fatal `invalid_index_type` otherwise)
		- Maps: string required (fatal `invalid_index_type` otherwise)
	- Path resolution behavior depends on context:
		- In expressions: fatal if path element doesn't exist (operations can't work with nothing)
		- In `/get`: returns `?` if path element doesn't exist (query semantic)
		- In mutations (`/set`, `/drop`, `/capture`, etc.): fatal if intermediate path doesn't exist
	- Being one of the value types, references are used as pointers to variables in storage, users can not put references into variables with any verb like how one will with literal of other types.

#### Type-to-String Conversion

All values can be converted to strings. This specification ensures consistent cross-language implementations.

| Type | String Representation |
|------|----------------------|
| `nothing` | `"?"` |
| `boolean` | `"true"` or `"false"` (lowercase) |
| `integer` | Decimal digits with `-` prefix for negative, no leading zeros |
| `double` | Decimal notation, always includes `.` (e.g., `42.0` not `42`) |
| `string` | Identity (returns unchanged) |
| `list` | `"[v1, v2, ...]"` with recursive conversion |
| `map` | `"{"k1": v1, ...}"` with quoted keys |
| `channel` | `"<name>"` |
| `reference` | `"*name"` or `"*name[i]..."` |
| `expression` | `` "`...`" `` |
| `verb` | `"/name ...;"` |

**Double formatting**:
- Always include decimal point: `42.0` not `42` (distinguishes from integer)
- Minimum precision to uniquely identify value
- Scientific notation for `|value| >= 1e15` or `|value| < 1e-4`
- Special values: `"Infinity"`, `"-Infinity"`, `"NaN"`

**Collection formatting**:
- String values within collections are quoted
- Map key order is implementation-defined

## Attributes
Inspired by C#, attributes are reusable decorators for verb calls.

Attributes are denoted as `[name:value]`, or `[name]` if the value is not required. In the latter case, the value is a `nothing`. All types are allowed for attribute values.

Attribute names are case-insensitive, can not start with digits and can not contain whitespaces or reserved characters. Multiple attributes with the same name are allowed and are left to verb drivers to handle.

Attributes are DATA, and ALWAYS OPTIONAL. While looking similar to named parameters, attributes have 2 purposes:
- Common concept as parameters shared by arbitrary verbs
- Marking individual verb calls for [handlers](#handlers) to achieve metaprogramming

For example, Fading is a concept useful to many verbs that start or end something, such as showing images, playing musics, etc., but is not a necessary parameter. It's a valid attribute candidate. 

```
*Jack <- "char_jack";
/converse [by: *Jack] "Hello, world!"; :: The /converse driver can see a [By] attribute whose value is a reference named "Jack" therefore can resolve it to "char_jack".
*var [required]; :: this attribute does not have value
```

# Verb
Verbs are the foundational building blocks of the language, which are the instructions to the runtime to perform actions.

Verbs calls can contain attributes, named parameters, and unnamed parameters, and are always terminated with a `;`.

Named parameters are denoted as `name:value`, while unnamed parameters are denoted as `value`. Named and unnamed parameters can be freely interleaved—there is no required ordering.

Verbs always return a value, even if it's a `nothing`. This value is kept by the runner context and can be kept into a variable by calling `/capture "variable_name";`.

Verbs always return a list of diagnostics (if any) or `nothing`. The last diagnostics returned is kept by the runner context and can be returned by `/diagnose;`. This applies to `/diagnose` itself, therefore, the last diagnostics can be returned only once before overriden by `/diagnose`.

Verb behaviors are entirely determined by runtime implementation. If a runtime does not have a verb driver registered for a verb, simply nothing would happen and the playback continues. Authors should utilize required verbs metadata if certain verbs are required.

Verb names are case-insensitive, can NOT start with digits and can NOT contain whitespaces or reserved characters. Verbs can have aliases, which should be implemented as extra thin wrappers to the original verb.

Verbs may have syntactic sugar forms, but these only exist in the source code and should be translated to the standard form by pre-processors.

## Syntax

Verbs can be written in two forms:

- Standard form: `/verb [attribute1] [attribute2]... param1, param2...;`
	- Attributes must be delimited by spaces, this includes the space after the verb name.
	- Parameters must be delimited by commas. Trailing comma after the last parameter is NOT allowed.
- Block form: designed to improve readability for multi-line verb calls. A block form starts with a additional tailing `/` and ends with `/;`, attributes and parameters are delimited by spaces and newlines. Standard form verb parameters can still be used within block form. The best practice is to align the enclosing `/` and `/;` for best readability.
```
/verb/
	[attribute1] [attribute2]
	named_param1_name:named_param1_value
	param2 param3
	named_param2_name: named_param2_value :: this still works despite the space due to the semicolon hint the parser
	param5
/;

/capture "return_into_var";

/verb [attribute1] [attribute2] ... param1, param2...; -> *return_into_var; 
:: -> *return_into_var; is syntactic sugar of /capture "return_into_var";
```

## Standard Diagnostics
These are diagnostics that maybe returned by any verb.

- Fatal: `invalid_type`: Any supplied parameter is not of valid types.
- Fatal: `parameter_not_found`: Any required parameter is not provided.
- Error: `invalid_attribute`: Any supplied attribute is invalid. Should be returned by all verbs when a attribute is found and handled but the value is invalid.
- Fatal: `runtime_error`: Any undefined error/exception from runtime implenmentation.
- Info: `timeout`: The timeout was reached (operation cancelled per `timeout` parameter).

## Core Verbs
Core verbs are expected to be implemented by all runtimes, while [standard verbs](./std_verbs.md) are expected to play most stories without issues.

### Core.Set

The verb declares a variable or updates an existing variable.

#### Parameters
- `target`: the variable to set. Accept `*reference` or `*reference[path...]`. The reference name identifies the variable; the optional path navigates into nested collections.
- `value`: the value to set. Accept `any`. In case it's a reference, the value of the reference is used. Results in a fatal error in case the variable is already typed and the value is not of the same type.

#### Diagnostics
- Fatal: `invalid_index`: The path provided does not exist (for intermediate path elements).
- Fatal: `invalid_index_type`: The index provided is of the wrong type for the collection.
- Fatal: `invalid_value`: The value provided is not listed in the provided OneOf list.
- Fatal: `required`: The variable is not assigned and the value is not provided.
- Fatal: `type_mismatch`: The value type does not match the variable's declared type (via `[typed]` attribute).

#### Returns
A nothing.

#### Attributes
- **resolve**: Instead of storing the `/verb` or `` `expr` ``, store the value resolved from the `/verb` or `` `expr` `` parameter.
- **scope** (string: `context`/`story`): the scope to set the variable to. Requires a value of either `context` or `story`. The runtime should default to `story` too if the attribute not specified. Accept `"string"`.
- **required**: requires the variable to be assigned. Does not requires a value. If the variable is not already assigned and the value is not provided, `/set` verb validator should output an fatal error.
- **typed** (string: `string`/`integer`/`double`/`boolean`/`list`/`map`/`verb`/`channel`): set the type for the variable without needing a value or inferring from value. Accept `"string"`.
- **OneOf** (`[list]`/`*[list]`): Checks the variable to be one of the values in the list.

#### Examples

```
/set *var_name;
:: Declaration of a variable named "var_name" of type nothing, and the value is a nothing

/set [typed: "string"] *var_name;
:: Declaration of a variable named "var_name" of type string, and the value is still a nothing

/set *var_name, *value;
:: Declaration of a variable named "var_name" of type of value, and the value is the value

/set [required] *var_name;      :: Declaration with required attribute

/set *list[0], *value;          :: Update list element
/set *map["key"], *value;       :: Update map entry
/set *data["users"][0]["name"], "Alice"; :: Nested path assignment

/set *var_name, /verb;;         :: set the variable to the verb call
/set [resolve] *var_name, /verb;; :: set the variable to the return value of the verb call
```

#### Syntactic Sugar Forms
```
*var_name;                          :: /set *var_name;
*var_name [required];               :: /set [required] *var_name;
*var_name <- *value;                :: /set *var_name, *value;
*var_name [required] <- "string";   :: /set [required] *var_name, "string";
*map["key"] [required] <- "string"; :: /set [required] *map["key"], "string";
*list[0] [required] <- "string";    :: /set [required] *list[0], "string";
*data["users"][0]["name"] <- "Alice"; :: /set *data["users"][0]["name"], "Alice";
*var_name<-*value;                	:: Spaces around `<-` are optional
```

##### Notes

The nature of the language requires user awareness of the difference between `/set` and `/capture`. To take the return value of a verb, the user should use `/capture`. This requires extra attention when pairing with other verbs, especially the syntactic sugar forms.

```
*var <- `expr`;        :: works, *var now holds the expression
*var <- *`expr`;       :: works, *var now holds the value of the expression reference
*var <- /eval `expr`;; :: works, BUT *var now holds THE /EVAL call, not the evaluated value of the call
*var <- /`expr`;;      :: works, BUT *var now holds THE /EVAL call, not the evaluated value of the call

*var <- /"expr";;      :: works, BUT *var now holds THE /INTERPOLATE call, not a interpolated string
```

### Core.Get

The verb returns the value of a variable.

#### Parameters
- `target`: the variable to get. Accept `*reference` or `*reference[path...]`. The reference name identifies the variable; the optional path navigates into nested collections.

#### Returns
The value of the variable. Returns `?` if the variable does not exist or path navigation encounters a missing element.

#### Attributes
- `required`: outputs an error if returning a nothing.

#### Diagnostics
- Fatal: `invalid_index_type`: path navigation failed due to invalid index type.
- Error: `not_found`: returning `nothing` but the verb is marked `[required]`.

#### Examples
```
/get *var_name;
/get *list[0];
/get *map["key"];
/get *data["users"][0]["name"];
/get [required] *var_name;
```

#### Syntactic Sugar Forms
```
<- *var_name;                     :: /get *var_name;
<- *var_name [attribute];         :: /get [attribute] *var_name;
<- *list[*index];                 :: /get *list[*index];
<- *data["users"][0]["name"];     :: /get *data["users"][0]["name"];
<-*var_name;                      :: Spaces around `<-` are optional
```

### Core.Drop

A drop verb deletes a variable from story or context, or sets a nested element to `?`.

#### Parameters
- `target`: the variable to drop. Accept `*reference` or `*reference[path...]`. For root variables (no path), removes the variable entirely. For nested paths, sets the element to `?`.

#### Attributes
- **scope** (string: `context`/`story`): the scope to drop the variable from. Requires a value of either `context` or `story`. The runtime should default to `story` if the attribute not specified. Accept `"string"`.

#### Diagnostics
- Fatal: `invalid_index_type`: path navigation failed due to invalid index type.

#### Returns
A nothing.

#### Examples
```
/drop *var_name;                  :: Remove variable entirely
/drop *list[0];                   :: Set list[0] to ?
/drop *data["users"][0]["name"];  :: Set nested element to ?
```

### Core.Capture

A capture verb takes the return value of the last verb call in the context and stores it into a variable.

Internally should be implemented with a `/set`.

#### Parameters
- `target`: the variable to capture into. Accept `*reference` or `*reference[path...]`. The reference name identifies the variable; the optional path navigates into nested collections.

#### Diagnostics
- Fatal: `invalid_index_type`: path navigation failed due to invalid index type.

#### Returns
A nothing.

#### Examples
```
/capture *var_name;
/capture *list[*index];
/capture *data["result"]["value"];
```

#### Syntactic Sugar Forms
```
-> *var_name;
-> *list[*index];
-> [attribute] *map["key"];
-> *data["users"][0]["name"];
->*var_name; :: Space is not enforced
```

### Core.Diagnose

A diagnose verb returns the last list of diagnostics returned by the last verb call in the context.

If no diagnostics are returned, it returns a nothing.

#### Returns
A map of the following structure: `{"fatal": ["string"], "error": ["string"], "warning": ["string"], "info": ["string"]}`, or a nothing if no diagnostics are returned. Each type of diagnostics is a list of strings. Each type of diagnostics could be omitted if there are no diagnostics of that type.

#### Examples
```
/diagnose;
```

### Core.Try

A try verb executes a verb and downgrades any fatal diagnostics to error level, preventing context termination. The original fatal is converted to an error and available via `/diagnose`.

#### Parameters
- `verb`: The verb to execute. Accept `/verb` or `*/verb`.

#### Named Parameters
- `catch`: The verb to execute if `verb` produces any fatal diagnostic (now downgraded to error). Accept `/verb` or `*/verb`.

#### Attributes
- **suppress**: Suppress diagnostics from `/verb`. The context kept diagnostics will only contain diagnostics from the catch verb.

#### Returns
- If no fatal diagnostic: the return value of `verb`.
- If fatal diagnostic downgraded and `catch` provided: the return value of the catch verb.
- If fatal diagnostic downgraded and no catch provided: a `nothing`.

#### Diagnostics
The original fatal diagnostics are moved to the `error` list with their codes preserved. For example, if the inner verb returns `{fatal: ["invalid_type: ..."]}`, after `/try` the diagnostics become `{error: ["invalid_type: ..."]}`.

Diagnostics from the `cache` verb (if provided) should be appended. Therefore, if the `catch` verb emit fatal diagnostics, the context still get terminated.

#### Examples
```
:: Basic try - downgrades fatal to error, context continues
/try /risky_verb;;

:: Try with catch handler (runs if fatal occurred)
/try /risky_verb;, catch: /handle_error;;

:: Block form
/try/
    /parse *user_input, "integer";
    catch: /sequence/
        /warning "Invalid input, using default";
        /set "result", 0;
    /;
/;

:: Suppress diagnostics after handling
/try [suppress] /verb;, catch: /log_and_continue;;

:: Check diagnostics after try
/try /risky_verb;;
/diagnose; -> *diag;
/if *diag, /handle *diag;;  :: *diag contains downgraded error if fatal occurred
```

#### Behavior Notes

1. **Downgrade, not catch**: `/try` converts fatal diagnostics to error level. Since errors don't terminate the context, execution continues normally.
2. **Errors pass through**: Error, warning, and info diagnostics from the inner verb are NOT modified - only fatals are downgraded.
3. **Scope**: Only affects diagnostics from the immediate verb execution, not from nested contexts (forks/calls).
4. **Propagation**: If the catch verb itself produces a fatal diagnostic, that fatal terminates the context normally (not downgraded).
5. **Return Value**: When a fatal is downgraded without a handler, the return value from the failed verb (if any was produced before the fatal) is discarded and a nothing is returned.
6. **Diagnostics Preservation**: The downgraded diagnostics are available via `/diagnose` after `/try` completes (unless `[suppress]` is used). Original error codes are preserved.

### Core.Interpolate

An interpolate verb interpolates a string ONCE. Any value enclosed with un-escaped brackets `{}` is replaced with the value of the variable. Undefined variables are treated as `nothing` (stringifying to `?` or `fallback`), instead of raising a fatal error.

For exmaple, `"Hello, ${*name}!"` should be interpolated to "Hello, John!" if the runtime/context resolves `*name` to a string "John". (Without `/Interpolate`, It would otherwise requires `` /eval `"Hello, " + *name + "!` ``) 

The brackets can be escaped with `\{` and `\}`.

Interpolation should also support:
- Formatting: C# composite format string style `${var[,width][:formatString]}`.
- Unrolling: `${*var..."delim"}` to expand `*[list]` and `*{map}` into `{element}{delim}{element}...` where `element`s are the list elements or map k:v pairs.
- calling `/count` for `$#{*var}`.
- calling `/if` for `$?{*var1? *var2 : *var3}` with `var1` as the condition, `var2` as the true case value, and `var3` as the else case value.
- calling `/any` for `$?{*var1|*var2...|*varN}`.
- calling `/get` for `${*var1|*var2...|*varN}[#]` with the options as a list parameter and `#` as the index. For example: `${1|2|3}[*i]`. Optionally, the `#` can be prefixed with `!` for a modulo operation to wrap around the cases.
- calling `/roll` for `${*var1|*var2...|*varN}[%]` with the options as parameters.
- calling `/wroll` for `${*var1:<w>|*var2:<w>...|*varN:<w>}[%]` with the options as parameters and `w` as the weights.

#### Aliases
- `/itpl`

#### Named Parameters
- `fallback`: Optional. The string to use when a variable resolves to `nothing` (explicitly or undefined). Defaults to `?`.

#### Parameters
- `str`: the string to interpolate. Accept `any`. In case of `*reference`, the value of the reference is used. In case of types other than `string`, the parameter is simply formatted to string. `` `expr` `` is not evaluated.

#### Diagnostics
- Fatal: `invalid_syntax`: Malformed interpolation syntax (unclosed `${`, invalid escape sequence).

#### Returns
The interpolated string.

#### Examples
```
*name <- "John";
/"Hello ${*name}"; :: returns "Hello, John!"

/"Hello ${*missing}"; :: returns "Hello ?" (assuming *missing is undefined)
/interpolate "Hello ${*missing}", fallback: "Stranger"; :: returns "Hello Stranger"

*format <- "Hello, ${*format}!";
/interpolate *format; :: returns "Hello, Hello, {*format}!!"
```

#### Syntactic Sugar Forms
C# style string interpolation:
```
/"It's a me, ${*name}!";
/'Hello, ${*name}!';
```

### Core.Evaluate

An eval verb evaluates a expression ONCE and returns the result.

On top of arithmetic operation, it should support this special syntax:
- calling `/interpolate` ONCE for `$"string"` or `$*var`. For example: `$"Hello, ${*name}!"`.
- calling `/count` for `$#(*var)`.
- calling `/if` for `$?(*var1? *var2 : *var3)` with `var1` as the condition, `var2` as the true case value, and `var3` as the else case value.
- calling `/any` for `$?(*var1|*var2...|*varN)`.
- calling `/get` for `$(*var1|*var2...|*varN)[#]` with the options as a list parameter and `#` as the index. For example: `$(1|2|3)[*i]`. Optionally, the `#` can be prefixed with `!` for a modulo operation to wrap around the cases.
- calling `/roll` for `$(*var1|*var2...|*varN)[%]` with the options as parameters.
- calling `/wroll` for `$(*var1:<w>|*var2:<w>...|*varN:<w>)[%]` with the options as parameters and `w` as the weights.

Verb calls are not supported.

#### Aliases
- `eval`

#### Parameters
- `expr`: the expression to evaluate. Accept `` `expr` `` or `` *`expr` ``.

#### Diagnostics
- Fatal: `invalid_syntax`: Malformed expression syntax.
- Fatal: `invalid_type`: Type error during expression evaluation (e.g., adding string to integer).
- Fatal: `undefined_var`: Any variable used in the expression is undefined.

#### Returns
The result of the expression, type depends on the expression.

#### Examples
```
/eval `*var+1`;
/eval `(*int*10-1)/5.5`;
/eval `*str1+*str2`;
/eval `*list1+*list2`;
/eval `*verb1+*verb2`;
/eval `*bool1==*bool2`;
```
#### Syntactic Sugar Forms
```
/`*var+1`;                 :: /eval `*var+1`;
/`*var+1` [attr];     	   :: /eval [attr] `*var+1`;
/`*var+1` [attr1] [attr2]; :: /eval [attr1] [attr2] `*var+1`;
```

##### Examples
```
/`*var+1`;
<- *var[`*index+1`];; :: /get *var[`*index+1`];;
```

#### Interactions & Edge Cases

- **Operator Precedence**: `$"..."` and `$*...` are Primary Expressions (highest precedence).
  - `1 + $*num` is equivalent to `1 + (*num evaluated as template string and interpolated)`.
- **Interpolation Semantics**: 
  - `$"string"`: The string literal is interpolated.
  - `$*var`: The value of `*var` is resolved, treated as a string template, and interpolated.
    - Example: If `*t <- "Hello ${*n}"` and `*n <- "World"`, then `$*t` -> `"Hello World"`.
- **String Escaping**: `$"..."` follows standard ZOH string escaping.
  - `$"Value: \"\${*var}\""` -> `Value: "ContentsOfVar"`
  - `$"Literal: \\{*var}"` -> `Literal: {*var}`

### Store.Write

A write verb write or update a variable's type and value to persistent storage. The persisted variable should be available across runtimes.

Persistence storage is global, concurrent safe and accessible across all runtimes/contexts/stories.

#### Aliases
- Save

#### Named Parameters
- `store`: a specific "container" for persistence time scoping. Defaults to `?`, which indicates the verb to use the default global container.

#### Parameters
- `*var...`: Variables to write. Accept repeating references of `nothing`, `integer`, `double`, `boolean`, `string`, `list`, `map`.

#### Diagnostics
- Fatal: `invalid_type`: A value cannot be persisted (e.g., verb, channel, expression).
- Fatal: `runtime_error`: Persistence layer failure (I/O error, storage unavailable).

#### Returns
A nothing.

#### Examples
```
/write *score;                        :: Single variable
/write *score, *name, *inventory;     :: Multiple variables
/write store:"save1", *score, *name;  :: With store name
/write/
    *score
    *name
    *inventory
/;                                    :: Block form
```

### Store.Read

A read verb retrieves a value from persistent storage and `/set` it to the provided variable. If no value with the same name is found in the persistent storage, the default value is returned.

Persistence storage is global, concurrent safe and accessible across all runtimes/contexts/stories.

#### Aliases
- Load

#### Named Parameters
- `store`: a specific "container" for persistence time scoping. Defaults to `?`, which indicates the verb to use the default container.
- `default`: the default value to return if the variable is not found. Accept `integer`, `double`, `boolean`, `string`, `list`, `map` or `?`. Defaults to `?`.

#### Parameters
- `*var...`: Variables to read. Accept repeating references.

#### Attributes
- `required`: Output error if the variable is not found.
- `scope` (string: `context`/`story`): the scope to set the variable to. Requires a value of either `context` or `story`. The runtime should default to `story` if the attribute not specified. Accept `"string"`.

#### Diagnostics
- Info: `not_found`: A variable key does not exist in storage (returns default value).
- Fatal: `runtime_error`: Persistence layer failure (I/O error, storage unavailable).

#### Returns
A nothing.

#### Examples
```
/read *score;                         :: Single variable
/read *score, *name, *inventory;      :: Multiple variables
/read default:0, *score, *level;      :: With default value
/read store:"save1", *score;          :: From named store
/read [scope:"context"] *score;       :: Read to context scope
```

### Store.Erase

An erase verb erases a variable from persistent storage.

#### Named Parameters
- `store`: a specific "container" for persistence time scoping. Defaults to `?`, which indicates the verb to use the default container.

#### Parameters
- `*var`: the variable to erase. Accept references.

#### Diagnostics
- Info: `not_found`: The variable key does not exist in storage.
- Fatal: `runtime_error`: Persistence layer failure (I/O error, storage unavailable).

#### Returns
A nothing.

#### Examples
```
/erase *var;
```

### Store.Purge

Purges all variables from persistent storage.

#### Parameters
- `store`: a specific "container" for persistence time scoping. Defaults to `?`, which indicates the verb to use the default container.

#### Diagnostics
- Fatal: `runtime_error`: Persistence layer failure (I/O error, storage unavailable).

#### Returns
A nothing.

#### Examples
```
/purge;
```

### Core.Type

A type verb returns the type of a variable.

#### Parameters
- `var`: the variable to type. Accept references.

#### Diagnostics
- Fatal: `invalid_index_type`: Path navigation failed due to invalid index type.

#### Returns
String. `string`/`integer`/`double`/`boolean`/`list`/`map`/`verb`/`channel`/`expression`/`nothing`/`unknown`.

#### Examples
```
/type *list; :: returns "list"
/type *map;  :: returns "map"
/type *str;  :: returns "string"
```

### Core.Count

A count verb returns the size of a variable.

#### Parameters
- `target`: the variable to count. Accept `*reference[path...]`. The reference navigates to the value to count. The value must be `string`, `list`, `map`, `channel`, or `nothing`.

#### Diagnostics
- Fatal: `invalid_type`: the value to count is not of type `string`/`list`/`map`/`channel`/`nothing`.
- Fatal: `invalid_index`: Path element missing.
- Fatal: `invalid_index_type`: Index is wrong type for collection.

#### Returns
The integer size of the variable. For `nothing`, returns 0.

#### Examples
```
/count *list; -> *size;
/count *map; -> *size;
/count *map["key"]; -> *size;
/count *data["users"][0]["items"]; -> *size;
```

### Core.Do
A do verb executes a verb referenced by the parameter, or a verb returned by the parameter.

#### Parameters
- `verb`: the verb to execute. Accept `*/verb` or verb-returning `/verb`.

#### Diagnostics
- Fatal: `invalid_type`: The parameter is not a verb value.

#### Returns
The return value of the verb executed.

#### Examples
```
/do *verb_var;;
/do /verb_returning;; :: This execute the verb returned by /verb_returning
```

### Core.Sequence

A sequence verb executes verbs in order.

#### Named Parameters
- `breakif`: Optional. The condition to break the loop or execute next verb. Accept `*boolean`, `/verb`/`*verb` or `` `expr` ``/`` *`expr` ``. In case of reference, the , the value is used. In case of `` `expr` ``, it is evaluated. In case of `/verb`, the returned value is used.

#### Parameters
- Repeating:
	- `/verb`: The verbs to execute. Accept `/verb`, `*/verb` or `*[list]` of verbs. In case of `*[list]`, the list is iterated over and each element is executed if it is a verb.

#### Diagnostics
- Fatal: `invalid_type`: The breakif condition is not a boolean or nothing after evaluation.

#### Returns
The return value of the last verb executed.

#### Examples
```
/sequence /verb1;, /verb2;, /verb3;;
/sequence/
	/verb1;
	/verb2;
	/verb3;
/;

/if *condition, /sequence
	/verb1;,
	/verb2;,
	/verb3;
;;

/if/ *condition,
	/sequence/
		/verb1;
		/verb2;
		/verb3;
	/;
/;
```

### Core.If
A if verb conditionally executes a verb.

#### Named Parameters
- `is`: Value to be compared to the subject. Optional. Default to `true`. Accept `any`. In case of reference, the value is used. In case of `` `expr` ``, it is evaluated.
- `else`: the verb to execute if the condition is false. Accept `/verb` or `*/verb`.

#### Parameters
- `subject`: Accept `any`. In case of reference, the value is used. In case of `` `expr` ``, it is evaluated. In case of `/verb`, the returned value is used.
- `verb`: the verb to execute if the condition is true. Accept `/verb` or `*/verb`.

#### Diagnostics
- Fatal: `invalid_type`: The evaluated condition is not a boolean or nothing.

#### Returns
A nothing.

#### Examples
```
/if *cond, /verb;;
/if *char, is: "someone", /verb;, else: /verb;;
/if/
	*cond
	is: false
	/verb;
/;
/if /bool_returning;, /verb;;
/if `*var==1`, /verb;;
```

### Core.Loop

A loop verb repeatedly executes a verb a specified number of times.

#### Named Parameters
- `breakif`: Optional. The condition to break the loop or start next iteration. Accept `*boolean`, `/verb`/`*verb` or `` `expr` ``/`` *`expr` ``. In case of reference, the , the value is used. In case of `` `expr` ``, it is evaluated. In case of `/verb`, the returned value is used.

#### Parameters
- `times`: The number of times to loop. Accept `int` or `*int`, in case of the latter, the runtime copies the value. In case of `-1`, the loop runs indefinitely. For dynamic loops, use [/while](#while) instead.
- `/verb`: The verb to execute.

#### Diagnostics
- Fatal: `invalid_type`: The count parameter is not an integer.

#### Returns
A nothing.

```
/loop *times, /verb;;
/loop 5, /verb;;
```

### Core.While

A while verb repeats a verb as long as the condition is met.

#### Named Parameters
- `is`: Value to be compared to the subject. Optional. Default to `true`. Accept `any`. In case of reference, the value is used. In case of `` `expr` ``, it is evaluated.

#### Parameters
- `subject`: Accept `any`. In case of reference, the value is used. In case of `` `expr` ``, it is evaluated. In case of `/verb`, the returned value is used.
- `verb`: the verb to execute if the condition is true. Accept `/verb` or `*/verb`.

#### Diagnostics
- Fatal: `invalid_type`: The evaluated condition is not a boolean or nothing.

#### Returns
A nothing.

#### Examples
```
/while *condition, /verb;;
/while `*a>0`, /verb;;
/while /bool_returning;, /verb;;
```

### Core.Foreach

A foreach verb iterate over a list or map and set each element to the provided reference, then execute the verb.

#### Named Parameters
- `breakif`: Optional. The condition to break the loop or start next iteration. Accept `*boolean`, `/verb`/`*verb` or `` `expr` ``/`` *`expr` ``. In case of reference, the , the value is used. In case of `` `expr` ``, it is evaluated. In case of `/verb`, the returned value is used.

#### Parameters
- `subject`: The list to iterate over. Accept `[list]`,`*[list]`, `{map}` or `*{map}`.
- `it`: The reference to set each element to. Accept references. In case the subject is a map, it's set to a map with exactly one key-value pair, where the key is the map key and the value is the map value. If the variable is already assigned in the **story scope**, it will be dropped from the **story scope** first to avoid type conflicts.
- `/verb`: The verb to execute for each element. Accept `/verb`. 

#### Diagnostics
- Fatal: `invalid_type`: The subject is not a list or map.

#### Returns
A nothing.

#### Examples
```
/foreach *list, *it, /verb *it;;
```

### Core.Switch

A switch verb conditionally returns the value of the first matching condition case.

#### Parameters
- `subject`: the subject to compare. Accept `/verb`, `` `expr` `` or any reference. In case of reference, the value will be used. In case of `/verb`, it takes the return value of the verb. In case of `` `expr` ``, it evaluates the expression.
- Repeating pairs of:
    - `case`: the case to compare. Accept `any`. In case of reference, the value will be used. In case of `/verb`, it takes the return value of the verb. In case of `` `expr` ``, it evaluates the expression.
    - `value`: the value to return if the case matches. Accept `any`. In case of reference, it takes the value of the reference.
- `default`: Optional. The value to return if no case matches. Accept `any`. In case of reference, it takes the value of the reference.

#### Diagnostics
- Fatal: `invalid_type`: Subject or case expression/verb evaluation resulted in an error.

#### Returns
The value of the case, or the default value if no case matches, or a nothing if no default is provided.

#### Examples
```
/switch *subject,
case,
	*case_value,
case,
	*case_value,
case,
	*case_value,
*default_value
;

/switch/ *subject
	case
		*case_value
	case
		*case_value
	case
		*case_value
	*default_value
/;
```


### Core.Has

A has verb checks if a variable is in a list or is it a key in a map.

#### Parameters
- `collection`: The collection to check. Accept `[list]`/`*[list]` or `{map}`/`*{map}`.
- `subject`: The variable to check. Accept `any`. In case of reference, the value is used. If the collection is a list, the subject is checked for equality with each element. If the collection is a map, the subject is checked for equality with each key.

#### Diagnostics
- Fatal: `invalid_type`: The collection is not a list or map.
- Fatal: `invalid_index_type`: Path navigation failed due to invalid index type.

#### Returns
A boolean.

#### Examples
```
/has *list, *subject;
/has *map, *subject;
```

### Core.Any

An any verb returns `true` if a `/get` against the provided variable and index does not return a nothing.

#### Parameters
- `var`: The variable to check. Accept `any`. If it is a reference, the value is used.
- `index`: The index to check. Accept `?`, `integer`/`*integer` for [list] or `"string"`/`*"string"` for {map}. Optional, defaults to `?`. In case of `?`, the runtime checks the variable itself.

#### Diagnostics
- Fatal: `invalid_index_type`: Index is wrong type for collection (e.g., string index on list).

#### Returns
A boolean.

#### Examples
```
/any *list, *subject;
/any *map, *subject;
```

### Core.First

A First verb returns the first value that `/any` returns `true` for.

#### Parameters
- Repeating:
    - `case`: the case to check. Accept `any` EXCEPT `nothing`. In case of reference, the value will be used. In case of `/verb`, it takes the return value of the verb. In case of `` `expr` ``, it evaluates the expression.

#### Returns
The first value that `/any` returns `true` for. If no case matches, returns a `nothing`.

#### Examples
```
/first /verb1;, /verb2;;
```

### Core.Jump
A jump verb instruct the context to jump to a [checkpoint](#checkpoints) in a story.

In case of jumping to other stories, story-scoped variables are dropped while context-scoped variables are preserved. In order to *transfer* story-scoped variables, use the `var` parameters.

**Validation**: If the target checkpoint defines a contract (required variables), the runtime validates that these variables exist and are not `nothing` in the context (after applying any `var` parameters). If validation fails, a fatal error is raised.

#### Parameters
- `story`: the story to jump to. Accept `"string"`/`*"string"`, or `?`.
- `checkpoint`: the checkpoint to jump to. Accept `"string"`/`*"string"`.
- Repeating:
    - `var`: references to story-scoped variables to *transfer*. Accept references. The name of the references are used by the `/set` operation.

#### Diagnostics
- Fatal: `invalid_type`: Story or checkpoint parameter is not a string (or nothing for story).
- Fatal: `invalid_story`: The story to jump to does not exist.
- Fatal: `invalid_checkpoint`: The checkpoint to jump to does not exist.
- Fatal: `checkpoint_violation`: A required variable for the checkpoint is missing or `nothing`.

#### Returns
A nothing.

#### Examples
```
/jump "story", "checkpoint", *var1, *var2;
/jump/
	?
	"checkpoint"
	*var1
	*var2
/;
```

#### Syntactic Sugar Forms
```
====> @story:checkpoint *var1 *var2;
====> @checkpoint *var1 *var2;       :: omitting story

====> [attr] @checkpoint;

====> [attr1] [attr2]
@story:checkpoint
*var1
*var2
;
```

### Core.Fork
A fork is similar to jump but starts a new context to execute from the destination in parallel.

To `/set` variables in the newly forked context, use the `var` parameters.

**Validation**: Validates the target checkpoint's contract against the *newly forked context's state*.

#### Parameters
- `story`: the story to jump to. Accept `"string"`/`*"string"`, or `?`.
- `checkpoint`: the checkpoint to jump to. Accept `"string"`/`*"string"`.
- Repeating:
    - `var`: references to variables that will be used to `/set` in the forked context before execution. Scope of the variables are preserved. The name of the references are used by the `/set` operation.

#### Diagnostics
- Fatal: `invalid_type`: Story or checkpoint parameter is not a string (or nothing for story).
- Fatal: `invalid_story`: The story to jump to does not exist.
- Fatal: `invalid_checkpoint`: The checkpoint to jump to does not exist.
- Fatal: `checkpoint_violation`: A required variable for the checkpoint is missing or `nothing`.

#### Attributes
- `clone`: if the forked context should be a clone of the current context. A cloned context have all the variables and flags of the current context.

#### Returns
A nothing.

#### Examples
```
/fork "story", "checkpoint", *var1, *var2;
```

#### Syntactic Sugar Forms
```
====+ @story:checkpoint *var1 *var2;
====+ @checkpoint clone:true;
====+ [clone] @checkpoint;

====+
@story:checkpoint
*var1
*var2
;

====+ [attr1] [attr2]
@story:checkpoint
*var1
*var2
;

====+
[attr]
@checkpoint
;
```

### Core.Call

A call verb is similar to fork but it blocks the current context until the forked context is terminated (as in no more instructions to execute).

The last return value in the forked context before termination is returned by the call verb.

**Validation**: Validates the target checkpoint's contract against the *newly forked context's state*.

#### Parameters
- `story`: the story to jump to. Accept `"string"`/`*"string"`, or `?`.
- `checkpoint`: the checkpoint to jump to. Accept `"string"`/`*"string"`.
- Repeating:
    - `var`: references to variables that will be used to `/set` in the forked context before execution. Accept references. Scope of the variables are preserved. When the call is marked with `inline` attribute, the context will use the values in the forked context to `/set` in the current context after the fork is terminated. If the forked context terminates with an error, the driver should forfeit the `/set` operation.

#### Diagnostics
- Fatal: `invalid_type`: Story or checkpoint parameter is not a string (or nothing for story).
- Fatal: `invalid_story`: The story to jump to does not exist.
- Fatal: `invalid_checkpoint`: The checkpoint to jump to does not exist.
- Fatal: `checkpoint_violation`: A required variable for the checkpoint is missing or `nothing`.

#### Attributes
- `inline`: if the context should copy the value of the specified `var`s in the forked context back when it's terminated.
- `clone`: if the forked context should be a clone of the current context. A cloned context have all the variables and flags of the current context.

#### Returns
The last return value in the forked context before termination.

#### Examples
```
/call "story", "checkpoint", *var1, *var2;
/call
story:"story",
checkpoint:"checkpoint",
;
```

#### Syntactic Sugar Forms
```
<===+ @story:checkpoint *var1 *var2;
<===+ @checkpoint;
<===+ [attr] @checkpoint;

<===+
[attr]
@checkpoint
;

<===+
@story:checkpoint
*var1
*var2
;
<===+ [attr1] [attr2]
@story:checkpoint
*var1
*var2
;
```

#### Core.Flag
A flag verb set named parameters for the context that is visible to all verb drivers.

Context flags are copied to forks.

#### Parameters
- `name`: the name of the flag. Accept `"string"` or `*"string"`.
- `value`: the value of the flag. Accept `any`. In case of reference, it takes the value of the reference.

#### Returns
A nothing.

#### Examples
```
/flag "flag_name", value;
/flag [attr] "flag_name", *value;
```

#### Syntactic Sugar Forms
```
#flag flag_name value;
#flag [attr] flag_name value;
```

### Sleep
A sleep verb simply blocks the context execution for a specified amount of seconds.

#### Parameters
- `seconds`: The number of seconds to sleep. Accept `double`/`*double`, double-returning `/verb`/`*verb` or `` `expr` ``/`` *`expr` ``. In case of reference, the value is used. In case of `` `expr` ``, it is evaluated. In case of `/verb`, the returned value is used.

#### Returns
A nothing.

```
/sleep 0.1;
/sleep /rand 0.1, 1;
```

### Core.Wait

A wait verb blocks until the runtime receives a message with matching name.

A runtime may get external or internal messages that propagate to all contexts.

#### Named Parameters
- `timeout`: The timeout in seconds. Accept `double` or `*double`.

#### Parameters
- `name`: The name of the message to wait for. Accept `"string"` or `*"string"`.

#### Returns
The message received. Could be `integer`, `double`, `boolean`, `string`, `list`, `map`. If the timeout is reached, returns a nothing.

#### Examples
```
/wait "message_name";
/wait "message_name", timeout: 0.1;
```
### Signal
A signal verb broadcasts a message to all contexts.

#### Parameters
- `name`: The name of the message to send. Accept `"string"` or `*"string"`.
- `msg`: The message to send. Accept `integer`, `double`, `boolean`, `string`, `list`, `map`.

#### Returns
A nothing.

#### Examples
```
signal "message_name", msg;
```

### Channel.Push

A push verb pushes a variable to a channel. Creates a new channel if it does not exist or closed.

#### Parameters
- `channel`: The channel to push to. Accept `<channel>` or `*<channel>`.
- `var`: The variable to push. Accept `any`. If it is a reference, the runtime takes the value.

#### Returns
A nothing.

#### Examples
```
/push *channel, *var;
/push <channel>, *var;
```

### Channel.Pull

A pull verb takes the first variable from a channel, or wait until a variable is available.

#### Named Parameters
- `timeout`: The timeout in seconds. Accept `double` or `*double`. Optional. Default to `?`.

#### Parameters
- `channel`: The channel to pull from. Accept `<channel>` or `*<channel>`.

#### Returns
The variable pulled from the channel.

#### Diagnostics
- Error: `not_found`: The channel does not exist.
- Error: `closed`: The channel is closed.
- Info: `timeout`: The timeout was reached.

#### Examples
```
/pull *channel;
/pull <channel>, timeout: 0.1;
```

### Channel.Close

A close verb closes a channel.

#### Parameters
- `channel`: The channel to close. Accept `<channel>` or `*<channel>`.

#### Diagnostics
- Error: `not_found`: The channel does not exist.
- Error: `closed`: The channel is already closed.

#### Returns
A nothing.

#### Examples
```
/close *channel;
/close <channel>;
```

### Core.Append

An append verb appends a value to a list, or a key-value pair to a map.

#### Parameters
- `collection`: The collection to append to. Accept `*[list]` or `*{map}`.
- `var`: The value to add. Accept any type. If it is a reference, the runtime takes the value. For maps, the value should be a key-value pair.

#### Diagnostics
- Fatal: `invalid_type`: The collection is not a list or map.
- Fatal: `invalid_index_type`: Path navigation failed due to invalid index type.
- Error: `key_conflict`: The key already exists in the map.

#### Returns
The size of the list after the addition. Integer.

#### Examples
```
/append *list, *var;
/append *map, {"key":value};
```

### Core.Remove

A remove verb removes a value from a list or map by index or key.

No effect if the index is out of bounds or the key does not exist.

#### Parameters
- `collection`: Accept `*[list]` or `*{map}`.
- `index`: The index to remove. Accept `integer`/`*integer` for list or `"string"`/`*"string"` for map. Accept negative index to remove from the end.

#### Diagnostics
- Fatal: `invalid_type`: The collection is not a list or map.
- Fatal: `invalid_index_type`: Path navigation failed due to invalid index type.

#### Returns
The size of the list or map after the removal. Integer.

#### Examples
```
/remove *list, *index;
/remove *map, *key;
```

### Core.Insert

An insert verb inserts a value into a list.

#### Parameters
- `list`: Accept `*[list]`.
- `index`: The index to insert. Accept `integer`/`*integer`. Accept negative index to insert from the end.
- `value`: The value to insert. Accept any type. If it is a reference, the runtime takes the value. 

#### Diagnostics
- Fatal: `invalid_type`: The collection is not a list.
- Fatal: `invalid_index_type`: Path navigation failed due to invalid index type.
- Error: `invalid_index`: The index is out of bounds.

#### Returns
The size of the list after the insertion. Integer. Defaults to 0.

#### Examples
```
/insert *list, *index, *var;
```

### Core.Clear

A clear verb clears a list or map.

#### Parameters
- `collection`: Accept `*[list]` or `*{map}`.

#### Diagnostics
- Fatal: `invalid_type`: The collection is not a list or map.
- Fatal: `invalid_index_type`: Path navigation failed due to invalid index type.

#### Returns
A nothing.

#### Examples
```
/clear *list;
/clear *map;
```

### Core.Exit

An exit verb stop the context from continuing the current story.

#### Parameters
A `any` value that will be returned. Useful in context management.

#### Returns
A nothing.

#### Examples
```
/exit;
/exit 1;
```

### Core.Increase

An increase verb increases a variable by a value.

#### Parameters
- `target`: The variable to increase. Accept `*reference[path...]` pointing to an `integer` or `double`.
- `value`: The value to increase by. Accept `integer`/`*integer`, `double`/`*double`, `/verb`/`*verb` or `` `expr` ``/`` *`expr` ``. In case of reference, the value is used. In case of `/verb`, the returned value is used. In case of `` `expr` ``, it's evaluated. Optional, defaults to `1` or `1.0`.

#### Diagnostics
- Fatal: `invalid_type`: The target value is not numeric (integer or double).
- Fatal: `invalid_index`: Path element missing.
- Fatal: `invalid_index_type`: Index is wrong type for collection.

#### Returns
The value of the variable after the increase. Integer or double.

#### Examples
```
/increase *var;
/increase *var, *value;
/increase *var, `expr`;
/increase *list[0];
/increase *map["key"], 5;
/increase *data["stats"]["score"], 10;
```

### Core.Decrease

A decrease verb decreases a variable by a value.

#### Parameters
- `target`: The variable to decrease. Accept `*reference[path...]` pointing to an `integer` or `double`.
- `value`: The value to decrease by. Accept `integer`/`*integer`, `double`/`*double`, `/verb`/`*verb` or `` `expr` ``/`` *`expr` ``. In case of reference, the value is used. In case of `/verb`, the returned value is used. In case of `` `expr` ``, it's evaluated. Optional, defaults to `1` or `1.0`.

#### Diagnostics
- Fatal: `invalid_type`: The target value is not numeric (integer or double).
- Fatal: `invalid_index`: Path element missing.
- Fatal: `invalid_index_type`: Index is wrong type for collection.

#### Returns
The value of the variable after the decrease. Integer or double.

#### Examples
```
/decrease *var;
/decrease *var, *value;
/decrease *var, `expr`;
/decrease *list[0];
/decrease *map["key"], 5;
/decrease *data["stats"]["score"], 10;
```

### Debug Verbs

Debug verbs make the runtime emit a diagnostic message.

#### Parameters
- `message`: The message to emit. Accept `"string"`, `*"string"`, `` `expr` ``, or `` *`expr` ``. In case of reference, the value is used. In case of `` `expr` ``, the expression is evaluated.

#### Returns
A nothing.

#### Examples
```
/info `"Hello, world! {*user}!"`;
/info `"Hello, world! " + *user + "!"`;
/warning "Hello, world!";
/error "Hello, world!";
/fatal "Hello, world!";
```

### Core.Roll

A roll verb returns a random value.

#### Parameters
- Repeating:
	- `value`: The value to return. Accept `any`. In case the value is a reference, the runtime takes the value.

#### Diagnostics
- Fatal: `parameter_not_found`: No options provided.

#### Returns
Random one from the values.

#### Examples
```
/roll *v1, *v2, *v3;
```

### Core.WRoll

A Wroll verb returns one of the values randomly, with weights.

#### Parameters
- Repeating:
	- `value`: The value to return. Accept any type. In case the value is a reference, the runtime takes the value.
	- `weight`: The weight of the value. Accept `integer` or `*integer`.

#### Diagnostics
- Fatal: `parameter_not_found`: No options provided.
- Fatal: `invalid_type`: A weight is not a number.
- Fatal: `invalid_value`: A weight is negative.

#### Returns
Random one from the values.

#### Examples
```
/wroll *v1, 1, *v2, 2, *v3, 3;
/wroll
*v1, 1,
*v2, 2,
*v3, 3;
```

### Core.Rand

A rand verb returns a random value between min and max.

#### Named Parameters
- `inclmax`: Whether to include max in the range. Accept `boolean`/`*boolean`. Optional, defaults to `false`.

#### Parameters
- `min`: The minimum value. Accept `integer`/`*integer` or `double`/`*double`.
- `max`: The maximum value. Accept `integer`/`*integer` or `double`/`*double`.

#### Diagnostics
- Fatal: `invalid_type`: Min or max is not a number.
- Fatal: `invalid_value`: Min is greater than max.

#### Returns
A random value between min and max. Depends on the type of min and max.

#### Examples
```
/rand 1, 10;
/rand 1.0, 10.0;
```

### Core.Parse

A parse verb returns a value parsed from a string.

#### Parameters
- `value`: The value to parse. Accept `string`/`*string`.
- `type`: The type to parse to. Optional. Accept `"integer"`, `"double"`, `"boolean"`, `"list"`, `"map"`.

#### Diagnostics
- Fatal: `invalid_type`: The value cannot be parsed into the specified type, or the type can not be inferred.

#### Returns
The parsed value. Defaults to a nothing.

#### Examples
```
/parse "1";
/parse "1", "integer";
/parse "1.0", "double";
/parse "true", "boolean";
/parse "[]", "list";
/parse "{}", "map";
```

### Core.Defer

A defer verb defers the execution of a verb. Deferred verbs are executed when the context is leaving thes story if the scope attribute is set to `story`, or when the context is being terminated if the scope attribute is set to `context`.

Deferred verbs are executed in LIFO order.

#### Parameters
- `verb`: The verb to defer. Accept `verb`/`*verb`.

#### Attributes
- **scope**: the scope of the defer. Accept `"string"` or `*"string"`. Valid values are `context` or `story`.

#### Diagnostics
- Fatal: `invalid_type`: The parameter is not a verb value.

#### Returns
A nothing.

#### Examples
```
/defer /verb;
```

## Embed

The language should support embedding of other script files.

Denoted as `#embed` in their own lines, embeds are one of the rare exceptions in the language that are non-verbs that are first-class citizens as verbs.

The implementation should simply replace the embed syntax with the content of the designated file with a standard pre-processor.

The path must be absoulute or relative to the current file.

If the file is not found, the runtime should emit a compile error.

During each embed resolution, each file can only be embedded once.

```
#embed "relative/path/to/file.zoh";
```

## Macro

The language supports story body templating with macro.

Denoted as `#macro` and `#expand` in their own lines, macros are one of the rare exceptions in the language that are non-verbs that are first-class citizens as verbs.

A macro definition starts `#macro MACRO_NAME param1, param2...;`, then any number of lines, finally terminated with a line of `#macro;`.

The implementation should replace all `|#placeholder#|` with provided named string with matching name, in a standard pre-processor.

Default body for placeholders can be defined by additional `|` in the following format: `|#placeholder|default_body#|`.

`|` inside macro body can be escaped with `\|`. `\` can be escaped with `\\`.

Macro is local, expanding macros defined in other stories will not work.

Embedding preprocessors should run before macro expansion.


```
#macro MACRO_NAME pname1, pname2, pname3;
/converse "Here we have |#pname1|Jack#|, |#pname2|John#| and |#pname3|Alice#|!";
#macro;

#expand MACRO_NAME;
#expand MACRO_NAME p_name1:p_value1, p_name2:p_value2...;
#expand MACRO_NAME |#p_name1|p_value1#| |#p_name2|p_value2#|;

#macro SYSTEM_OUTPUT placeholder;
/converse [By: *System] |#placeholder#|
#macro;

#expand SYSTEM_OUTPUT placeholder:"Hello, world!";
:: becomes
/converse [By: *System] "Hello, world!";

```

## Checkpoint

A checkpoint, denoted as `@checkpoint`, is a named node in a stoty, that enables `/jump`, `/fork` and `/call`.

Checkpoints must be in its own line and must be denoted at root level.

Checkpoint names are case-insensitive, can not caontain whitespace or reserved characters.

Checkpoint names must be unique within each story.

A checkpoint can be suffixed with variable references. All referenced variables must not be resolved to `nothing` when the context is about to execute across or jump/fork/call to the checkpoint.

### Examples
```
@checkpoint *var1 *var2 *var3
```

# Runtime Design

Runtime should be layered in the following manner: Runtime, Context, Story.

A runtime may have multiple contexts while each context may only be at a certain position of a story.

## Variable Resolution

The runtime should lookup for variables in story scope first then context scope.

In case of name conflicts, context variables are shadowed by story variables.

It is recommended to be verbose and explicit when naming variables.

## Syntactic Sugar

Verbs may have syntactic sugar forms, but these only exist in the source code and should be translated to the standard form by pre-processors.

## Handlers

A runtime should be designed in a modular fashion that it can be extended with the following:

### Pre-processor
Pre-processors are called before the runtime parser and can take action against the raw story data.
They are provided with the story name, the metadata entries and the story body text, and can temper them in anyway.
Pre-processors can invalidate the story by returning a fatal diagnostic.

#### Standard Preprocessors
- TODO

### Compiler
Compilers are called to verify and convert the story to runtime data structures.
Compilers work by repeatedly asking if any registered compiler is interested in the current node. An interested compiler may read ahead and perform its logics to generate runtime data structures or modify already generated ones, then the next compiler is asked, and so on, until no compiler is interested in the current node. The runtime should move ahead and repeat the process until the end of the story. Noted that once the runtime moves on, all compiled elements up to the point should be finalized.
In other words, because compilers are called in order of priority, compilers with lower priority can modify results from compilers with higher priority.
Compiled elements should track their source line number and story body line number in the story, the difference is story body line number is based on actual parsed story body.

#### Standard Compilers
- TODO

### Story Validator
Story validators are called before running the story. Any story validator can output diagnostics and prevent the story from running.

### Verb Validator
Verb validators are called by a standard story validator to verify the compiled verb calls are valid after compilers are done.

### Verb Driver
Verb drivers are called by the runtime/context to perform the verb call at runtime.

## Handler Priority

Handlers should be registered with priority value so the runtime can call them in order.

For verb drivers, the runtime only calls the one with hightest priority, if multiple verb drivers map to the same verb name.

The highest priorty is -2^31 and the lowest is 2^31-1.

## Error Handling

Handlers can return diagnostics to the runtime. Diagnostics are leveled in severity of "info", "warning", "error" and "fatal". Fatal diagnostics immediately terminate the context, while other diagnostics should be logged and reported.

Contexts should keep a list of diagnostics, which may be used by the runtime, or other contexts in case it is a forked context.

## State Management
Handlers should be stateless and concurrent safe. State should only exists in runtime and context.

## Asset Management
Asset paths should be abstracted away and assets should be identified by "address". Addresses can be file paths but not limited to.

## Channel Management

Noted that closing a channel should instantly notify all pullers. No puller should be mistakenly pull from a new channel with the same name.