# Canonical IR Spec — Proposal

> **Status:** Draft
> **Created:** 2026-02-25
> **Author:** agent
> **Related Projex:** 20260225-compiled-ir-eval.md

---

## Summary

Introduce `spec/ir.md` — a normative specification for a canonical JSON Intermediate Representation (IR) of a zoh story. The IR captures the post-parse, pre-compiler AST: handler-independent and portable across any conforming runtime, environment, or handler set. Loading from IR skips the source-processing pipeline (lex, parse, preprocess) but always runs compiler and validator passes — the same safety guarantees as loading from source. This enables cross-environment sharing, source-free distribution, and portable tooling.

Compilation caching (skip the full pipeline including compile + validate) is a separate, per-runtime concern and is explicitly out of scope for the canonical IR spec.

---

## Problem Statement

### Current State

The zoh compilation pipeline produces a `CompiledStory` object in memory, cached within a session and discarded when the process exits. There is no serializable external form. Every `loadStory()` call re-runs the full pipeline from source.

The impl spec (`impl/02_parser.md`) defines AST node types but does not define a canonical interchange encoding. The spec folder has no IR document.

### Gap / Need / Opportunity

Without a canonical IR:
- Every host language implementing zoh must re-implement the full source-processing pipeline (lexer, parser, preprocessor, expression parser) independently
- Stories cannot be distributed in compiled form — source is always required
- Tooling (linters, static analyzers, editors) has no stable, language-agnostic analysis target
- There is no portable artifact a build tool can produce and a different runtime can consume

### Why Now?

