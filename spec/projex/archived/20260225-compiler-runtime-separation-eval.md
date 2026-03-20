# Compiler / Runtime Separation — Evaluation

> **Created:** 2026-02-25
> **Author:** agent
> **Subject:** Whether the zoh compiler and runtime should be separate packages, and what "separation" even means given the handler architecture
> **Type:** Comparative + Gap Analysis
> **Related Projex:** 20260225-compiled-ir-eval.md, 20260225-canonical-ir-spec-proposal.md, 20260225-runtime-impl-friction-memo.md

---

## Executive Summary

The question "should compiler and runtime be separate packages?" cannot be cleanly answered yes or no for zoh, because the handler registry deliberately couples what other languages call "compilation" into the runtime. All four handler types — preprocessors, compilers, validators, and drivers — are registered on the same `Runtime` object, and validators critically depend on that registry to know which verbs exist. The only natural split point today is at the post-parse AST boundary (lexer/parser/preprocessor vs everything else), which separates the easy 5% from the hard 95% and has limited practical value.

A meaningful separation — where a standalone compiler tool can fully validate a story — requires verb implementations to declare their parameter and attribute schemas declaratively rather than through imperative handler code. That is a language design question, not a packaging question. If zoh moved in that direction, separation would become both possible and valuable. Until then, enforced separation creates an incomplete compiler and a fragmented developer experience.

---

## Evaluation Scope

### Subject

Whether to architect zoh as two separate packages — a compiler/build tool and an embedded runtime — versus the current integrated design where a single runtime object owns the full pipeline.

### Questions Addressed

1. What does "compiler" and "runtime" actually mean in zoh's architecture?
2. Where are the natural split points in the current pipeline?
3. What value does each split point provide?
4. What prevents a clean, complete separation today?
5. What language design changes would make separation genuinely valuable?

### Out of Scope

- Specific package manager or deployment mechanisms
- The IR format question (addressed in `20260225-canonical-ir-spec-proposal.md`)
- Performance optimisation

---

## Context Analysis

### Current State

From `impl/09_runtime.md`, the `Runtime` interface owns everything:

```
Runtime:
  preprocessors: List<Preprocessor>    # source text → source text
  compilers: List<Compiler>             # AST node → CompiledStory
  storyValidators: List<StoryValidator> # CompiledStory → diagnostics
  verbValidators: Map<string, VerbValidator>
  verbDrivers: Map<string, VerbDriver>  # execution
```

The pipeline is strictly sequential and all phases run through the same registry:

```
Source → Preprocessors → Lexer → Parser → Compilers → Validators → Execution
```

Crucially, the `NamespaceValidator` requires the runtime's `HandlerRegistry` directly — it resolves verb calls by suffix-matching against registered `verbDrivers` to detect `unknown_verb` and `namespace_ambiguity`. Validation is not a closed-world operation; it requires knowing what the target runtime has registered.

### Key Architectural Property

The handler registry is the extension point. Hosts register custom preprocessors, compilers, validators, and drivers — all on the same runtime. This is intentional and load-bearing: it is what makes progressive enhancement work. A minimal runtime registers only core verb drivers; a rich runtime adds more. Compilation and validation adapt to whatever is registered.

---

## Analysis

### Analysis Area 1: What "Compiler" Means in Zoh

**Finding:** "Compiler" in zoh refers to two very different things that are conflated by the name.

The pipeline has a **source processor** (preprocessors + lexer + parser) and a **semantic processor** (compilers + validators). These are architecturally distinct:

| Phase | Input | Output | Handler-dependent? |
|-------|-------|--------|--------------------|
| Preprocessors | Raw source text | Transformed text | Yes — custom preprocessors |
| Lexer | Text | Token stream | No — fixed grammar |
| Parser | Token stream | AST | No — fixed grammar |
| Compilers | AST nodes | CompiledStory | Yes — per-verb compilers |
| Validators | CompiledStory | Diagnostics | Yes — per-verb validators + NamespaceValidator needs registry |
| Drivers | CompiledVerbCall | Execution | Yes — per-verb drivers |

The source processor (lexer + parser) is handler-independent. Everything else depends on what handlers are registered.

**Implications:** The only handler-independent split is at the post-parse AST. Everything below that — compile, validate, execute — requires handler context.

---

### Analysis Area 2: The Two Split Points

#### Split A: Post-parse (source processor vs runtime)

```
[zoh-parser]                    [zoh-runtime]
preprocessors + lexer + parser  →  IR  →  compilers + validators + drivers
```

- `zoh-parser` produces IR (post-parse AST). Handler-independent.
- `zoh-runtime` loads IR, runs its registered handler pipeline, executes.

