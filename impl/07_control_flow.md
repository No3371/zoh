# 07: Control Flow Implementation

## Purpose

Control flow verbs manage program execution order: conditionals, loops, branching, and sequencing.

---

## Control Flow Verbs Overview

| Verb | Purpose |
|------|---------|
| `/sequence` | Execute verbs in order |
| `/if` | Conditional execution |
| `/loop` | Fixed iteration count |
| `/while` | Condition-based loop |
| `/foreach` | Collection iteration |
| `/switch` | Pattern matching |
| `/first` | First non-nothing result |
| `/break` | Exit current loop (via breakif) |
| `/try` | Execute with fatal error recovery |

---

## Sequence

**Purpose**: Execute verbs in order, optionally with break condition.

### Signature
```
/sequence breakif:condition?, verb1, verb2, ...;
```

### Implementation

```
SequenceDriver.execute(call, context):
    breakCondition = getNamedParam(call, "breakif")
    lastResult = Nothing
    
    for param in call.unnamedParams:
        # Check break condition before each verb
        if breakCondition != null:
            if shouldBreak(breakCondition, context):
                break
        
        verb = resolveToVerb(param, context)
        lastResult = executeVerb(verb, context)
    
    return lastResult

shouldBreak(condition, context): bool | fatal
    if condition is ReferenceValue:
        value = resolve(condition, context)
    elif condition is ExpressionValue:
        value = evaluate(condition, context)
    elif condition is VerbValue:
        value = executeVerb(condition, context)
    else:
        value = condition

    # Validate condition is boolean or nothing
    if value is not BoolValue and not value.isNothing():
        return fatal("invalid_type", "Break condition must be boolean or nothing, got: " + value.getType())

    return value.toBool()
```

### Block Form

```zoh
/sequence/
    /verb1;
    /verb2;
    /verb3;
/;
```

---

## If

**Purpose**: Conditional verb execution.

### Signature
```
/if subject, is:value?, verb, else:verb?;
```

### Parameters

| Parameter | Type | Default | Notes |
|-----------|------|---------|-------|
| `subject` | any | required | Condition to evaluate |
| `is` | any | `true` | Value to compare against |
| `verb` | verb | required | Execute if condition matches |
| `else` | verb | nothing | Execute if condition fails |

### Implementation

```
IfDriver.execute(call, context):
    subject = resolveValue(call.params[0], context)
    compareValue = getNamedParam(call, "is", BoolValue(true))
    thenVerb = call.params[1]
    elseVerb = getNamedParam(call, "else")

    # Evaluate subject if needed
    if subject is VerbValue:
        subject = executeVerb(subject, context)
    if subject is ExpressionValue:
        subject = evaluate(subject, context)

    # Validate subject type when comparing to default (true)
    if compareValue == BoolValue(true):
        if subject is not BoolValue and not subject.isNothing():
            return fatal("invalid_type", "Condition must be boolean or nothing, got: " + subject.getType())

    # Evaluate comparison value if needed
    compareValue = resolveValue(compareValue, context)

    # Compare
    matches = equals(subject, compareValue)

    if matches:
        return executeVerb(thenVerb, context)
    elif elseVerb != null:
        return executeVerb(elseVerb, context)

    return ok()
```

### Examples

```zoh
/if *cond, /verb;;                           # Simple if
/if *char, is: "Alice", /speak;;             # Equality check
/if `*x > 5`, /verb;, else: /other;;         # With else
/if /some_check;, /verb;;                    # Verb as condition
```

---

## Loop

**Purpose**: Execute a verb a fixed number of times.

### Signature
```
/loop times, breakif:condition?, verb;
```

### Implementation

```
LoopDriver.execute(call, context):
    timesValue = resolve(call.params[0], context)

    # Validate count is integer
    if timesValue is not IntegerValue:
        return fatal("invalid_type", "Loop count must be integer, got: " + timesValue.getType())

    times = timesValue.toInt()
    breakCondition = getNamedParam(call, "breakif")
    verb = call.params[1]

    # Special case: -1 means infinite (or until break)
    isInfinite = (times == -1)

    iteration = 0
    while isInfinite or iteration < times:
        # Check break condition
        if breakCondition != null:
            if shouldBreak(breakCondition, context):
                break

        executeVerb(verb, context)
        iteration++

    return ok()
```

