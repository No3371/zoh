# Intermediate Format for Compiled Zoh — Evaluation

> **Created:** 2026-02-25
> **Author:** agent
> **Subject:** Whether a serializable intermediate representation (IR) between compilation and execution would benefit the zoh runtime
> **Type:** Comparative + Gap Analysis
> **Related Projex:** 20260208-project-structure-map.md, 20260222-specs-nav.md, 20260225-canonical-ir-spec-proposal.md, 20260225-compiler-runtime-separation-eval.md

---

## Executive Summary

Zoh already has a de-facto in-memory IR: `CompiledStory`, produced by the full compilation pipeline and cached in the runtime's story map. What it lacks is a **serializable external form** of that IR. Four concrete use cases would benefit from such a format — compilation caching, source-free distribution, cross-language portability, and offline tooling — but none are pressing today. The most strategically valuable case is cross-language portability: a canonical IR spec would let any compliant zoh runtime load a compiled story without re-implementing the full pipeline, directly serving the multi-host embedding goal. The recommended path is to design `CompiledStory` fields to be IR-serializable now (no extra cost), document a canonical IR schema as a spec artefact, and defer implementation until a concrete driver (distribution, multi-language host) materializes.

---

## Evaluation Scope

### Subject

An intermediate representation is a structured, serializable encoding of a compiled zoh story, sitting between the source compilation pipeline and runtime execution. It captures everything a runtime needs to execute a story without re-parsing the source.

### Questions Addressed

1. Does the current architecture have a gap where an IR could fit?
2. What use cases would a serializable IR concretely enable?
3. What form should an IR take given zoh's design constraints?
4. What are the costs and risks of introducing one?
5. When, if ever, is the right time to act?

### Evaluation Criteria

| Criterion | Weight | Description |
|-----------|--------|-------------|
| Use-case validity | High | Are there real, non-hypothetical drivers for an IR? |
| Fit with extensibility model | High | Does IR survive handler/driver customisation? |
| Implementation cost | Medium | How much work is a well-designed IR? |
| Spec coherence | Medium | Does IR belong in the spec or only in impl? |
| Future-proofing | Low | Does it prevent lock-in or enable ecosystem growth? |

### Out of Scope

- Bytecode optimisation / ahead-of-time optimisation passes
- JIT compilation
- A debugger or profiler using the IR (a secondary benefit, not the driver)

---

## Context Analysis

### Current State

The zoh compilation pipeline produces a `CompiledStory` object that lives entirely in memory:

```
Source
  → Preprocessors (embed, macro, sugar)
  → Lexer (tokens)
  → Parser (AST: StoryAst, StatementAst, ValueAst)
  → Compilers (priority-ordered, read-ahead, transform AST)
  → Validators (story-level, verb-level)
  → CompiledStory (cached in Runtime.stories)
```

`CompiledStory` is the natural IR. It is cached per-session in `Runtime.stories: Map<string, CompiledStory>` and reused across contexts. What it is **not** is serializable: it is an object graph in host-language memory, tied to the host runtime's type system, and thrown away when the process exits.

The `loadStory(path: string): CompiledStory` operation always re-runs the full pipeline from source.

### Historical Context

Zoh is designed as an **embedded scripting language for storytelling**. Its primary hosts are game engines and interactive narrative runtimes. The spec is explicitly language-agnostic, with the C# code as the reference implementation and the `impl/` markdown docs as the portable spec.

### Constraints

- Zoh's handler system is extensible: custom verb drivers, compilers, and validators are registered at runtime. An IR cannot encode registered behaviour, only the structural data (verb calls, parameters, attributes).
- Expressions are lazy: backtick expressions are stored as data and evaluated at runtime against live variables. They have their own AST but cannot be pre-evaluated.
- Scripts are typically small (narrative content). Compilation cost is low on modern hardware.
- The project is in an active specification phase. Premature serialisation format lock-in would create migration burden.

### Stakeholders

- **Runtime implementors** (any language targeting the impl/ spec): would benefit most from a canonical IR
- **Story authors**: transparent — IR is invisible at the authoring level
- **Host application developers**: benefit from predictable load times and distribution options
- **Tooling authors** (linters, editors): could target IR as a stable analysis target

---

## Foundations

### Underlying Principles

