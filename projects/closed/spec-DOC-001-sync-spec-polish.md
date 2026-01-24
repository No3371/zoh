# spec-DOC-001: Sync Documentation with Spec Polish

## Priority
**Medium** - The specification (`spec.md`) has evolved (Attributes as Data, Block Syntax flex, Sugar), but implementation guides and standard definitions are lagging. This causes confusion during implementation.

## Problem

Recent updates to `spec.md` introduced or clarified several key concepts that are missing or misrepresented in the `impl/` guides and `std_*.md` files:

1.  **Attributes as Decorators**: `spec.md` now defines attributes as "Data" and "Decorators" for metaprogramming (Handlers), whereas `std_attributes.md` treats them simply as "Common Parameters".
2.  **Expression Sugar**: The `` /`expr` `` syntax (sugar for `/evaluate`) is the primary way expressions are used imperatively, but `impl/04_expressions.md` only covers the internal grammar.

3.  **Usage Patterns**: `impl/06_core_verbs.md` lacks examples for the `->` capture sugar (which is a standalone statement) and the idiomatic "one-liner chaining" pattern (`/verb; -> *var;`), as well as Block Form usage.

**References:**
- `spec.md` (Attributes section, Block Form section, Sugar section)
- `std_attributes.md` (Preamble)
- `impl/04_expressions.md` (Missing usage section)


## Proposed Change

Update the documentation to align with `spec.md`. This is a documentation-only project.

### 1. `std_attributes.md`
- Rewrite the Preamble to define attributes as "Data decorations" that serve two purposes: common parameters and metaprogramming signals.
- Remove the "implementation-defined" ambiguity if it contradicts the spec's definition of how they should be parsed (though handling is still implementation-dependent).

### 2. `impl/04_expressions.md`
- Add a "**Usage & Sugar**" section.
- Explain that `/`expr`` desugars to `/evaluate `expr``.
- Explain that expressions can be used as arguments (e.g., `/if `*a > 5`, ...`).



### 4. `impl/06_core_verbs.md`
- Add a "**Syntax Notes**" section or update existing examples to include:
    - **Capture Sugar**: `/core.verb; -> *var;`
    - **Block Form**: `/sequence/ ... /;`
    - **Sugar Usage**: `` /`1+1`; ``

## Acceptance Criteria

1.  `std_attributes.md` preamble aligns with `spec.md`'s definition of attributes.
2.  `impl/04_expressions.md` documents the `/`expr`` syntax.

4.  `impl/06_core_verbs.md` provides examples of Capture and Block syntax usage.