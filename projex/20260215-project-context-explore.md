# Project Context & Scenario Exploration

> **Created:** 2026-02-15 | **Author:** Antigravity
> **Type:** Domain
> **Related Projex:** 

---

## Summary
ZOH is a mature, verb-based embedded scripting language designed for "stage direction" storytelling. It features a robust C# runtime with advanced features like parallel contexts and channel-based concurrency. However, it currently lacks "end-to-end" validation scenarios that test these features in concert at scale. The exploration identified capabilities for complex logic, state management, and concurrency, but also highlighted risks in resource management and race conditions. A list of end-to-end scenarios has been generated to bridge this gap.

**Guiding Questions:**
1. What is the core purpose and functionality of the ZOH language/system?
2. What are the key components and how do they interact?
3. What are the existing capabilities that support end-to-end user scenarios?
4. What are potential gaps or areas for new end-to-end scenarios?

**Scope:** 
- Core language specifications (`spec/`, `intro.md`)
- Runtime implementation status (`c#/`)
- Existing project documentation and plans (`projex/`)
- Goal: Generate a list of end-to-end scenario ideas.

---

## Context

**Why Now:** User requested a deep exploration to prepare a list of end-to-end scenario ideas for future design.
**Current State:** Runtime is feature-complete but untested in complex, integrated scenarios. Redteam analysis flags concurrency/resource risks that such scenarios should target.
**Stakeholders:** User, Developers.


---

## Investigation Targets

> Targets are identified upfront and revised during exploration.

### Target: `intro.md`
**Rationale:** Understand the high-level vision and "why" of ZOH.
**Status:** Done
**Findings:**
- **Philosophy:** ZOH aligns closer to "stage directions" than pure text generation (like Ink/Twine). It commands a runtime to "perform" the story.
- **Key Concepts:**
    - **Everything is a Verb:** Uniform syntax (`/verb`) for dialogue, logic, and presentation.
    - **Parallel Contexts:** Running limits/loops alongside main dialogue (e.g., ambient noise, NPC schedules).
    - **Inversion of Control:** Script drives the runtime (presentation), rather than runtime polling script for text.
- **Example:** "The Last Coffee Shop" demonstrates:
    - `====+` for parallel contexts.
    - `/push` / `/pull` for channel synchronization between contexts.
    - Mixed presentation (`/show`, `/play`) with logic.

---

### Target: `spec/`
**Rationale:** Understand the specific capabilities and "how" of the language to construct valid scenarios.
**Status:** Done
**Findings:**
- **Core Mechanics:**
    - **Verbs:** The fundamental unit. Can have attributes `[attr]`, named/unnamed parameters.
    - **Variables:** Typed (or `nothing`), scoped to Story or Context.
    - **Types:** Primitives (`string`, `int`, `double`, `bool`), Collections (`list`, `map`), System (`channel`, `verb`, `expression`, `reference`).
- **Data & Control Flow:**
    - `/set`, `/get`, `/drop` for variable management.
    - `/if`, `/while`, `/loop`, `/sequence` (likely in std_verbs, implied by intro).
    - `/capture` to get verb results.
    - `/try` for error handling.
- **Inter-Context Communication:**
    - Channels (`<name>`) permit `/push` and `/pull` for data exchange between parallel contexts.
- **Expression System:**
    - Backtick expressions (`` `expr` ``) for math/logic evaluation.
    - Interpolation (`${*var}`) for string construction.
- **Structure:**
    - Story files (`.zoh`) with headers and metadata.
    - Explicit parsing phases (Pre-processor -> Story Body).

---

### Target: `c#/README.md` (and structure)
**Rationale:** Assess the current implementation status to know what scenarios are strictly "future design" vs "currently possible".
**Status:** Done
**Findings:**
- **Structure:** well-organized C# solution with `Zoh.Runtime` and `Zoh.Tests`.
- **Components:**
    - `Lexing`, `Parsing`, `Preprocessing` (supports `#embed` and macros).
    - `Execution` (Runtime, Context, Channels).
    - `Verbs` (Core, Flow, Signals, Store).
    - `Expressions` & `Interpolation`.
- **Maturity:** High coverage of spec features. Implementation is far along.
- **Risks:** Redteam analysis (`20260207-spec-impl-redteam.md`) identified:
    - Channel race conditions (recreating closed channels).
    - Unbounded resource consumption (fork bombs, infinite lists).
    - Expression injection via `${*var}` if `*var` contains malformed/malicious content.