- **Verb-driven uniformity**: every executable unit is a verb call with a uniform shape. This makes a flat IR natural — there is no deep semantic diversity to encode.
- **Progressive enhancement**: the same story runs in text consoles and full 3D engines. Any IR must be runtime-agnostic (no host-specific resolution encoded).
- **Extensible by design**: the handler registry means compilation is not a closed-world operation. An IR produced under one handler configuration may not be valid under another.
- **Source-first authoring**: zoh stories are human-authored text, not compiled artefacts. Source is the canonical form today.

### Key Assumptions

| Assumption | Validity | Risk if Wrong |
|------------|----------|---------------|
| Compilation cost is negligible for typical story sizes | Likely valid today, questionable at scale | Large stories or many stories at startup would justify a caching IR sooner |
| Cross-language deployment is a future goal | Valid — multi-host embedding is a stated design goal | If single-runtime, portability argument is weaker |
| Handler configuration is stable per deployment | Questionable — different hosts may register different verbs | IR produced with custom handlers may not load cleanly on another host |
| Source availability is acceptable for all deployment contexts | Questionable — game distribution may need source-free binaries | If source must be hidden, an IR becomes urgent |

### Prior Work (Industry Reference)

| System | IR Form | Key Insight |
|--------|---------|-------------|
| Lua | Bytecode (.luac) | Flat instruction set, fast load, source-hiding |
| Python | .pyc marshalled AST | Per-version compatibility problem, not cross-impl |
| JVM | Class file bytecode | Stable cross-impl contract enabled the JVM ecosystem |
| WebAssembly | Binary module | Canonical IR enabled multi-language compilation targets |
| Ink (narrative scripting) | JSON compiled story | Source→JSON→runtime; runtime only needs JSON |

The Ink comparison is most relevant: Ink compiles `.ink` → `.json` (a structured representation of the story graph), and runtimes consume the JSON. This decouples the compiler from the runtime completely and enables runtime implementations in any language without re-implementing the parser/compiler.

---

## Analysis

### Analysis Area 1: The De-Facto IR Already Exists

**Finding:** `CompiledStory` is already an IR in function, just not in form.

**Evidence:**
- It is the output of all compilation phases and the input to all execution
- It is cached and reused across contexts in the same session
- Its data is structural (verb calls, parameters, attributes, checkpoints, metadata) with no host-specific handles

**Implications:**
An external IR is not a new concept to introduce — it is a serialisation of what already exists. The design question is not *whether to have an IR* but *how to make the existing IR externalisable*. This dramatically reduces the perceived novelty and risk of the feature.

---

### Analysis Area 2: Use Cases — Ranked by Strength

**Finding:** Four use cases exist; they differ significantly in urgency.

#### 2a. Cross-Language Portability (Strongest)

Zoh targets multi-host embedding. The C# reference implementation is one host. A Unity project, a Godot GDScript plugin, a Rust game engine, and a JavaScript web player are plausible future hosts. Each would need to implement the full compilation pipeline independently.

