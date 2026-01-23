# Deep Think: Namespace Ambiguity Diagnostic Emission

## 1. Core Idea
**Objective**: Define the precise timing and mechanism for emitting `namespace_ambiguity` diagnostics.
**Proposed Change**: Implement suffix-based resolution logic that detects multiple matches and halts execution with a fatal error.

## 2. Context & Motivation
**Problem**: The spec introduces a "Suffix Matching" rule. A call like `/converse` could match both `std.converse` and `custom.converse`. This ambiguity must be detected and reported.
**Constraint**: The spec explicitly states "fatal diagnostic at runtime".

## 3. Approaches

### Option 1: Strict Compile-Time Validation (Pre-Check)
**Concept**: During the "Verb Validator" phase (post-compilation, pre-execution), iterate through all compiled verb calls and resolve them against the *currently registered* verbs.
**Pros**:
- Fails fast (before story starts).
- No runtime performance penalty (resolution happens once).
**Cons**:
- Assumes all verbs are registered *before* validation. If verbs can be loaded dynamically (e.g., plugins loaded after validation?), this might miss conflicts or flag false positives.

### Option 2: Just-In-Time Runtime Resolution (On-Demand)
**Concept**: When the runtime executes a `VerbCallNode`, it performs the suffix match against the *current* verb registry.
**Pros**:
- Handles dynamic verb registration perfectly.
- "Lazy" evaluation (only checks path actually taken).
**Cons**:
- Runtime crash (user playing might encounter it mid-game).
- Performance overhead (resolving every call every time).

### Option 3: Hybrid (Cached Resolution)
**Concept**: Resolve at first execution (or validation time if possible) and cache the result.
**Pros**: Balance of performance and correctness.
**Cons**: Complexity of cache invalidation if verbs change.

## 4. Analysis & Recommendation

**Spec Interpretation**: "fatal diagnostic at runtime" suggests it's a runtime error, preventing the specific ambiguous action from proceeding. However, failing *before* the story starts is generally a better developer experience (static analysis).

**Technical Viability**: 
- ZOH Runtime seems to have a `VerbRegistry`. 
- Suffix matching is computationally cheap but doing it on every tick/call for every user interaction is wasteful.

**Recommendation**: **Option 1 (Verb Validator)** is the robust choice for a compiled language feel.
- **Timing**: Triggered by `StandardStoryValidator` (or `VerbValidator`).
- **Emission**: Returns `fatal: namespace_ambiguity`.
- **Caveat**: If `impl/` allows dynamic verb registration *during* story execution, we must fallback to Option 2. Assuming verbs are static during a story session:
    - **Resolution**: `Validator` iterates all `VerbCallNode`s.
    - **Logic**: `Registry.FindVerbs(suffix)`. Count results.
    - **Result**: If >1, emit Fatal.

## 5. Execution Plan (for `spec-NS-001`)

1.  **Refine `impl/09_runtime.md`**: Define `VerbValidator` phase explicitly performing this check.
2.  **Algorithm**:
    ```csharp
    foreach (var call in story.VerbCalls) {
        var matches = registry.Find(call.Name);
        if (matches.Count > 1) Emit(Fatal, "namespace_ambiguity", ...);
        if (matches.Count == 0) Emit(Fatal, "unknown_verb", ...);
        // If 1, bind it (optimization: bake into compiled node?)
    }
    ```
3.  **Ambiguity Error Message**: Should list the conflicting FQNs (e.g., "Ambiguous verb '/run'. Matches: 'core.run', 'dry.run'. Please be more specific.").

## 11. Open Questions
1.  **Dynamic Loading**: Does ZOH support loading new verbs *while* a story is running? (Assume No for now, as it destabilizes specs).
2.  **Aliases**: Do aliases count as separate matches? (Yes, they are registered verbs/names).
