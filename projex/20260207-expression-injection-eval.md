# Evaluation: Expression Injection via Interpolation (Finding 3)

> **Date:** 2026-02-07 ~ 2026-02-12
> **Subject:** Finding 3 from [Red Team Report](20260207-spec-impl-redteam.md)
> **Status:** **FALSE POSITIVE (Runtime Safe)** / **VALID (Spec Ambiguity)**

## Executive Summary

The Red Team identified a potential vulnerability where user-controlled strings containing `${...}` patterns, when interpolated, might trigger recursive evaluation, leading to information disclosure (e.g., reading `*secret_var`).

**Evaluation Result:**
- **Runtime Behavior:** The current C# runtime (`Zoh.Runtime`) is **SAFE**. The interpolation logic is **non-recursive**. Patterns found within an interpolated value are treated as literal text and are not re-evaluated.
- **Specification:** The spec states "evaluates the variable as a template and interpolates it ONCE", which implies safety, but "interpolates it ONCE" could be misinterpreted as "one pass over the final string" (unsafe) vs "one pass over the template" (safe). This needs clarification.

---

## 1. Investigation Findings

### 1.1. Code Analysis

We analyzed `InterpolateDriver.cs`, `ZohInterpolator.cs`, and `ScannerUtility.cs`.

- The driver resolves the template string *first*.
- `ZohInterpolator.Interpolate` scans this template string.
- When it finds `${*var}`, it calls `ExpressionEvaluator`.
- `ExpressionEvaluator` returns a `ZohValue` (e.g., the user input string "World${*secret}").
- `ZohInterpolator` appends this value's string representation (`.ToString()`) directly to the `StringBuilder`.
- **Crucially:** It does *not* recursively scan the appended result for more `${...}` tokens. It advances the scan index past the original token.

### 1.2. Verification (Proof of Concept)

We created a reproduction test case `Zoh.Tests.Vulnerabilities.ExpressionInjectionTests`:

```csharp
[Fact]
public void Interpolation_ShouldNotRecurse_WhenInjectedValueContainsTemplateSyntax()
{
    var variables = new VariableStore(new Dictionary<string, Variable>());
    variables.Set("secret", new ZohStr("SECRET_VALUE"));
    variables.Set("user_input", new ZohStr("World${*secret}"));
    
    // *template = "Hello ${*user_input}!"
    var template = "Hello ${*user_input}!"; 
    
    var interpolator = new ZohInterpolator(variables);
    var result = interpolator.Interpolate(template);
    
    // SAFE: "Hello World${*secret}!"
    // UNSAFE: "Hello WorldSECRET_VALUE!"
    Assert.Equal("Hello World${*secret}!", result);
}
```

**Result:** The test **passed**, confirming safety.

---

## 2. Recommendations

Although the runtime is safe, the investigation validates the Red Team's concern about *ambiguity*.

### 2.1. Spec Clarification (Documentation)

Update `spec.md` to explicitly forbid recursive interpolation.

**Current Spec:**
> "...evaluates the variable as a template and interpolates it ONCE..."

**Proposed Change:**
> "...evaluates the variable as a template and interpolates it **non-recursively**. Any interpolation syntax (`${...}`) contained within the resolved values of variables is treated as literal text and is **not** processed."

### 2.2. Regression Testing

Keep the created test case `Zoh.Tests.Vulnerabilities.ExpressionInjectionTests.cs` in the test suite to prevent future regressions if the interpolation engine is refactored.

---

## 3. Conclusion

No code changes are required for the runtime. The "Remediation" step from the Red Team report should be scoped to **Documentation Only**.

**Action Items:**
1.  [x] Verify Runtime Safety (Completed)
2.  [ ] Update `spec.md` with explicit non-recursion guarantee.