### Examples

```zoh
/loop 5, /verb;;                    # Execute 5 times
/loop *count, /verb;;               # Variable count
/loop -1, breakif: *done, /verb;;   # Until done
/loop 999, /sequence/               # Multi-verb loop
    /verb1;
    /verb2;
/;;
```

---

## While

**Purpose**: Loop while condition is true.

### Signature
```
/while subject, is:value?, verb;
```

### Implementation

```
WhileDriver.execute(call, context):
    subjectExpr = call.params[0]
    compareValue = getNamedParam(call, "is", BoolValue(true))
    verb = call.params[1]

    while true:
        # Re-evaluate subject each iteration
        subject = resolveValue(subjectExpr, context)
        if subject is VerbValue:
            subject = executeVerb(subject, context)
        if subject is ExpressionValue:
            subject = evaluate(subject, context)

        # Validate subject type when comparing to default (true)
        if compareValue == BoolValue(true):
            if subject is not BoolValue and not subject.isNothing():
                return fatal("invalid_type", "Condition must be boolean or nothing, got: " + subject.getType())

        compare = resolveValue(compareValue, context)

        if not equals(subject, compare):
            break

        executeVerb(verb, context)

    return ok()
```

### Examples

```zoh
/while *running, /tick;;              # While true
/while `*count > 0`, /process;;       # Expression condition
/while /has_more;, /process;;         # Verb as condition
```

---

## Foreach

**Purpose**: Iterate over a collection.

### Signature
```
/foreach collection, iterator, breakif:condition?, verb;
```

### Implementation

```
ForeachDriver.execute(call, context):
    collection = resolve(call.params[0], context)
    iteratorRef = call.params[1]  # Must be ReferenceValue
    breakCondition = getNamedParam(call, "breakif")
    verb = call.params[2]
    
    iteratorName = iteratorRef.name
    
    # Drop existing variable from Story scope to avoid type conflicts
    context.drop(iteratorName, scope: STORY)
    
    if collection is ListValue:
        for element in collection.elements:
            context.set(iteratorName, element)
            
            if breakCondition != null and shouldBreak(breakCondition, context):
                break
            
            executeVerb(verb, context)
    
    elif collection is MapValue:
        for (key, value) in collection.entries:
            # Iterator receives single-entry map
            iterValue = MapValue({ key: value })
            context.set(iteratorName, iterValue)
            
            if breakCondition != null and shouldBreak(breakCondition, context):
                break
            
            executeVerb(verb, context)
    
    else:
        return fatal("invalid_type", "Cannot iterate over type: " + collection.getType())
    
    return ok()
```

### Examples

```zoh
/foreach *items, *item, /process *item;;

/foreach *characters, *char, /sequence/
    /converse [By: *char] "Hello!";
/;;

:: Iterating a map - each *entry is a single-entry map like {"key": value}
/foreach *inventory, *entry, /sequence/
    :: Use the entry directly in interpolation
    /converse "Item: ${*entry}";
/;;
```

---

## Switch

**Purpose**: Pattern matching with multiple cases.

### Signature
```
/switch subject, case1, value1, case2, value2, ..., default?;
```

### Implementation

```
SwitchDriver.execute(call, context):
    subject = resolveValue(call.params[0], context)
    
    # subject can be verb or expression
    if subject is VerbValue:
        subject = executeVerb(subject, context)
    if subject is ExpressionValue:
        subject = evaluate(subject, context)
    
    # Cases are in pairs (case, value), final param may be default
    params = call.unnamedParams.slice(1)  # Skip subject
    
    i = 0
    while i < params.length:
        if i + 1 >= params.length:
            # Last param is default
            return resolveValue(params[i], context)
        
        caseValue = resolveValue(params[i], context)
        resultValue = params[i + 1]
        
        if equals(subject, caseValue):
            return ok(resolveValue(resultValue, context))
        
        i += 2
    
    return ok()
```

### Block Form