The IR eval (`20260225-compiled-ir-eval.md`) identified cross-language portability as the primary value driver. An AST audit (conducted as part of that eval's recommended follow-up) confirms the feasibility:

**AST Audit Results — `impl/02_parser.md` + `impl/04_expressions.md`:**

| Node | Fields | IR-serializable? |
|------|--------|-----------------|
| `StoryNode` | name (string), metadata (Map<string, primitive/list/map>), body (List<Statement>), checkpoints (Map<string, CheckpointNode>) | ✅ All plain data |
| `VerbCallNode` | namespace (string?), name (string), isBlock (bool), attributes, namedParams, unnamedParams, position (line/col ints) | ✅ All plain data |
| `AttributeNode` | name (string), value (Value?) | ✅ |
| `Value` variants | Primitives, ReferenceValue (string + List<Value>), ListValue, MapValue, ExpressionValue (expr string), ChannelValue (string), VerbCallValue (recursive) | ✅ All plain data |
| `CheckpointNode` | name (string), contract (List<ContractParam>), position | ✅ |
| `ContractParam` | name (string), type (string?) | ✅ |
| `ExprNode` | BinaryExpr, UnaryExpr, LiteralExpr, ReferenceExpr, GroupExpr, SpecialExpr — all with primitive fields and recursive children | ✅ All plain data |

**Conclusion:** No host-language object references, closures, or non-serializable types exist anywhere in the AST. Serialization is straightforward.

---

## Proposed Change

### Overview

Add `spec/ir.md` defining:
1. The IR boundary: post-preprocessor, post-parse, pre-compiler AST
2. The envelope format: a JSON object with version and story fields
3. The JSON encoding of each AST node type
4. Load behaviour: compile and validate passes always run on IR load
5. Versioning

The IR spec is normative: any conforming zoh runtime must be able to produce and consume IR files in this format.

---

### Approach Options

#### Option A: Post-parse JSON IR (Recommended)

**Description:** The IR captures the output of preprocessor + lexer + parser — the AST before any compiler passes. Each runtime loading the IR runs its own compiler and validator passes from that neutral form.

- **Pros:**
  - **Handler-independent:** IR contains only verb calls (name, namespace, attributes, parameters) and checkpoints — no enrichments from any handler set. Any conforming runtime, regardless of which verbs or compilers it has registered, can load the same IR.
  - **Progressive enhancement preserved:** A story shared between a minimal text runtime and a rich 3D runtime loads correctly on both. Each runtime compiles and validates against its own handler set. Unknown verbs are handled by each runtime's own policy.
  - **Matches already-defined AST node types** in `impl/02_parser.md` — no new data model work needed.
  - **Validators always run,** providing the same safety guarantees as loading from source regardless of IR origin.
  - **Compiler changes never invalidate IR files** — IR is stable across handler configuration changes.
- **Cons:**
  - Runtimes run compiler and validator passes at every IR load (though this is fast compared to source-processing).
  - Does not encode compiler-enriched metadata — each loading runtime produces its own.

#### Option B: Post-compiler JSON IR

**Description:** The IR captures the fully compiled output after all compiler passes, encoding handler-specific enrichments alongside verb calls.

- **Pros:** Loading skips compiler passes in addition to source-processing.
- **Cons:**
  - **Breaks progressive enhancement.** Compiler enrichments are specific to the producing runtime's handler set. A different runtime loading the IR would find foreign enrichments it does not understand, and would be missing the enrichments its own compilers would have produced for its own drivers. The only fix — running compilers at load time anyway — eliminates the advantage.
  - **Different handler sets produce different IR for the same source.** A story compiled by a Unity runtime and the same story compiled by a Godot runtime would produce non-identical IR files, defeating the goal of a canonical shareable artifact.
  - Compiler changes that alter enrichment shape require an IR version bump, tying the IR format to handler evolution.

**Note on structural rewriting:** An earlier analysis raised the concern that custom compilers might expand one verb into many, making post-compiler IR host-specific. This concern stands but is secondary — the handler-enrichment problem above applies even to purely additive, non-structural compilers.

#### Option C: Binary CBOR IR

**Description:** Same as Option A but encoded as CBOR instead of JSON.

- **Pros:** Smaller files, faster parse than JSON.
- **Cons:** Not human-readable; tooling requires CBOR support; premature optimisation at this stage.

---

### Recommended Approach

**Option A — post-parse JSON IR.**

The canonical IR must follow the same portability principle as zoh source: shareable across different environments, implementations, and handler sets. Post-compile IR (Option B) violates this because compiler enrichments are handler-set-specific. Option A is handler-independent by construction.

Validators always run at IR load time. This is intentional: validators are the loading runtime's safety check regardless of where the IR came from. The compile + validate cost at load time is modest compared to the source-processing phases that IR does eliminate (lex, parse, preprocess, expression parse — including any `#embed` file I/O).

**On caching:** Some runtimes may want to skip compile + validate passes for stories they have already loaded and cached locally (same handler set, known provenance). This is a valid per-runtime optimisation, but it is not the purpose of the canonical IR and is explicitly out of scope for `spec/ir.md`. Runtimes may maintain their own compiled caches in whatever form they choose.

---

## Impact Analysis

### Affected Areas

- **`spec/`**: New file `spec/ir.md`. No changes to existing spec files (IR is purely additive).
- **`impl/09_runtime.md`**: Downstream — add `loadStoryFromIR(path)` to the runtime interface; specify that compile + validate passes run on IR load.
- **`impl/12_validation.md`**: Downstream — note that IR load follows the same compile + validate pipeline as source load; only lex, parse, and preprocess are skipped.
- **`csharp/src/Zoh.Runtime/`**: Downstream — implement IR serializer/deserializer.
- **`spec/versioning.md`**: The IR envelope carries its own version field (`zoh_ir_version`), independent of the language spec version (`zoh_version`).

### Dependencies

- Depends on: `spec/versioning.md` (for referencing `zoh_version` in the IR envelope) — see `20260225-versioning-concept-spec-plan.md`
- Blocks: downstream impl plans for `loadStoryFromIR()` and IR serializer

### Risks

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| IR format changes before it stabilises, breaking stored IR files | Medium | Medium | Mandatory `zoh_ir_version` field from day one; runtimes reject unknown versions |
| `SugarStatement` or `PreprocessorDirective` nodes reaching the IR boundary (if preprocessor runs after parse) | Low | Medium | Clarify in spec: IR boundary is post-preprocessor; no sugar or directive nodes appear in IR |
| Expression AST in IR is verbose/large | Low | Low | IR is a distribution artefact, not a size-critical format; gzip available |

### Breaking Changes

None. IR is a new, optional addition. Existing source-based `loadStory()` is unchanged.

---

## Open Questions

- [ ] Confirm that the compiler phase always receives a fully desugared, preprocessor-resolved AST. If sugar and directive nodes never survive into compiler input, they will not appear in the IR by definition.
- [ ] Should `position` (source location) fields be included in the IR? (Recommendation: yes, as optional fields — enables debuggers to map IR back to source; runtimes that don't need them can ignore them.)
- [ ] Should the IR envelope carry the story's `zoh_version`? (Recommendation: yes — allows runtimes to check language version compatibility before deserializing the full AST.)

---

## Next Steps

If accepted:
1. Write `spec/ir.md` with the canonical JSON schema (one plan, targeting `spec/` only)
2. Downstream: update `impl/09_runtime.md` to add `loadStoryFromIR()` and specify compile + validate run at load time
3. Downstream: update `impl/12_validation.md` to note IR load skips only lex/parse/preprocess; compile + validate pipeline is unchanged
4. Downstream: implement IR serializer/deserializer in the C# reference runtime

---

## Appendix

### Research / References

- `20260225-compiled-ir-eval.md` — evaluation that identified IR use cases and recommended this approach
- `impl/02_parser.md` — AST node definitions (audited above)
- `impl/04_expressions.md` — expression AST nodes (audited above)
- `impl/09_runtime.md` — runtime interface (`loadStory`, `CompiledStory` cache)
- `spec/0_basic.md` — progressive enhancement principle
- Ink JSON format — narrative scripting precedent for source→JSON→runtime pipeline

### Alternatives Considered

- **"Post-compiler IR as the canonical format"**: Rejected — compiler enrichments are handler-set-specific, breaking cross-environment portability. Progressive enhancement requires each runtime compile against its own handler set. Post-compiler IR is appropriate only as a per-runtime cache, which is out of scope for the canonical spec.
- **"Don't spec it, let each runtime define their own IR"**: Defeats cross-environment sharing. Each runtime producing incompatible IR files fragments the ecosystem.
- **"Defer until a second runtime exists"**: Valid, but spec can be written before implementation without premature lock-in. A spec document is cheap to revise.
- **"Binary bytecode instead of JSON"**: Rejected — adds complexity, removes human-readability and tooling compatibility, with no demonstrated performance need.
