# PRD: BFM Metadata - Phase 1

**Contract**: ./contract.md
**Phase**: 1 of 3
**Focus**: Specification and shared test fixtures

## Phase Overview

Phase 1 establishes the formal specification for all metadata features and creates the shared fixture suite that both implementations will test against. This must come first because the spec is the source of truth — without it, the JS and PHP implementations would diverge.

After this phase, the BFM spec will define YAML front-matter, inline hashtags, document metadata model, computed fields, and front-matter merging. The fixture suite will provide concrete input/output pairs for conformance testing.

## User Stories

1. As a BFM implementer, I want a formal front-matter specification so that I know exactly how to parse `---` delimited YAML blocks
2. As a BFM implementer, I want a hashtag syntax specification so that I know how `#tag` interacts with existing inline elements
3. As a BFM implementer, I want a document metadata model specification so that I know what computed fields to produce and their exact shapes
4. As a BFM implementer, I want merging rules specified so that I know how to combine front-matter from multiple files
5. As a BFM implementer, I want shared fixtures so that I can verify my implementation matches the spec

## Functional Requirements

### YAML Front-matter

- **FR-1.1**: Specify YAML front-matter as an optional block delimited by `---` on its own line, appearing before any other content
- **FR-1.2**: The opening `---` MUST be the first line of the document (no preceding whitespace or blank lines)
- **FR-1.3**: The closing `---` MUST appear on its own line
- **FR-1.4**: Content between delimiters is parsed as YAML 1.2
- **FR-1.5**: Front-matter produces a `yaml` (or `frontmatter`) node in the AST with parsed key-value data
- **FR-1.6**: Documents without front-matter are valid BFM (front-matter is optional)
- **FR-1.7**: Invalid YAML between delimiters SHOULD produce a parse error, not silent failure

### Inline Hashtags

- **FR-1.8**: Specify `#identifier` as an inline element where `identifier` matches `[a-zA-Z][a-zA-Z0-9_-]*`
- **FR-1.9**: The `#` MUST be preceded by whitespace, punctuation, or appear at the start of inline content
- **FR-1.10**: Hashtags MUST NOT conflict with ATX headings (headings are block-level, hashtags are inline)
- **FR-1.11**: Hashtags produce a `hashtag` AST node with an `identifier` field
- **FR-1.12**: Hashtags inside code spans/blocks are not parsed (consistent with other inline elements)

### Document Metadata Model

- **FR-1.13**: Define a `DocumentMetadata` structure that implementations produce from a parsed BFM document
- **FR-1.14**: Specify 5 built-in computed fields: `wordCount`, `readingTime`, `tasks`, `tags`, `links`
- **FR-1.15**: `wordCount` counts words in body content (excluding front-matter, code blocks TBD)
- **FR-1.16**: `readingTime` is `ceil(wordCount / wordsPerMinute)` where default WPM is 200
- **FR-1.17**: `tasks` is an object with `all` array plus arrays per state (`open`, `done`, `scheduled`, `migrated`, `irrelevant`, `event`, `priority`)
- **FR-1.18**: Each extracted task includes: `text` (raw text content), `state` (TaskState), `modifiers` (array of key-value pairs), `line` (source line number)
- **FR-1.19**: `tags` is a deduplicated array of tag strings from both front-matter `tags` field and inline `#hashtag` nodes
- **FR-1.20**: `links` is an array of `{url, title?, line}` extracted from markdown links in body content

### Computed Field Resolver

- **FR-1.21**: Define a `ComputedFieldResolver` interface: given a document AST and front-matter, return a key-value pair
- **FR-1.22**: Implementations MUST support registering custom computed field resolvers
- **FR-1.23**: Custom computed fields are computed after built-in fields and can reference them

### Front-matter Merging

- **FR-1.24**: Define a merge algorithm for combining front-matter from N documents in order
- **FR-1.25**: Scalar values: last-wins (later document overrides earlier)
- **FR-1.26**: Array values: concatenation (later document's items appended)
- **FR-1.27**: Object values: recursive deep merge (apply same rules at each level)
- **FR-1.28**: Body content: concatenation with double newline (`\n\n`) separation
- **FR-1.29**: Computed fields are recomputed on the merged result
- **FR-1.30**: Implementations SHOULD support configurable merge strategies (last-wins, first-wins, error-on-conflict, custom resolver)

### Backlinks

- **FR-1.31**: Define backlinks as a collection-level computed field — given a set of documents, for each document compute which other documents link to it
- **FR-1.32**: Backlink computation operates on extracted `links` from all documents in the collection

## Non-Functional Requirements

- **NFR-1.1**: Spec language must be precise enough to write deterministic tests
- **NFR-1.2**: Fixtures must cover happy paths, edge cases, and error cases
- **NFR-1.3**: New spec sections must follow existing writing style and structure conventions

## Dependencies

### Prerequisites

- Current BFM spec (v0.1.0-draft) as baseline
- Existing fixture suite structure

### Outputs for Next Phase

- Updated `bfm-spec.md` with sections 5-9 (front-matter, hashtags, metadata, computed fields, merging)
- New fixtures in `fixtures/` directory
- Updated conformance section referencing new features

## Acceptance Criteria

- [ ] `bfm-spec.md` contains formal sections for: YAML front-matter, inline hashtags, document metadata model, computed fields, front-matter merging
- [ ] Grammar summary appendix updated with new productions
- [ ] Conformance section updated to include metadata features
- [ ] Fixture: `frontmatter-basic.md/.ast.json` — simple front-matter parsing
- [ ] Fixture: `frontmatter-empty.md/.ast.json` — empty front-matter block
- [ ] Fixture: `frontmatter-complex.md/.ast.json` — nested objects, arrays, various YAML types
- [ ] Fixture: `hashtags-basic.md/.ast.json/.html` — inline hashtag parsing
- [ ] Fixture: `hashtags-edge-cases.md/.ast.json` — hashtags near headings, in code, adjacent to mentions
- [ ] Fixture: `metadata-tasks.md/.metadata.json` — task extraction output
- [ ] Fixture: `metadata-tags.md/.metadata.json` — tag extraction from front-matter + inline
- [ ] Fixture: `merge-basic.md` pair + `.merged.md/.merged.ast.json` — basic merge scenario
- [ ] Fixture: `merge-deep.md` pair + output — nested object merge

---

*Review this PRD and provide feedback before spec generation.*
