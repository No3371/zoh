# 03: Preprocessor Implementation

## Purpose

The preprocessor handles text-level transformations before parsing. It processes `#embed`, `#macro`, `#expand`, and `#flag` directives by manipulating the raw source text.

---

## Processing Order

The preprocessor **is not monolithic**—it's a pipeline of modular handlers, each with a priority. The runtime calls them in order:

```
Runtime.preprocessors: List<Preprocessor>  # Ordered by priority
```

**Logical phases** (each can have multiple handlers):

```
┌──────────────────┐
│  Source Text     │
└────────┬─────────┘
         ▼
┌──────────────────┐
│  1. Strip BOM    │  (Optional: handle byte-order marks)
└────────┬─────────┘
         ▼
┌──────────────────┐
│  2. #embed       │  ← EmbedPreprocessor (can be extended/replaced)
└────────┬─────────┘
         ▼
┌──────────────────┐
│  3. #macro/#exp  │  ← MacroPreprocessor
│                  │    USER-DEFINED SYNTAX via text templates
└────────┬─────────┘
         ▼
┌──────────────────┐
│  4. Built-in     │  ← SugarPreprocessor
│   Sugar Transform│    (*var<-val → /set, ====> → /jump, etc.)
└────────┬─────────┘
         ▼
┌──────────────────┐
│  Tokenizer       │
└──────────────────┘
```

### Extensibility

Users can register custom preprocessors at any priority:

```
# Example: Register a custom preprocessor before built-in sugar
runtime.registerPreprocessor(MyCustomSyntax(), priority: 50)

# Built-in preprocessors might be at priorities like:
# - EmbedPreprocessor:  priority 100
# - MacroPreprocessor:  priority 200
# - SugarPreprocessor:  priority 300
```

> [!NOTE]
> **Custom syntactic sugar**: Users define custom patterns via `#macro` OR by registering 
> a custom `Preprocessor`. Macros are simpler (text templates); custom preprocessors give 
> full programmatic control. Both run before built-in sugar transformation.

---

## Directive Syntax

### #embed

```zoh
#embed "relative/path/to/file.zoh";
#embed "absolute/path/to/file.zoh";
```

- Path is relative to current file or absolute
- File content replaces the `#embed` line
- Each file can only be embedded once per resolution pass (cycle detection)
- Throws compile error if file not found

### Macro Definition

```zoh
|%MACRO_NAME%|
<macro body with placeholders>
|%MACRO_NAME%|
```

- Macro names are identifiers (case-sensitive)
- Placeholders: `|%N|` (index), `|%|` (auto-inc), `|%+N|` (relative), `|%-N|` (relative)
- Escape `|` as `\|`
- macros are local to the story file

### Macro Expansion

```zoh
|%MACRO_NAME%|
|%MACRO_NAME|arg0|arg1...|%|
```

- `|%MACRO_NAME%|` expands with no arguments
- `|%MACRO_NAME|...|%|` expands with positional arguments
- Positional placeholders replace arguments
- Unused arguments are ignored; missing arguments for placeholders result in replacement with `?` (nothing)

### #flag

```zoh
#flag flag_name value;
#flag [attr] flag_name value;
```

Syntactic sugar for `/flag "flag_name", value;`

---

## Implementation Steps

### Step 1: File Resolution

(No changes to file resolution logic)

### Step 2: Embed Processing

(No changes to embed processing logic)

### Step 3: Macro Collection

```
MacroDefinition:
  name: string
  body: string
  sourceFile: string
  sourceLine: int

collectMacros(source: string): (string, Map<string, MacroDefinition>)
    macros = Map<string, MacroDefinition>()
    result = StringBuilder()
    
    # Regex to capture macro definitions:
    # ^\s*\|%(\w+)%\|\s*$
    
    while processing lines:
        if line matches macro start:
            name = captured_name
            body = collect lines until matching |%NAME%| closure
            macros[name] = MacroDefinition { name, body, ... }
        else:
            result.append(line)
    
    return (result.toString(), macros)
```

### Step 4: Macro Expansion

