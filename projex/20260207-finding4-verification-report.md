# Verification Report: Spec/Impl Inconsistencies (Finding 4)

> **Date:** 2026-02-07
> **Source:** `projex/20260207-spec-impl-redteam.md`
> **Workflow:** Verification via Code Inspection

## Executive Summary

We verified the inconsistencies reported in "Finding 4" by inspecting the Language Specification (`spec.md`), Implementation Guides (`impl/*.md`), and the C# Runtime Source Code (`Zoh.Runtime`).

| Issue | Finding Veridct | Status | Action Required |
|-------|-----------------|--------|-----------------|
| `/pull` on closed channel | **VALID** | Inconsistent | Update `impl/08` to match Spec/Code (Error vs Result Object) |
| Map key order | **INVALID** | Consistent | None (Spec allows implementation-defined) |
| `/foreach` iterator drop | **INVALID** | Consistent | None (Impl explicitly mentions it, Code matches) |
| Macro syntax | **PARTIALLY VALID** | Confusing | Fix `impl/03` intro text (mentions `#macro` but defines `|%`) |

---

## Detailed Findings

### 1. `/pull` on Closed Channel
- **Claim:** Spec says Error `closed`; Impl/08 says `PullResult { status: "closed" }`.
- **Verification:**
    - **Spec:** "Throws error if channel ... is closed".
    - **Impl/08:** Description says "Returns a PullResult: { status: 'closed' }".
    - **Code (`ChannelVerbs.cs`):** Returns `VerbResult.Error(..., "closed", ...)` via `PullVerbDriver.Execute`.
- **Conclusion:** The Implementation Guide (`impl/08`) is **INCORRECT**. The Spec and Code agree (it returns an error, not a value object).
- **Remediation:** Update `impl/08` `pull` pseudo-code to throw error.

### 2. Map Key Order in `toString()`
- **Claim:** Spec says "impl-defined"; Impl says "Not mentioned".
- **Verification:**
    - **Spec:** "Map key order is implementation-defined".
    - **Impl/05:** "Order is implementation-defined" (Note: Redteam report claimed it was not mentioned, but `impl/05` explicitly states it).
    - **Code (`ZohMap.cs`):** Uses `ImmutableDictionary` iteration (arbitrary).
- **Conclusion:** **CONSISTENT**. The Spec explicitly leaves this open. `Impl/05` is also explicit.
- **Remediation:** None required.

### 3. `/foreach` Iterator Drop
- **Claim:** Spec says "dropped from story scope first"; Impl says "Not explicitly mentioned"; Code behavior unknown.
- **Verification:**
    - **Spec:** "dropped from story scope first".
    - **Impl/07:** Explicitly includes: `context.drop(iteratorName, scope: STORY)`.
    - **Code (`ForeachDriver.cs`):** Calls `context.Variables.Drop(varName)`.
    - **Code (`VariableStore.cs`):** `Drop` with no scope defaults to `_storyVariables.Remove(name)`, leaving Context variables touched only if they shadow.
- **Conclusion:** **CONSISTENT**. The Redteam report's claim that Impl/07 doesn't mention it is **FALSE**. The Code behavior matches the Spec exactly (targeting story scope by default).
- **Remediation:** None.

### 4. Macro Syntax
- **Claim:** Spec says `|%NAME%|`; Impl says `#macro`.
- **Verification:**
    - **Spec:** `|%NAME%|` (pipe-delimited).
    - **Impl/03:** The *definition section* uses `|%NAME%|`. However, the *introduction* text mentions `#macro`.
    - **Code (`MacroPreprocessor.cs`):** Implements `|%` syntax (`Regex(@"^|%(\w+)%|$")`).
- **Conclusion:** **CONSISTENT** (Functionally). The documentation in `impl/03` contains a minor text error in the intro ("#macro") but strictly defines the correct syntax later.
- **Remediation:** Correct `impl/03` introduction text to remove reference to `#macro`.
