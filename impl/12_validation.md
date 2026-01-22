# 12: Validation & Diagnostics

## Purpose

Validation ensures ZOH scripts are correct before and during execution. This covers compilation, story validation, verb validation, and diagnostic handling.

---

## Validation Pipeline

```
┌──────────────────────────────────────────────────────────────────┐
│                      VALIDATION PIPELINE                         │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌────────────────────┐                                          │
│  │  1. LEXER PHASE    │  Syntax errors in tokenization           │
│  │     Errors         │  - Unterminated strings                  │
│  │                    │  - Invalid characters                    │
│  └─────────┬──────────┘  - Malformed numbers                     │
│            │                                                     │
│            ▼                                                     │
│  ┌────────────────────┐                                          │
│  │  2. PARSER PHASE   │  Structure errors in parsing             │
│  │     Errors         │  - Missing semicolons                    │
│  │                    │  - Unbalanced brackets                   │
│  └─────────┬──────────┘  - Invalid syntax                        │
│            │                                                     │
│            ▼                                                     │
│  ┌────────────────────┐                                          │
│  │  3. PREPROCESSOR   │  Directive errors                        │
│  │     Errors         │  - Missing files (#embed)                │
│  │                    │  - Circular embeds                       │
│  └─────────┬──────────┘  - Unknown macros                        │
│            │                                                     │
│            ▼                                                     │
│  ┌────────────────────┐                                          │
│  │  4. STORY          │  Story-level validation                  │
│  │     Validators     │  - Duplicate labels                      │
│  │                    │  - Invalid metadata                      │
│  └─────────┬──────────┘  - Required verbs check                  │
│            │                                                     │
│            ▼                                                     │
│  ┌────────────────────┐                                          │
│  │  5. VERB           │  Verb-specific validation                │
│  │     Validators     │  - Parameter type/count                  │
│  │                    │  - Attribute validation                  │
│  └─────────┬──────────┘  - Reference resolution                  │
│            │                                                     │
│            ▼                                                     │
│  COMPILE COMPLETE (if no fatal errors)                           │
│                                                                  │
│  ┌────────────────────┐                                          │
│  │  6. RUNTIME        │  Execution-time errors                   │
│  │     Errors         │  - Type mismatches                       │
│  │                    │  - Undefined variables                   │
│  └────────────────────┘  - Division by zero                      │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Diagnostic Structure

```
Diagnostic:
  level: DiagnosticLevel
  code: string                # Error code (e.g., "type_mismatch", "undefined_var")
  message: string             # Human-readable message
  storyName: string?          # Story where error occurred
  sourceLine: int?            # Original source line
  sourceColumn: int?          # Original source column
  context: string?            # Code snippet for context
  suggestion: string?         # Fix suggestion
  
DiagnosticLevel:
  INFO     = 0   # Informational, no action needed
  WARNING  = 1   # Potential issue, execution continues
  ERROR    = 2   # Real problem, execution continues
  FATAL    = 3   # Critical error, execution stops
```

---

## Story Validators

Story validators run after compilation, before execution:

### Label Validator

```
LabelValidator:
  validate(story: CompiledStory): List<Diagnostic>
    diagnostics = []
    seen = Map<string, int>()
    
    for (name, index) in story.labels:
      lowerName = name.toLowerCase()
      if lowerName in seen:
        diagnostics.add(Diagnostic {
          level: FATAL,
          code: "duplicate_label",
          message: "Duplicate label: @" + name,
          storyName: story.name,
          sourceLine: getSourceLine(story, index),
          suggestion: "Rename one of the duplicate labels"
        })
      seen[lowerName] = index
    
    return diagnostics
```

### Required Verbs Validator

```
RequiredVerbsValidator:
  validate(story: CompiledStory): List<Diagnostic>
    diagnostics = []
    
    # Check story metadata for required_verbs
    required = story.metadata["required_verbs"]
    if required == null:
      return []
    
    requiredSet = Set(required.elements.map(v => v.toString()))
    availableVerbs = runtime.getRegisteredVerbs()
    
    for verbName in requiredSet:
      if verbName not in availableVerbs:
        diagnostics.add(Diagnostic {
          level: FATAL,
          code: "missing_required_verb",
          message: "Required verb not available: /" + verbName,
          storyName: story.name,
          suggestion: "Register the verb driver or remove from required_verbs"
        })
    
    return diagnostics
```

### Jump Target Validator

```
JumpTargetValidator:
  validate(story: CompiledStory): List<Diagnostic>
    diagnostics = []
    
    for stmt in story.statements:
      if stmt is CompiledVerbCall and stmt.name in ["jump", "fork", "call"]:
        # Check if label exists (for same-story jumps)
        storyParam = stmt.params[0]
        labelParam = stmt.params[1]
        
        if storyParam.isNothing() or storyParam.toString() == "?":
          # Same-story jump, validate label at compile time if literal
          if labelParam is LiteralValue:
            labelName = labelParam.value.toLowerCase()
            if labelName not in story.labels:
              diagnostics.add(Diagnostic {
                level: WARNING,
                code: "unknown_jump_target",
                message: "Jump to unknown label: @" + labelParam.value,
                storyName: story.name,
                sourceLine: stmt.sourceLine,
                suggestion: "Create label @" + labelParam.value
              })
    
    return diagnostics
```

---

## Standard Diagnostics

Per spec, these diagnostics may be returned by **any verb at runtime**. They are generated by verb drivers during execution, not at compile time (since parameter types are only known after resolution).

### Runtime Generation

Verb drivers use the diagnostic helpers to return standard errors:

```
# In any verb driver:
if param is not expected_type:
    return fatal("type_mismatch", 
        "Expected " + expected + ", got " + param.getType())