```zoh
/switch/ *subject
    "case1"
        *result1
    "case2"
        *result2
    "case3"
        /verb;
    *default_result
/;
```

---

## Try

**Purpose**: Execute a verb with fatal diagnostic recovery, downgrading fatal to error level.

### Signature
```
/try verb, catch:handler?;
```

### Parameters

| Parameter | Type | Default | Notes |
|-----------|------|---------|-------|
| `verb` | verb | required | The verb to execute |
| `catch` | verb | nothing | Execute if fatal occurred |

### Attributes
- **suppress**: Clear the downgraded diagnostics from context

### Implementation

```
TryDriver.execute(call, context):
    verb = call.params[0]
    catchHandler = getNamedParam(call, "catch")
    suppressDiagnostics = hasAttribute(call, "suppress")

    # Execute verb and capture result
    result = executeVerb(verb, context)

    # Check if fatal occurred
    hasFatal = context.diagnostics.fatal.length > 0

    if hasFatal:
        # Downgrade fatal to error (preserve original codes)
        for diagnostic in context.diagnostics.fatal:
            context.diagnostics.error.append(diagnostic)
        context.diagnostics.fatal.clear()

        # Execute catch handler if provided
        if catchHandler != null:
            result = executeVerb(catchHandler, context)
        else:
            result = Nothing

    # Apply suppress attribute
    if suppressDiagnostics:
        context.diagnostics.clear()

    return result
```

### Examples

```zoh
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

---

## Break Behavior

The `breakif` named parameter is used for loop control:

```
breakif: condition
```

Where condition can be:
- `*boolean` reference
- `` `expression` ``
- `/verb` returning boolean

### Break Check Timing

- **Before each iteration**: Check breakif before executing the loop body
- Not after: This means if breakif becomes true during execution, the current iteration still completes

### Example: Early Exit

```zoh
*found <- false;
/foreach *items, *item, breakif: *found, /sequence/
    /if `*item == "target"`, /sequence/
        *found <- true;
        /converse "Found it!";
    /;;
/;;
```

---

## Nested Control Flow

Control flow verbs can be freely nested:

```zoh
:: Data structure: *rooms is a list of maps
::   *rooms = [ {"sword": {"name": "Iron Sword", "count": 2}},
::              {"potion": {"name": "Health Potion", "count": 5}},
::              {} ]  :: empty room

/foreach *rooms, *room, /sequence/
    :: $#(*room) = count of items in room
    /if `$#(*room) > 0`, /sequence/
        /foreach *room, *item, /sequence/
            /while `*item["count"] > 0`, /sequence/
                /converse "Taking ${*item["name"]}";
                /decrease *item, "count";
            /;;
        /;;
    /;, else: /converse "Empty room";;
/;;
```

---

## Testing Checklist

### Sequence
- [ ] Execute multiple verbs in order
- [ ] Return value is last verb's return
- [ ] breakif stops execution
- [ ] Empty sequence returns a nothing

### If
- [ ] Simple boolean condition
- [ ] Comparison with `is`
- [ ] Else branch execution
- [ ] Expression as condition
- [ ] Verb as condition

### Loop
- [ ] Fixed count iteration
- [ ] Variable count
- [ ] Infinite loop with break
- [ ] Zero iterations (count = 0)

### While
- [ ] Condition re-evaluated each iteration
- [ ] Expression condition
- [ ] Verb condition
- [ ] Comparison with `is`

### Foreach
- [ ] List iteration
- [ ] Map iteration (key-value pairs)
- [ ] breakif early exit
- [ ] Nested foreach

### Switch
- [ ] Multiple cases
- [ ] Default case
- [ ] First match wins
- [ ] Verb/expression as subject

### First
- [ ] Finds first non-nothing
- [ ] All nothing returns a nothing
- [ ] Verb options

### Nesting
- [ ] Deeply nested control flow
- [ ] Mixed control flow types
- [ ] Variable scope in loops

### Try
- [ ] Fatal downgraded to error
- [ ] Catch handler executes on fatal
- [ ] No catch returns a nothing on fatal
- [ ] Non-fatal diagnostics pass through unchanged
- [ ] Suppress attribute clears diagnostics
- [ ] Catch fatal terminates normally
