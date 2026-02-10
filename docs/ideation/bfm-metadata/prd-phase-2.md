# PRD: BFM Metadata - Phase 2

**Contract**: ./contract.md
**Phase**: 2 of 3
**Focus**: JavaScript (markdown-js) implementation

## Phase Overview

Phase 2 implements all metadata features in the `markdown-js` library using the micromark/remark/unified ecosystem. This phase follows the spec established in Phase 1 and must pass all shared fixtures.

The JS library is the primary implementation — it's used in Node.js applications, static site generators, and build pipelines. After this phase, JS consumers can parse front-matter, use inline hashtags, extract document metadata, register custom computed fields, and merge documents.

## User Stories

1. As a JS developer, I want to parse YAML front-matter from BFM documents so that I can access structured metadata
2. As a JS developer, I want `#hashtag` syntax recognized in my markdown so that I can tag content inline
3. As a JS developer, I want extracted tasks grouped by state so that I can build task dashboards from markdown
4. As a JS developer, I want a unified tag array so that I can categorize and filter documents
5. As a JS developer, I want to merge multiple BFM documents so that I can compose content from fragments
6. As a JS developer, I want to register custom computed fields so that I can derive project-specific metadata

## Functional Requirements

### Front-matter Parsing

- **FR-2.1**: Implement micromark syntax extension for YAML front-matter tokenization
- **FR-2.2**: Implement mdast from-markdown extension to produce a `yaml` node with parsed data
- **FR-2.3**: Implement mdast to-markdown extension to serialize front-matter back to `---` blocks
- **FR-2.4**: Integrate with existing `remarkBfm` plugin (front-matter registered automatically)

### Hashtag Syntax

- **FR-2.5**: Implement micromark syntax extension for `#hashtag` tokenization
- **FR-2.6**: Implement mdast from-markdown extension to produce `hashtag` AST nodes
- **FR-2.7**: Implement mdast to-markdown extension to serialize hashtags
- **FR-2.8**: Extend `types.ts` with `HashtagNode` type and token type map entries

### Document Metadata

- **FR-2.9**: Implement `extractMetadata(tree: Root, frontmatter: Record<string, unknown>): DocumentMetadata` function
- **FR-2.10**: Implement `wordCount` computed field (walk text nodes, split on whitespace)
- **FR-2.11**: Implement `readingTime` computed field (configurable WPM, default 200)
- **FR-2.12**: Implement `tasks` computed field (walk list items for task markers, group by state)
- **FR-2.13**: Implement `tags` computed field (union front-matter tags + hashtag nodes, deduplicate)
- **FR-2.14**: Implement `links` computed field (walk link/image nodes, extract URLs)

### Computed Field Resolver

- **FR-2.15**: Define `ComputedFieldResolver` type: `(tree: Root, frontmatter: Record<string, unknown>, builtins: BuiltinMetadata) => Record<string, unknown>`
- **FR-2.16**: `remarkBfm` plugin accepts `computedFields` option (array of resolvers)
- **FR-2.17**: Custom fields merged into metadata output after builtins

### Front-matter Merging

- **FR-2.18**: Implement `mergeDocuments(docs: BfmDocument[], options?: MergeOptions): BfmDocument`
- **FR-2.19**: Default strategy: last-wins for scalars, concatenate for arrays, recursive for objects
- **FR-2.20**: `MergeOptions` supports strategy override: `'last-wins' | 'first-wins' | 'error' | MergeResolver`
- **FR-2.21**: Body content concatenated with `\n\n` separator
- **FR-2.22**: Metadata recomputed on merged result

### Backlinks

- **FR-2.23**: Implement `computeBacklinks(docs: BfmDocument[]): Map<string, string[]>` for collection-level backlink computation

## Non-Functional Requirements

- **NFR-2.1**: No new runtime dependencies beyond what micromark/mdast already provide (except a YAML parser — `yaml` package)
- **NFR-2.2**: Front-matter parsing adds <5ms overhead for typical documents
- **NFR-2.3**: All exported types properly declared for TypeScript consumers
- **NFR-2.4**: Tree-shakeable exports (front-matter, hashtags, metadata each independently importable)

## Dependencies

### Prerequisites

- Phase 1 complete (spec + fixtures)
- Existing `markdown-js` codebase with micromark extensions

### Outputs for Next Phase

- Reference implementation for PHP to match
- API surface that PHP should mirror

## Acceptance Criteria

- [ ] All Phase 1 fixtures pass in `markdown-js`
- [ ] Front-matter: `src/blocks/frontmatter/` with syntax, from-markdown, to-markdown modules
- [ ] Hashtags: `src/inlines/hashtags/` with syntax, from-markdown, to-markdown modules
- [ ] Metadata: `src/metadata/` with extraction and computed field logic
- [ ] Merging: `src/merge/` with deep merge implementation
- [ ] Contracts: `src/contracts/computed-field-resolver.ts` added
- [ ] `types.ts` extended with new node types and token types
- [ ] `plugin.ts` registers front-matter and hashtag extensions
- [ ] Package exports updated for new subpaths (`./frontmatter`, `./hashtags`, `./metadata`, `./merge`)
- [ ] All existing tests pass (no regressions)
- [ ] New unit tests for each feature module
- [ ] `bun run typecheck` passes
- [ ] `bun run test` passes
- [ ] `bun run build` succeeds

---

*Review this PRD and provide feedback before spec generation.*
