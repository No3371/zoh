# 05: Type System

## Purpose

ZOH uses dynamic typing with optional type annotations. This document covers variable types, storage, scoping, and type operations.

---

## Variable Types

| Type | Literal | Example | Notes |
|------|---------|---------|-------|
| `nothing` | `?` | `?` | Default/null value |
| `boolean` | `true`/`false` | `true` | Case-insensitive |
| `integer` | `[+-]?\d+` | `42`, `-7` | 64-bit signed |
| `double` | `[+-]?\d+\.\d+` | `3.14` | 64-bit IEEE 754 |
| `string` | `"..."` or `'...'` | `"hello"` | UTF-8 |
| `list` | `[...]` | `[1, 2, 3]` | Ordered, mixed-type |
| `map` | `{...}` | `{"a": 1}` | String-keyed |
| `channel` | `<...>` | `<events>` | Runtime-managed |
| `verb` | `/verb ...;` | `/converse;` | Objectified verb call |
| `expression` | `` `...` `` | `` `*a + 1` `` | Unevaluated expression |
| `reference` | `*name` | `*score` | Pointer to variable |

---

## Type Hierarchy

```
Value
├── Nothing             (singleton)
├── Boolean             (true/false)
├── Number
│   ├── Integer        (i64)
│   └── Double         (f64)
├── String             (UTF-8)
├── Collection
│   ├── List           (ordered, 0-indexed)
│   └── Map            (string-keyed)
├── Channel            (runtime-managed)
├── Verb               (objectified call)
├── Expression         (unevaluated)
└── Reference          (pointer)
```

---

## Value Representation

### Interface

```
Value:
  getType(): string           # "nothing", "boolean", "integer", etc.
  isNothing(): bool
  toBool(): bool              # For conditionals
  toInt(): int?               # Type conversion
  toDouble(): double?
  toString(): string
  equals(other: Value): bool
  clone(): Value              # Deep copy
```

### String Formatting Helpers

```
formatElement(value: Value): string
  # Format a value for use in collection toString()
  # Strings are quoted, others use their toString()
  if value is StringValue:
    return "\"" + escapeString(value.value) + "\""
  return value.toString()

escapeString(s: string): string
  # Escape special characters for string representation
  return s.replace("\\", "\\\\")
          .replace("\"", "\\\"")
          .replace("\n", "\\n")
          .replace("\t", "\\t")
```

### Nothing

```
NothingValue:
  # Singleton - only one instance
  getType() = "nothing"
  isNothing() = true
  toBool() = false
  toString() = "?"
```

### Boolean

```
BoolValue:
  value: bool
  getType() = "boolean"
  toBool() = value
  toInt() = value ? 1 : 0
  toString() = value ? "true" : "false"
```

### Numbers

```
IntegerValue:
  value: int64
  getType() = "integer"
  toBool() = value != 0
  toInt() = value
  toDouble() = value as double
  toString() = value.toString()

DoubleValue:
  value: float64
  getType() = "double"
  toBool() = value != 0.0
  toInt() = truncate(value)     # Round toward zero
  toDouble() = value
  toString():
    # Always include decimal point to distinguish from integer
    # Use scientific notation for very large/small values
    if isNaN(value): return "NaN"
    if isInfinity(value): return value > 0 ? "Infinity" : "-Infinity"
    if value == floor(value):
      return formatInteger(value) + ".0"  # e.g., 42.0 not 42
    return formatDecimal(value)           # Minimum precision needed
```

### String

```
StringValue:
  value: string
  getType() = "string"
  toBool() = value.length > 0
  toInt() = parse if valid, else error
  toDouble() = parse if valid, else error
  toString() = value
  equals(other) = value == other.value  # Case-sensitive
```

### List

```
ListValue:
  elements: List<Value>
  getType() = "list"
  toBool() = elements.length > 0
  toString():
    # Format: [v1, v2, ...] with recursive conversion
    # String elements are quoted: ["a", "b"]
    parts = elements.map(e => formatElement(e))
    return "[" + parts.join(", ") + "]"

  # Operations
  get(index: int): Value
  set(index: int, value: Value)
  append(value: Value)
  insert(index: int, value: Value)
  remove(index: int): Value
  count(): int
  contains(value: Value): bool
  clone(): ListValue           # Deep clone elements
```

