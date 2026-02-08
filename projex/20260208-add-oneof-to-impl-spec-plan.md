# Plan: Add OneOf to Implementation Spec

> **Status:** Ready
> **Created:** 2026-02-08
> **Author:** Antigravity
> **Source:** `projex/20260208-verify-type-system.md` (Major Issue)
> **Related Projex:** `c#/projex/20260208-add-oneof-to-runtime-plan.md` (Derivative)

---

## Summary

Documents the `OneOf` attribute as a persistent constraint in `impl/05_type_system.md`. This aligns the impl spec with `spec.md` L485.

**Scope:** `impl/05_type_system.md`
**Estimated Changes:** 1 file

---

## Objective

### Problem / Gap / Need
`impl/05_type_system.md` describes `Variable` with only `type` constraint. The spec defines `OneOf` for `/set` but the impl doc doesn't show how to persist it.

### Success Criteria
- [ ] `Variable` structure includes `oneOf: List<Value>?`
- [ ] `set` pseudocode validates against `oneOf` on updates

### Out of Scope
- C# runtime implementation (separate plan)

---

## Implementation

### Step 1: Update Variable Structure

**File:** `impl/05_type_system.md` (L307-310)

**Before:**
```
Variable:
  value: Value
  type: string?          # Optional type constraint
  isTyped: bool          # Once typed, type is locked
```

**After:**
```
Variable:
  value: Value
  type: string?          # Optional type constraint
  isTyped: bool          # Once typed, type is locked
  oneOf: List<Value>?    # Optional value constraint
```

### Step 2: Update set() Logic

**File:** `impl/05_type_system.md` (L320-330)

**Before:**
```
set(name: string, value: Value, scope: string = "story")
  name = name.toLowerCase()
  target = scope == "story" ? storyVars : contextVars
  
  if name in target:
    var = target[name]
    if var.isTyped and getType(value) != var.type:
      error("Type mismatch: expected " + var.type)
    var.value = value
  else:
    target[name] = Variable { value, type: null, isTyped: false }
```

**After:**
```
set(name: string, value: Value, scope: string = "story", oneOf: List<Value>? = null)
  name = name.toLowerCase()
  target = scope == "story" ? storyVars : contextVars
  
  if name in target:
    var = target[name]
    if var.isTyped and getType(value) != var.type:
      error("Type mismatch: expected " + var.type)
    if var.oneOf and value not in var.oneOf:
      error("Value not in allowed list")
    var.value = value
  else:
    target[name] = Variable { value, type: null, isTyped: false, oneOf: oneOf }
```

---

## Verification Plan

- [ ] Review pseudocode for consistency with spec.md L485
