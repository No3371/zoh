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

Diagnostics from the `catch` verb (if provided) should be appended. Therefore, if the `catch` verb emit fatal diagnostics, the context still get terminated.

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

An interpolate verb interpolates string with `${*var}` syntax **non-recursively** (ONCE) for dynamic text generation. Any interpolation syntax (`${...}`) contained within the resolved values of variables is treated as literal text and is **not** processed. Undefined variables are treated as `nothing` (stringifying to `?` or provided `fallback` parameter).

For exmaple, `"Hello, ${*name}!"` should be interpolated to "Hello, John!" if `*name` is resolved to a string "John". (Without `/Interpolate`, It would otherwise requires `` /eval `"Hello, " + *name + "!` ``) 

Interpolation should also support:
- Formatting: C# composite format parity `${var[,width][:formatString]}`.
    - Examples: `/itpl "|${"Left",-7}|${"Right",7}|"` is `"|Left   |  Right|`. `/itpl "My balance is ${*balance,8:N1}."` is `"My balance is     100.0"`
    - This can not be used with the following feature syntaxes.
- Unrolling: `${*var..."delim"}` to expand `*[list]` and `*{map}` into `{element}{delim}{element}...` where `element`s are the list elements or map k:v pairs.
    - Example: `/itpl "I have ${*inv...", "}."` is `"I have potion, sword, pie."`
- Parity of `/count` for `$#{*var}`.
    - Example: `/itpl "I have $#{*inv_size} items."` is `"I have 3 items."`
- Parity of `/if` for `$?{*cond? *true_case | *false_case}`.
    - Example: `/itpl "You $?{*win? "win"|"lose"}."` is `"You win."`
- Parity of `/first` for `$?{*var1|*var2...|*varN}`.
    - Example: `/itpl "I have $?{?|"something"}".` is `"I have something."`
- Picking one option for `${*var1|*var2...|*varN}[#]` with the options as a list parameter and `#` as the index. For example: `${1|2|3}[*i]`. Optionally, the `#` can be prefixed with `!` for a modulo operation to wrap around the cases.
    - Examples: `/itpl "The ${1|2|3}[1]nd."` is `"The 2nd."`. `/itpl "The ${1|2|3}[!5]rd."` is `"The 3rd."`. 
- Parity of  `/roll` for `${*var1|*var2...|*varN}[%]` with the options as parameters.
- Parity of  `/wroll` for `${*var1:<w>|*var2:<w>...|*varN:<w>}[%]` with the options as parameters and `w` as the weights.

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
- Parity of `/interpolate` ONCE for `$"string"` or `$*var`. For example: `$"Hello, ${*name}!"`.
- Parity of `/count` for `$#(*var)`.
- Parity of `/if` for `$?(*var1? *var2 : *var3)` with `var1` as the condition, `var2` as the true case value, and `var3` as the else case value.
- Parity of `/first` for `$?(*var1|*var2...|*varN)`.
- Picking one option for `$(*var1|*var2...|*varN)[#]` with the options as a list parameter and `#` as the index. For example: `$(1|2|3)[*i]`. Optionally, the `#` can be prefixed with `!` for a modulo operation to wrap around the cases.
- Parity of `/roll` for `$(*var1|*var2...|*varN)[%]` with the options as parameters.
- Parity of `/wroll` for `$(*var1:<w>|*var2:<w>...|*varN:<w>)[%]` with the options as parameters and `w` as the weights.

Verb calls are not supported.

#### Aliases
- `eval`

#### Parameters
- `expr`: the expression to evaluate. Accept `` `expr` `` or `` *`expr` ``.

#### Diagnostics
- Fatal: `invalid_syntax`: Malformed expression syntax.
- Fatal: `invalid_type`: Type error during expression evaluation (e.g., subtracting integer from string, or using arithmetic on `nothing`).
- Fatal: `undefined_var`: Any variable used in the expression is undefined.
- Fatal: `division_by_zero`: Division or modulo operation with a zero divisor.
- Fatal: `invalid_index`: Index out of bounds in an `$(options)[n]` indexed form.

#### Returns
The result of the expression, type depends on the expression.

#### Examples
```
/eval `*var+1`;
/eval `(*int*10-1)/5.5`;
/eval `*str1+*str2`;
/eval `$*str1+$"str2str";
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

### Channel.Open

An open verb creates a new channel or re-creates a closed channel.

#### Parameters
- `channel`: The channel to open. Accept `<channel>` or `*<channel>`.

#### Returns
A nothing.

#### Examples
```
/open *channel;
/open <channel>;
```

### Channel.Push

A push verb pushes a variable to a channel. By default, the push blocks until the value is consumed by a puller (rendezvous semantics). Use `wait: false` for fire-and-forget behavior.

#### Named Parameters
- `wait`: Whether to block until the value is consumed. Accept `boolean`. Optional. Default to `true`.
- `timeout`: The timeout in seconds when `wait` is `true`. Accept `double` or `*double`. Optional. Default to `?`. Ignored when `wait` is `false`.

#### Parameters
- `channel`: The channel to push to. Accept `<channel>` or `*<channel>`.
- `var`: The variable to push. Accept `any`. If it is a reference, the runtime takes the value.

#### Returns
A nothing.

#### Diagnostics
- Error: `not_found`: The channel does not exist.
- Error: `closed`: The channel is closed, or closed while waiting.
- Info: `timeout`: The push timed out before the value was consumed (only with `wait: true` and `timeout`).

#### Examples
```
:: Blocking push (default) — waits until consumed
/push <channel>, *var;

:: Blocking push with timeout
/push <channel>, *var, timeout: 5;

:: Fire-and-forget push
/push <channel>, *var, wait: false;
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
- `message`: The message to emit. Accept `"string"`, `*"string"`, `` `expr` ``, or `` *`expr` ``. In case of reference, the value is used. In case of string, the value is interpolated ONCE. In case of `` `expr` ``, the expression is evaluated.

#### Returns
A nothing.

#### Examples
```
/info "Hello, world! ${*user}!";
/info `"Hello, world! " + *user + "!"`;
/warning "Hello, world!";
/error "Hello, world!";
/fatal "Hello, world!";
```

### Core.Assert

An assert verb checks a condition and emits a fatal diagnostic with the given message if the condition is not met (falsy: `false`, `nothing`, `0`, `0.0`, `""`, `[]`, `{}`). If the condition is met (truthy: anything that is not falsy), no diagnostic is emitted.

#### Named Parameters
- `is`: Value to be compared to the subject. Optional. Default to `true`. Accept `any`. In case of reference, the value is used. In case of `` `expr` ``, it is evaluated.

#### Parameters
- `subject`: The condition to assert. Accept `any`. In case of reference, the value is used. In case of `` `expr` ``, it is evaluated. In case of `/verb`, the returned value is used.
- `message`: The message to emit on failure. Accept `"string"`, `*"string"`, `` `expr` ``, or `` *`expr` ``. In case of reference, the value is used. In case of string, the value is interpolated ONCE. In case of `` `expr` ``, the expression is evaluated. Optional — defaults to `"assertion failed"`.

#### Diagnostics
- Fatal: `assertion_failed`: The asserted condition was falsy. Includes the evaluated message.

#### Returns
A nothing.

#### Examples
```
/assert *is_valid;
/assert *health, "health must be truthy";
/assert `*health > 0`, "health must be positive: ${*health}";
/assert *mode, is: "combat", "expected combat mode";
/assert /has *inventory, "sword";, "player must have sword";
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

A parse verb returns a value parsed from a string. The verb first trims any leading and trailing whitespace from the input string.

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