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

```
#embed "relative/path/to/file.zoh";
#embed "absolute/path/to/file.zoh";
```

- Path is relative to current file or absolute
- File content replaces the `#embed` line
- Each file can only be embedded once per resolution pass (cycle detection)
- Throws compile error if file not found

### #macro / #macro;

```
#macro MACRO_NAME param1, param2, param3;
<macro body with |#param1#| placeholders>
#macro;
```

- Macro names are identifiers (case-sensitive)
- Parameters are comma-separated identifiers
- Placeholders: `|#paramName#|` for required, `|#paramName|default#|` for optional
- Escape `|` as `\|`, `\` as `\\`
- Macros are local to the story file

### #expand

```
#expand MACRO_NAME;
#expand MACRO_NAME p1:"value1", p2:"value2";
```

- Named parameters replace placeholders
- Missing required placeholders cause compile error
- Optional placeholders use default if not provided

### #flag

```
#flag flag_name value;
#flag [attr] flag_name value;
```

Syntactic sugar for `/flag "flag_name", value;`

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

### Step 2: Embed Processing

```
processEmbeds(source: string, sourceFile: string, embedded: Set<string>): string
    result = StringBuilder()
    lines = source.split('\n')
    
    for line in lines:
        if matches(line, /^#embed\s+"(.+)"\s*;/):
            path = extractPath(line)
            absPath = resolve(path, sourceFile)
            
            if absPath in embedded:
                error("Circular embed: " + absPath)
            
            embedded.add(absPath)
            content = readFile(absPath)
            
            # Recursively process embeds in included file
            content = processEmbeds(content, absPath, embedded)
            result.append(content)
        else:
            result.append(line + '\n')
    
    return result.toString()
```

### Step 3: Macro Collection

```
MacroDefinition:
  name: string
  params: List<string>
  body: string
  sourceFile: string
  sourceLine: int

collectMacros(source: string): (string, Map<string, MacroDefinition>)
    macros = Map<string, MacroDefinition>()
    result = StringBuilder()
    lines = source.split('\n')
    
    i = 0
    while i < lines.length:
        line = lines[i]
        
        if matches(line, /^#macro\s+(\w+)\s*(.*);/):
            name, paramsStr = extractMacroHeader(line)
            params = parseParams(paramsStr)
            
            # Collect body until #macro;
            body = StringBuilder()
            i++
            while i < lines.length and not matches(lines[i], /^#macro\s*;/):
                body.append(lines[i] + '\n')
                i++
            
            if i >= lines.length:
                error("Unterminated macro: " + name)
            
            macros[name] = MacroDefinition { name, params, body.toString() }
        else:
            result.append(line + '\n')
        i++
    
    return (result.toString(), macros)
```

### Step 4: Macro Expansion

```
expandMacros(source: string, macros: Map<string, MacroDefinition>): string
    result = StringBuilder()
    lines = source.split('\n')
    
    for line in lines:
        if matches(line, /^#expand\s+(\w+)\s*(.*);/):
            name, argsStr = extractExpandCall(line)
            
            if name not in macros:
                error("Unknown macro: " + name)
            
            macro = macros[name]
            args = parseNamedArgs(argsStr)
            
            expanded = expandMacroBody(macro.body, macro.params, args)
            result.append(expanded)
        else:
            result.append(line + '\n')
    
    return result.toString()

expandMacroBody(body: string, params: List<string>, args: Map<string, string>): string
    result = body
    
    # Find all placeholders
    for match in findAll(body, /\|#(\w+)(?:\|([^#]*))?\#\|/):
        paramName = match.groups[0]
        defaultValue = match.groups[1]  # May be null
        
        if paramName in args:
            replacement = args[paramName]
        elif defaultValue != null:
            replacement = defaultValue
        else:
            error("Missing required parameter: " + paramName)
        
        result = result.replace(match.fullMatch, replacement)
    
    # Handle escapes
    result = result.replace("\\|", "|")
    result = result.replace("\\\\", "\\")
    
    return result
```

### Step 5: Syntactic Sugar Transformation

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
    # Flag:      #flag name val; →  /flag "name", val;
    
    # Implementation note: These transformations are complex
    # and parser-level handling is recommended. See 02_parser.md.
    
    return source
```

---

## Placeholder Syntax

| Pattern | Meaning |
|---------|---------|
| `\|#name#\|` | Required parameter |
| `\|#name\|default#\|` | Optional with default |
| `\\|` | Escaped pipe |
| `\\\\` | Escaped backslash |

### Placeholder Example

```
#macro DIALOG speaker, line;
/converse [By: |#speaker#|] |#line|"..."#|;
#macro;

#expand DIALOG speaker:"Narrator", line:"Hello!";
:: Becomes:
/converse [By: "Narrator"] "Hello!";

#expand DIALOG speaker:"Bob";
:: Becomes (using default):
/converse [By: "Bob"] "...";
```

---

## Error Handling

### Compile-Time Errors

| Error | Condition |
|-------|-----------|
| File not found | `#embed` path doesn't exist |
| Circular embed | File embeds itself (directly or indirectly) |
| Unterminated macro | `#macro` without closing `#macro;` |
| Unknown macro | `#expand` references undefined macro |
| Missing parameter | Required placeholder not provided |

### Error Message Format

```
[ERROR] file.zoh:15 - #embed: File not found: "missing.zoh"
[ERROR] file.zoh:20 - #macro: Unterminated macro "DIALOG"
[ERROR] file.zoh:30 - #expand: Unknown macro "TYPO_DIALOG"
[ERROR] file.zoh:35 - #expand: Missing required parameter "speaker" for macro "DIALOG"
```

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
