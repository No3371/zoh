# Debug Verb Spec Update Plan

> **Status:** Complete
> **Created:** 2026-02-19
> **Author:** Antigravity
> **Source:** Direct request
> **Related Projex:** N/A

---

## Summary

Update the `spec.md` to clarify that debug verbs (info, warning, etc.) perform single-pass string interpolation on their arguments. Also fixes incorrect syntax in the provided examples.

**Scope:** `spec.md` and `impl/06_core_verbs.md`.
**Estimated Changes:** 2 files, ~20 lines.

---

## Objective

### Problem / Gap / Need
- The current `Debug Verbs` section in `spec.md` does not specify that string messages should be interpolated.
- The examples use incorrect syntax (e.g., `{}` for interpolation instead of `${}`, and unnecessary backticks around string literals).

### Success Criteria
- [ ] `spec.md` explicitly states that string arguments are interpolated once.
- [ ] Examples use correct Zoh interpolation syntax (`${}`).
- [ ] Examples demonstrate intended usage clearly.

### Out of Scope
- Implementation changes in C# runtime (will be covered by a separate plan).

---

## Context

### Current State
`spec.md` currently says:
> - `message`: The message to emit. Accept `"string"`, `*"string"`, `` `expr` ``, or `` *`expr` ``. In case of reference, the value is used. In case of `` `expr` ``, the expression is evaluated.

Example:
```
/info `"Hello, world! {*user}!"`;
```

### Key Files
| File | Purpose | Changes Needed |
|------|---------|----------------|
| `spec.md` | Language Specification | Update `Debug Verbs` section |
| `impl/06_core_verbs.md` | Implementation Specification | Update `Debug Verbs` implementation logic |

### Dependencies
- **Blocks:** Implementation of debug verb interpolation in runtimes (e.g. C# `DebugDriver.cs`).

---

## Implementation

### Step 1: Update Debug Verbs Section

**Objective:** specificy interpolation behavior and fix examples.

**Files:**
- `spec.md`

**Changes:**

```markdown
// Before:
#### Parameters
- `message`: The message to emit. Accept `"string"`, `*"string"`, `` `expr` ``, or `` *`expr` ``. In case of reference, the value is used. In case of `` `expr` ``, the expression is evaluated.

#### Returns
A nothing.

#### Examples
```
/info `"Hello, world! {*user}!"`;
/info `"Hello, world! " + *user + "!"`;
/warning "Hello, world!";
/error "Hello, world!";
/fatal "Hello, world!";
```

// After:
#### Parameters
- `message`: The message to emit. Accept `"string"`, `*"string"`, `` `expr` ``, or `` *`expr` ``. In case of reference, the value is used. In case of string, the value is interpolated once. In case of `` `expr` ``, the expression is evaluated.

#### Returns
A nothing.

#### Examples
```
/info "Hello, world! ${*user}!";
/info `"Hello, world! " + *user + "!"`;
/warning "Hello, world!";
/error "Hello, world!";
/fatal "Hello, world!";
```

**Rationale:**
- Explicitly mentioning interpolation ensures consistent behavior across runtimes.
- Correcting example syntax avoids confusion.

**Verification:**
- Manual review of the generated markdown.

---

### Step 2: Update Debug Verbs Implementation Spec

**Objective:** Update implementation reference to include interpolation.

**Files:**
- `impl/06_core_verbs.md`

**Changes:**

```markdown
// Before:
DebugDriver.execute(call, context):
    message = resolve(call.params[0], context)
    
    if message is ExpressionValue:
        message = evaluate(message, context)
    
    severity = getSeverityFromVerbName(call.name)

// After:
DebugDriver.execute(call, context):
    message = resolve(call.params[0], context)
    
    if message is ExpressionValue:
        message = evaluate(message, context)
        
    # Interpolate if string
    if message is StringValue:
        message = interpolate(message.value, context)
    
    severity = getSeverityFromVerbName(call.name)
```

**Rationale:**
- Ensure implementation guide matches the spec.

**Verification:**
- Manual review.

---

## Verification Plan

### Manual Verification
- [ ] Read the updated `spec.md` to ensure clarity and correctness.

---

## Notes

### Risks
- Implementations might lag behind spec implies this feature won't work immediately until runtimes are updated.