**Value:**
- New runtime implementations skip the lexer/parser/preprocessor
- A standalone syntax-check tool is possible
- Smaller embedded runtime (no parser shipped)

**Limitation:** `zoh-parser` can only catch syntax errors. It has no knowledge of verbs, so it cannot catch parameter errors, type mismatches, unknown verbs, or any semantic issues. Meaningful validation still requires the runtime.

#### Split B: Post-compile (build tool vs embedded runtime)

```
[zohc build tool]                        [zoh-runtime embedded]
preprocessors + lexer + parser           IR loader
+ compilers + validators         →  IR  →  + drivers (execution only)
```

- `zohc` is a standalone tool: validates and compiles stories to IR
- `zoh-runtime` loads IR and only dispatches verb drivers — no compilation or validation at load time

**Value:**
- Full validation at build time
- Smallest possible embedded runtime (no parser, no compiler, no validator)
- Clean separation of authoring-time from execution-time concerns

**Critical problem:** `zohc` must know which verbs exist to run `NamespaceValidator`. But verb sets are host-specific — a Unity runtime registers different verbs than a Godot runtime. `zohc` cannot validate a story against an unknown host's verb set. It can only validate against the standard verb set.

This means either:
- Validation is incomplete (can miss `unknown_verb` for custom verbs)
- Or hosts provide `zohc` with a capabilities manifest describing their registered verbs

Without a capabilities manifest mechanism, Split B produces a `zohc` that gives false confidence — it validates the story against standard verbs but silently passes custom verb calls that don't exist in the target runtime.

---

### Analysis Area 3: Why the Handler Registry Prevents Clean Separation

**Finding:** The handler registry is the core coupling mechanism, and it was designed that way.

The handler registry enables zoh's progressive enhancement: hosts register whatever verbs they support. Validation adapts to that registration. This is not a bug — it is the primary extensibility mechanism.

The consequence is that compilation and validation are not closed-world operations. A `zohc` standalone tool cannot replicate what the runtime's handler pipeline does without having the same handler registrations. The options are:

1. **Accept incomplete validation in `zohc`** — validate syntax and standard verbs only. Custom verb errors surface only at runtime. This is a worse developer experience than just running the story.

2. **Ship a capabilities manifest with each runtime** — hosts publish a machine-readable description of their registered verbs, attributes, and parameter schemas. `zohc` loads the manifest and can validate fully. This is a significant additional mechanism to design and maintain.

3. **Move verb declarations to be declarative** — instead of imperative handler code, verbs declare their parameters and attributes in a schema. `zohc` reads the schemas to validate without running the handler code. This is the deepest change but makes separation genuinely clean.

---

### Analysis Area 4: What Declarative Verb Declarations Would Enable

**Finding:** Declarative verb parameter schemas are the key that unlocks clean separation.

Currently, verb validation is imperative: each `VerbValidator` contains code that checks its verb's parameter constraints. That code can only run when the validator is registered and invoked by the runtime.

If instead, verbs declared their parameters declaratively:

```
VerbDeclaration:
  name: string
  namespace: string
  parameters: List<ParamSchema>
  attributes: List<AttributeSchema>

ParamSchema:
  name: string?        # null = unnamed
  type: string         # "string" | "integer" | ... | "any"
  required: bool
  ...
```

Then:
- `zohc` loads verb declarations (from a standard library + a host-provided extension set) and validates stories against them without running any handler code
- The runtime uses the same declarations to drive validation (the declarative schema replaces imperative validator code for the common case)
- Custom validators for complex constraints can supplement the schema but the schema handles the common case

This is analogous to how OpenAPI specs, GraphQL schemas, or protobuf definitions decouple interface definition from implementation.

Partial steps toward this already exist in zoh: `spec/2_verbs.md` and `spec/std_verbs.md` define verb parameters and attributes in structured prose. Formalizing this into a machine-readable schema would be the change needed.

---

### Comparative Analysis

| Aspect | No split (status quo) | Split A (post-parse) | Split B (post-compile) | Split B + declarative schemas |
|--------|----------------------|----------------------|----------------------|-------------------------------|
| New runtime impl burden | Full pipeline | Skip parser only | Skip parser + compile + validate | Skip parser + compile + validate |
| Validation completeness | Full | Syntax only | Standard verbs only | Full (with manifest) |
| Embedded runtime size | Large | Smaller (no parser) | Smallest | Smallest |
| Build tool usefulness | N/A | Syntax check only | Incomplete (custom verbs) | Full validation |
| Implementation complexity | Low | Low | Medium | High (schema design) |
| Progressive enhancement | ✅ | ✅ | ⚠️ (validation gap) | ✅ |
| Fits current architecture | ✅ | ✅ | ❌ (registry coupling) | Requires redesign |

