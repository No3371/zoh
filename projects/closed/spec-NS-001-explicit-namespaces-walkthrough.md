# Walkthrough: spec-NS-001-explicit-namespaces

**Status**: Completed & Verified
**Date**: 2026-01-24

## 1. Objective

Standardized namespace resolution using a "Suffix Matching" strategy to handle the explicit namespace requirement and ambiguity detection introduced in `spec.md`. This ensures consistent resolution for both Verbs and Attributes.

## 2. File Changes

### `spec.md` (User Updated)
- Defined `namespace_ambiguity` diagnostic.
- Defined explicit namespace prefixing rule if ambiguous.

### `impl/02_parser.md`
- **Grammar**: Updated `namespaced_id` to `IDENTIFIER (DOT IDENTIFIER)*`.
- **Note**: Added explicit note that `IDENTIFIER` cannot contain dots, they are strictly separators.

### `impl/09_runtime.md`
- **Lookup**: Updated `Context.run` to use `runtime.verbDrivers.resolveBySuffix(suffix)` instead of direct map lookup.
- **Validator**: Added `NamespaceValidator` to the Validation Phase.
    - Logic: Iterates compiled verb calls and attributes.
    - Performs suffix matching against current registries.
    - Emits `fatal: namespace_ambiguity` if >1 match.
    - Emits `fatal: unknown_verb` if 0 matches (for verbs).
    - Emits `fatal: unknown_attribute` if 0 matches (for attributes, strict mode).

## 3. Verification

Verified that:
1. `impl/02_parser.md` grammar correctly separates components.
2. `impl/09_runtime.md` explicitly calls for Compile-Time Validation (Story Validator) as per the "Deep Think" analysis.
3. Validator logic covers both Verbs and Attributes.

## 4. Key Insights

- **Suffix Matching**: This allows succinct invalidation of ambiguous calls (e.g. `/log` matching `app.log` and `sys.log`) while keeping the syntax clean for unique names.
- **Compile-Time Validation**: By moving ambiguity checks to a Validator phase, we prevent runtime scenarios where a story works until it hits a specific ambiguous line (which might be rare). This improves developer confidence.
- **Attributes Namespacing**: Attributes are treated similarly to verbs, meaning they can be namespaced (e.g., `[std.By]`) and checked for ambiguity, keeping the language consistent.
