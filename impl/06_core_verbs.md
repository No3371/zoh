# 06: Core Verbs Implementation

## Purpose

Core verbs are the foundational building blocks that all ZOH runtimes must implement. They handle variable management, value operations, and basic utilities.

---

## Verb Driver Architecture

```
VerbDriver:
  namespace: string                 # e.g., "core", "std"
  name: string                      # Verb name
  priority: int                     # Execution priority
  
  validate(call: VerbCall, context: Context): List<Diagnostic>
  execute(call: VerbCall, context: Context): ExecutionResult

ExecutionResult:
  value: Value                # Return value
  diagnostics: List<Diagnostic> | nothing

Helpers to return ExecutionResult of nothing value and specified diagnostics:
ok(value = nothing)
info(code, message)
warning(code, message)
error(code, message)
fatal(code, message)
```

All drivers share a common pattern:
1. Extract and validate parameters
2. Resolve references to values, unless in rare cases references are expected
3. Perform the operation
4. Return ExecutionResult

---

## Core.Set

**Purpose**: Declare or update a variable.

### Signature
```
/set [scope:story|context] [typed:type] [required] [resolve] [OneOf:list]
     *target[path...], value?;
```

### Implementation

```
SetDriver.execute(call, context):
    # Extract target reference
    target = call.params[0]
    if target is not ReferenceValue:
        return fatal("invalid_type", "Expected reference")

    varName = target.name
    path = target.path
    value = call.params.length > 1 ? call.params[1] : Nothing

    # Handle [resolve] attribute
    if hasAttribute(call, "resolve"):
        if value is VerbValue:
            value = executeVerb(value, context)
        elif value is ExpressionValue:
            value = evaluate(value, context)
    else:
        value = resolve(value, context)

    # Handle [required] attribute
    if hasAttribute(call, "required"):
        existingValue = context.get(varName)
        if value.isNothing() and existingValue.isNothing():
            return fatal("required", "Required variable has no value: " + varName)

    # Handle [typed] attribute
    typedAttr = getAttribute(call, "typed")
    if typedAttr:
        expectedType = typedAttr.value.toString()
        if not value.isNothing() and value.getType() != expectedType:
            return fatal("type_mismatch",
                "Expected " + expectedType + ", got " + value.getType())
        context.setTyped(varName, value, expectedType, getScope(call))
    else:
        # Set at path
        result = setAtPath(context, varName, path, value, getScope(call))
        if result.isFatal():
            return result

    # Handle [OneOf] attribute
    oneOfAttr = getAttribute(call, "OneOf")
    if oneOfAttr:
        allowed = resolve(oneOfAttr.value, context)
        if not listContains(allowed, value):
            return fatal("invalid_value", "Value not in allowed list")

    return ok()

setAtPath(context, varName, path, value, scope):
    if path.isEmpty():
        context.set(varName, value, scope)
        return ok()

    # Navigate to parent of final element
    current = context.get(varName)
    if current.isNothing():
        return fatal("invalid_index", "Variable does not exist: " + varName)

    for i in range(0, path.length - 1):
        index = resolve(path[i], context)
        current = getIndexed(current, index)
        if current.isNothing():
            return fatal("invalid_index", "Path element does not exist at index " + i)

    # Set final element
    finalIndex = resolve(path[path.length - 1], context)
    return setIndexed(current, finalIndex, value)

getScope(call): string
    scopeAttr = getAttribute(call, "scope")
    if scopeAttr:
        return scopeAttr.value.toString()
    return "story"  # Default
```

---

## Core.Get

**Purpose**: Return the value of a variable.

### Signature
```
/get [required] *target[path...];
```

### Implementation

```
GetDriver.execute(call, context):
    target = call.params[0]
    if target is not ReferenceValue:
        return fatal("invalid_type", "Expected reference")

    varName = target.name
    path = target.path

    value = getAtPath(context, varName, path)

    if hasAttribute(call, "required") and value.isNothing():
        return error("not_found", "Required variable not found: " + varName)

    return ok(value)

getAtPath(context, varName, path):
    value = context.get(varName)

    for pathElement in path:
        index = resolve(pathElement, context)
        value = getIndexed(value, index)
        if value.isNothing():
            return Nothing  # Path navigation stops at nothing

    return value
```

