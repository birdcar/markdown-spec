# PRD: BFM Metadata - Phase 3

**Contract**: ./contract.md
**Phase**: 3 of 3
**Focus**: PHP (markdown-php) implementation

## Phase Overview

Phase 3 implements all metadata features in the `markdown-php` library using the league/commonmark extension system. This mirrors the JS implementation from Phase 2, ensuring both libraries produce identical output for the shared fixture suite.

The PHP library serves Laravel applications, Filament admin panels, and PHP-based content systems. After this phase, PHP consumers have full parity with the JS implementation.

## User Stories

1. As a PHP developer, I want to parse YAML front-matter from BFM documents so that I can access structured metadata
2. As a PHP developer, I want `#hashtag` syntax recognized in my markdown so that I can tag content inline
3. As a PHP developer, I want extracted tasks grouped by state so that I can build task management features
4. As a PHP developer, I want a unified tag array so that I can categorize and filter documents
5. As a PHP developer, I want to merge multiple BFM documents so that I can compose content
6. As a PHP developer, I want to register custom computed fields so that I can derive project-specific metadata

## Functional Requirements

### Front-matter Parsing

- **FR-3.1**: Implement `FrontmatterExtension` for league/commonmark
- **FR-3.2**: Implement `FrontmatterBlockStartParser` to detect opening `---`
- **FR-3.3**: Implement `FrontmatterBlock` AST node class
- **FR-3.4**: Implement `FrontmatterRenderer` (renders nothing — front-matter is metadata, not content)
- **FR-3.5**: Parse YAML content using Symfony YAML component (already a common PHP dependency)
- **FR-3.6**: Register in `BfmEnvironmentFactory::create()`

### Hashtag Syntax

- **FR-3.7**: Implement `HashtagExtension` for league/commonmark
- **FR-3.8**: Implement `HashtagParser` (inline parser)
- **FR-3.9**: Implement `Hashtag` AST node class
- **FR-3.10**: Implement `HashtagRenderer` (renders as `<span class="hashtag">#tag</span>` or configurable)

### Document Metadata

- **FR-3.11**: Implement `DocumentMetadata` class with same shape as JS implementation
- **FR-3.12**: Implement `MetadataExtractor::extract(Document $doc): DocumentMetadata`
- **FR-3.13**: Implement all 5 built-in computed fields matching JS behavior
- **FR-3.14**: Task extraction walks AST for `TaskMarker` nodes, groups by `TaskState`
- **FR-3.15**: Tag extraction unions front-matter `tags` + `Hashtag` nodes, deduplicates

### Computed Field Resolver

- **FR-3.16**: Define `ComputedFieldResolverInterface` matching the existing resolver pattern
- **FR-3.17**: `BfmEnvironmentFactory` accepts computed field resolvers as configuration
- **FR-3.18**: Custom fields merged into metadata after builtins

### Front-matter Merging

- **FR-3.19**: Implement `DocumentMerger::merge(array $documents, ?MergeOptions $options): MergedDocument`
- **FR-3.20**: Default strategy: last-wins for scalars, concatenate for arrays, recursive for objects
- **FR-3.21**: `MergeOptions` supports strategy configuration
- **FR-3.22**: Body concatenation with `\n\n` separator
- **FR-3.23**: Metadata recomputed on merged result

### Backlinks

- **FR-3.24**: Implement `BacklinkComputer::compute(array $documents): array` for collection-level backlinks

## Non-Functional Requirements

- **NFR-3.1**: PHP 8.2+ compatibility maintained
- **NFR-3.2**: No new dependencies beyond `symfony/yaml` (likely already present via league/commonmark)
- **NFR-3.3**: Follow existing extension pattern (Extension, Parser, Node, Renderer classes)
- **NFR-3.4**: Strict types declared in all new files

## Dependencies

### Prerequisites

- Phase 1 complete (spec + fixtures)
- Phase 2 complete (JS reference implementation to mirror)
- Existing `markdown-php` codebase with league/commonmark extensions

### Outputs for Next Phase

- Feature-complete PHP implementation
- Laravel/Filament packages can consume new features in future iterations

## Acceptance Criteria

- [ ] All Phase 1 fixtures pass in `markdown-php`
- [ ] Front-matter: `src/Block/Frontmatter/` with Extension, Parser, Block, Renderer classes
- [ ] Hashtags: `src/Inline/Hashtag/` with Extension, Parser, Node, Renderer classes
- [ ] Metadata: `src/Metadata/` with MetadataExtractor, DocumentMetadata, computed field implementations
- [ ] Merging: `src/Merge/` with DocumentMerger, MergeOptions
- [ ] Contracts: `src/Contracts/ComputedFieldResolverInterface.php` added
- [ ] `BfmEnvironmentFactory` registers new extensions
- [ ] All existing tests pass (no regressions)
- [ ] New unit tests for each feature
- [ ] New fixture-based tests for all metadata fixtures
- [ ] `composer run test` passes
- [ ] PHPStan/Psalm analysis passes (if configured)

---

*Review this PRD and provide feedback before spec generation.*
