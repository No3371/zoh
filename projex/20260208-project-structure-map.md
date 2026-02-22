# ZOH Repository Structure Map

**Created**: 2026-02-08
**Last Revised**: 2026-02-12
**Author**: Antigravity
**Scope**: Repository-wide structural index and orientation guide

---

## Overview

ZOH is an embedded scripting language for interactive storytelling where everything is a "verb" — characters `/converse`, scenes `/show`, music `/play`, even variables (`/set`) and control flow (`/if`, `/while`). The repository contains the complete language specification, a C# reference implementation with comprehensive test coverage, implementation specs for building runtimes, project management documents, and examples demonstrating the language in action.

---

## Structure

```
S:\Repos\zoh/
├── spec/                           — Complete language specification directory
├── expr.md                         — Expression grammar and evaluation rules
├── std_verbs.md                    — Standard verb definitions (variables, control flow, collections)
├── std_attributes.md               — Standard attribute definitions ([scope], [required], [resolve], etc.)
├── std_flags.md                    — Runtime flag definitions
├── std_metadata.md                 — Story metadata schema
├── readme.md                       — Project readme with "At a Glance" example
├── intro.md                        — Narrative introduction comparing ZOH to Ink/Twine
├── AGENT.md                        — Agent/AI assistant instructions
├── CLAUDE.md                       — Symlink to AGENT.md
├── GEMINI.md                       — Symlink to AGENT.md
│
├── c#/                             — C# reference implementation
│   ├── Zoh.sln                     — Visual Studio solution file
│   ├── src/Zoh.Runtime/            — Runtime library source code
│   │   ├── Lexing/                 — Tokenization (Lexer, Token, TokenType, TextPosition)
│   │   ├── Parsing/                — AST construction (Parser, StoryAst, StatementAst, ValueAst)
│   │   ├── Preprocessing/          — Macro expansion and #embed directives
│   │   ├── Expressions/            — Expression parsing and evaluation (ExpressionParser, ExpressionEvaluator)
│   │   ├── Interpolation/          — String interpolation (ZohInterpolator)
│   │   ├── Execution/              — Runtime execution (ZohRuntime, Context, ChannelManager, SignalManager)
│   │   ├── Types/                  — Type system implementation
│   │   ├── Variables/              — Variable storage and scoping
│   │   ├── Verbs/                  — Verb implementations
│   │   │   ├── Core/               — Core verbs (set, get, capture, type, count, etc.)
│   │   │   ├── Flow/               — Control flow verbs (if, sequence, loop, while, foreach, switch)
│   │   │   ├── Signals/            — Channel/signal verbs (push, pull, close)
│   │   │   └── Store/              — Collection verbs (append, insert, remove, clear, has)
│   │   ├── Validation/             — Semantic validation and diagnostics
│   │   ├── Diagnostics/            — Diagnostic system (Diagnostic, DiagnosticBag, DiagnosticSeverity)
│   │   ├── Storage/                — Persistence layer
│   │   └── Helpers/                — Utility functions
│   │
│   ├── tests/Zoh.Tests/            — Comprehensive test suite
│   │   ├── Lexing/                 — Lexer tests (LexerTests, LexerSpecComplianceTests)
│   │   ├── Parsing/                — Parser tests (ParserTests, ParserSpecComplianceTests, ParserComplexTests)
│   │   ├── Preprocessing/          — Preprocessor tests
│   │   ├── Expressions/            — Expression tests (ExpressionTests, ExpressionParserComplianceTests)
│   │   ├── Interpolation/          — Interpolation tests
│   │   ├── Execution/              — Runtime execution tests (RuntimeTests, ContextTests, ChannelManagerTests, SignalTests)
│   │   ├── Runtime/                — Type system tests (MapStringKeyTests)
│   │   ├── Types/                  — Type-specific tests
│   │   ├── Variables/              — Variable tests
│   │   ├── Verbs/                  — Verb behavior tests organized by category (Core/, Flow/, Store/)
│   │   └── Vulnerabilities/        — Security and edge-case tests
│   │
│   ├── projex/                     — C# implementation project management
│   │   ├── closed/                 — Completed C# implementation tasks
│   │   ├── 20260207-csharp-runtime-nav.md
│   │   ├── 20260208-parse-whitespace-trimming-csharp-plan.md
│   │   └── 20260211-csharp-crlf-handling-explore.md
│   │
│   ├── projects/                   — Legacy C# project tracking
│   │   ├── closed/                 — Archived C# projects
│   │   └── .agent/                 — Agent configurations
│   │
│   └── report/                     — Build/test reports (currently empty)
│
├── impl/                           — Implementation specifications for building ZOH runtimes
│   ├── 00_overview.md              — Runtime implementation overview
│   ├── 01_lexer.md                 — Lexer implementation spec
│   ├── 02_parser.md                — Parser implementation spec
│   ├── 03_preprocessor.md          — Preprocessor implementation spec
│   ├── 04_expressions.md           — Expression evaluator implementation spec
│   ├── 05_type_system.md           — Type system implementation spec
│   ├── 06_core_verbs.md            — Core verb implementation spec
│   ├── 07_control_flow.md          — Control flow verb implementation spec
│   ├── 08_concurrency.md           — Concurrency/channel implementation spec
│   ├── 09_runtime.md               — Runtime architecture spec
│   ├── 10_std_verbs.md             — Standard verb implementation spec
│   ├── 11_storage.md               — Storage/persistence implementation spec
│   ├── 12_validation.md            — Validation implementation spec
│   ├── readme.md                   — Implementation specs index
│   └── projex/                     — Implementation-specific project documents
│       └── closed/                 — Completed implementation spec tasks
│
├── projex/                         — Project management and task tracking (Projex framework)
│   ├── .agent/                     — Projex framework agent configurations
│   ├── closed/                     — Completed cross-cutting projects/tasks
│   ├── 20260207-spec-impl-redteam.md
│   ├── 20260208-checkpoint-type-contract-proposal.md
│   ├── 20260208-newline-handling-explore.md
│   ├── 20260208-parse-whitespace-trimming-plan.md
│   ├── 20260208-project-structure-map.md
│   ├── 20260208-remove-truthiness-proposal.md
│   └── 20260208-verify-type-system.md
│
├── projects/                       — Legacy project tracking system
│   ├── closed/                     — Archived specification and test projects (25+ completed items)
│   ├── archived/                   — Older archived projects (18+ items)
│   ├── feat-*.md                   — Active feature specifications
│   └── (agent configs)             — Agent instruction files
│
├── examples/                       — Example ZOH scripts
│   ├── example_murder_mystery.zoh  — Interactive story example
│   └── original.ink                — Reference Ink script for comparison
│
├── discussions/                    — Design discussions and rationale documents
│   ├── deep-think-NS-001-ambiguity.md  — Namespace ambiguity analysis
│   └── why_not_break_and_continue.md   — Design decision rationale
│
├── review/                         — Code review and analysis documents (currently empty)
├── reviews/                        — Code review archives (currently empty)
│
└── .vscode/                        — VS Code workspace settings
    └── settings.json
```

