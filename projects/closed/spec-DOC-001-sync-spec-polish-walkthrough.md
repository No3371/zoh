# Walkthrough: spec-DOC-001-sync-spec-polish

**Status**: Completed & Verified
**Date**: 2026-01-24

## 1. Objective

This project synchronized the implementation guides (`impl/`) and standard attributes documentation (`std_attributes.md`) with recent changes in the main specification (`spec.md`). Key updates included defining attributes as data/decorators, documenting imperative expression sugar, and adding usage notes for statement chaining and block syntax.

## 2. File Changes

### `std_attributes.md`

Rewrote the Preamble to explicitly define attributes as "Data decorations" that serve for both common parameters and metaprogramming signals, correcting the previous "common definitions" simplification.

### `impl/04_expressions.md`

Added a **Usage & Sugar** section documenting:
- Imperative evaluation sugar (`` /`expr`; ``)
- Usage of expressions as arguments (`/if `*a > 5`, ...`)

### `impl/06_core_verbs.md`

Added a **Usage & Syntax** section clarifying:
- Statement Chaining (`/verb; -> *var;`) is idiomatic, not a single syntactic unit.
- Block Form (`/verb/ ... /;`) usage.
- Standard verb sugar tables replaced by cross-reference to parser guide.

## 3. Verification

Since this was a documentation-only project, verification involved reviewing the generated markdown to ensure:
- It matches the `spec.md` definitions.
- It provides clear, actionable guidance for implementers.
- No technical inaccuracies were introduced (e.g., correctly identifying `->` as a standalone statement).

## 4. Key Insights

- **Documentation Lag**: "Sugar" features often get implemented in the parser but missed in the implementation guides, leaving a gap where implementers might miss them or misimplement them.
- **Sugar vs Syntax**: It's crucial to distinguish between parser-level sugar (like `->`) and verb-level syntax to avoid confusion. Documenting idiomatic patterns helps bridge this gap without misleading about the technical grammar.
