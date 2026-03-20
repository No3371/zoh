# Remove #flag Syntactic Sugar (Spec)

> **Status:** Complete
> **Created:** 2026-02-07
> **Completed:** 2026-02-07
> **Walkthrough:** [Walkthrough](20260207-remove-flag-sugar-walkthrough.md)
> **Author:** Agent
> **Source:** Direct request
> **Related Projex:** [C# Implementation](../csharp/projex/20260207-remove-flag-sugar-csharp-plan.md)

---

## Summary

Remove the `#flag name value;` syntactic sugar form from ZOH specification and implementation guides. The `/flag` verb remains unchanged.

**Scope:** Spec docs, impl specs, AGENT.md  
**Estimated Changes:** 5 files

---

## Objective

### Problem / Gap / Need

The `#flag` sugar creates confusion with preprocessor directives and adds unnecessary complexity.

### Success Criteria

- [ ] All `#flag` references removed from spec and impl docs
- [ ] AGENT.md updated

### Out of Scope

- C# implementation (separate projex)
- The `/flag` verb itself

---

## Implementation

### Step 1: Update spec.md

**Files:** `spec.md`

**Changes:**
1. **Line 175** - Remove `#flag flag_name on`
2. **Line 229** - Remove `#flag flag_name off`
3. **Lines 1403-1407** - Remove "Syntactic Sugar Forms" subsection under Core.Flag

---

### Step 2: Update impl/01_lexer.md

**Files:** `impl/01_lexer.md`

**Changes:**
1. **Line 50** - Remove `#flag` from `HASH_DIRECTIVE` list
2. **Line 150** - Remove `#flag` from preprocessor list

---

### Step 3: Update impl/02_parser.md

**Files:** `impl/02_parser.md`

**Changes:**
1. **Line 79** - Remove `FlagSugar` from `SugarStatement`
2. **Lines 137-138** - Remove `flag_sugar` from grammar
3. **Line 152** - Remove `flag_sugar` production
4. **Lines 431-432** - Remove `#flag` rows from sugar table

---

### Step 4: Update impl/03_preprocessor.md

**Files:** `impl/03_preprocessor.md`

**Changes:**
1. **Line 5** - Remove `#flag` from purpose statement
2. **Lines 126-134** - Remove entire `### #flag` section
3. **Line 302** - Remove `#flag` from sugar pseudocode

---

### Step 5: Update AGENT.md

**Files:** `AGENT.md`

**Changes:**
1. **Line 88** - Remove `#flag` row from sugar table

---

## Verification

- [ ] `grep -rn "#flag" spec.md impl/ AGENT.md` returns empty

---

## Rollback

`git checkout -- spec.md impl/ AGENT.md`
