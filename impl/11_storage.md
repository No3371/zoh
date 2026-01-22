# 11: Persistent Storage Implementation

## Purpose

Persistent storage enables variables to survive across runtime sessions. The `/write`, `/read`, `/erase`, and `/purge` verbs manage persistent state.

---

## Storage Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        RUNTIME                                  │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                  PERSISTENT STORAGE                        │  │
│  │                                                            │  │
│  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐   │  │
│  │  │  Store: null  │  │  Store: "A"   │  │  Store: "B"   │   │  │
│  │  │  (default)    │  │               │  │               │   │  │
│  │  │               │  │               │  │               │   │  │
│  │  │  score: 100   │  │  chapter: 3   │  │  unlocked: T  │   │  │
│  │  │  name: "Kai"  │  │  time: 45.2   │  │               │   │  │
│  │  │  flags: [...] │  │               │  │               │   │  │
│  │  └───────────────┘  └───────────────┘  └───────────────┘   │  │
│  │                                                            │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Key Properties

| Property | Description |
|----------|-------------|
| Global | Accessible from any context/story |
| Concurrent-safe | Thread-safe operations |
| Persistent | Survives runtime restarts |
| Namespaced | Stores provide containers |

---

## Storage Interface

```
PersistentStorage:
  # Backend abstraction
  backend: StorageBackend
  
  # Operations
  write(store: string?, varName: string, value: Value): void
  read(store: string?, varName: string): Value?
  erase(store: string?, varName: string): void
  purge(store: string?): void
  exists(store: string?, varName: string): bool

StorageBackend:
  # Abstraction for different storage implementations
  save(store: string, data: Map<string, bytes>): void
  load(store: string): Map<string, bytes>?
  delete(store: string): void
```

---

## Store.Write

**Purpose**: Persist one or more variable values.

### Signature
```
/write store:name?, *var1, *var2, ...;
```

### Implementation

```
WriteDriver.execute(call, context):
    store = getNamedParam(call, "store")

    refs = call.params.filter(p => p is ReferenceValue)
    if refs.isEmpty():
        return fatal("parameter_not_found", "Write requires at least one reference parameter")

    for varRef in refs:
        varName = varRef.name
        value = context.get(varName)

        # Validate type (only serializable types)
        if value.getType() in ["verb", "channel", "expression"]:
            return fatal("invalid_type", "Cannot persist type: " + value.getType())

        storeName = store?.toString()
        serialized = serialize(value)

        try:
            context.runtime.storage.write(storeName, varName, serialized)
        catch IOError as e:
            return fatal("runtime_error", "Storage write failed: " + e.message)

    return ok()
```

### Serialization

```
serialize(value: Value): bytes
    match value.getType():
        "nothing":
            return encodeNothing()
        "boolean":
            return encodeBoolean(value.value)
        "integer":
            return encodeInteger(value.value)
        "double":
            return encodeDouble(value.value)
        "string":
            return encodeString(value.value)
        "list":
            return encodeList(value.elements.map(serialize))
        "map":
            entries = {}
            for (k, v) in value.entries:
                entries[k] = serialize(v)
            return encodeMap(entries)
```

---

## Store.Read

**Purpose**: Load one or more persisted variables.

### Signature
```
/read [required] [scope:story|context] store:name?, default:value?, *var1, *var2, ...;
```

### Implementation

```
ReadDriver.execute(call, context):
    store = getNamedParam(call, "store")
    defaultValue = getNamedParam(call, "default") ?? Nothing
    scope = getScope(call)

    refs = call.params.filter(p => p is ReferenceValue)
    if refs.isEmpty():
        return fatal("parameter_not_found", "Read requires at least one reference parameter")

    diagnostics = []

    for varRef in refs:
        varName = varRef.name
        storeName = store?.toString()

        # Read from storage
        try:
            serialized = context.runtime.storage.read(storeName, varName)
        catch IOError as e:
            return fatal("runtime_error", "Storage read failed: " + e.message)

        if serialized == null:
            diagnostics.add(info("not_found", "Variable not in storage: " + varName))
            if hasAttribute(call, "required"):
                return error("not_found", "Required variable not in storage: " + varName)
            value = resolve(defaultValue, context)
        else:
            value = deserialize(serialized)

        # Type check against existing variable
        existingVar = context.getVariable(varName)
        if existingVar != null and existingVar.isTyped:
            if value.getType() != existingVar.type:
                return fatal("type_mismatch", "Type mismatch: stored " + value.getType() +
                      ", variable expects " + existingVar.type)

        # Set the variable
        context.set(varName, value, Nothing, scope)

    return ok().withDiagnostics(diagnostics)
```

---

## Store.Erase

**Purpose**: Remove a variable from storage.

### Signature
```
/erase store:name?, *var;
```

### Implementation