A canonical IR (e.g., a JSON schema spec'd in `spec/`) would let each host implement only an IR *loader* rather than a full compiler. This is the same value proposition as JVM bytecode or Ink's JSON: the compiler becomes a separate, reusable tool; runtimes only need to consume the IR.

This directly serves the "language-agnostic spec" design intent of `impl/`.

#### 2b. Source-Free Distribution (Moderate)

Game developers may want to ship story content without readable source (IP protection, obfuscation). An opaque compiled format (binary or encrypted JSON) serves this. This need is real but not urgent — Ink solved it the same way.

#### 2c. Compilation Caching (Weak today)

For small stories, recompilation on every startup is negligible. For large projects with hundreds of stories, a compiled-story cache on disk would improve startup time. This need may emerge at scale but is not present today.

#### 2d. Offline Tooling (Weak today)

A stable IR would let linters, formatters, and editors target a normalised representation rather than raw source. This is a secondary benefit; tooling can also work directly on the AST.

---

### Analysis Area 3: Design Constraints Specific to Zoh

**Finding:** Three zoh-specific properties complicate IR design.

#### 3a. Handler-Transformed AST

Compilers in the handler registry can read ahead and transform the AST before validation. This means the IR has two plausible shapes:

- **Pre-compiler IR** (raw parser output): portable across handler configurations, but requires each runtime to run its own compiler passes
- **Post-compiler IR** (fully compiled): fast to load, but handler-specific — an IR produced with custom verb compilers may not be loadable by a vanilla runtime

The Ink model resolves this by having the compiler produce a canonical flat representation that runtimes consume without any compiler passes. Zoh's extensible compiler complicates this: a post-compiler IR is host-specific.

**Resolution path**: Define a *canonical IR* as the post-parser, post-preprocessor, pre-compiler AST. This is portable and deterministic regardless of handler registration. Each runtime runs its compiler/validator passes at load time from the canonical IR, not from source. This skips only the expensive lexer/parser/preprocessor steps.

#### 3b. Lazy Expressions

Backtick expressions (`\`*a + *b\``) are stored as unevaluated AST and resolved at runtime. They cannot be pre-evaluated in an IR. However, their AST is fully encodable — the expression nodes (`BinaryExpr`, `ReferenceExpr`, `SpecialExpr`, etc.) map cleanly to a JSON-serialisable schema.

No special handling needed: IR carries expression ASTs verbatim.

#### 3c. Dynamic References

Path segments in references can be dynamic (`*map[*key]`). Like expressions, these are structural data that serialise straightforwardly. No special handling needed.

---

### Comparative Analysis: IR Form Options

| Aspect | Structured JSON | Binary Bytecode | Canonical AST (JSON/CBOR) |
|--------|----------------|-----------------|--------------------------|
| Human readability | High | None | Medium |
| Tooling compatibility | Excellent | Poor | Good |
| Load performance | Slow for large | Fast | Medium |
| Cross-impl stability | High | Requires spec | High |
| Compression | Moderate (gzip) | Intrinsic | Good (CBOR) |
| Fit with verb-driven model | Good | Awkward | Excellent |
| Implementation cost | Low | High | Low-Medium |
| Story of Ink | This is what Ink does | Not used | Variant of JSON |

**Verdict**: A **Canonical AST JSON** (or CBOR for performance) is the best fit for zoh. It is human-readable, toolable, and structurally faithful to the verb-call model. Binary bytecode adds complexity without clear value — the bottleneck in narrative scripting is not instruction dispatch.

---

## Evaluation Against Criteria

| Criterion | Score | Rationale |
|-----------|-------|-----------|
| Use-case validity | Adequate | Cross-language portability is real; others are future/weak |
| Fit with extensibility model | Adequate | Pre-compiler canonical IR avoids handler coupling; compiler passes re-run at load |
| Implementation cost | Strong | Serialising a clean AST schema is low-cost if data model is plain |
| Spec coherence | Strong | A canonical IR schema belongs in spec/ alongside the AST spec |
| Future-proofing | Strong | Decouples compiler from runtime; enables multi-language hosts |

**Overall Assessment:** The case is real but not urgent. The right move is to spec-preserve optionality cheaply now: ensure `CompiledStory` fields are IR-serializable (plain data, no host handles), and draft a canonical IR schema in the spec. Full implementation waits for a concrete driver.

---

## Challenges and Risks

### Identified Challenges

1. **Handler-IR coupling**: If IR is post-compiler, it is handler-configuration-specific. Solution: define IR as post-preprocessor/parser, pre-compiler. Compiler passes run at IR-load time.
2. **Format versioning**: An IR spec must carry a version field. Schema evolution must be managed. Minor spec changes (e.g., new standard attributes) require IR version bumps or backward-compatible extensions.
3. **Preprocessor boundary**: The `#embed` directive inlines files; the `#macro`/`#expand` directives expand templates. IR would capture post-preprocessor text, losing the embed/macro source mapping. Debug diagnostics referencing original source locations need a source-map companion if that is required.

### Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| IR format locks in prematurely before spec stabilises | Medium | Medium | Defer IR implementation until spec is stable; spec the schema now |
| Handler-coupled IR creates cross-runtime incompatibility | High if post-compiler | High | Define IR boundary as pre-compiler |
| Binary format chosen prematurely, tooling suffers | Low | Medium | Default to JSON; binary is an optimisation, not a requirement |
| Versioning neglected, migration breaks existing compiled stories | Medium | Medium | Mandate version field in schema from day one |

---

## Findings

### Key Findings

1. **`CompiledStory` is already the IR.** The gap is serialisability, not concept. This makes the feature an extension, not a new abstraction.
2. **Cross-language portability is the primary driver.** The multi-host embedding intent of zoh makes a canonical IR a natural complement to the language-agnostic `impl/` spec. It completes the story: spec tells you how to implement the compiler; IR tells you how to skip it.
3. **Pre-compiler is the right IR boundary.** Post-preprocessor, post-parse, pre-compiler AST is handler-independent and portable. Compiler and validator passes run at IR-load time, which is fast since they skip lexing and parsing.
4. **JSON is the right initial format.** Ink validates this choice in the narrative scripting domain. CBOR can be added later for performance-sensitive deployments.
5. **The design prerequisite is a clean data model.** If `CompiledStory` (or the pre-compiler AST) contains any host-language object references, serialisation becomes hard. Ensuring plain-data fields costs nothing now and enables IR later.

### Gaps Identified

- No spec currently describes the shape of `CompiledStory` as a serialisable schema
- `impl/02_parser.md` defines AST node types but not a canonical interchange encoding
- No versioning story exists for the compiled representation

### Opportunities

- A canonical IR schema in `spec/` would be a natural extension of the existing AST spec
- A `zohc` standalone compiler tool (compiles source → IR file) would be a clean complement to the embedded runtime
- IR opens a path to a pure-interpreter runtime that only implements a loader, not a full parser/compiler — lowering the barrier to new host implementations

---

## Recommendations

### Primary Recommendation

**Do not implement a serialisable IR now.** No concrete deployment driver exists today. Instead:

1. **Ensure the pre-compiler AST data model is IR-serialisable** — all AST node types should contain only plain data (no host object handles, no closure references). This is a zero-cost design constraint that keeps the option open.
2. **Draft an IR schema spec** as a `spec/ir.md` document, describing the canonical JSON encoding of the post-preprocessor AST. This costs one spec document and commits to nothing.
3. **Add a version field hint** to story metadata or a new IR envelope so that when serialisation is implemented, versioning is not an afterthought.

### Conditional Recommendations

- **If a second host implementation is started** (e.g., a Godot or Unity plugin): implement the IR immediately. The second implementor should not have to re-implement the full pipeline.
- **If source-free distribution is required** for a game release: implement IR with an encryption/obfuscation layer over the JSON.
- **If startup time becomes a problem** at scale (hundreds of stories): implement disk-based compilation cache using the IR schema.

### Suggested Next Steps

1. Audit `impl/02_parser.md` AST node definitions: confirm all fields are plain data with no host-language object references
2. Draft `spec/ir.md`: canonical JSON encoding of the post-preprocessor AST, with version field and envelope schema
3. Track this eval as a reference when a second host implementation or distribution requirement emerges

---

## Open Questions

- [ ] Does the C# `CompiledStory` currently hold any host-language handles (delegates, object references) that would block serialisation?
- [ ] Should the IR boundary be post-preprocessor (canonical source) or post-parser (AST)? The post-parser form is smaller and faster to load; the post-preprocessor form preserves more for debugging.
- [ ] If the IR is post-parser but pre-compiler, do compiler passes on IR-load run fast enough to be unnoticeable? (Likely yes for narrative story sizes.)
- [ ] Should `spec/ir.md` be a normative spec or a non-normative reference?

---

## Appendix

### Methodology

Reviewed `impl/00_overview.md` through `impl/12_validation.md`, the C# source structure under `csharp/src/Zoh.Runtime/`, the Ink narrative scripting comparison (external), and existing projex documents. No implementation was examined at the line level; this is an architectural evaluation.

### Sources

- `impl/00_overview.md` — pipeline phases and dependency graph
- `impl/09_runtime.md` — `CompiledStory` cache, `loadStory` interface
- `impl/12_validation.md` — compilation phases and diagnostics
- `impl/02_parser.md` — AST node definitions
- Ink language JSON format (external reference for narrative IR precedent)
- `20260208-project-structure-map.md` — repo organisation

### Dissenting Views

- **"Don't spec what you don't implement"**: A counterargument is that speccing an IR without implementing it creates dead spec. Counter-counter: the schema is a natural extension of the AST spec already in `impl/02_parser.md` and costs little to write while the AST is still being defined.
- **"Binary bytecode is faster"**: For narrative scripting at typical story sizes, JSON parse time is negligible. Binary optimisation is premature. Revisit if profiling shows otherwise.
