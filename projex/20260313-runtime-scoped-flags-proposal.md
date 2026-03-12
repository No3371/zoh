# Runtime-Scoped Flags

> **Status:** Draft
> **Created:** 2026-03-13
> **Author:** Agent
> **Related Projex:** 20260313-embed-variable-locale-flag-proposal.md, 20260312-script-level-localization-proposal.md, 20260313-runtime-scoped-flags-spec-plan.md, 20260313-runtime-scoped-flags-impl-plan.md

---

## Summary

Extend the flag system with a **runtime scope** alongside the existing context scope. Runtime flags are global, persist across contexts, and are available to the preprocessor as preprocessor variables. Flag reading falls back from context to runtime. `/flag` accepts a `[scope]` attribute to select the target scope, mirroring how `[scope]` works for variables with `/set`.

---

## Problem Statement

### Current State

Flags exist only at context scope:

```zoh
/flag "interactive", false;  :: sets on the current context
```

- Set with `/flag "name", value;`
- Visible to all verb drivers within the context
- Copied to forked contexts
- No runtime-level flags exist

Variables have two scopes (context, story) selected via `[scope]`. Flags have only one (context).

### Gap

The preprocessor runs before any context exists. It cannot read context flags. This creates a phase problem: `locale` needs to be available at both preprocess time (for `#embed` path resolution) and runtime (for verb drivers). With only context-scoped flags, there is no flag value the preprocessor can see.

More broadly, some flags are inherently runtime-global — they describe the environment, not the context. `locale`, `platform`, `debug` are properties of the runtime, not of any individual context.

### Why Now?

The variable-aware embedding proposal (`20260313-embed-variable-locale-flag-proposal.md`) requires preprocessor access to flag values. Without runtime-scoped flags, the proposal must invent a separate "preprocessor variable" concept. Runtime-scoped flags unify the model.

---

## Proposed Change

### Flag Scopes

| Scope | Lifetime | Visibility | Set by |
|-------|----------|------------|--------|
| **Runtime** | Runtime lifetime | All contexts, preprocessor | Runtime API, `/flag [scope: "runtime"]` |
| **Context** | Context lifetime | Current context, copied to forks | `/flag` (default) |

### Reading: Fallback Chain

When a flag value is read (by a verb driver, preprocessor, or `/flag` query):

```
Context flag → Runtime flag → (not found)
```

Context flags shadow runtime flags with the same name, just like story variables shadow context variables.

### Writing: `[scope]` Attribute

```zoh
/flag "locale", "fr";                        :: default → context scope
/flag [scope: "context"] "locale", "fr";     :: explicit context scope
/flag [scope: "runtime"] "debug", true;      :: runtime scope
```

The `[scope]` attribute mirrors its use in `/set`:

```zoh
/set [scope: "context"] *var, value;   :: existing variable scope attribute
/flag [scope: "runtime"] "name", value; :: same pattern for flags
```

### Runtime API

The runtime provides an API to set runtime-scoped flags before story loading:

```
runtime.setFlag("locale", "fr")    // runtime-scoped
runtime.setFlag("debug", true)     // runtime-scoped
runtime.setFlag("platform", "mobile")
```

These flags are:
- Available to the preprocessor during `#embed` resolution (as `${locale}`, `${debug}`, etc.)
- Available to verb drivers in all contexts
- Overridable per-context via `/flag` at context scope

### Preprocessor Access

Runtime-scoped flags are exposed to the preprocessor as the preprocessor variable map. No separate "preprocessor variable" concept needed — runtime flags ARE the preprocessor variables.

Built-in preprocessor variables (`${filename}`) are a separate, fixed set that supplements runtime flags.

```
Preprocessor variable resolution:
  ${name} → built-in (filename, etc.) → runtime flag → story metadata → ""
```

### Interaction with Fork/Clone

- **Runtime flags:** Shared (not copied — all contexts see the same runtime flags)
- **Context flags:** Copied to forks (existing behavior, unchanged)
- **Shadow rule:** A context flag `locale` shadows the runtime flag `locale` for that context only

### Standard Flag: `locale`

Add to `std_flags.md`:

**locale** — The active locale identifier (BCP 47: `"fr"`, `"ja"`, `"pt-BR"`). Default: `""` (empty — no locale). Typically set at runtime scope.

At runtime scope, `locale` is available to the preprocessor for `#embed` path resolution and to all contexts for locale-aware verb behavior.

---

## Design Rationale

| Decision | Rationale |
|----------|-----------|
| Runtime scope for flags | Mirrors variable scoping; flags describe environment (runtime) or context |
| Fallback chain (context → runtime) | Same pattern as variable resolution (story → context); natural shadowing |
| `[scope]` attribute | Consistent with `/set [scope]`; no new syntax |
| Runtime flags = preprocessor variables | One concept instead of two; runtime flags are the single source of truth |
| Context scope remains default | Backwards compatible; existing `/flag` calls don't change |

---

## Impact Analysis

### Affected Areas
- **Spec: `1_concepts.md`** — Flag scoping concept (parallel to variable scoping)
- **Spec: `2_verbs.md`** — `/flag` verb: add `[scope]` attribute, document reading fallback
- **Spec: `std_flags.md`** — Add `locale` standard flag, document runtime vs context scope
- **Spec: `3_runtime.md`** — Runtime flag storage, API for setting flags before story load
- **Impl: `03_preprocessor.md`** — Preprocessor reads runtime flags as `${}` variables
- **Impl: `09_runtime.md`** — Runtime-level flag storage

### Dependencies
- Variable-aware embedding proposal (`20260313-embed-variable-locale-flag-proposal.md`) — consumes runtime flags as preprocessor variables

### Risks
| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Shadowing confusion (context flag hides runtime flag) | Low | Low | Same pattern as variable shadowing — already understood |
| Scripts writing runtime flags unexpectedly | Low | Medium | Runtime flags writable from script only with explicit `[scope: "runtime"]` |
| Flag namespace collision across contexts | Low | Low | Runtime flags are shared intentionally; contexts that need isolation use context scope |

### Breaking Changes
None. Existing `/flag` calls default to context scope (unchanged). Runtime scope is opt-in.

---

## Open Questions

- [ ] Should runtime flags be writable from scripts at all, or only via the runtime API? (Allowing script writes enables dynamic runtime config; restricting prevents unintended global side effects)
- [ ] Should there be a story scope for flags (parallel to story-scoped variables)?
- [ ] Should `/flag` support a read/query mode (e.g., `/flag "locale"; -> *val;`) or should flag reading remain verb-driver-only?

---

## Next Steps

If accepted:
1. Spec: Add flag scoping to `1_concepts.md`
2. Spec: Update `/flag` verb in `2_verbs.md` with `[scope]` attribute and fallback
3. Spec: Add `locale` to `std_flags.md`
4. Spec: Document runtime flag storage in `3_runtime.md`
5. Update embed proposal to reference runtime flags instead of separate preprocessor variables