```
EraseDriver.execute(call, context):
    store = getNamedParam(call, "store")
    varRef = call.params[0]

    if varRef is not ReferenceValue:
        return fatal("invalid_type", "Expected variable reference")

    storeName = store?.toString()

    try:
        exists = context.runtime.storage.exists(storeName, varRef.name)
        if not exists:
            return ok().withDiagnostics([info("not_found", "Variable not in storage: " + varRef.name)])
        context.runtime.storage.erase(storeName, varRef.name)
    catch IOError as e:
        return fatal("runtime_error", "Storage erase failed: " + e.message)

    return ok()
```

---

## Store.Purge

**Purpose**: Clear all variables from a store.

### Signature
```
/purge store:name?;
```

### Implementation

```
PurgeDriver.execute(call, context):
    store = getNamedParam(call, "store")
    storeName = store?.toString()

    try:
        context.runtime.storage.purge(storeName)
    catch IOError as e:
        return fatal("runtime_error", "Storage purge failed: " + e.message)

    return ok()
```

---

## Backend Implementations

### File-Based Backend

```
FileStorageBackend:
  basePath: string  # Storage directory
  
  getStorePath(store: string?): string
    storeName = store ?? "default"
    return join(basePath, storeName + ".zohstore")
  
  write(store, varName, value):
    path = getStorePath(store)
    data = loadOrCreate(path)
    data[varName] = serialize(value)
    writeFile(path, encode(data))
  
  read(store, varName):
    path = getStorePath(store)
    if not exists(path):
      return null
    data = decode(readFile(path))
    return data[varName]
  
  erase(store, varName):
    path = getStorePath(store)
    if not exists(path):
      return
    data = decode(readFile(path))
    data.remove(varName)
    writeFile(path, encode(data))
  
  purge(store):
    path = getStorePath(store)
    deleteFile(path)
```

### SQLite Backend

```
SQLiteStorageBackend:
  dbPath: string
  connection: SQLiteConnection
  
  init():
    connection = connect(dbPath)
    execute("CREATE TABLE IF NOT EXISTS storage (
      store TEXT NOT NULL,
      var_name TEXT NOT NULL,
      value BLOB,
      PRIMARY KEY (store, var_name)
    )")
  
  write(store, varName, value):
    store = store ?? ""
    execute("INSERT OR REPLACE INTO storage VALUES (?, ?, ?)",
            store, varName, serialize(value))
  
  read(store, varName):
    store = store ?? ""
    result = query("SELECT value FROM storage WHERE store=? AND var_name=?",
                   store, varName)
    if result.isEmpty():
      return null
    return deserialize(result[0])
  
  erase(store, varName):
    store = store ?? ""
    execute("DELETE FROM storage WHERE store=? AND var_name=?",
            store, varName)
  
  purge(store):
    store = store ?? ""
    execute("DELETE FROM storage WHERE store=?", store)
```

### In-Memory Backend (Testing)

```
MemoryStorageBackend:
  stores: Map<string, Map<string, bytes>>
  
  write(store, varName, value):
    store = store ?? ""
    if store not in stores:
      stores[store] = {}
    stores[store][varName] = value
  
  read(store, varName):
    store = store ?? ""
    if store not in stores:
      return null
    return stores[store][varName]
  
  # Optionally persist to disk on shutdown for testing
```

---

## Serialization Format

Use a simple, language-agnostic format. Options:

### Option 1: JSON-Based
```json
{
  "type": "integer",
  "value": 42
}

{
  "type": "list",
  "value": [
    {"type": "string", "value": "a"},
    {"type": "integer", "value": 1}
  ]
}
```

### Option 2: MessagePack/CBOR
Binary format, more compact, supports the same types.

### Type Tags

| Tag | Type |
|-----|------|
| 0 | nothing |
| 1 | boolean |
| 2 | integer |
| 3 | double |
| 4 | string |
| 5 | list |
| 6 | map |

---

## Concurrency Safety

Storage must be thread-safe:

```
RuntimeStorage:
  lock: RWMutex
  backend: StorageBackend
  
  write(store, varName, value):
    lock.writeLock()
    try:
      backend.write(store, varName, value)
    finally:
      lock.writeUnlock()
  
  read(store, varName):
    lock.readLock()
    try:
      return backend.read(store, varName)
    finally:
      lock.readUnlock()
```

---

## Testing Checklist

### Basic Operations
- [ ] Write variable to default store
- [ ] Read variable from default store
- [ ] Write/read from named store
- [ ] Erase single variable
- [ ] Purge entire store

### Repeating Parameters
- [ ] Write multiple variables in single call
- [ ] Read multiple variables in single call
- [ ] Mixed named params with multiple refs
- [ ] No reference params returns fatal error

### Type Handling
- [ ] All serializable types (nothing, bool, int, double, string, list, map)
- [ ] Non-serializable type error (verb, channel, expression)
- [ ] Type mismatch on read into typed variable

### Attributes
- [ ] `[required]` error on missing
- [ ] `[default]` fallback value

### Concurrency
- [ ] Concurrent reads
- [ ] Concurrent writes
- [ ] Read-write interleaving

### Persistence
- [ ] Values survive runtime restart
- [ ] Corrupted storage handling