---

## Key Root Files

| File | Purpose |
|------|---------|
| `spec/` | **Canonical language specification** — Complete ZOH syntax, semantics, type system, and verb definitions |
| `expr.md` | **Expression grammar** — EBNF grammar for expressions, operator precedence, special forms |
| `std_verbs.md` | **Standard verb reference** — Definitions for core verbs (set, get, if, loop, push, pull, etc.) |
| `std_attributes.md` | **Attribute reference** — Standard attributes like [scope], [required], [resolve], [typed] |
| `readme.md` | **Project introduction** — "At a Glance" example showcasing parallel contexts and channels |
| `intro.md` | **Design philosophy** — Narrative comparing ZOH to Ink/Twine, explaining design choices |
| `AGENT.md` | **AI assistant instructions** — Project overview, build commands, conventions (symlinked to CLAUDE.md, GEMINI.md) |

---

## Conventions

### Naming Patterns
- **Project documents**: `YYYYMMDD-description.md` (dated for chronological tracking)
- **Closed projects**: Prefixed with category (e.g., `fix-LEX-001-`, `spec-CORE-002-`, `impl-SUGAR-001-`)
- **Feature proposals**: `feat-AREA-NNN-description.md` format
- **Test files**: `*Tests.cs` for unit tests, `*ComplianceTests.cs` for spec compliance tests

### Directory Organization
- **Separated concerns**: Spec files (root), implementation (c#/), impl specs (impl/), project mgmt (projex/)
- **Test mirroring**: Test directory structure mirrors source directory structure exactly
- **Closure tracking**: All project directories have `closed/` subdirectories for completed work
- **Agent configs**: `.agent/` directories for AI assistant workflows, skills, and rules

### Code Organization (C#)
- **Namespace alignment**: Directory structure matches namespace hierarchy (e.g., `Verbs/Core/` → `Zoh.Runtime.Verbs.Core`)
- **Separation of concerns**: Distinct directories for each compilation stage (Lexing → Parsing → Preprocessing → Execution)
- **Build artifacts**: `bin/` and `obj/` directories excluded from version control
- **Test categories**: Tests organized by component (Lexing, Parsing, Expressions, Execution, Verbs)

### Documentation Standards
- **Spec files**: Markdown with EBNF grammar notation, example code blocks in `zoh` language
- **Implementation specs**: Numbered sequence (00-12) matching compilation/execution pipeline stages
- **Project docs**: Follow Projex framework templates (Plan, Walkthrough, Log, Review formats)

---

## Unmapped Areas

- **S:\Repos\zoh\.agent/**: Repository-level agent configurations (skills, workflows, rules) — not explored in detail
- **S:\Repos\zoh\.claude/**: Claude Code-specific configuration — purpose unclear
- **S:\Repos\zoh\.git/**: Git internals — standard version control, excluded from map
- **Build artifacts**: All `bin/`, `obj/`, `TestResults/` directories — transient, excluded from map
- **Database files**: `.repomap`, `.repomap.db`, `repomap.db` — agent memory databases, not code
- **Archive files**: `zoh.7z` — compressed backup, not actively maintained

---

## Revision Log

| Date | Author | Changes |
|------|--------|---------|
| 2026-02-08 | Claude Sonnet 4.5 | Initial map creation — surveyed entire repository structure, documented all major directories and key files |
| 2026-02-12 | Antigravity | Revised map — updated spec/ structure, added symlink details for CLAUDE.md/GEMINI.md, listed active projex items in c#/ and root directories. |