---

## Core.Drop

**Purpose**: Delete a variable or set nested element to nothing.

### Signature
```
/drop [scope:story|context] *target[path...];
```

### Implementation

```
DropDriver.execute(call, context):
    target = call.params[0]
    if target is not ReferenceValue:
        return fatal("invalid_type", "Expected reference")

    varName = target.name
    path = target.path
    scope = getAttribute(call, "scope")?.value.toString() ?? "story"

    if path.isEmpty():
        # Drop the entire variable
        context.drop(varName, scope)
    else:
        # Set nested element to nothing
        result = setAtPath(context, varName, path, Nothing, scope)
        if result.isFatal():
            return result

    return ok()
```

---

## Core.Capture

**Purpose**: Store the return value of the previous verb into a variable.

### Signature
```
/capture *target[path...];
```

### Implementation

```
CaptureDriver.execute(call, context):
    target = call.params[0]
    if target is not ReferenceValue:
        return fatal("invalid_type", "Expected reference")

    value = context.getLastReturnValue()
    varName = target.name
    path = target.path

    result = setAtPath(context, varName, path, value, "story")
    if result.isFatal():
        return result

    return ok()
```

---

## Core.Diagnose

**Purpose**: Return the last diagnostics from previous verb execution.

### Signature
```
/diagnose;
```

### Implementation

```
DiagnoseDriver.execute(call, context):
    diagnostics = context.lastDiagnostics
    # Return the captured diagnostics as the value
    # Since this call itself returns empty diagnostics, it will 
    # effectively "clear" the context buffer after execution.
    return ok(diagnostics ?? Nothing)
```

---

## Core.Interpolate

**Purpose**: Perform string interpolation.

### Signature
```
/interpolate str;
```

### Special Syntax in Strings

| Pattern | Action |
|---------|--------|
| `${*var}` | Insert variable value (References only, NO expressions) |
| `${*var,width}` | Format with width |
| `${*var:format}` | Format with specifier |
| `${*var..."\|"}` | Unroll list/map with delimiter |
| `$#{*var}` | Insert count |
| `$?{*a? *b : *c}` | Conditional |
| `$?{*a\|*b\|*c}` | First non-nothing |
| `${a\|b\|c}[#]` | Index select |
| `${a\|b\|c}[%]` | Random select |

### Implementation

```
InterpolateDriver.execute(call, context):
    input = resolve(call.params[0], context)
    if input is not StringValue:
        return ok(StringValue(input.toString()))
    
    result = StringBuilder()
    str = input.value
    i = 0
    
    while i < str.length:
        if str[i] == '\\' and i+1 < str.length and str[i+1] in ['{', '}']:
            result.append(str[i+1])
            i += 2
        elif str[i] == '$' and i+1 < str.length:
            i++
            interpolated, consumed = parseInterpolation(str, i, context)
            result.append(interpolated)
            i += consumed
        else:
            result.append(str[i])
            i++
    
    return ok(StringValue(result.toString()))

parseInterpolation(str, start, context): (string, int)
    # Handle special prefixes
    if str[start] == '#':
        # Count form: $#{*var}
        return parseCountInterpolation(str, start+1, context)
    if str[start] == '?':
        # Conditional or any form
        return parseConditionalInterpolation(str, start+1, context)
    if str[start] == '{':
        return parseBasicInterpolation(str, start, context)
    
    return fatal("invalid_syntax", "Invalid interpolation")

parseBasicInterpolation(str, start, context): (string, int)
    # Find matching }
    # Parse contents, handle |, :, [...] for select/roll
    # Resolve and format
    ...
```

---

## Core.Evaluate

**Purpose**: Evaluate an expression and return the result.

### Signature
```
/evaluate expr;
```

### Implementation