### Map

```
MapValue:
  entries: Map<string, Value>
  getType() = "map"
  toBool() = entries.size > 0
  toString():
    # Format: {"k1": v1, "k2": v2} with quoted keys and recursive values
    # Order is implementation-defined
    parts = entries.map((k, v) => "\"" + escapeString(k) + "\": " + formatElement(v))
    return "{" + parts.join(", ") + "}"

  # Operations
  get(key: string): Value?
  set(key: string, value: Value)
  remove(key: string): Value?
  count(): int
  hasKey(key: string): bool
  keys(): List<string>
  values(): List<Value>
  clone(): MapValue            # Deep clone entries
```

### Channel

```
ChannelValue:
  name: string                 # Unique identifier
  getType() = "channel"
  toString() = "<" + name + ">"
  # Actual queue managed by Runtime, not the value itself
```

### Verb

```
VerbValue:
  namespace: string?
  name: string
  attributes: List<Attribute>
  namedParams: Map<string, Value>
  unnamedParams: List<Value>
  getType() = "verb"
  toString():
    # Reconstruct verb syntax: /name attr params;
    result = "/" + (namespace ? namespace + "." : "") + name
    for attr in attributes:
      result += " [" + attr.name + (attr.value ? ":" + formatElement(attr.value) : "") + "]"
    for (k, v) in namedParams:
      result += " " + k + ":" + formatElement(v)
    for v in unnamedParams:
      result += " " + formatElement(v)
    return result + ";"

  # Verbs are compared by structure, not identity
  equals(other): deep structural comparison
```

### Expression

```
ExpressionValue:
  source: string               # Original text
  ast: ExprNode                # Parsed expression tree
  getType() = "expression"
  toString() = "`" + source + "`"

  # Expressions are compared by source text
  equals(other) = source == other.source
```

### Reference

```
ReferenceValue:
  name: string                 # Variable name
  path: List<Value>            # Path of indexes for nested access
  getType() = "reference"
  toString():
    result = "*" + name
    for idx in path:
      result += "[" + formatElement(idx) + "]"
    return result

  # References resolve to values at evaluation time
  resolve(context: Context): Value
```

---

## Variable Storage

### Scopes

```
┌───────────────────────────────────────────┐
│                  Runtime                   │
│  ┌─────────────────────────────────────┐  │
│  │              Channels               │  │ (Global, concurrent-safe)
│  └─────────────────────────────────────┘  │
│  ┌─────────────────────────────────────┐  │
│  │          Persistent Storage         │  │ (Cross-session)
│  └─────────────────────────────────────┘  │
│                                           │
│  ┌─────────────────────────────────────┐  │
│  │            Context A                │  │
│  │  context_var1, context_var2...      │  │ (Context-scoped)
│  │  ┌───────────────────────────────┐  │  │
│  │  │          Story X              │  │  │
│  │  │  story_var1, story_var2...    │  │  │ (Story-scoped)
│  │  └───────────────────────────────┘  │  │
│  └─────────────────────────────────────┘  │
└───────────────────────────────────────────┘
```

### Scope Behavior

| Scope | Lifetime | Visibility |
|-------|----------|------------|
| **Story** | Until context leaves story | Current story only |
| **Context** | Until context terminates | All stories in context |
| **Persistent** | Cross-session | All runtimes |

### Lookup Order

1. Story scope (shadows context if name conflicts)
2. Context scope
3. Return a nothing if not found

### Implementation