---

## Findings

### Key Findings

1. **"Compiler" in zoh is really two things:** a handler-independent source processor (lexer/parser/preprocessor) and handler-dependent semantic processing (compile passes, validators). These have different split implications.

2. **The handler registry prevents meaningful Split B today.** `NamespaceValidator` requires the handler registry; validation cannot run standalone without knowing what verbs are registered. This is architectural, not accidental.

3. **Split A (post-parse) is clean but limited.** It separates the easy, well-specified part and leaves the hard, host-specific part in the runtime. Value is modest.

4. **Declarative verb schemas are the architectural enabler.** Moving verb parameter declarations from imperative validator code to a machine-readable schema would allow `zohc` to validate fully against a declared capability set, making Split B genuinely clean and valuable.

5. **The IR proposal and this question are linked.** If Split A is adopted, the IR is the post-parse AST (as the current proposal concludes). If Split B becomes viable (via declarative schemas), the IR could be the fully validated compiled form, and the embedded runtime truly becomes just an execution engine.

### Gaps Identified

- Verb parameter and attribute declarations exist in spec prose but not as machine-readable schemas
- No capabilities manifest mechanism exists for describing a host's registered verbs to external tools
- The `impl/` spec does not separate lexer/parser from the broader runtime concern

### Opportunities

- Formalizing verb declarations as machine-readable schemas would benefit tooling (IDE completion, linting) independently of the compiler/runtime separation question
- A thin `zoh-parser` package is achievable today with no architectural changes — it just wraps the lexer/parser/preprocessor and produces IR
- The declarative schema direction aligns with reducing the 95% implementation burden (see `20260225-runtime-impl-friction-memo.md`) — if verb schemas are machine-readable, new runtimes might consume them to drive validation automatically rather than writing imperative validator code

---

## Recommendations

### Primary Recommendation

**Do not force a compiler/runtime split now.** The current integrated design is appropriate given the handler registry architecture. A forced split today either yields an incomplete build tool (Split A: syntax only) or breaks the progressive enhancement model (Split B without declarative schemas).

### Conditional Recommendations

- **If a standalone `zoh-parser` is wanted** (for syntax-checking, IR generation, editor tooling): Split A is achievable today at low cost. The lexer/parser/preprocessor can be factored into a thin package that produces post-parse IR. Value is modest but real.

- **If a fully validating `zohc` build tool is wanted**: Design declarative verb parameter schemas first. This is a language design investment that pays dividends in tooling, documentation, and reduced implementation burden for new runtimes — not just in enabling the split.

- **If the goal is a smaller embedded runtime**: Declarative schemas are still the prerequisite. Once validation can be driven by schemas rather than imperative handlers, the runtime sheds significant weight.

### Suggested Next Steps

1. Evaluate whether verb parameter declarations should be formalized as a machine-readable schema — this is the higher-leverage question that the compiler/runtime separation depends on
2. If yes: that schema design is the path toward a genuinely useful `zohc` tool and a lighter embedded runtime
3. If no: keep the integrated design; optionally factor out the parser as a thin utility package if tooling demand warrants it

---

## Open Questions

- [ ] Is the intent for zoh runtimes to always ship the full pipeline (parse + compile + validate + execute), or is there a desired mode where the embedded component is execution-only?
- [ ] Would declarative verb schemas replace imperative verb validators entirely, or supplement them for the common case?

---

## Appendix

### Methodology

Analysis based on `impl/09_runtime.md` (full runtime interface and handler types), `impl/03_preprocessor.md` (preprocessor pipeline), `impl/02_parser.md` (AST node types), and the current conversation thread establishing IR and architecture context.

### Sources

- `impl/09_runtime.md` — Runtime interface, handler types, NamespaceValidator
- `impl/03_preprocessor.md` — Preprocessor pipeline and extensibility
- `impl/02_parser.md` — AST node types
- `20260225-compiled-ir-eval.md` — IR use case analysis
- `20260225-canonical-ir-spec-proposal.md` — IR boundary decision
- `20260225-runtime-impl-friction-memo.md` — 95% implementation burden

### Dissenting Views

- **"Just do Split A, it's better than nothing"**: Valid if there's concrete demand for a standalone syntax checker or IR generator today. The cost is low and the value, while modest, is real.
- **"The coupling is a design flaw, not a feature"**: An argument can be made that the handler registry over-couples concerns. Other extensible scripting languages (Lua, Wren) separate the language core from extension registration. Zoh could take a similar approach. This is a legitimate design alternative but represents a deeper architectural revision than is appropriate to evaluate here.
