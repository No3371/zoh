# Walkthrough: spec-VERB-001-standard-timeout

**Status**: Completed & Verified
**Date**: 2026-01-23

## 1. Objective

Standardized the usage of the `timeout` named parameter across blocking user interaction verbs. Previously inconsistent or missing, now `Std.Prompt`, `Std.Choose`, `Std.ChooseFrom`, and `Std.Converse` support a `timeout` parameter, behaving similarly to `Core.Pull` and `Core.Wait`.

## 2. File Changes

### `spec.md`

| Location | Change |
|----------|--------|
| Standard Diagnostics | Updated `info:timeout` description to explicitly mention the `timeout` parameter. |

### `impl/10_std_verbs.md`

Added `timeout` parameter and logic to:

| Verb | Change |
|------|--------|
| **`Std.Converse`** | Added `timeout` param. Logic added to check `timeout <= 0` for immediate return. Calls `runtime.waitForContinue` with timeout value. |
| **`Std.Choose`** | Added `timeout` param. Logic added for immediate return. Calls `runtime.presentChoice` with timeout value. Returns `info:timeout` on timeout (-1 index). |
| **`Std.ChooseFrom`** | Added `timeout` param. Same logic as `Std.Choose`. |
| **`Std.Prompt`** | Added `timeout` param. Logic added for immediate return. Calls `runtime.presentPrompt` with timeout value. Returns `info:timeout` on timeout (null result). |

**Runtime Interface Updates**:
The pseudo-code implies updates to the Runtime interface signatures to accept `timeout: double?`:
- `waitForContinue(context, timeout)`
- `presentChoice(request, context, timeout)`
- `presentPrompt(request, context, timeout)`

## 3. Verification

Verified that:
1. `info:timeout` is documented in `spec.md`.
2. All target verbs in `impl/10_std_verbs.md` have updated signatures.
3. Logic for `timeout <= 0` returns `info("timeout", "Immediate timeout")`.
4. Logic for runtime timeout returns `info("timeout", "Operation timed out")`.
5. Return values on timeout are consistent with "irrelevant/manual handling" (returning the diag info implies `?` value in many implementation contexts or explicitly handling the diag).

## 4. Key Insights

- **Manual Timeout Handling**: By returning `info:timeout`, we allow the runtime/script to handle what happens next, rather than forcing a specific default value (like `0` or `false`).
- **Immediate Timeout**: The `>= 0` check is crucial for robust math, preventing infinite waits if a calculation goes wrong.
- **Runtime Interface**: This change propagates to the Runtime interface, so concrete implementations (C#, Unity, etc.) will need to update their `IZohRuntime` or equivalent interfaces.

## Addendum 2026-01-24

### `std_verbs.md`

Updated the high-level standard verbs documentation which was initially missed.

| Verb | Change |
|------|--------|
| **`Std.Converse`** | Added `timeout` named parameter and `Info: timeout` diagnostic. |
| **`Std.Choose`** | Added `timeout` named parameter and `Info: timeout` diagnostic. |
| **`Std.ChooseFrom`** | Added `timeout` named parameter and `Info: timeout` diagnostic. |
| **`Std.Prompt`** | Added `timeout` named parameter and `Info: timeout` diagnostic. |
