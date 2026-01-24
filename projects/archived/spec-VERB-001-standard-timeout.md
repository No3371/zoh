# spec-VERB-001: Standard Timeout Named Parameter

## Priority
**Medium** - Standardizes behavior for blocking operations, which is critical for robust application logic and user experience.

## Problem

Currently, "timeout" functionality is implemented inconsistently across verbs:
- `Core.Pull` and `Core.Wait` use a named parameter `timeout`.
- User interactions like `Std.Prompt`, `Std.Choose`, and `Std.Converse` (when waiting) block execution indefinitely until user action, with no mechanism to enforce time limits (e.g., timed decisions).
- The `info:timeout` diagnostic is defined in `spec.md` but not consistently used.

## Proposed Change

Standardize the usage of the `timeout` **named parameter** across user interaction verbs.

### The `timeout` Parameter
- **Definition**: A named parameter `timeout`. Accept `double`, `*double`, or `?`.
- **Value**: The duration in seconds before the blocking operation is cancelled.
- **Behavior**: 
    1. If the value is `<= 0`, the operation times out immediately.
    2. When the timeout duration elapses:
        - The blocking operation is interrupted/cancelled immediately.
        - Any partial input or pending selection is discarded.
        - The return value is **not modified** by the timeout mechanism (it is entirely up to the specific verb implementation to determine what to return).
        - The verb emits an `info` diagnostic with code `timeout`.

### Affected Verbs

The following verbs will support the `timeout` named parameter:

#### Core Verbs
1. **`Core.Pull`**: Already supports `timeout`.
2. **`Core.Wait`**: Already supports `timeout`.

#### Standard Verbs
3. **`Std.Prompt`**: Adds `timeout` parameter. 
    - on timeout: partial text input is discarded.
4. **`Std.Choose`**: Adds `timeout` parameter. 
    - on timeout: no default option is selected; manual handling required.
5. **`Std.ChooseFrom`**: Adds `timeout` parameter.
    - on timeout: no default option is selected; manual handling required.
6. **`Std.Converse`**: Adds `timeout` parameter, effective only when `Wait` is true (either via flag or attribute).

## Spec Updates

### `spec.md`
- Update `Standard Diagnostics` to explicitly link `info:timeout` to the `timeout` parameter behavior.

### `impl/10_std_verbs.md`
- **`Std.Prompt`**: Update signature and implementation.
- **`Std.Choose`** / **`Std.ChooseFrom`**: Update signature and implementation.
- **`Std.Converse`**: Update signature and implementation.

## Acceptance Criteria

1. **Standard Verbs**: `/prompt`, `/choose`, `/chooseFrom`, `/converse` accept `timeout` named parameter and behave correctly (emit `info:timeout`, handle <=0 immediately).
2. C# implementation plan is updated to reflect these changes.
