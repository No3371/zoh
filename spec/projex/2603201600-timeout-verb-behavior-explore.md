# Timeout Verb Behavior Exploration

> **Created:** 2026-03-20 | **Author:** agent
> **Type:** Question
> **Related Projex:** 2603201530-core-verb-namespace-restructure-plan.md

---

## Summary

Three verbs have `timeout` parameters: `Channel.Pull`, `Channel.Push`, and `Core.Wait`. All share the same diagnostic (`Info: timeout`) and return `nothing` on timeout. The spec is consistent on the happy-path behavior but leaves two behaviors **unspecified**: what happens with `timeout: 0` and `timeout: ?` (explicit nothing passed vs. omitted).

**Guiding Questions:**
1. Which verbs time out, and what are their exact parameter types, defaults, and return values?
2. What happens when `timeout` is `0`? When it is `nothing`?

**Scope:** `spec/` only — `2_verbs.md`, `1_concepts.md`. No runtime implementation.

---

## Context

**Why Now:** Implementing or testing timeout behavior requires knowing edge cases the spec may have left implicit.

**Current State:** Spec covers the normal positive-seconds case clearly; edge-case values (`0`, `?`) are not defined.

---

## Investigation Targets

### Target: Channel.Pull
**Rationale:** Primary blocking verb; most common timeout usage.
**Status:** Done
**Findings:**

```
/pull <channel>, timeout: 0.1;
```

- `timeout`: `double` or `*double`. **Optional. Default: `?`** (`spec/2_verbs.md:1059`).
- **Returns:** The pulled value normally. On timeout, returns **`nothing`** (`spec/2_verbs.md:1063–1066`).
- **Diagnostic:** `Info: timeout` — the timeout was reached (`spec/2_verbs.md:1065`).
- No `invalid_type` mentioned for timeout specifically beyond the global rule.

Ambiguity — **what does default `?` mean?** Implied: no timeout → block indefinitely. Not stated explicitly.

---

### Target: Channel.Push
**Rationale:** Push can also block (rendezvous semantics) and has its own timeout.
**Status:** Done
**Findings:**

```
/push <channel>, *var, timeout: 5;
/push <channel>, *var, wait: false;   :: timeout ignored
```

- `timeout`: `double` or `*double`. **Optional. Default: `?`.** Only applies when `wait: true` (default). Ignored when `wait: false` (`spec/2_verbs.md:1027`).
- **Returns:** `nothing` normally (always, regardless of timeout).
- **Diagnostic on timeout:** `Info: timeout` — "The push timed out before the value was consumed" (`spec/2_verbs.md:1039`).
- Timeout cannot be distinguished from success by return value alone — caller must check `/diagnose`.

---

### Target: Core.Wait
**Rationale:** Signal/message wait is another blocking verb with timeout.
**Status:** Done
**Findings:**

```
/wait "message_name";
/wait "message_name", timeout: 0.1;
```

- `timeout`: `double` or `*double`. **No default stated** (`spec/2_verbs.md:977`). Omitting it implies block indefinitely (consistent with Pull/Push pattern, but not explicit).
- **Returns:** The message value on success. **`nothing`** on timeout (`spec/2_verbs.md:983`).
- **Diagnostic:** `Info: timeout` (global standard, `spec/1_concepts.md:240`).

Inconsistency with Pull/Push: Pull and Push explicitly state `Default: ?`. Core.Wait's timeout entry omits the default.

---

### Target: Global Timeout Diagnostic
**Rationale:** Understand the diagnostic contract shared across all timeout verbs.
**Status:** Done
**Findings:**

From `spec/1_concepts.md:240`:
> `Info: timeout` — The timeout was reached (operation cancelled per `timeout` parameter).

- Severity: **Info** (not Error, not Fatal). The verb completes gracefully.
- Diagnostics are available via `/diagnose` after the verb call.
- The verb's return value on timeout is always `nothing`.
- Because `Info` is below `Error`, a wrapping `/try` will **not** catch it as an error — the timeout is considered a normal outcome.

---

### Target: Edge Case Values (0 and nothing)
**Rationale:** Core of the user's question — spec says nothing about these.
**Status:** Done
**Findings:**

**`timeout: 0`:**
- Not addressed anywhere in the spec.
- Two plausible interpretations:
  1. **Immediate timeout** — check for a value; if not present, return nothing + `Info: timeout` immediately. (Poll semantics.)
  2. **Treated as no timeout** — `0` and `?` both mean "no limit." (Less useful, surprising.)
- The spec example `timeout: 0.1` (a very small positive value) suggests small positive values are valid, but `0` itself is never demonstrated or prohibited.
- No `invalid_attribute` rule is stated for non-positive timeout values.