```
EvaluateDriver.execute(call, context):
    expr = call.params[0]
    
    if expr is ReferenceValue:
        expr = context.get(expr.name)
    
    if expr is not ExpressionValue:
        return fatal("invalid_type", "Expected expression, got: " + expr.getType())
    
    # Evaluate expression, catching undefined variable errors
    try:
        result = evaluateExpression(expr.ast, context)
        return ok(result)
    catch UndefinedVariableError as e:
        return fatal("undefined_var", "Undefined variable: " + e.varName)
```

See [04_expressions.md](./04_expressions.md) for full evaluation logic.

---

## Core.Type

**Purpose**: Return the type name of a value.

### Signature
```
/type var;
```

### Implementation

```
TypeDriver.execute(call, context):
    value = resolve(call.params[0], context)
    return ok(StringValue(value.getType()))
```

---

## Core.Count

**Purpose**: Return the size/length of a value.

### Signature
```
/count *target[path...];
```

### Implementation

```
CountDriver.execute(call, context):
    target = call.params[0]
    if target is not ReferenceValue:
        return fatal("invalid_type", "Expected reference")

    # getAtPath returns Nothing on invalid path (query semantic)
    value = getAtPath(context, target.name, target.path)

    # Nothing (including invalid path) returns 0
    if value.isNothing():
        return ok(IntegerValue(0))

    match value:
        StringValue:
            return ok(IntegerValue(value.value.length))
        ListValue:
            return ok(IntegerValue(value.elements.length))
        MapValue:
            return ok(IntegerValue(value.entries.size))
        ChannelValue:
            return ok(IntegerValue(context.runtime.getChannelSize(value.name)))
        _:
            return fatal("invalid_type", "Cannot count type: " + value.getType())
```

---

## Core.Do

**Purpose**: Execute a verb stored in a variable.

### Signature
```
/do verb_ref;;
```

### Implementation

```
DoDriver.execute(call, context):
    verbParam = call.params[0]
    
    # If it's a verb call, execute it first to get the verb to execute
    if verbParam is VerbCallValue:
        verbValue = executeVerb(verbParam, context)
    elif verbParam is ReferenceValue:
        verbValue = resolve(verbParam, context)
    else:
        verbValue = verbParam
    
    if verbValue is not VerbValue:
        return fatal("invalid_type", "Expected verb, got: " + verbValue.getType())
    
    # Execute the verb and return its result
    return executeVerb(verbValue, context)
```

---

## Core.Increase / Core.Decrease

**Purpose**: Modify a numeric variable.

### Signature
```
/increase *target[path...], value?;
/decrease *target[path...], value?;
```

### Implementation

```
IncreaseDriver.execute(call, context):
    target = call.params[0]
    if target is not ReferenceValue:
        return fatal("invalid_type", "Expected reference")

    varName = target.name
    path = target.path
    amount = call.params.length > 1 ? resolve(call.params[1], context) : IntegerValue(1)

    # getAtPath returns Nothing on invalid path (query semantic)
    current = getAtPath(context, varName, path)
    if current.isNothing():
        current = IntegerValue(0)

    # Validate target is numeric
    if current is not IntegerValue and current is not DoubleValue and not current.isNothing():
        return fatal("invalid_type", "Target must be numeric, got: " + current.getType())

    # Calculate new value
    if amount is IntegerValue and current is IntegerValue:
        newValue = IntegerValue(current.value + amount.value)
    else:
        newValue = DoubleValue(current.toDouble() + amount.toDouble())

    # setAtPath fatals on invalid intermediate path (mutation semantic)
    result = setAtPath(context, varName, path, newValue, "story")
    if result.isFatal():
        return result

    return ok(newValue)

DecreaseDriver.execute(call, context):
    # Same as increase but subtract
    ...
```

---

## Core.Roll / Core.WRoll / Core.Rand

**Purpose**: Random value selection.

### Signatures
```
/roll value1, value2, ...;
/wroll value1, weight1, value2, weight2, ...;
/rand min, max, inclmax?;
```

### Implementation

