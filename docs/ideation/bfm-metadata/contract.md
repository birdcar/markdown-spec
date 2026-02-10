# BFM Metadata Contract

**Created**: 2026-02-10
**Confidence Score**: 95/100
**Status**: Draft

## Problem Statement

BFM currently specifies syntax parsing only — it transforms markdown into an AST and HTML output, but has no concept of document-level metadata. Real-world markdown-based systems (blogs, wikis, task managers, knowledge bases) need structured data extracted from documents: front-matter fields, tags, tasks grouped by state, word counts, and the ability to merge multiple documents.

Without a metadata layer, every BFM consumer reimplements these features differently, leading to inconsistent behavior across the ecosystem. The JS and PHP libraries have no way to express or extract this information, forcing downstream applications to walk the AST themselves.

## Goals

1. **Specify YAML front-matter** as a first-class BFM feature with formal parsing rules and AST representation
2. **Add inline hashtag syntax** (`#tag`) as a new BFM inline element, alongside the existing mention (`@user`) pattern
3. **Define a document metadata model** with built-in computed fields (wordCount, readingTime, tasks, tags, links/backlinks) and an extensible computed field resolver interface
4. **Specify front-matter merging** with recursive deep merge semantics (last-wins for scalars, concatenation for arrays, recursive for objects)
5. **Implement all features** in both `markdown-js` (remark/micromark) and `markdown-php` (league/commonmark)

## Success Criteria

- [ ] BFM spec updated with sections for: YAML front-matter, hashtags, document metadata model, computed fields, and front-matter merging
- [ ] Shared fixture suite includes front-matter, hashtag, merge, and metadata test cases
- [ ] `markdown-js` parses YAML front-matter into AST and exposes document metadata API
- [ ] `markdown-js` tokenizes `#hashtag` syntax with proper AST nodes
- [ ] `markdown-js` implements all 5 built-in computed fields (wordCount, readingTime, tasks, tags, links)
- [ ] `markdown-js` implements front-matter merging with deep merge semantics
- [ ] `markdown-js` exposes a `ComputedFieldResolver` interface for custom fields
- [ ] `markdown-php` implements the same feature set as `markdown-js`
- [ ] All existing tests continue to pass (no regressions)
- [ ] Both libraries pass the shared fixture suite for all new features

## Scope Boundaries

### In Scope

- YAML front-matter parsing (standard `---` delimited block)
- `#hashtag` inline syntax as a new BFM element
- Document metadata model with 5 built-in computed fields
- Computed field resolver interface (extensible)
- Front-matter merging algorithm (deep recursive, configurable strategy)
- Task extraction from AST (grouped by state)
- Tag extraction (unified from front-matter + inline hashtags, deduplicated)
- Link extraction from body content
- Backlink computation across document collections
- Spec updates to `bfm-spec.md`
- Shared test fixtures
- `markdown-js` implementation
- `markdown-php` implementation

### Out of Scope

- TOML or JSON front-matter — YAML only for this iteration
- Schema validation of front-matter fields — parsed as-is
- Database persistence or indexing — libraries produce metadata, consumers store it
- Full-text search — beyond metadata extraction
- Front-matter-based routing or path resolution
- Changes to Laravel or Filament packages — they consume the core library
- Tree-sitter grammar updates — separate project

### Future Considerations

- Front-matter schema validation (define expected types/required fields)
- TOML front-matter support
- Computed field caching/memoization
- Cross-document link graph visualization
- Front-matter inheritance (child documents inherit from parent)

---

*This contract was generated from brain dump input. Review and approve before proceeding to PRD generation.*