---

### Target: Existing Projex (`projex/`)
**Rationale:** Avoid duplicating existing ideas and build upon current momentum.
**Status:** Done
**Findings:**
- **Active:**
    - `20260207-spec-impl-redteam.md`: Critical security/stability analysis.
    - `20260208-project-structure-map.md`: High-level index.
    - `20260208-checkpoint-type-contract-proposal.md`: Proposal for typed checkpoints.
- **Closed:** Many spec/impl tasks.
- **Gap:** No "end-to-end scenario" list found. The redteam doc suggests stress tests, but not necessarily user-facing scenarios.

---

## Patterns & History

**Patterns Found:**
- **Inversion of Control:** Script drives the runtime (Presentation) rather than Runtime querying Script (Text).
- **Fork-Join Concurrency:** Heavy use of `/fork`, `/call` and channels for non-linear storytelling and system simulation.
- **Data-Driven Logic:** Extensive use of `list` and `map` for state, enabling RPG-like systems within the script.

**Evolution:** Originated as a text-generation tool, evolved into a general-purpose state machine/director.

---

## Findings

### Discoveries
1. **Implementation is Advanced:** Leaps ahead of "prototype" stage. AST, Lexing, Parsing are solid.
2. **Concurrency is the Frontier:** Features exist (`/fork`, `<channel>`) but are the source of most theoretical risks (race conditions, deadlocks).
3. **"Verbs" are Powerful:** The uniform syntax allows for metagaming and easy extension.

### Mental Model
ZOH is a "Director" that shouts commands (`/verbs`) to a "Stage" (Runtime). The Director can hire generic assistants (Contexts) to do background tasks, communicating via walkie-talkies (Channels).

### Implications
- **Scenarios must be concurrent:** Single-threaded scenarios won't stress the system's unique value props or risks.
- **Scenarios must be state-heavy:** To test the limits of variable scoping and collection management.

---

## End-to-End Scenario Ideas

The following scenarios are proposed to test ZOH's capabilities and identified risks.

### 1. The "MUD Server" (Concurrency & Channels)
**Concept:** Simulating a Multi-User Dungeon where multiple "Players" (contexts) interact in "Rooms" (channels).
**Goal:** Stress test channel concurrency, race conditions, and context isolation.
**Key Mechanics:** `/fork` per player, `<room_channel>` for chat/events, `/push`/`/pull` generic message passing.

### 2. The "Ecosystem Simulation" (Resource limits)
**Concept:** 100+ "Creature" contexts living, eating, and reproducing in a shared world.
**Goal:** Stress test resource limits (context count, memory), scheduler fairness, and performance.
**Key Mechanics:** `/loop` life cycles, dynamic `/fork` (reproduction), shared `<world_state_channel>`.

### 3. The "Card Battler" (Complex State)
**Concept:** A Hearthstone-lite game with Deck, Hand, Board, and Graveyard collections.
**Goal:** Verify complex list/map manipulations, deep copy/reference behavior, and state consistency.
**Key Mechanics:** `/append`, `/remove`, `/shuffle` (via std lib patterns), nested `map` structures.

### 4. The "Metagame Tutorial" (Injection & Reflection)
**Concept:** A game that modifies its own rules or specifically asks the user to "hack" it by inputting ZOH code.
**Goal:** Test the "Expression Injection" risk and runtime safety/sandboxing.
**Key Mechanics:** `/parse` user input, `/interpolate` dynamic strings, `/eval` (if exposed/safe).

---

## Answers

**What is the core purpose and functionality of the ZOH language/system?**
To act as a "stage director" layer that controls narrative flow, state, and presentation in a unified, verb-based syntax.

**What are the key components and how do they interact?**
Runtime executes Scripts. Scripts use Verbs to manipulate State (Variables) and control Flow (Contexts). Contexts communicate via Channels.

**What are the existing capabilities that support end-to-end user scenarios?**
Full control flow (`/if`, `/while`), Concurrency (`/fork`, channels), State (`map`, `list`), and IO (`/converse`, `/show`).

**What are potential gaps or areas for new end-to-end scenarios?**
Complex integrated concurrency (MUD), heavy resource load (Sim), and hostile input handling (Injection).

---

## Open Questions

- [ ] [Unresolved question]

---

## Appendix

**Sources:**
**Limitations:**