```
RollDriver.execute(call, context):
    if call.params.isEmpty():
        return fatal("parameter_not_found", "Roll requires at least one option")

    values = call.params.map(p => resolve(p, context))
    index = randomInt(0, values.length)
    return ok(values[index])

WRollDriver.execute(call, context):
    if call.params.length < 2:
        return fatal("parameter_not_found", "WRoll requires at least one value-weight pair")

    # Parse value-weight pairs
    pairs = []
    for i in range(0, call.params.length, 2):
        value = resolve(call.params[i], context)
        weightValue = resolve(call.params[i+1], context)

        if weightValue is not IntegerValue and weightValue is not DoubleValue:
            return fatal("invalid_type", "Weight must be a number, got: " + weightValue.getType())

        weight = weightValue.toInt()
        if weight < 0:
            return fatal("invalid_value", "Weight cannot be negative: " + weight.toString())

        pairs.add({ value, weight })

    totalWeight = sum(p.weight for p in pairs)
    roll = randomInt(0, totalWeight)

    cumulative = 0
    for pair in pairs:
        cumulative += pair.weight
        if roll < cumulative:
            return ok(pair.value)

    return ok(pairs.last.value)

RandDriver.execute(call, context):
    min = resolve(call.params[0], context)
    max = resolve(call.params[1], context)
    inclmax = getNamedParam(call, "inclmax", false)

    # Validate types
    if min is not IntegerValue and min is not DoubleValue:
        return fatal("invalid_type", "Min must be a number, got: " + min.getType())
    if max is not IntegerValue and max is not DoubleValue:
        return fatal("invalid_type", "Max must be a number, got: " + max.getType())

    # Validate range
    if min.toDouble() > max.toDouble():
        return fatal("invalid_value", "Min cannot be greater than max")

    if min is DoubleValue or max is DoubleValue:
        result = randomDouble(min.toDouble(), max.toDouble())
        return ok(DoubleValue(result))
    else:
        upperBound = max.toInt()
        if inclmax:
            upperBound++
        result = randomInt(min.toInt(), upperBound)
        return ok(IntegerValue(result))
```

---

## Core.Parse

**Purpose**: Parse string to typed value.

### Signature
```
/parse value, type?;
```

### Implementation

```
ParseDriver.execute(call, context):
    str = resolve(call.params[0], context).toString()
    
    if call.params.length > 1:
        targetType = resolve(call.params[1], context).toString()
    else:
        targetType = inferType(str)
    
    try:
        match targetType:
            "integer": return ok(IntegerValue(parseInt(str)))
            "double": return ok(DoubleValue(parseDouble(str)))
            "boolean": return ok(BoolValue(parseBool(str)))
            "list": return ok(parseList(str))
            "map": return ok(parseMap(str))
            _: return fatal("invalid_type", "Unknown type: " + targetType)
    catch ParseError as e:
        return fatal("invalid_type", "Cannot parse '" + str + "' as " + targetType)

inferType(str): string
    if matches(str, /^-?\d+$/): return "integer"
    if matches(str, /^-?\d+\.\d+$/): return "double"
    if str.toLowerCase() in ["true", "false"]: return "boolean"
    if startsWith(str, "["): return "list"
    if startsWith(str, "{"): return "map"
    return "string"
```

---

## Core.Defer

**Purpose**: Schedule a verb to run when leaving scope.

### Signature
```
/defer [scope:story|context] verb;
```

### Implementation

```
DeferDriver.execute(call, context):
    verb = call.params[0]

    # Resolve if reference
    if verb is ReferenceValue:
        verb = resolve(verb, context)

    # Validate verb type
    if verb is not VerbValue:
        return fatal("invalid_type", "Expected verb, got: " + verb.getType())

    scope = getAttribute(call, "scope")?.value.toString() ?? "story"

    if scope == "story":
        context.addStoryDefer(verb)
    else:
        context.addContextDefer(verb)

    return ok()

# On story exit:
Context.executeStoryDefers():
    while storyDefers.notEmpty():
        verb = storyDefers.pop()  # LIFO order
        executeVerb(verb, this)

# On context termination:
Context.executeContextDefers():
    while contextDefers.notEmpty():
        verb = contextDefers.pop()
        executeVerb(verb, this)
```

---

## Debug Verbs

