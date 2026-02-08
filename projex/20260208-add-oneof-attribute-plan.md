# Plan: Implement Persistent OneOf Constraint

> **Status:** Ready
> **Created:** 2026-02-08
> **Author:** Antigravity
> **Source:** Direct request (Fix for `projex/20260208-verify-type-system.md`)
> **Related Projex:** `projex/20260208-verify-type-system.md`

---

## Summary

This plan implements persistent support for the `OneOf` attribute in the ZOH type system. Currently, `OneOf` is only validated transiently during a `/set` verb call, but the specification and issue report imply it should be a persistent constraint on the variable, similar to `[typed]`.

**Scope:** `impl/05_type_system.md` and `Zoh.Runtime` (C#).
**Estimated Changes:** 3 files.

---

## Objective

### Problem / Gap / Need
The `VariableStorage.set` implementation in `impl/05_type_system.md` ignores the `OneOf` attribute. While `SetDriver.cs` currently checks it, `VariableStore.cs` does not store or enforce it for future assignments. This creates a discrepancy where strict constraints (like Enums) cannot be reliably enforced on variables.

### Success Criteria
- [ ] `impl/05_type_system.md` documents `OneOf` constraint in `Variable` structure and `set` logic.
- [ ] `Variable` record in C# stores `OneOfConstraint`.
- [ ] `VariableStore` enforces `OneOfConstraint` on value updates.
- [ ] `SetDriver` correctly sets the `OneOfConstraint` when the attribute is present.
- [ ] Unit tests verify that a variable with `OneOf` constraint rejects invalid values in subsequent updates.

### Out of Scope
- Changes to other attributes or verbs.

---

## Context

### Current State
- `Variable` record only has `Value` and `TypeConstraint`.
- `SetDriver` checks `OneOf` attribute against the *current* value being set, but discards the list afterwards.
- `VariableStore` has no knowledge of `OneOf`.

### Key Files
| File | Purpose | Changes Needed |
|------|---------|----------------|
| `impl/05_type_system.md` | Implementation Spec | Add `oneOf` to `Variable` and validation logic to `set`. |
| `c#/src/Zoh.Runtime/Variables/Variable.cs` | Variable Data Structure | Add `ZohList? OneOfConstraint` property. |
| `c#/src/Zoh.Runtime/Variables/VariableStore.cs` | Variable Storage Logic | Enforce `OneOf` check in `UpdateOrAdd` and allow setting it. |
| `c#/src/Zoh.Runtime/Verbs/Core/SetDriver.cs` | `/set` Verb Implementation | Pass `OneOf` list to `VariableStore` to persist it. |

---

## Implementation

### Step 1: Update Implementation Spec

**Objective:** detailed the design in documentation.

**Files:**
- `impl/05_type_system.md`

**Changes:**
- Update `Variable` structure pseudocode to include `oneOf: List<Value>?`.
- Update `set` pseudocode to check `if var.oneOf and not var.oneOf.contains(value): error(...)`.

### Step 2: Update Runtime Core (Variable & Store)

**Objective:** Add storage and validation logic.

**Files:**
- `c#/src/Zoh.Runtime/Variables/Variable.cs`
- `c#/src/Zoh.Runtime/Variables/VariableStore.cs`

**Changes:**
```csharp
// Variable.cs
public record Variable(ZohValue Value, ZohValueType? TypeConstraint = null, ZohList? OneOfConstraint = null)
{
    public Variable WithValue(ZohValue value)
    {
        // ... Check Type ...
        // ... Check OneOf ...
        return this with { Value = value };
    }
}
```

```csharp
// VariableStore.cs
public void Set(string name, ZohValue value, Scope? specificScope = null, ZohList? oneOf = null)
{
    // ...
    // Pass oneOf to UpdateOrAdd or logic to set it
}
```

### Step 3: Update SetDriver

**Objective:** Connect verb attribute to storage.

**Files:**
- `c#/src/Zoh.Runtime/Verbs/Core/SetDriver.cs`

**Changes:**
- In `Execute`, when `OneOf` attribute is present, pass it to `VariableStore.Set`.

### Step 4: Verification Tests

**Objective:** Verify persistence.

**Files:**
- `c#/tests/Zoh.Tests/Variables/VariableStoreTests.cs` (Create if needed or add to existing) or `c#/tests/Zoh.Tests/Verbs/Core/SetTests.cs`

**Tests:**
```csharp
[Fact]
public void OneOf_Constraint_Is_Persistent() {
    // 1. /set [OneOf: [1, 2]] *x, 1;
    // 2. /set *x, 3; -> Should Fail
    // 3. /set *x, 2; -> Should Succeed
}
```

---

## Verification Plan

### Automated Checks
- [ ] Run `dotnet test` to ensure no regressions.
- [ ] Run new tests for `OneOf` persistence.

### Manual Verification
- [ ] N/A

---
