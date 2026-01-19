# ZOH

**ZOH** is a narrative scripting language designed for interactive storytelling. Whether you're building visual novels, branching dialogues, text adventures, or cinematic sequences—ZOH provides a clean, expressive syntax that lets writers focus on the story while giving developers the power they need.

## Why ZOH?

Narrative scripting languages often have original design that provides very different development experience from general programming languages. While they usually packs powerful features specialized for presenting text-based stories, 

**For Writers:**
- Write dialogue naturally with minimal syntax noise
- Focus on story flow, not implementation details
- Preview and test stories without a full game build

**For Developers:**
- First-class concurrency for parallel storylines
- Extensible verb system—add custom commands without forking
- Clean separation between story logic and presentation

## Core Philosophy

1. **Verbs, not functions.** Every action is a *verb* that the runtime performs. This keeps the language close to natural storytelling: characters *converse*, scenes *show*, music *plays*.

2. **Stories are isolated.** Variables live in scopes. Contexts run independently. When stories fork, they don't accidentally break each other.

3. **Presentation is separate.** ZOH doesn't dictate how your game looks. The same script can power a text console, a 2D visual novel, or a 3D cinematic—just swap the verb drivers.

4. **Extend, don't modify.** Need a custom verb? Register a driver. Need preprocessing? Add a compiler pass. The runtime is built for plugins.

## At a Glance

```zoh
Murder Mystery
author: Jane Doe;
===

@intro
/converse [By: "Detective"] "The body was found at midnight.";

/choose "What do you do?",
    "Examine the body", true, "examine",
    "Question the witness", true, "witness"
; -> *choice;

/if *choice, is: "examine", /jump ?, "examine_body";;
/if *choice, is: "witness", /jump ?, "question_witness";;

@examine_body
/converse "You notice a strange marking on the victim's hand...";
```

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
/verb p_name:p_v;             :: Named parameters
/verb p1, p2, p3;             :: Unnamed parameters
/verb p_name:p_v, p1, p2, p3; :: Named before unnamed
/verb [attribute1] [attribute_with_value:attr_value] [even_more_attr] param_name1:param_value1, param_name2:param_value2..., unnamed_param1, unnamed_param2...;
/capture "return_into_var";
-> *return_into_var;          :: Syntactic sugar of /capture "return_into_var";
/verb; -> *return_into_var;
/verb;                        :: Return value is not required.
   /verb;                     :: Verbs don't have to start at the beginning of a line

/verb [attr1] [attr2];        :: Spaces between attributes are required
/verb p1,p2,p3;               :: Spaces between parameters are optional
/verb;->*return_into_var;     :: Spaces around -> are optional

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

Namespaces are used to group verbs, attributes. Each verb and attribute should be under a namespace. Namespace can be nested. Namespace can be omitted if the verb or attribute name is unique.

```
/core.set "var";
/std.converse "Hello";
```

# Variables

Variables are values of a certain type.

Verbs may require variables passed-in to be of certain types. Attributes may require values to be of certain types. 

Variable stroage is located in the runtime or individual context and variables are always accessible to verb drivers, regardless if references are passed in the verb call. Passing references only serves as a "pointer" to the variable.

Variables in storage are either not-yet-typed or typed. When typed, the runtime throws error and aborts `/set` operations if the value to be set is not of the same type.

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
	- A map is a collection of string-variable pairs. Denoted as `{"key": value, *string: value...}`. Accept `"string"` or `*"string"` for keys. In case of a reference, the value is used.
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
	- Denoted as `*variable_name`.
	- A reference value optionally carries an index for collection variables.
	- Being one of the value types, references are used as pointers to variables in storage, users can not put references into variables with any verb like how one will with literal of other types.

## Attributes
Inspired by C#, attributes are reusable components that can be marked on verb calls.

Attributes are denoted as `[name:value]`, or `[name]` if the value is not required. In the latter case, the value is a `Nothing`. All types are allowed for attribute values. Attribute names are case-insensitive, can not start with digits and can not contain whitespaces or reserved characters, and allows up to one `.` if namespace is used.

Attributes are ALWAYS OPTIONAL and common parameters shared by indefinite verbs that map to common concepts that could be useful in various scenarios.

Attributes are DATA. Multiple attributes with the same name are allowed and are left to verb drivers to handle.

For example, Fading is a concept useful to various verbs, but are not a necessary parameter for such verbs. It suits well as an attribute.