```
VariableStorage:
  # Per-context storage
  storyVars: Map<string, Variable>
  contextVars: Map<string, Variable>
  
  # Variable metadata
  Variable:
    value: Value
    type: string?          # Optional type constraint
    isTyped: bool          # Once typed, type is locked

  get(name: string): Value
    name = name.toLowerCase()   # Case-insensitive
    if name in storyVars:
      return storyVars[name].value
    if name in contextVars:
      return contextVars[name].value
    return Nothing

  set(name: string, value: Value, scope: string = "story")
    name = name.toLowerCase()
    target = scope == "story" ? storyVars : contextVars
    
    if name in target:
      var = target[name]
      if var.isTyped and getType(value) != var.type:
        error("Type mismatch: expected " + var.type)
      var.value = value
    else:
      target[name] = Variable { value, type: null, isTyped: false }

  setTyped(name: string, value: Value, type: string, scope: string = "story")
    name = name.toLowerCase()
    target = scope == "story" ? storyVars : contextVars
    target[name] = Variable { value, type, isTyped: true }

  drop(name: string)
    name = name.toLowerCase()
    storyVars.remove(name)
    contextVars.remove(name)

  clearStoryScope()
    storyVars.clear()
```

---

## Type Coercion Rules

### Implicit Coercion

| From | To | Rule |
|------|----|------|
| `integer` | `double` | Lossless promotion |
| `double` | `integer` | Truncate toward zero |
| `any` | `boolean` | Truthiness check |
| `any` | `string` | Format for display |

### Truthiness

| Type | Truthy | Falsy |
|------|--------|-------|
| `nothing` | - | Always falsy |
| `boolean` | `true` | `false` |
| `integer` | Non-zero | `0` |
| `double` | Non-zero | `0.0` |
| `string` | Non-empty | `""` |
| `list` | Non-empty | `[]` |
| `map` | Non-empty | `{}` |
| `channel` | Always truthy | - |
| `verb` | Always truthy | - |
| `expression` | Always truthy | - |

---

## Reference Resolution

References (`*var`) are resolved at evaluation time:

```
resolveReference(ref: ReferenceValue, context: Context): Value
    baseValue = context.getVariable(ref.name)
    
    if baseValue == Nothing:
        return Nothing
    
    if ref.index == null:
        return baseValue
    
    # Handle indexed access
    index = evaluate(ref.index, context)
    
    if baseValue is ListValue:
        if index is not Integer:
            error("List index must be integer")
        i = index.toInt()
        if i < 0:
            i = baseValue.count() + i  # Negative indexing
        return baseValue.get(i)
    
    if baseValue is MapValue:
        if index is not StringValue:
            error("Map index must be string, got: " + index.getType())
        return baseValue.get(index.value) ?? Nothing
    
    error("Cannot index type: " + baseValue.getType())
```

---

## Collection Operations

### List Operations

| Operation | Description |
|-----------|-------------|
| `get(i)` | Get element at index (0-based) |
| `set(i, v)` | Set element at index |
| `append(v)` | Add element to end |
| `insert(i, v)` | Insert at position |
| `remove(i)` | Remove by index |
| `count()` | Number of elements |
| `contains(v)` | Check membership |

### Map Operations

| Operation | Description |
|-----------|-------------|
| `get(k)` | Get value by key |
| `set(k, v)` | Set key-value pair |
| `remove(k)` | Remove by key |
| `count()` | Number of entries |
| `hasKey(k)` | Check key exists |
| `keys()` | List of all keys |
| `values()` | List of all values |

---

## Testing Checklist

### Type Storage
- [ ] Declare variable without value
- [ ] Declare with value, infer type
- [ ] Typed variable enforces type
- [ ] Type mismatch error

### Scopes
- [ ] Story-scoped variable isolation
- [ ] Context-scoped variable sharing
- [ ] Story scope shadows context scope
- [ ] Story scope cleared on story exit

### Type Coercion
- [ ] Integer to double
- [ ] Double to integer (truncation)
- [ ] All types to boolean (truthiness)
- [ ] All types to string

### Collections
- [ ] List create, read, update, delete
- [ ] List negative indexing
- [ ] Map create, read, update, delete
- [ ] Nested collections

### References
- [ ] Simple reference resolution
- [ ] Indexed reference (list)
- [ ] Indexed reference (map)
- [ ] Undefined variable returns a nothing
