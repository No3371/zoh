# spec-CORE-001-interpolation-undefined-vars: Non-Fatal Undefined Variables in Interpolation

## Priority
**High** - Strict interpolation crashes are a major pain point for content authors and debugging, making the language fragile for text generation tasks.

## Problem

Currently, string interpolation (e.g., `/${*name};`) is strict: if `*name` is not defined, the runtime raises a fatal `undefined_var` error, terminating the story context.

This strictness is often undesirable for text generation, debug logging, or dynamic content where optional variables might be missing. Unlike logic evaluation (where `undefined + 1` should fail), string interpolation is often used for display, where showing a placeholder or `?` is preferable to a crash.

- `spec.md` (Core.Interpolate section) currently implies strict evaluation via `ExpressionEvaluator`.
- `ExpressionEvaluator.cs` throws `InvalidOperationException` for undefined variables.

## Proposed Change

Relax the diagnostics for `Core.Interpolate` (and related interpolation syntax) to treat undefined variables as `nothing` (stringifying to `?`) instead of raising a fatal `undefined_var` error.

Undefined variables referenced within interpolation syntax (variable references in `${}`) should resolve to `nothing`. Since `nothing` stringifies to `"?"`, the interpolation will simple output `?` (or a fallback) instead of crashing.

This applies to:
- `Core.Interpolate` verb.
- Syntactic sugar `${*var}` in strings.
- Interpolation special form `$(*var)` within expressions (when used for string interpolation).

**Note:** `Core.Evaluate` (logic expressions) remains strict. This proposal ONLY affects interpolation.

**Clarification on `$(expression)`**: The expression wrapper `$(...)` in `expr.md` evaluates the expression *strictly* before passing the result to interpolation. Therefore, `$(*undefined)` will still crash with `undefined_var`, but `$("${*undefined}")` will safely return `?`.

### New Parameter: `fallback`

Add a named parameter `fallback` to `Core.Interpolate`:
- **Name**: `fallback`
- **Type**: `string`
- **Default**: `"?"`
- **Description**: The string to use when a variable resolves to `nothing` (either because it was explicitly `nothing` or because it was undefined).

## Spec Updates

### `spec.md`

#### `Core.Interpolate`
- Update behavior description to state that undefined variables resolve to `nothing` instead of erroring.
- Add `fallback` parameter documentation.
- Add example:
  ```zoh
  :: *missing is undefined
  /interpolate "Hello ${*missing}"; -> "Hello ?";
  /interpolate "Hello ${*missing}", fallback: "stranger"; -> "Hello stranger";
  ```


## Acceptance Criteria

1. **Undefined Variable in Interpolation**: `${*undefined}` inside a string should evaluate to `?` (or `nothing`'s string representation) without error.
2. **Fallback Parameter**: `Core.Interpolate` with `fallback: "default"` should return "default" if the interpolated variable is undefined or `nothing`.
3. **Strict Logic Preserved**: `*undefined + 1` in a normal `/evaluate` or logic context MUST still raise `undefined_var` error.

## Challenge Questions (2026-01-24)

1. **Inline Fallback**: Should we support C#-style null coalescing or similar syntax inside the interpolation itself (e.g., `${*name ?? "Stranger"}`) to allow per-variable fallbacks instead of just a global `fallback` param?
A: NO, we have `$?{*var1|fallback}`
2. **Explicit Strictness**: If a user *wants* to ensure a variable exists for interpolation (to catch typos), should `Core.Interpolate` support the `[required]` attribute to force `undefined_var` errors?
A: NO
3. ~~**`expr.md` Alignment**: Does the leniency of `$(*var)` in expressions extend to `$(expression)`? If I write `$((*a + 1))` and `*a` is undefined, does it resolve to `?` or crash? (Since it involves math which is strict).~~
4. **Fallback Recursion**: Does the `fallback` string undergo interpolation? If `fallback: "See ${*other}"`, is it processed? (Likely not to avoid complexity, but worth defining).
A: NO
5. **Invalid Index vs Undefined**: If specific array index is out of bounds (`${*list[100]}`), does this also resolve to `nothing`/fallback, or does it throw `invalid_index`? (Aligning with `Core.Get` implies `nothing`, but worth confirming).
A: YES
6. **Double vs Nothing**: If `*var` exists but is explicitly `nothing` (`/set *var;`), does it use the `fallback`? The proposal says yes ("resolves to `nothing` (either because it was explicitly `nothing` or... undefined)"), which is consistently handled.
7. ~~**Expression String Literals**: Does `/evaluate "User: ${*missing}"` stay strict? (String literals in expressions typically don't interpolate unless wrapped in `$(...)`, so they are just strings. But if `evaluate` supported native string interpolation, would it be strict?)~~
8. **Multiple Missing**: If `fallback` is `"foo"`, does `"A=${*miss1}, B=${*miss2}"` become `"A=foo, B=foo"`? (Implies yes, per-substitution).
A: YES
9. **Zero Values**: Confiming that `0`, `false`, `""` do NOT trigger fallback?
A: Only `nothing`, and `nothing` should be the only possibility
10. ~~**Performance**: Does this imply a `try-catch` block around every variable resolution in the interpolator, and does that introduce overhead compared to `ContainsKey` checks?~~

