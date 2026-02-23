# BFM Directives Expansion Contract

**Created**: 2026-02-23
**Confidence Score**: 96/100
**Status**: Draft

## Problem Statement

BFM's directive system (`@name`/`@endname`) is well-designed but ships with only two built-in directives: `@callout` and `@embed`. This leaves common document structures — collapsible sections, tabbed content, figures with captions, footnotes, tables of contents, math blocks, sidebars, file transclusion, and metadata queries — without first-class support. Authors either can't express these patterns or must rely on raw HTML, defeating the purpose of a structured markdown superset.

## Goals

1. **Add 9 new built-in directives** to the BFM spec, each with full syntax, parameters, AST shape, and HTML output definitions
2. **Add footnote inline syntax** (`[^label]` / `[^label]: content`) as a new inline extension following the Pandoc/PHP Markdown Extra convention
3. **Maintain spec consistency** — new directives follow the same patterns as `@callout` and `@embed` (container vs leaf classification, parameter syntax, AST structure)
4. **Provide fixture coverage** for every new directive and the footnote syntax

## Success Criteria

- [ ] Each of the 9 directives has a complete spec section: syntax, type (container/leaf), parameters table, example, AST shape, and HTML output
- [ ] Footnote inline syntax has a complete spec section with disambiguation rules
- [ ] The grammar summary (Appendix A) is updated with all new productions
- [ ] Conformance levels (§10) are updated to reference new directives
- [ ] Fixture `.md` files exist for each new directive and for footnotes
- [ ] `@query` has core required filter keys (`state`, `tag`) with an extensibility pattern for implementations
- [ ] All new directives integrate cleanly with existing BFM features (e.g., mentions, hashtags, and task modifiers work inside container directives)

## Scope Boundaries

### In Scope

- Spec sections for: `@details`, `@include`, `@query`, `@tabs`/`@tab`, `@figure`, `@toc`, `@math`, `@aside`
- Spec section for footnote inline syntax (`[^label]` markers + `[^label]:` definitions)
- An `@endnotes` leaf directive for controlling footnote placement
- Fixture `.md` files for each new directive
- Updates to grammar summary, conformance levels, and extension list at top of spec
- Document metadata updates if any new directives produce extractable metadata (e.g., `@toc`, footnotes)

### Out of Scope

- Parser implementation — this is spec-only work
- AST `.json` fixture files — only `.md` input fixtures for now
- HTML `.html` fixture files — only `.md` input fixtures
- Changes to existing directives (`@callout`, `@embed`)
- Changes to existing non-directive features (tasks, mentions, hashtags, front-matter, metadata, merging)

### Future Considerations

- `@diagram` directive for Mermaid/D2/PlantUML rendering
- `@code` directive as a richer alternative to fenced code blocks (with line highlighting, file name, diff markers)
- Collection-level `@query` that spans multiple documents (currently scoped to single document)

---

_This contract was generated from brain dump input. Review and approve before proceeding to specification._