```
*Jack <- "char_jack";
/converse [by: *Jack] "Hello, world!"; :: The /converse driver can see a [By] attribute whose value is a reference named "Jack" therefore can resolve it to "char_jack".
*var [required]; :: this attribute does not have value
```

Attributes are data. It's entirely up to implementation to perform action according to attribute it encounters.

# Verb
Verbs are the foundational building blocks of the language, which are the instructions to the runtime to perform actions.

Verbs calls can contain attributes, named parameters, and unnamed parameters, and are always terminated with a `;`.

Named parameters are denoted as `name:value`, while unnamed parameters are denoted as `value`. Named parameters can appear before or after unnamed parameters, but not in between.

Verbs always return a value, even if it's a `Nothing`. This value is kept by the runner context and can be kept into a variable by calling `/capture "variable_name";`.

Verbs always output a list of diagnostics or nothing. The last diagnostics returned is kept by the runner context and can be checked by calling `/diagnose;`.

Verb behaviors are entirely determined by runtime implementation. If a runtime does not have a verb driver registered for a verb, simply nothing would happen and the playback continues. Authors should utilize required verbs metadata if certain verbs are required.

Verb names are case-insensitive, can NOT start with digits and can NOT contain whitespaces or reserved characters, and allows up to one `.` if namespace is used. If both `a.talk` and `b.talk` is registered, only writing `/talk` is not allowed. Verbs can have aliases, which should be implemented as extra thin wrappers to the original verb.

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

## Core Verbs
Core verbs are expected to be implemented by all runtimes, while [standard verbs](./std.md) are expected to play most stories without issues.

All core verbs should be under `std.` namespace.

### Core.Set

The verb declares a variable or updates an existing variable.

#### Parameters
- `var_name`: the name of the variable to set. Accept `"string"` or `*"string"`.
- `index`: the index of the variable to set. Optional. If the index is `?` or not provided, the variable itself is declared or updated, otherwise, the variable must be a declared list or map. Accept `"string"`, `*"string"`, `integer` or `*integer`.
- `value`: the value to set. Accept `any`. In case it's a reference, the value of the reference is used. Throws error in case the variable is already typed and the value is not of the same type.

#### Returns
A Nothing.

#### Attributes
- **resolve**: Instead of storing the `/verb` or `` `expr` ``, store the value resolved from the `/verb` or `` `expr` `` parameter.
- **scope** (string: `context`/`story`): the scope to set the variable to. Requires a value of either `context` or `story`. The runtime should default to `story` too if the attribute not specified. Accept `"string"`.
- **required**: requires the variable to be assigned. Does not requires a value. If the variable is not assigned and the value is not provided, `/set` verb validator should output an fatal error.
- **typed** (string: `string`/`integer`/`double`/`boolean`/`list`/`map`/`verb`/`channel`): set the type for the variable without needing a value or inferring from value. Accept `"string"`.
- **OneOf** (`[list]`/`*[list]`): Checks the variable to be one of the values in the list. Accept `*[list]`.

#### Examples

```
/set "var_name";                 
:: Declaration of a variable named "var_name" of type Nothing, and the value is a Nothing

/set [typed: "string"] "var_name";                 
:: Declaration of a variable named "var_name" of type string, and the value is still a Nothing

/set "var_name", *value;         
:: Declaration of a variable named "var_name" of type of value, and the value is the value

/set "var_name", ?, *value;      :: Equal to the above

/set [required] "var_name";      :: Declaration with required attribute

/set *var_var_name *var_value;      
:: Declaration of a variable of type of var_value, named with the string value of var_var_name, and the value is the value of var_value

/set "var_name", *index, *value; :: Update list or map

/set "var_name", /verb;;         :: set the variable to the verb call
/set [resolve] "var_name", /verb;; :: set the varialbe to the return value of the verb call
```

#### Syntactic Sugar Forms
```
*var_name;                          :: /set "var_name";
*var_name [required];               :: /set [required] "var_name";
*var_name <- *value;                :: /set "var_name", *value;
*var_name[?] <- *value;             :: /set "var_name", ?, *value;
*var_name [required] <- "string";   :: /set [required] "var_name", "string";
*map["key"] [required] <- "string"; :: /set [required] "map", "key", "string";
*list[0] [required] <- "string";    :: /set [required] "list", 0, "string";
*var_name<-*value;                	:: Spaces around `<-` are optional
```

##### Notes

