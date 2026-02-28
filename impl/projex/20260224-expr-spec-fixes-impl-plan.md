# Expression Spec Fixes — Impl Doc

> **Status:** In Progress
> **Created:** 2026-02-24
> **Author:** agent
> **Source:** `20260224-expr-spec-eval.md`
> **Related Projex:** `20260224-expr-spec-eval.md`, `20260224-expr-spec-fixes-plan.md`

---

## Summary

Fixes two issues in `impl/04_expressions.md` identified by the expression grammar evaluation: a wrong separator character in the conditional grammar comment, and an unintended fallthrough in `parseOptionList()` that silently returns `AnyForm` instead of erroring for bare `$(options)` without a `[...]` suffix.

**Scope:** `impl/04_expressions.md` only — no spec files.
**Estimated Changes:** 1 file, 2 targeted edits.

---

## Objective

### Problem / Gap / Need
`impl/04_expressions.md` has:
1. **F3**: The grammar comment for `conditional` says `'?' expr '|' expr` but the correct separator is `':'`. The implementation code (`parseConditionalOrAny`) correctly uses `consume(COLON, ...)` — only the grammar comment is wrong.
2. **F4**: `parseOptionList()` falls through to `return AnyForm(options)` when no `[index]` or `[%]` follows. Per the spec and user decision, this is invalid syntax. The impl should emit a parse error instead.

### Success Criteria
- [ ] Grammar comment for `conditional` uses `':'` not `'|'`
- [ ] `parseOptionList()` raises a parse error when the `$(...)` form is not followed by `[index]` or `[%]`
- [ ] Error message clearly explains the mistake and suggests `$?()` as the correct form

### Out of Scope
- Spec file changes — covered in `20260224-expr-spec-fixes-plan.md`
- Implementing new operators or features — this plan is corrections only

---

## Context

### Current State
`impl/04_expressions.md` is the implementation specification for the C# expression evaluator. It defines grammar (EBNF), AST nodes, parser pseudocode, and evaluator pseudocode.

### Key Files
| File | Purpose | Changes Needed |
|------|---------|----------------|
| `impl/04_expressions.md` | Expression evaluator impl spec | Fix grammar comment; fix parseOptionList fallthrough |

### Dependencies
- **Requires:** `20260224-expr-spec-fixes-plan.md` (establishes the canonical spec that this impl doc follows)
- **Blocks:** nothing

---

## Implementation

### Overview
Two small, isolated edits to `impl/04_expressions.md`. No structural changes to the document.

---

### Step 1: Fix conditional grammar comment

**Objective:** Correct the separator in the `conditional` EBNF comment from `'|'` to `':'`.

**Files:** `impl/04_expressions.md`

**Changes — in the `## Expression Grammar (EBNF)` block, the `conditional` line:**

```diff
- conditional := '$?(' expr '?' expr '|' expr ')'        # /if ternary
+ conditional := '$?(' expr '?' expr ':' expr ')'        # /if ternary
```

**Rationale:** The grammar comment is wrong — it uses `'|'` which is the any-form separator. The `parseConditionalOrAny` code already correctly uses `consume(COLON, "Expected ':' in conditional")`, matching the spec and all examples. This is a documentation-only bug with no runtime impact, but it misleads implementers reading the grammar.

**Verification:** Grammar comment for `conditional` reads `'?' expr ':' expr`.

---

### Step 2: Fix `parseOptionList()` to error on bare `$(options)`

**Objective:** When `$(options)` is parsed without a following `[index]` or `[%]`, emit a parse error instead of silently returning `AnyForm`.

**Files:** `impl/04_expressions.md`

**Changes — in `### Step 3: Special Form Parsing`, the `parseOptionList()` pseudocode:**

```diff
  parseOptionList(): ExprNode
      # $( a | b )
      options = [parse()]
      while match(PIPE):
          options.add(parse())
      consume(RPAREN)

      # Check for index or roll
      if match(LBRACKET):
          if match(PERCENT):
              consume(RBRACKET)
              # Check for weighted
              if hasWeights(options):
                  return WRollForm(parseWeighted(options))
              return RollForm(options)
          else:
              # Indexed access
              wrap = match('!')
              index = parse()
              consume(RBRACKET)
              return IndexedForm(options, index, wrap)

-     return AnyForm(options)
+     error("'$(' option list requires '[index]' or '[%]' suffix; did you mean '$?(' for first-non-nothing selection?")
```

**Rationale:** The fallthrough to `AnyForm` was unintended. `$(a|b|c)` is not defined syntax — `$?(a|b|c)` is the correct form for first-non-nothing. Silently accepting invalid syntax masks author mistakes. The error message guides authors to the correct form.

**Verification:** `parseOptionList()` has no `return AnyForm(options)` fallthrough; final line is an `error(...)` call with a message referencing `$?(`.

---

## Verification Plan

### Manual Verification
- [ ] Read `impl/04_expressions.md` grammar block — `conditional` comment uses `':'`
- [ ] Read `parseOptionList()` pseudocode — no fallthrough `return AnyForm`; ends with `error(...)` call

### Acceptance Criteria Validation
| Criterion | How to Verify | Expected Result |
|-----------|---------------|-----------------|
| Conditional comment | Read EBNF grammar block | `'?' expr ':' expr` |
| parseOptionList fallthrough removed | Read parseOptionList pseudocode | Last statement is `error(...)`, not `return AnyForm(options)` |

---

## Rollback Plan

All changes are to a Markdown implementation spec. Rollback via `git revert` on the commit.

---

## Notes

### Assumptions
- The `parseConditionalOrAny` code is already correct (uses `COLON`) — no change needed to that function
- The error message text is pseudocode; the actual runtime message in any concrete implementation may differ in wording