```
expandMacros(source: string, macros: Map<string, MacroDefinition>): string
    result = StringBuilder()
    
    # Process text for expansions
    # Pattern: \|%(\w+)(?:\|(.*?))?\|%\|
    # Note: Regex must handle the "pipe or end" carefully
    
    for each match in source:
        name = match.name
        argsString = match.args_part
        
        if name not in macros:
             # If not a macro, maybe leave it (or error? Spec says error on unknown directive, but this is inline)
             # Implementation choice: Treat as text if not found, or error?
             # Spec implies it's a replacement.
             error("Unknown macro: " + name)
        
        args = split argsString by pipe `|` (respecting `\|` escape)
        
        expanded = expandMacroBody(macros[name].body, args)
        replace match with expanded
        
    return result.toString()

expandMacroBody(body: string, args: List<string>): string
    result = body
    
    # Replace placeholders
    # |%N| -> args[N]
    # |%| -> args[auto_inc_index]
    # |%+N| / |%-N| -> args[current_index +/- N]
    
    for each placeholder in body:
        targetIndex = resolveIndex(placeholder)
        replacement = args[targetIndex] if index in bounds else "?"
        result.replace(placeholder, replacement)
        
    # Handle escapes
    result.replace("\|", "|")
    
    return result
```

### Step 5: Syntactic Sugar Transformation

(No changes to sugar transformation logic)

---

## Placeholder Syntax

| Pattern | Meaning |
|---------|---------|
| `\|%N\|` | Argument at position N (0-indexed) |
| `\|%\|` | Next argument (auto-increment) |
| `\|%+N\|` | Relative: current + N |
| `\|%-N\|` | Relative: current - N |
| `\\\|` | Escaped pipe |

### Example

```zoh
|%DIALOG%|
/converse [By: "|%0|"] "|%1|";
|%DIALOG%|

|%DIALOG|Narrator|Hello!|%|
:: Becomes:
/converse [By: "Narrator"] "Hello!";

|%MultiArgs%|
/log "|%|";
/log "|%|";
|%MultiArgs%|

|%MultiArgs|First|Second|%|
:: Becomes:
/log "First";
/log "Second";
```

---

## Error Handling

### Compile-Time Errors

| Error | Condition |
|-------|-----------|
| Unterminated macro | `|%NAME%|` start without matching closing tag |
| Malformed expansion | `|%NAME|...` without closing `|%|` |

---

## Testing Checklist

### Macro
- [ ] Basic definition `|%NAME%|...|%NAME%|`
- [ ] No-arg expansion `|%NAME%|`
- [ ] Arg expansion `|%NAME|arg|%|`
- [ ] Positional placeholders `|%0|`
- [ ] Auto-inc placeholders `|%|`
- [ ] Relative placeholders `|%+1|`
- [ ] Escaped pipes `\|` in args
- [ ] Multiline arguments
- [ ] Missing args (replace with `?`)

---

## Source Mapping

To preserve original line numbers for error reporting:

```
SourceMap:
  # Maps transformed line → original file:line
  entries: List<SourceMapEntry>
  
SourceMapEntry:
  transformedLine: int
  originalFile: string
  originalLine: int

buildSourceMap(operations: List<PreprocessorOp>): SourceMap
    # Track how lines map as embeds/expansions occur
    # This enables accurate error messages post-preprocessing
```

---

## Testing Checklist

### #embed
- [ ] Basic embed: single file
- [ ] Nested embed: file A embeds B embeds C
- [ ] Circular embed detection: A embeds A
- [ ] Indirect circular: A embeds B embeds A
- [ ] File not found error
- [ ] Relative path resolution
- [ ] Absolute path resolution

### #macro
- [ ] Basic macro definition and expansion
- [ ] Multiple parameters
- [ ] Required placeholders
- [ ] Optional placeholders with defaults
- [ ] Escaped pipes in body
- [ ] Unterminated macro error
- [ ] Multiple macros in same file

### #expand
- [ ] All parameters provided
- [ ] Optional parameter omitted (uses default)
- [ ] Required parameter missing (error)
- [ ] Unknown macro name (error)
- [ ] Empty expansion

### Source Mapping
- [ ] Line numbers correct after embed
- [ ] Line numbers correct after macro expansion
- [ ] Error messages point to original source

---

## Output

The preprocessor outputs:
1. Transformed source text (ready for tokenization)
2. Source map (for error reporting)
3. List of diagnostics (warnings/errors encountered)

```
PreprocessorResult:
  source: string
  sourceMap: SourceMap
  diagnostics: List<Diagnostic>
```