**Purpose**: Emit diagnostic messages.

### Signatures
```
/info message;
/warning message;
/error message;
/fatal message;
```

### Implementation

```
DebugDriver.execute(call, context):
    message = resolve(call.params[0], context)
    
    if message is ExpressionValue:
        message = evaluate(message, context)
    
    severity = getSeverityFromVerbName(call.name)
    code = "debug_" + severity
    msg = message.toString()
    
    # Use the appropriate helper based on severity
    match severity:
        "info": return info(code, msg)
        "warning": return warning(code, msg)
        "error": return error(code, msg)
        "fatal": return fatal(code, msg)
```

---

## Core.Has

**Purpose**: Check if a value exists in a list or as a key in a map.

### Signature
```
/has collection, subject;
```

### Implementation

```
HasDriver.execute(call, context):
    collection = resolve(call.params[0], context)
    subject = resolve(call.params[1], context)
    
    if collection is ListValue:
        for element in collection.elements:
            if equals(element, subject):
                return ok(BoolValue(true))
        return ok(BoolValue(false))
    
    if collection is MapValue:
        if subject is not StringValue:
            return fatal("invalid_type", "Map key must be string, got: " + subject.getType())
        return ok(BoolValue(collection.hasKey(subject.value)))
    
    return fatal("invalid_type", "Expected list or map, got: " + collection.getType())
```

---

## Core.Any

**Purpose**: Return `true` if a variable or indexed value is not a nothing.

### Signature
```
/any var, index?;
```

### Implementation

```
AnyDriver.execute(call, context):
    value = resolve(call.params[0], context)
    
    if call.params.length > 1:
        index = resolve(call.params[1], context)
        if index.isNothing():
            return ok(BoolValue(not value.isNothing()))
        
        if value is ListValue:
            idx = index.toInt()
            if idx < 0 or idx >= value.elements.length:
                return ok(BoolValue(false))
            return ok(BoolValue(not value.elements[idx].isNothing()))
        
        if value is MapValue:
            if index is not StringValue:
                return fatal("invalid_type", "Map key must be string, got: " + index.getType())
            return ok(BoolValue(value.hasKey(index.value) and not value.get(index.value).isNothing()))
    
    return ok(BoolValue(not value.isNothing()))
```

---

## First

**Purpose**: Return first non-nothing value.

### Signature
```
/first option1, option2, ...;
```

### Implementation

```
FirstDriver.execute(call, context):
    for param in call.params:
        value = resolveValue(param, context)
        
        if value is VerbValue:
            value = executeVerb(value, context)
        if value is ExpressionValue:
            value = evaluate(value, context)
        
        if not value.isNothing():
            return ok(value)
    
    return ok()
```

### Examples

```zoh
/first *primary, *fallback, "default";  -> *result;
/first /try_a;, /try_b;, /try_c;;       -> *first_success;
```

---

## Core.Append

**Purpose**: Append a value to a list or a key-value pair to a map.

### Signature
```
/append collection, value;
```

### Implementation

```
AppendDriver.execute(call, context):
    collectionRef = call.params[0]
    if collectionRef is not ReferenceValue:
        return fatal("invalid_type", "Expected reference")
    
    collection = context.get(collectionRef.name)
    value = resolve(call.params[1], context)
    
    if collection is ListValue:
        collection.append(value)
        context.set(collectionRef.name, collection)
        return ok(IntegerValue(collection.count()))
    
    if collection is MapValue:
        # value should be a single-entry map {"key": val}
        if value is not MapValue or value.entries.size != 1:
            return fatal("invalid_type", "Expected single-entry map for map append")
        for (key, val) in value.entries:
            if collection.hasKey(key):
                return error("key_conflict", "Key already exists: " + key)
            collection.set(key, val)
        context.set(collectionRef.name, collection)
        return ok(IntegerValue(collection.count()))
    
    return fatal("invalid_type", "Expected list or map")
```

---

## Core.Remove

**Purpose**: Remove a value from a list or map by index/key.

### Signature
```
/remove collection, index;
```

### Implementation

