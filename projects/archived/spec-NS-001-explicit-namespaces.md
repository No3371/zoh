# spec-NS-001: Explicit Namespace Resolution

## Priority
**High** - Matches currently staged spec changes. Critical for parser/runtime correctness.

## Problem
The spec previously allowed "up to one `.`" in identifiers and had loose rules about namespace resolution, leading to potential ambiguity between `namespace.verb` and `verb_with_dot`.
The user has updated `spec.md` to:
1.  Require explicit namespaces when ambiguous.
2.  Remove allowance of `.` in names (reserving it for namespaces).
3.  Add `namespace_ambiguity` diagnostic.

We need to propagate these changes to the Implementation Guides (`impl/`) and ensure consistency.

## Proposed Change

### Spec Alignment
- **Resolution Rule (Suffix Matching)**:
    - Symbols are resolved by matching the provided name as a **suffix** against registered verbs' FQNs (Namespace + Name).
    - **Unique Match**: If the suffix matches exactly one referenced symbol (verb or attribute), it resolves to that symbol.
    - **Ambiguity**: If multiple symbols match the suffix, it results in a `namespace_ambiguity` fatal diagnostic.
    - **No Match**: If no symbols match, it raises a standard `unknown_verb` or `unknown_attribute`.
    - **Attributes**: Verification logic applies equally to Attributes. Attributes must be unambiguous relative to registered global/standard attributes.

### Implementation Approach: Story Validator
Based on [Deep Think Analysis](../discussions/deep-think-NS-001-ambiguity.md), `namespace_ambiguity` will be enforced at **Compile Time** (Post-Compilation, Pre-Execution) via a specific **Validator**.

- **Phase**: Validation Phase (`StandardStoryValidator` -> `NamespaceValidator`).
- **Timing**: After `CompiledStory` is generated, before `Runtime.run(context)`.
- **Logic**: 
    1. Iterate all `CompiledVerbCall` nodes in the story.
    2. **Verbs**: Perform Suffix Matching against the **current** Runtime Verb Registry.
    3. **Attributes**: Iterate all `attributes` in the call. Perform Suffix Matching against the **current** Runtime Attribute Registry (or Standard Attributes).
    4. If >1 match found (verb or attribute): Emit `fatal: namespace_ambiguity`.
    5. If 0 match found (verb): Emit `fatal: unknown_verb`. (Attributes are optional/data, so unknown attributes might be warnings or ignored depending on strictness?). *Refinement: Spec says attributes are data, but if they look like a namespace and match nothing, is it an error? If they match >1, it IS an error.*

### Examples

**Scenario 1: Simple Ambiguity**
Registered:
- `std.converse`
- `custom.converse`

Calls:
- `/converse` -> **Ambiguous** (matches both).
- `/std.converse` -> Resolves to `std.converse`.
- `/custom.converse` -> Resolves to `custom.converse`.

**Scenario 2: Nested Resolutions**
Registered:
- `app.utils.log`
- `sys.log`

Calls:
- `/log` -> **Ambiguous** (matches both).
- `/utils.log` -> Resolves to `app.utils.log` (unique match).
- `/app.utils.log` -> Resolves to `app.utils.log`.
- `/sys.log` -> Resolves to `sys.log`.

**Scenario 3: Partial Overlap**
Registered:
- `core.run`
- `dry.run`

Calls:
- `/run` -> **Ambiguous**.
- `/core.run` -> Resolves to `core.run`.
- `/dry.run` -> Resolves to `dry.run`.
- `/y.run` -> **Not Found** (Suffix must match full namespace segments, e.g., `dry` is a segment, `y` is not). 
    - *Clarification*: Suffix matching operates on dot-delimited segments, not string suffices. `dry.run` ends with `run` and `dry.run`. It does not match `y.run`.

### Affected Areas
- **Spec**: (Already staged by user). Check for `namespace_ambiguity` definition.
- **Impl Guides**:
    - `impl/02_parser.md`: Update identifier parsing rules (stop at `.`). Update Verb Call parsing to handle `ns.verb`.
    - `impl/05_type_system.md`: Update Reference resolution logic.
    - `impl/09_runtime.md`: Update lookup logic and error handling. Define `AttributeValidator` logic suitable for `namespace_ambiguity`.
    - `impl/09_runtime.md`: Update lookup logic and error handling. Define `AttributeValidator` logic suitable for `namespace_ambiguity`.

## Acceptance Criteria
1.  `spec.md`: `namespace_ambiguity` diagnostic is defined.
2.  `impl/02_parser.md`: Identifier parsing excludes `.`. Namespace parsing logic updated.
3.  `impl/05_type_system.md` & `impl/09_runtime.md`: Resolution logic handles explicit namespaces and checks ambiguity (Verb & Attribute).
3.  `impl/05_type_system.md` & `impl/09_runtime.md`: Resolution logic handles explicit namespaces and checks ambiguity (Verb & Attribute).
