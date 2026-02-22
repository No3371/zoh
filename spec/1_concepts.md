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
    - Denoted as `<channel_name>`, which serves as a pointer to the underlying channel hub uniquely identified by "channel_name" in the channel-dedicated storage in the runtime.
    - No white space is allowed between `<` and `>`.
    - `<channel>` points to one same logical channel for any executing contexts at the same time. Each context maintains its own outbox (push buffer) and inbox (pull buffer), coordinated by the channel hub. Contexts are auto-registered with the hub on first `/push` or `/pull`.
    - A channel can be closed. New channel can be created with the same name, but does not point to the old channel. Internally, channels have `generation` to distinguish channels with same names.
- [\`expression\`](./expr.md)
    - An expression is a special construct that can be evaluated by `/evalulate` at runtime.
    - Denoted as `` `expr` ``.
- *reference
    - Denoted as `*variable_name` or `*variable_name[index1][index2]...` for nested access.
    - A reference value carries a path of zero or more indexes for nested collection navigation.
    - Each index navigates into a collection: integers for lists (0-based, negative index supported), strings for map keys.
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

The language supports story body templating with macros.

- Macros are defined using the pipe-delimited syntax `|%NAME%|...|%NAME%|`.
- `|%MACRO_NAME%|` LINEs to open and close a definition. MACRO_NAME can not contain whitespaces or newlines, spaces on the left and right are ignored.
- `|%|` for positional parameter
- `|%1|` for mirroring 2nd parameter in the definition
- `|%+2|` for mirroring the second parameter found after this parameter
- `|%-2|` for mirroring the second parameter found before this parameter

### Definition
```zoh
|%MACRO_NAME%|
<body with placeholders>
|%MACRO_NAME%|

|%NAME_CHAR%|
/save *pc_name, "|%|";
/converse "Your name is |%0|;
/converse "Your name is |%-2|;
|%NAME_CHAR%|

|%NAME_CHAR|tommy123|%|
```

### Expansion

- `|%MACRO_NAME` starts a expansion. `|%|` closes an expansion.
- `|%MACRO_NAME|%|` for expanding the macro without replacement parameters. MACRO_NAME can not contain whitespaces or newlines, spaces on the left and right are ignored.
- `|%MACRO_NAME|PARAM1|PARAM2|%|`
- Macro replacement are optional with available positional parameters. `|%|` without matching parameters at matching index simply is replaced with an empty string:
- Macros can be denoted in multilines as long as long as each line starts with `|PARAM`.

```zoh
|%MACRO_NAME|%|
|%MACRO_NAME|arg0|arg1|...|%|

|% NOTIFY %|
/notify |%|, |%|;
|%NOTIFY%|

|%NOTIFY|
|tommy123
|"You got a mail!"
|%|
```


### Trimming
Arguments in macro expansion are trimmed symmetrically. The preprocessor calculates the minimum of leading and trailing whitespace, and removes that amount from both ends. This allows formatted macro calls without injecting unwanted whitespace.

### Escaping
- `\|` escapes the pipe `|` character.
- `\%` escapes the percent `%` character (e.g. to write literal `%|`).

### Indentation
The macro expansion preserves the indentation (continuous white spaces from line start) of the usage line. All lines in the expanded body are indented by the same amount of whitespace found before the `|%` token of the call.

```
    :: Indented macro call
    |%GENERATE_LOGIC|%|
```

Expands to:

```
    :: Indented lines
    /if *condition,
        /do_something;
    ;
```

## Checkpoint

A checkpoint, denoted as `@checkpoint`, is a named node in a stoty, that enables `/jump`, `/fork` and `/call`.

Checkpoints must be in its own line and must be denoted at root level.

Checkpoint names are case-insensitive, can not caontain whitespace or reserved characters.

Checkpoint names must be unique within each story.

A checkpoint can be suffixed with variable references. These references define the contract for the checkpoint.
The syntax for contract variables is `*var` or `*var:type`.
- `*var`: The variable must exist and not be `nothing`.
- `*var:type`: The variable must exist, not be `nothing`, and match the specified type.
Supported types are all types except `nothing` and `reference`: `string`, `integer`, `double`, `boolean`, `list`, `map`, `channel`, `verb`, `expression`.

Validation occurs when a context jumps, calls, or forks to the checkpoint, OR when execution naturally reaches it.
- If a required variable is missing or `nothing`, a fatal `checkpoint_violation` is raised.
- If a typed variable does not match the specified type, a fatal `checkpoint_violation` is raised.

### Examples
```
@checkpoint *var1 *var2:integer *var3:string
```