```
RemoveDriver.execute(call, context):
    collectionRef = call.params[0]
    if collectionRef is not ReferenceValue:
        return fatal("invalid_type", "Expected reference")
    
    collection = context.get(collectionRef.name)
    index = resolve(call.params[1], context)
    
    if collection is ListValue:
        idx = index.toInt()
        if idx < 0:
            idx = collection.count() + idx  # Negative indexing
        if idx >= 0 and idx < collection.count():
            collection.remove(idx)
        context.set(collectionRef.name, collection)
        return ok(IntegerValue(collection.count()))
    
    if collection is MapValue:
        if index is not StringValue:
            return fatal("invalid_type", "Map key must be string, got: " + index.getType())
        collection.remove(index.value)  # No-op if key doesn't exist
        context.set(collectionRef.name, collection)
        return ok(IntegerValue(collection.count()))
    
    return fatal("invalid_type", "Expected list or map")
```

---

## Core.Insert

**Purpose**: Insert a value into a list at a specific index.

### Signature
```
/insert list, index, value;
```

### Implementation

```
InsertDriver.execute(call, context):
    listRef = call.params[0]
    if listRef is not ReferenceValue:
        return fatal("invalid_type", "Expected reference")
    
    list = context.get(listRef.name)
    if list is not ListValue:
        return fatal("invalid_type", "Expected list")
    
    index = resolve(call.params[1], context).toInt()
    value = resolve(call.params[2], context)
    
    if index < 0:
        index = list.count() + index + 1  # Negative: insert from end
    if index < 0 or index > list.count():
        return error("invalid_index", "Index out of bounds: " + index.toString())
    
    list.insert(index, value)
    context.set(listRef.name, list)
    return ok(IntegerValue(list.count()))
```

---

## Core.Clear

**Purpose**: Clear all elements from a list or map.

### Signature
```
/clear collection;
```

### Implementation

```
ClearDriver.execute(call, context):
    collectionRef = call.params[0]
    if collectionRef is not ReferenceValue:
        return fatal("invalid_type", "Expected reference")
    
    collection = context.get(collectionRef.name)
    
    if collection is ListValue:
        collection.elements.clear()
    elif collection is MapValue:
        collection.entries.clear()
    else:
        return fatal("invalid_type", "Expected list or map")
    
    context.set(collectionRef.name, collection)
    return ok()
```

---

## Core.Exit

**Purpose**: Stop the context from continuing execution.

### Signature
```
/exit;
```

### Implementation

```
ExitDriver.execute(call, context):
    context.terminate()
    return ok()
```

---

## Testing Checklist


### Core.Set / Core.Get
- [ ] Simple set and get
- [ ] Set with path (list index)
- [ ] Set with path (map key)
- [ ] Set with nested path (*data["users"][0]["name"])
- [ ] Scoped variables (story vs context)
- [ ] Typed variable enforcement
- [ ] Required attribute
- [ ] Resolve attribute with verb
- [ ] OneOf validation
- [ ] Invalid path returns fatal

### Core.Capture
- [ ] Capture return value
- [ ] Capture into path (*list[0], *data["key"])

### Core.Interpolate
- [ ] Basic variable substitution
- [ ] Formatted output
- [ ] Conditional interpolation
- [ ] List unrolling
- [ ] Random selection

### Core.Evaluate
- [ ] Simple expression
- [ ] Expression with references
- [ ] Undefined variable error

### Random Verbs
- [ ] Roll with multiple options
- [ ] Weighted roll
- [ ] Rand integer range
- [ ] Rand double range
- [ ] Edge cases (single option, equal weights)

### Collection Verbs
- [ ] Has: value in list
- [ ] Has: key in map
- [ ] Any: variable exists
- [ ] Any: indexed value exists
- [ ] Append: to list
- [ ] Append: to map (single-entry map)
- [ ] Remove: from list by index
- [ ] Remove: from map by key
- [ ] Remove: negative indexing
- [ ] Insert: at index
- [ ] Insert: negative index
- [ ] Clear: list
- [ ] Clear: map

### Utility Verbs
- [ ] Exit: terminates context
- [ ] Exit: defers still execute

