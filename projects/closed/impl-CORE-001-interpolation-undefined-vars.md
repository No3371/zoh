# IMPL-CORE-001: Relaxed Interpolation & Undefined Variables

**Based on**: `projects/spec-CORE-001-interpolation-undefined-vars.md`
**Spec Updates**: `spec.md` (Core.Interpolate)

## Context
Relax `Core.Interpolate` to allow undefined variables to resolve to `nothing` (and thus `?` or a fallback string) instead of raising a fatal error. This improves DX for narrative text generation.

## Implementation Changes

### 1. Interpolator Logic (`Core.Interpolate`)
- `Core.Interpolate` now accepts a `fallback` parameter (default `?`).
- The interpolator must wrap variable resolution in a way that catches "undefined variable" errors or checks existence first.
- **Strictness**: This relaxation ONLY applies to variable references directly inside `${...}`.
- **Valid**: `${*undefined}` -> `?`
- **INVALID**: `${*undefined + 1}` -> Crash (Expression evaluation remains strict).

#### Pseudocode
```
function Interpolate(template, fallback="?"):
    result = ""
    for segment in ParseTemplate(template):
        if segment is Text:
            result += segment.Value
        else if segment is Variable:
            value = TryGetVariable(segment.Name)
            if value is undefined:
                result += fallback
            else:
                result += Stringify(value)
    return result
```

### 2. Expression Bridge (`$(...)`)
- The `expr.md` defines `$(expression)`.
- **CRITICAL**: The expression inside `$(...)` is evaluated **strictly** by the Expression Evaluator.
- If the expression evaluation fails (e.g. `*undefined`), it crashes *before* interpolation logic sees it.
- If the expression evaluates successfully to a string (e.g. `$( "str" )`), that result is then treated as a string.

### 3. Native String Interpolation (`Core.Evaluate`)
- If the runtime supports native string interpolation in expressions (e.g. `Evaluate("Hello ${*name}")`), it should defer to `Core.Interpolate` logic, thus inheriting the leniency.

## Verification Scenarios
1. **Basic Missing**: `/"${*missing}"` -> `"?"`
2. **Fallback**: `/interpolate "${*missing}", fallback:"empty"` -> `"empty"`
3. **Strict Math**: ``/eval `*missing + 1` `` -> **Fatal Error**
4. **Strict Expression Wrapper**: ``/eval `$( *missing_str )` `` -> **Fatal Error** (Expression evaluation happens first).
    - *Note*: If the user wants safe interpolation in expressions, they must use `"string literal with ${*var}"`.
5. **Multiple**: `/"${*a}-${*b}"` where both missing -> `"?-?"`