**`timeout: ?` (explicit nothing passed):**
- Pull and Push default to `?` meaning "no timeout."
- If a caller explicitly passes `timeout: ?`, it is reasonable to expect the same behavior as omitting it (no timeout). But the spec does not confirm this.
- The global type rule (`invalid_type` for wrong parameter types) would not fire since nothing is a valid "absence" value — but it is not a `double`.
- Tension: `timeout` accepts `double` or `*double` — `nothing` is neither, so strict type checking could raise `invalid_type`.

---

## Patterns & History

**Patterns Found:**
- **Consistent return on timeout:** All three verbs return `nothing` on timeout. Pull and Wait also return `nothing` for a legitimate "no value" state — callers must use `/diagnose` to distinguish.
- **Default `?` = block forever:** Pull and Push both explicitly default timeout to `?`. Wait is implied to follow the same pattern.
- **Info severity for timeout:** Timeout is not an error condition — it is a designed graceful outcome. This means `/try` does not intercept it automatically.

**Evolution:** No prior projex covers timeout specifically. The rendezvous model (`Channel.Push` blocking by default) is the most complex case since success and timeout are both `nothing` returns.

---

## Findings

### Discoveries
1. **Three verbs time out:** `Channel.Pull`, `Channel.Push` (only when `wait: true`), `Core.Wait`. `Sleep` does not have a timeout parameter.
2. **Timeout = Info diagnostic, nothing return:** All three emit `Info: timeout` and return `nothing`. For Push, this creates an ambiguity since the normal return is also `nothing`.
3. **`timeout: 0` is unspecified:** The spec gives no rule for zero or negative values. Implementations will diverge without a spec decision.
4. **`timeout: ?` type conflict:** The parameter is typed `double | *double`. Explicit `nothing` neither matches that type nor is it excluded by name. This is a latent inconsistency.
5. **Core.Wait omits its default:** Unlike Pull/Push, Wait's `timeout` parameter has no stated default. This is a spec gap — likely an oversight.

### Mental Model
> Timeout verbs block until satisfied or until `N` seconds elapse. On elapse, they return `nothing` and set `Info: timeout`. Omitting timeout (or passing `?`) means no limit. The diagnostic is the only reliable way to tell a timed-out `nothing` from a real `nothing`.

### Implications
- Callers using timeout on Push/Pull must always check `/diagnose` to distinguish timeout from a legitimately pushed `nothing` value.
- `timeout: 0` needs a spec decision before implementation — "immediate poll" is the most useful semantic.
- `timeout: ?` should either be explicitly equivalent to "omit" or explicitly raise `invalid_type`. Currently undefined.
- `/try` does not protect against timeouts — `Info` is below its catch threshold.

---

## Answers

**Which verbs have timeout, and what are their exact behaviors?**

| Verb | Param type | Default | On timeout: return | On timeout: diagnostic |
|------|-----------|---------|-------------------|----------------------|
| `Channel.Pull` | `double \| *double` | `?` (no timeout) | `nothing` | `Info: timeout` |
| `Channel.Push` | `double \| *double` | `?` (no timeout) | `nothing` | `Info: timeout` |
| `Core.Wait` | `double \| *double` | unstated (implied `?`) | `nothing` | `Info: timeout` |

**Do they timeout when `timeout: 0`?**
Unspecified. No rule exists for zero or negative values. Needs a spec decision.

**Do they timeout when `timeout: ?` (nothing)?**
Default behavior for Pull/Push is `?` = no timeout. Explicitly passing `?` is likely equivalent but the type signature (`double | *double`) creates a latent `invalid_type` conflict. Also unspecified.

---

## Open Questions

- [ ] Should `timeout: 0` trigger immediate timeout (poll), or be equivalent to no timeout, or raise `invalid_attribute`?
- [ ] Should `timeout: ?` (explicit nothing) be valid (= no timeout) or raise `invalid_type`?
- [ ] Should `Core.Wait` explicitly document its timeout default as `?` to match Pull/Push?
- [ ] How should callers reliably detect a timed-out `nothing` vs. a pushed `nothing` on a channel? Should there be a dedicated check verb or attribute?

---

## Appendix

**Sources:**
- `spec/2_verbs.md` — Channel.Pull (L1053–1070), Channel.Push (L1021–1051), Core.Wait (L970–989), Sleep (L956–968)
- `spec/1_concepts.md` — Standard diagnostics (L238–240), verb return contract (L202)

**Limitations:** Implementation files (`csharp/`) not examined — runtime behavior may already have resolved some of these gaps in practice.