The nature of the language requires user awareness of the difference between `/set` and `/capture`. To take the return value of a verb, the user should use `/capture`. This requires extra attention when pairing with other verbs, especially the syntactic sugar forms.

```
*var <- `expr`;        :: works, *var now holds the expression
*var <- *`expr`;       :: works, *var now holds the value of the expression reference
*var <- /eval `expr`;; :: works, BUT *var now holds THE /EVAL call, not the evaluated value of the call
*var <- $`expr`;;      :: works, BUT *var now holds THE /EVAL call, not the evaluated value of the call

*var <- $"expr";;      :: works, BUT *var now holds THE /INTERPOLATE call, not a interpolated string
```

### Core.Get

The verb returns the value of a variable.

#### Parameters
- `var_name`: the name of the variable to get. Accept `"string"` or `*"string"`.
- `index`: the index of the variable to get. Optional. Accept `?`, `integer`/`*integer`, `string`/`*string`, or `` `expr` ``/ `` *`expr` ``. Defaults to `?` if not provided. In case of reference, the value is used. In case of `?`, the variable itself is returned. In case of `` `expr` ``, the expression is evaluated and the result is used.

#### Returns
The value of the variable. Or a Nothing if the variable is not found by the index. Returns a Nothing if the variable does not exist.

#### Attributes
- `required`: outputs an error if returning a Nothing.

#### Examples
```
/get "var_name";
/get "var_name", *index;
/get "var_name", `expr`;
/get [required] "var_name", *index;
/get [required] "var_name";
```

#### Syntactic Sugar Forms
```
<- *var_name;                     :: /get "var_name";
<- *var_name [attribute];         :: /get "var_name";
<- *var_name[*index];             :: /get "var_name", *index;
<- *var_name[`expr`];             :: /get "var_name", `expr`;
<-*var_name;                      :: Spaces around `<-` are optional
```

### Core.Drop

A drop verb delete a variable from story or context.

#### Parameters
- `var_name`: the name of the variable to drop. Accept `"string"` or `*"string"`.

#### Returns
A Nothing.

#### Examples
```
drop "var_name";
```

### Core.Capture

A capture verb takes the return value of the last verb call in the context and stores it into a variable.

Internally should be implemented with a `/set`.

#### Parameters
- `var_name`: the name of the variable to set. Accept `"string"` or `*"string"`.
- `index`: the index of the variable to set. Optional. If the index is `?` or not provided, the variable itself is declared or updated, otherwise, the variable must be a declared list or map. Accept `"string"`, `*"string"`, `integer` or `*integer`.

#### Returns
A Nothing.

#### Examples
```
/capture "var_name";
/capture "a_list", *index;
/capture "a_map", *key;
```

#### Syntactic Sugar Forms
```
-> *var_name;
-> *a_list[*index];
-> [attribute] *a_map[*key];
```

### Core.Diagnose

A diagnose verb returns the last list of diagnostics returned by the last verb call in the context.

If no diagnostics are returned, it returns a Nothing.

#### Returns
A list of diagnostics or a Nothing.

#### Examples
```
/diagnose;
```

### Core.Interpolate

An interpolate verb interpolates a string ONCE. Any value enclosed with un-escaped brackets `{}` is replaced with the value of the variable.

#### Aliases
- `/itpl`

For exmaple, `"Hello, ${*name}!"` should be interpolated to "Hello, John!" if the runtime/context resolves `*name` to a string "John". (Without `/Interpolate`, It would otherwise requires `` /eval `"Hello, " + *name + "!` ``) 

The brackets can be escaped with `\{` and `\}`.

Interpolation should also support:
- Formatting: C# composite format string style `${var[,width][:formatString]}`.
- Unrolling: `{*var..."delim"}` to expand `*[list]` and `*{map}` into `{element}{delim}{element}...` where `element`s are the list elements or map k:v pairs.
- calling `/count` for `$#{*var}`.
- calling `/if` for `$?{*var1? *var2 : *var3}` with `var1` as the condition, `var2` as the true case value, and `var3` as the else case value.
- calling `/any` for `$?{*var1|*var2...|*varN}`.
- calling `/get` for `${*var1|*var2...|*varN}[#]` with the options as a list parameter and `#` as the index. For example: `${1|2|3}[*i]`. Optionally, the `#` can be prefixed with `!` for a modulo operation to wrap around the cases.
- calling `/roll` for `${*var1|*var2...|*varN}[%]` with the options as parameters.
- calling `/wroll` for `${*var1:<w>|*var2:<w>...|*varN:<w>}[%]` with the options as parameters and `w` as the weights.