if required_param.isNothing():
    return fatal("parameter_not_found", 
        "Required parameter not provided: " + param_name)

if unknown_attribute in call.attributes:
    return error("invalid_attribute", 
        "Unknown attribute: [" + attr_name + "]")
```

---

## Verb Validators

Each verb can have its own validator:

### Set Validator

```
SetVerbValidator:
  verbName = "set"
  
  validate(call: CompiledVerbCall, story: CompiledStory): List<Diagnostic>
    diagnostics = []
    
    if call.params.length == 0:
      diagnostics.add(Diagnostic {
        level: FATAL,
        code: "missing_parameter",
        message: "/set requires at least a variable name",
        sourceLine: call.sourceLine
      })
      return diagnostics
    
    varParam = call.params[0]
    if varParam is not StringValue and varParam is not ReferenceValue:
      diagnostics.add(Diagnostic {
        level: FATAL,
        code: "invalid_type",
        message: "/set first parameter must be string or reference",
        sourceLine: call.sourceLine
      })
    
    # Validate [typed] attribute
    typedAttr = getAttribute(call, "typed")
    if typedAttr != null:
      validTypes = ["string", "integer", "double", "boolean", 
                    "list", "map", "verb", "channel"]
      typeValue = typedAttr.value.toString()
      if typeValue not in validTypes:
        diagnostics.add(Diagnostic {
          level: FATAL,
          code: "invalid_attribute_value",
          message: "Invalid type for [typed]: " + typeValue,
          sourceLine: call.sourceLine,
          suggestion: "Use one of: " + join(validTypes, ", ")
        })
    
    return diagnostics
```

### Jump/Fork/Call Validator

```
JumpVerbValidator:
  validate(call: CompiledVerbCall, story: CompiledStory): List<Diagnostic>
    diagnostics = []
    
    if call.params.length < 2:
      diagnostics.add(Diagnostic {
        level: FATAL,
        code: "missing_parameter",
        message: "/" + call.name + " requires story and label parameters"
      })
    
    return diagnostics
```

---

## Runtime Error Handling

Runtime errors occur during execution:

```
Context.executeVerb(call: CompiledVerbCall):
  try:
    driver = findDriver(call)
    
    # Execute verb (returns ExecutionResult)
    result = driver.execute(call, this)
    
    # Handle diagnostics
    if result.diagnostics.hasFatal():
        state = FAULTED
        diagnostics.addAll(result.diagnostics)
        return
    
    # Store results
    lastDiagnostics = result.diagnostics
    lastReturnValue = result.value
    
  catch TypeMismatchError as e:
    diagnostics.add(Diagnostic {
      level: FATAL,
      code: "type_mismatch",
      message: "Type mismatch: " + e.message
    })
    state = FAULTED
    
  catch UndefinedVariableError as e:
    diagnostics.add(Diagnostic {
      level: FATAL,
      code: "undefined_variable",
      message: "Undefined variable: " + e.varName
    })
    state = FAULTED
    
  catch DivisionByZeroError as e:
    diagnostics.add(Diagnostic {
      level: FATAL,
      code: "division_by_zero",
      message: "Division by zero"
    })
    state = FAULTED
```

---

## Diagnostic Formatting

```
formatDiagnostic(d: Diagnostic): string
  prefix = match d.level:
    INFO: "[INFO]"
    WARNING: "[WARN]"
    ERROR: "[ERROR]"
    FATAL: "[FATAL]"
  
  location = ""
  if d.storyName:
    location = d.storyName + ":"
  if d.sourceLine:
    location += d.sourceLine.toString()
  if d.sourceColumn:
    location += ":" + d.sourceColumn.toString()
  
  result = prefix + " " + location + " - " + d.message
  
  if d.context:
    result += "\n    " + d.sourceLine.toString() + " | " + d.context
    if d.sourceColumn:
      result += "\n      | " + repeat(" ", d.sourceColumn - 1) + "^"
  
  if d.suggestion:
    result += "\n    Suggestion: " + d.suggestion
  
  return result
```

Example output:
```
[FATAL] my_story.zoh:42:15 - Undefined variable: *player_name
    42 | /converse "Hello, ${*player_name}!";
       |               ^
    Suggestion: Initialize the variable with /set before use
```

---

## Diagnostic Collection

```
DiagnosticCollector:
  diagnostics: List<Diagnostic>
  
  add(d: Diagnostic):
    diagnostics.add(d)
  
  hasErrors(): bool
    return any(d => d.level >= ERROR for d in diagnostics)
  
  hasFatal(): bool
    return any(d => d.level == FATAL for d in diagnostics)
  
  getByLevel(level: DiagnosticLevel): List<Diagnostic>
    return diagnostics.filter(d => d.level == level)
  
  report(): string
    return diagnostics.map(formatDiagnostic).join("\n")
```

---

## Testing Checklist

### Lexer Errors
- [ ] Unterminated string reported with line
- [ ] Invalid character reported
- [ ] Multi-error recovery

### Parser Errors
- [ ] Missing semicolon
- [ ] Unbalanced brackets
- [ ] Error recovery continues parsing

### Story Validation
- [ ] Duplicate labels detected
- [ ] Required verbs checked
- [ ] Jump targets warned

### Verb Validation
- [ ] Each core verb has validator
- [ ] Parameter count checked
- [ ] Parameter types checked
- [ ] Attribute values validated

### Runtime Errors
- [ ] Type mismatch caught
- [ ] Undefined variable caught
- [ ] Division by zero caught
- [ ] Fatal terminates context

### Diagnostic Quality
- [ ] Line numbers accurate
- [ ] Context snippets helpful
- [ ] Suggestions actionable
- [ ] Error codes documented
