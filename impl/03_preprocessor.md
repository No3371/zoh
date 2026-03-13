# 03: Preprocessor Implementation

## Purpose

The preprocessor handles text-level transformations before parsing. It processes `#embed` and `|%macro%|` directives by manipulating the raw source text.

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
#embed "${filename}.${locale}.zoh";
#embed? "${filename}.${locale}.zoh";
```

- Path is relative to current file or absolute
- File content replaces the `#embed` / `#embed?` line
- Each file can only be embedded once per resolution pass (cycle detection)
- `#embed`: throws compile error if file not found
- `#embed?`: silently removes the directive if file not found (other errors remain fatal)
- Paths may contain `${name}` interpolation — resolved before file lookup (see Step 2 below)

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
|%MACRO_NAME|%|
|%MACRO_NAME|arg0|arg1...|%|
```

- `|%MACRO_NAME%|%|` expands with no arguments
- `|%MACRO_NAME|...|%|` expands with positional arguments
- Positional placeholders replace arguments
- Unused arguments are ignored; missing arguments for placeholders result in replacement with an empty string

### Multiline Parameters

Parameters can span multiple lines. Each continuation line starts with `|`:

```zoh
|%MACRO|arg1
|arg2
|arg3|%|

or

|%MACRO|arg1
|arg2
|arg3
|%|
```

This is equivalent to `|%MACRO|arg1|arg2|arg3|%|`.

---

## Implementation Steps

### Step 1: File Resolution

```
FileResolver:
  basePath: string              # Path of currently processing file
  embeddedFiles: Set<string>    # Already embedded (for cycle detection)
  
  resolve(path: string): string
    if isAbsolute(path):
        return normalizePath(path)
    return normalizePath(join(basePath, path))
  
  read(path: string): string
    resolved = resolve(path)
    if resolved in embeddedFiles:
        error("Circular embed detected: " + resolved)
    return readFile(resolved)
```

### Step 2: Embed Path Interpolation

Before file resolution, `${name}` placeholders in embed paths are replaced with values from the following sources, checked in order:

```
BuiltInVars:
  filename: string    # Base name of current file without extension

InterpolationContext:
  builtIns: BuiltInVars
  runtimeFlags: Map<string, Value>    # From runtime-scoped flags
  metadata: Map<string, string>       # From story header

interpolatePath(path: string, ctx: InterpolationContext): string
    return path.replaceAll(/\$\{(\w+)\}/, (match, name) =>
        if name in ctx.builtIns:
            return ctx.builtIns[name]
        if name in ctx.runtimeFlags:
            return toString(ctx.runtimeFlags[name])
        if name in ctx.metadata:
            return ctx.metadata[name]
        return ""   # Unknown name resolves to empty string
    )
```

Resolution order: built-in variables > runtime-scoped flags > story metadata > empty string.

Built-in variables cannot be shadowed by flags or metadata of the same name.

### Step 3: Embed Processing

```
processEmbeds(source: string, sourceFile: string, embedded: Set<string>, ctx: InterpolationContext): string
    result = StringBuilder()
    lines = source.split('\n')

    # Update built-in for current file
    ctx.builtIns.filename = stripExtension(basename(sourceFile))

    for line in lines:
        if matches(line, /^#embed(\??)\s+"(.+)"\s*;/):
            optional = match[1] == "?"
            rawPath = match[2]

            # Interpolate variables in path
            path = interpolatePath(rawPath, ctx)
            absPath = resolve(path, sourceFile)

            if absPath in embedded:
                error("Circular embed: " + absPath)

            if not fileExists(absPath):
                if optional:
                    continue    # #embed? — silently skip
                else:
                    error("File not found: " + absPath)

            embedded.add(absPath)
            content = readFile(absPath)

            # Recursively process embeds in included file
            content = processEmbeds(content, absPath, embedded, ctx)
            result.append(content)
        else:
            result.append(line + '\n')

    return result.toString()
```

### Step 4: Macro Collection

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

### Step 5: Macro Expansion

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

        # Smart Symmetric Trimming
        # Remove min(leading_spaces, trailing_spaces) from both ends
        for each arg in args:
            L = countLeadingSpaces(arg)
            T = countTrailingSpaces(arg)
            trim = min(L, T)
            arg = substring(arg, trim, length - trim)
        
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
        replacement = args[targetIndex] if index in bounds else ""
        result.replace(placeholder, replacement)
        
    # Handle escapes
    result.replace("\|", "|")
    
    return result
```

### Indentation Preservation

The macro expansion preserves the indentation of the call site. All lines in the expanded body are prefixed with the whitespace found before the `|%` token.

```zoh
    |%LOGIC|%|
```

If `LOGIC` macro body is:
```zoh
/if *condition,
    /do_something;
;
```

Expanded result:
```zoh
    /if *condition,
        /do_something;
    ;
```

### Step 6: Syntactic Sugar Transformation

Transform sugar forms to standard verb calls at text level (or defer to parser):

```
transformSugar(source: string): string
    # These can also be handled by parser, but text-level is simpler
    
    # Set sugar: *var <- value;  →  /set "var", value;
    # Get sugar: <- *var;        →  /get "var";
    # Capture:   -> *var;        →  /capture "var";
    # Jump:      ====> @label;   →  /jump ?, "label";
    # Fork:      ====+ @label;   →  /fork ?, "label";
    # Call:      <===+ @label;   →  /call ?, "label";
    
    # Implementation note: These transformations are complex
    # and parser-level handling is recommended. See 02_parser.md.
    
    return source
```

---

## Placeholder Syntax

| Pattern | Meaning |
|---------|---------|
| `\|%N\|` | Argument at position N (0-indexed) |
| `\|%\|` | Next argument (auto-increment) |
| `\|%+N\|` | Relative: current + N |
| `\|%-N\|` | Relative: current - N |
| `\\\|` | Escaped pipe |
| `\%` | Escaped percent (literal `%`) |

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
- [ ] Basic embed: single file (static path)
- [ ] Nested embed: file A embeds B embeds C
- [ ] Circular embed detection: A embeds A
- [ ] Indirect circular: A embeds B embeds A
- [ ] File not found error (`#embed`)
- [ ] Relative path resolution
- [ ] Absolute path resolution
- [ ] Interpolation: `${filename}` resolves to current file base name
- [ ] Interpolation: `${flag}` resolves from runtime-scoped flags
- [ ] Interpolation: `${meta}` resolves from story metadata
- [ ] Interpolation: unknown `${name}` resolves to empty string
- [ ] Interpolation: resolution order (built-in > flag > metadata)
- [ ] Optional embed: `#embed?` skips silently when file not found
- [ ] Optional embed: `#embed?` still errors on circular embed
- [ ] Optional embed with interpolation: `#embed? "${filename}.${locale}.zoh";`
- [ ] Mixed: static `#embed` and interpolated `#embed?` in same file

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