#### Parameters
- `str`: the string to interpolate. Accept `any`. In case of `*reference`, the value of the reference is used. In case of types other than `string`, the parameter is simply formatted to string. `` `expr` `` is not evaluated.

#### Returns
The interpolated string.

#### Examples
```
*name <- "John";
/interpolate "Hello, ${*name}!"; :: returns "Hello, John!"

*format <- "Hello, ${*format}!";
/interpolate *format; :: returns "Hello, Hello, {*format}!!"
```

#### Syntactic Sugar Forms
C# style string interpolation:
```
$"It's a me, ${*name}!";
$'Hello, ${*name}!';
```

### Core.Evaluate

An eval verb evaluates a expression ONCE and returns the result.

On top of arithmetic operation, it should support this special syntax:
- calling `/interpolate` for `$(*var)`. For example: `$("Hello, {*name}!")`.
- calling `/count` for `$#(*var)`.
- calling `/if` for `$?(*var1? *var2 : *var3)` with `var1` as the condition, `var2` as the true case value, and `var3` as the else case value.
- calling `/any` for `$?(*var1|*var2...|*varN)`.
- calling `/get` for `$(*var1|*var2...|*varN)[#]` with the options as a list parameter and `#` as the index. For example: `$(1|2|3)[*i]`. Optionally, the `#` can be prefixed with `!` for a modulo operation to wrap around the cases.
- calling `/roll` for `$(*var1|*var2...|*varN)[%]` with the options as parameters.
- calling `/wroll` for `$(*var1:<w>|*var2:<w>...|*varN:<w>)[%]` with the options as parameters and `w` as the weights.

Verb calls are not supported.

In case of undefined variables, the expression should throw an error.

#### Aliases
- `eval`

#### Parameters
- `expr`: the expression to evaluate. Accept `` `expr` `` or `` *`expr` ``.

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
$`*var+1`;
$`*var+1` [attribute];
$`*var+1` [attr1] [attr2];
```

##### Examples
```
$`*var+1`;
-> *var+1;     :: /capture the returned value
<- *var[$`*index+1`];; :: /get "var", `*var+1`;;
```

### Store.Write

A write verb write or update a variable's type and value to persistent storage. The persisted variable should be available across runtimes.

Persistence storage is global, concurrent safe and accessible across all runtimes/contexts/stories.

#### Aliases
- Save

#### Named Parameters
- `store`: a specific "container" for persistence time scoping. Defaults to `?`, which indicates the verb to use the default global container.

#### Parameters
- `*var`: the variable to write. Accept references of `integer`, `double`, `boolean`, `string`, `list`, `map`.

#### Returns
A Nothing.

#### Examples
```
/write *var;
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
- `*var`: the variable to read. Accept references. Throws error in case the variable is already typed and the value is not of the same type.

#### Attributes
- `required`: Output error if the variable is not found.

#### Returns
A Nothing.

#### Examples
```
/read *var;
/read [default: 0] *var;
```

### Store.Erase

An erase verb erases a variable from persistent storage.

#### Named Parameters
- `store`: a specific "container" for persistence time scoping. Defaults to `?`, which indicates the verb to use the default container.

#### Parameters
- `*var`: the variable to erase. Accept references.

#### Returns
A Nothing.

#### Examples
```
/erase *var;
```

### Store.Purge

Purges all variables from persistent storage.

#### Parameters
- `store`: a specific "container" for persistence time scoping. Defaults to `?`, which indicates the verb to use the default container.

#### Returns
A Nothing.

#### Examples
```
/purge;
```

### Core.Type

A type verb returns the type of a variable.

#### Parameters
- `var`: the variable to type. Accept references. 

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
- `var`: the variable to count. Accept `"string"`/`*"string"`, `[list]`/`*[list]`, `{map}`/`*{map}`, or `<channel>`/`*<channel>`. In case of reference, the value is used. In case of `list` or `map`, the size of the list or map is returned. In case of `string`, the length of the string is returned. Throws error in case of other types. In case of `<channel>`/`*<channel>`, the number of elements in the channel is returned.

#### Returns
The integer size of the variable.

#### Examples
```
/count *list; -> *size;
/count *map; -> *size;
/count *str; -> *length;
```

### Core.Do
A do verb executes a verb referenced by the parameter, or a verb returned by the parameter.

#### Parameters
- `verb`: the verb to execute. Accept `*/verb` or verb-returning `/verb`.

#### Returns
The return value of the verb executed.

#### Examples
```
/do *verb_var;;
/do /verb_returning;; :: This execute the verb returned by /verb_returning
```

### Sequence

A sequence verb executes verbs in order.

#### Named Parameters
- `breakif`: Optional. The condition to break the loop or execute next verb. Accept `*boolean`, `/verb`/`*verb` or `` `expr` ``/`` *`expr` ``. In case of reference, the , the value is used. In case of `` `expr` ``, it is evaluated. In case of `/verb`, the returned value is used.

#### Parameters
- Repeating:
	- `/verb`: The verbs to execute. Accept `/verb`, `*/verb` or `*[list]` of verbs. In case of `*[list]`, the list is iterated over and each element is executed if it is a verb.

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

#### Returns
A Nothing.

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
- `times`: The number of times to loop. Accept `int` or `*int`, in case of the latter, the runtime copies the value. In case of dynamic loop, use [/while](#while) instead.
- `/verb`: The verb to execute.

#### Returns
A Nothing.

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

#### Returns
A Nothing.

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
- `it`: The reference to set each element to. Accept references. In case the subject is a map, it's set to a map with exactly one key-value pair, where the key is the map key and the value is the map value. If the reference is already assigned, it will be dropped first to avoid type conflicts.
- `/verb`: The verb to execute for each element. Accept `/verb`. 

#### Returns
A Nothing.

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

#### Returns
The value of the case, or the default value if no case matches, or Nothing if no default is provided.

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

#### Returns
A boolean.

#### Examples
```
/has *list, *subject;
/has *map, *subject;
```

### Core.Any

An any verb returns `true` if a `/get` against the provided variable and index does not return a Nothing.

#### Parameters
- `var`: The variable to check. Accept `any`. If it is a reference, the value is used.
- `index`: The index to check. Accept `?`, `integer`/`*integer` for [list] or `"string"`/`*"string"` for {map}. Optional, defaults to `?`. In case of `?`, the runtime checks the variable itself.

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
    - `case`: the case to check. Accept `any` EXCEPT `Nothing`. In case of reference, the value will be used. In case of `/verb`, it takes the return value of the verb. In case of `` `expr` ``, it evaluates the expression.

#### Returns
The first value that `/any` returns `true` for. If no case matches, returns a `Nothing`.

#### Examples
```
/first /verb1;, /verb2;;
```

### Core.Jump
A jump verb instruct the context to jump to a [label](#label) in a story.

In case of jumping to other stories, story-scoped variables are dropped while context-scoped variables are preserved. In order to *transfer* story-scoped variables, use the `var` parameters.

Throws fatal if the story or label does not exist.

#### Parameters
- `story`: the story to jump to. Accept `"string"`/`*"string"`, or `?`.
- `label`: the label to jump to. Accept `"string"`/`*"string"`.
- Repeating:
    - `var`: references to story-scoped variables to *transfer*. Accept references. The name of the references are used by the `/set` operation.

#### Returns
A Nothing.

#### Examples
```
/jump "story", "label", *var1, *var2;
/jump/
	?
	"label"
	*var1
	*var2
/;
```

#### Syntactic Sugar Forms
```
====> @story:label *var1 *var2;
====> @label *var1 *var2;       :: omitting story

====>
/story
label
*var1
*var2
;

====> [attr] @label;

====> [attr1] [attr2]
@story:label
*var1
*var2
;
```

### Core.Fork
A fork is similar to jump but starts a new context to execute from the destination in parallel.

To `/set` variables in the newly forked context, use the `var` parameters.

Throws fatal if the story or label does not exist.

#### Parameters
- `story`: the story to jump to. Accept `"string"`/`*"string"`, or `?`.
- `label`: the label to jump to. Accept `"string"`/`*"string"`.
- Repeating:
    - `var`: references to variables that will be used to `/set` in the forked context before execution. Scope of the variables are preserved. The name of the references are used by the `/set` operation.

#### Attributes
- `clone`: if the forked context should be a clone of the current context. A cloned context have all the variables and flags of the current context.

#### Returns
A Nothing.

#### Examples
```
/fork "story", "label", *var1, *var2;
```

#### Syntactic Sugar Forms
```
====+ @story:label *var1 *var2;
====+ @label clone:true;
====+ [clone] @label;

====+
@story:label
*var1
*var2
;

====+ [attr1] [attr2]
@story:label
*var1
*var2
;

====+
[attr]
@label
;
```

### Core.Call

A call verb is similar to fork but it blocks the current context until the forked context is terminated (as in no more instructions to execute).

The last return value in the forked context before termination is returned by the call verb.

Throws fatal if the story or label does not exist.

#### Parameters
- `story`: the story to jump to. Accept `"string"`/`*"string"`, or `?`.
- `label`: the label to jump to. Accept `"string"`/`*"string"`.
- Repeating:
    - `var`: references to variables that will be used to `/set` in the forked context before execution. Accept references. Scope of the variables are preserved. When the call is marked with `inline` attribute, the context will use the values in the forked context to `/set` in the current context after the fork is terminated. If the forked context terminates with an error, the driver should forfeit the `/set` operation.

#### Attributes
- `inline`: if the context should copy the value of the specified `var`s in the forked context back when it's terminated.
- `clone`: if the forked context should be a clone of the current context. A cloned context have all the variables and flags of the current context.

#### Returns
The last return value in the forked context before termination.

#### Examples
```
/call "story", "label", *var1, *var2;
/call
story:"story",
label:"label",
;
```

#### Syntactic Sugar Forms
```
<===+ @story:label *var1 *var2;
<===+ @label;
<===+ [attr] @label;

<===+
[attr]
@label
;

<===+
@story:label
*var1
*var2
;
<===+ [attr1] [attr2]
@story:label
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
A Nothing.

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
- `seconds`: The number of seconds to sleep. Accept `double` or `*double`.

#### Returns
A Nothing.

```
/sleep 0.1;
```

### Core.Wait

A wait verb blocks until the runtime receives a message with matching name.

A runtime may get external or internal messages that propagate to all contexts.

#### Named Parameters
- `timeout`: The timeout in seconds. Accept `double` or `*double`.

#### Parameters
- `name`: The name of the message to wait for. Accept `"string"` or `*"string"`.

#### Returns
The message received. Could be `integer`, `double`, `boolean`, `string`, `list`, `map`. If the timeout is reached, returns Nothing.

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
A Nothing.

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
A Nothing.

#### Examples
```
/push *channel, *var;
/push <channel>, *var;
```

### Channel.Pull

A pull verb takes the first variable from a channel, or wait until a variable is available.

Throws an error if the channel does not exist or is closed.

#### Named Parameters
- `timeout`: The timeout in seconds. Accept `double` or `*double`. Optional. Default to `?`.

#### Parameters
- `channel`: The channel to pull from. Accept `<channel>` or `*<channel>`.

#### Returns
The variable pulled from the channel.

#### Examples
```
/pull *channel;
/pull <channel>, timeout: 0.1;
```

### Channel.Close

A close verb closes a channel. Throws an error if the channel does not exist or is already closed.

#### Parameters
- `channel`: The channel to close. Accept `<channel>` or `*<channel>`.

#### Returns
A Nothing.

#### Examples
```
/close *channel;
/close <channel>;
```

### Core.Append

An append verb appends a value to a list, or a key-value pair to a map.

Throws an error if the collection in case of key conflict for maps.

#### Parameters
- `collection`: The collection to append to. Accept `*[list]` or `*{map}`.
- `var`: The value to add. Accept any type. If it is a reference, the runtime takes the value. For maps, the value should be a key-value pair.

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
- `index`: The index to insert. Accept `integer`/`*integer`. Accept negative index to insert from the end. Throws an error if the index is out of bounds.
- `value`: The value to insert. Accept any type. If it is a reference, the runtime takes the value. 

#### Returns
The size of the list after the insertion. Integer.

#### Examples
```
/insert *list, *index, *var;
```

### Core.Clear

A clear verb clears a list or map.

#### Parameters
- `collection`: Accept `*[list]` or `*{map}`.

#### Returns
A Nothing.

#### Examples
```
/clear *list;
/clear *map;
```

### Core.Exit

An exit verb stop the context from continuing the current story.

#### Parameters
None.

#### Returns
A Nothing.

#### Examples
```
/exit;
```

### Core.Increase

An increase verb increases a variable by a value.

#### Parameters
- `var`: The variable to increase. Accept `*integer` or `*double*`
- `index`: the index of the variable to set. Optional. If the index is `?` or not provided, the variable itself is declared or updated, otherwise, the variable must be a declared list or map. Accept `"string"`, `*"string"`, `integer` or `*integer`.
- `value`: The value to increase by. Accept `integer`/`*integer`, `double`/`*double`, `/verb`/`*verb` or `` `expr` ``/`` *`expr` ``. In case of reference, the value is used. In case of `/verb`, the returned value is used. In case of `` `expr` ``, it's evaluated. Optional defaults to `1` or `1.0`.

#### Returns
The value of the variable after the increase. Integer or double.

#### Examples
```
/increase *var;
/increase *var, *value;
/increase *var, `expr`;
/increase *var, /verb;
/increase *list, 0, *value;
/increase *map, "key", *value;
```

### Core.Decrease

A decrease verb decreases a variable by a value.

#### Parameters
- `var`: The variable to decrease. Accept `*integer` or `*double*`
- `index`: the index of the variable to set. Optional. If the index is `?` or not provided, the variable itself is declared or updated, otherwise, the variable must be a declared list or map. Accept `"string"`, `*"string"`, `integer` or `*integer`.
- `value`: The value to decrease by. Accept `integer`/`*integer`, `double`/`*double`, `/verb`/`*verb` or `` `expr` ``/`` *`expr` ``. In case of reference, the value is used. In case of `/verb`, the returned value is used. In case of `` `expr` ``, it's evaluated. Optional, defaults to `1` or `1.0`.

#### Returns
The value of the variable after the decrease. Integer or double.

#### Examples
```
/decrease *var;
/decrease *var, *value;
/decrease *var, `expr`;
/decrease *var, /verb;
/decrease *list, 0, *value;
/decrease *map, "key", *value;
```

### Debug Verbs

Debug verbs make the runtime emit a diagnostic message.

#### Parameters
- `message`: The message to emit. Accept `"string"`, `*"string"`, `` `expr` ``, or `` *`expr` ``. In case of reference, the value is used. In case of `` `expr` ``, the expression is evaluated.

#### Returns
A Nothing.

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

#### Returns
A random value between min and max. Depends on the type of min and max.

#### Examples
```
/rand 1, 10;
/rand 1.0, 10.0;
```

### Core.Parse

A parse verb returns a value parsed from a string.

Throws an error if the value cannot be parsed into the specified type or the type can not be inferred.

#### Parameters
- `value`: The value to parse. Accept `string`/`*string`.
- `type`: The type to parse to. Optional. Accept `"integer"`, `"double"`, `"boolean"`, `"list"`, `"map"`.

#### Returns
The parsed value. Defaults to a Nothing.

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

#### Returns
A Nothing.

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

#macro SYSTEM_OUTPUT placeholder;
/converse [By: *System] |#placeholder#|
#macro;

#expand SYSTEM_OUTPUT placeholder:"Hello, world!";
:: becomes
/converse [By: *System] "Hello, world!";

```

## Label

A label, denoted as `@label`, is a named node in a story, for `/jump` and `/fork` verbs to work.

Labels must be in its own line and can only be denoted at root level.

Label names are case-insensitive, can not contain whitespace or reserved characters.

Duplicate labels are not allowed and the runtime should emit a compile error.

### Examples
```
@label
/verb; :: the label points to this statement.
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

#### Pre-processor
Pre-processors are called before the runtime parser and can take action against the raw story data.
They are provided with the story name, the metadata entries and the story body text, and can temper them in anyway.
Pre-processors can invalidate the story by returning a fatal diagnostic.

#### Compiler
Compilers are called to verify and convert the story to runtime data structures.
Compilers work by repeatedly asking if any registered compiler is interested in the current node. An interested compiler may read ahead and perform its logics to generate runtime data structures or modify already generated ones, then the next compiler is asked, and so on, until no compiler is interested in the current node. The runtime should move ahead and repeat the process until the end of the story. Noted that once the runtime moves on, all compiled elements up to the point should be finalized.
In other words, because compilers are called in order of priority, compilers with lower priority can modify results from compilers with higher priority.
Compiled elements should track their source line number and story body line number in the story, the difference is story body line number is based on actual parsed story body.

#### Story Validator
Story validators are called before running the story. Any story validator can output diagnostics and prevent the story from running.

#### Verb Validator
Verb validators are called by a standard story validator to verify the compiled verb calls are valid after compilers are done.

#### Verb Driver
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