# Implementation Spec: BFM Metadata - Phase 1

**PRD**: ./prd-phase-1.md
**Estimated Effort**: L

## Technical Approach

Phase 1 is spec-only — no library code. We update `bfm-spec.md` with new sections covering YAML front-matter, inline hashtags, the document metadata model, computed fields, and front-matter merging. Then we create shared test fixtures.

The spec follows the existing structure: each feature gets a numbered section with syntax rules, AST examples, and conformance requirements. Fixtures follow the existing pattern (`name.md` + `name.ast.json` + `name.html`) with new `.metadata.json` files for metadata output and `.merged.*` files for merge scenarios.

The new features integrate with the existing conformance levels: front-matter and hashtags join BFM Core. Metadata extraction, computed fields, and merging are specified as BFM Extended — implementations SHOULD support them but Core conformance doesn't require it.

## File Changes

### Modified Files

| File Path | Changes |
|-----------|---------|
| `bfm-spec.md` | Add sections 5-9 (front-matter, hashtags, metadata, computed fields, merging). Update conformance section. Update grammar appendix. |

### New Files

| File Path | Purpose |
|-----------|---------|
| `fixtures/blocks/frontmatter-basic.md` | Simple front-matter with scalar and array values |
| `fixtures/blocks/frontmatter-basic.ast.json` | Expected AST for basic front-matter |
| `fixtures/blocks/frontmatter-empty.md` | Empty front-matter block (`---\n---`) |
| `fixtures/blocks/frontmatter-empty.ast.json` | Expected AST for empty front-matter |
| `fixtures/blocks/frontmatter-complex.md` | Nested objects, arrays, various YAML types |
| `fixtures/blocks/frontmatter-complex.ast.json` | Expected AST for complex front-matter |
| `fixtures/inlines/hashtags-basic.md` | Inline hashtags in various positions |
| `fixtures/inlines/hashtags-basic.ast.json` | Expected AST for hashtag parsing |
| `fixtures/inlines/hashtags-basic.html` | Expected HTML for hashtags |
| `fixtures/inlines/hashtags-edge-cases.md` | Hashtags near headings, in code, next to mentions |
| `fixtures/inlines/hashtags-edge-cases.ast.json` | Expected AST for edge cases |
| `fixtures/metadata/tasks-extraction.md` | Document with various task states and modifiers |
| `fixtures/metadata/tasks-extraction.metadata.json` | Expected metadata output with tasks grouped by state |
| `fixtures/metadata/tags-extraction.md` | Document with front-matter tags + inline hashtags |
| `fixtures/metadata/tags-extraction.metadata.json` | Expected metadata with unified deduplicated tags |
| `fixtures/metadata/computed-fields.md` | Document for testing wordCount, readingTime, links |
| `fixtures/metadata/computed-fields.metadata.json` | Expected metadata with all built-in computed fields |
| `fixtures/merge/a.md` | First document in merge pair |
| `fixtures/merge/b.md` | Second document in merge pair |
| `fixtures/merge/ab-merged.md` | Expected merged output |
| `fixtures/merge/ab-merged.ast.json` | Expected AST of merged document |
| `fixtures/merge/deep-a.md` | First document with nested objects |
| `fixtures/merge/deep-b.md` | Second document with nested objects |
| `fixtures/merge/deep-ab-merged.md` | Expected deep merge output |

## Implementation Details

### Section 5: YAML Front-matter

**Overview**: Define YAML front-matter as an optional preamble block.

**Spec content to write**:

```markdown
## 5. YAML Front-matter

BFM supports YAML front-matter as an optional metadata block at the beginning
of a document.

### 5.1 Syntax

A front-matter block begins with `---` on the first line of the document and
ends with a closing `---` on its own line:

\`\`\`
---
<yaml content>
---
\`\`\`

- The opening `---` MUST be the very first line of the document. No preceding
  blank lines, whitespace, or content is permitted.
- The closing `---` MUST appear on its own line.
- Content between delimiters is parsed as YAML 1.2.
- If the opening `---` is not the first line, it is treated as a thematic
  break (per CommonMark), not front-matter.
- An empty front-matter block (`---\n---`) is valid and produces an empty
  metadata object.
- Invalid YAML content SHOULD produce a parse error.

### 5.2 AST

Front-matter produces a `yaml` node as the first child of the root:

\`\`\`json
{
  "type": "yaml",
  "value": "key1: value1\ntags:\n  - tag1",
  "data": {
    "key1": "value1",
    "tags": ["tag1"]
  }
}
\`\`\`

The `value` field contains the raw YAML string. The `data` field contains
the parsed key-value structure.

### 5.3 Interaction with CommonMark

The opening `---` of front-matter would normally be parsed as a thematic
break in CommonMark. BFM resolves this by only treating `---` as front-matter
when it appears on the very first line of the document. All other occurrences
of `---` on their own line are thematic breaks as usual.
```

### Section 6: Inline Hashtags

**Overview**: Define `#identifier` inline syntax, similar to mentions.

**Spec content to write**:

```markdown
## 6. Inline Hashtags

Hashtags are inline references to topics or categories, using the `#` prefix.

### 6.1 Syntax

\`\`\`
#<identifier>
\`\`\`

- The `#` MUST be preceded by whitespace, punctuation, or appear at the start
  of inline content (not mid-word).
- `identifier` matches `[a-zA-Z][a-zA-Z0-9_-]*` — alphanumeric with
  underscores and hyphens (no dots, unlike mentions).
- The identifier terminates at whitespace, punctuation (except `_`, `-`
  within the identifier), or end of input.

### 6.2 Disambiguation

- `#` at the start of a line followed by a space is an ATX heading, not a
  hashtag. Hashtags are inline-only.
- `#` inside code spans or code blocks is literal text.
- `#identifier` at the start of a line (no space after identifier) is a
  hashtag if the line is within a paragraph context.

### 6.3 AST

\`\`\`json
{ "type": "hashtag", "identifier": "typescript" }
\`\`\`

### 6.4 HTML Output

Default rendering:
\`\`\`html
<span class="hashtag">#typescript</span>
\`\`\`

Implementations MAY provide configurable rendering (e.g., linking to a
tag page).
```

### Section 7: Document Metadata Model

**Spec content to write**:

```markdown
## 7. Document Metadata

BFM defines a document metadata model that implementations extract from
parsed documents. This enables structured data access without consumers
needing to walk the AST directly.

### 7.1 Metadata Structure

\`\`\`
DocumentMetadata {
  frontmatter: Record<string, unknown>   // parsed YAML front-matter
  computed: {
    wordCount: number                     // body word count
    readingTime: number                   // minutes, ceil(wordCount / wpm)
    tasks: TaskCollection                 // extracted tasks by state
    tags: string[]                        // unified deduplicated tags
    links: LinkReference[]                // extracted links
  }
  custom: Record<string, unknown>         // user-defined computed fields
}
\`\`\`

### 7.2 Built-in Computed Fields

#### wordCount
Count of words in body content (all text nodes). Code blocks are included.
Front-matter content is excluded.

#### readingTime
`ceil(wordCount / wordsPerMinute)` where default WPM is 200. Implementations
SHOULD allow configuring the WPM value.

#### tasks
\`\`\`
TaskCollection {
  all: ExtractedTask[]
  open: ExtractedTask[]
  done: ExtractedTask[]
  scheduled: ExtractedTask[]
  migrated: ExtractedTask[]
  irrelevant: ExtractedTask[]
  event: ExtractedTask[]
  priority: ExtractedTask[]
}

ExtractedTask {
  text: string               // raw text content of the task (excluding marker)
  state: TaskState           // one of the 7 states
  modifiers: TaskModifier[]  // [{key, value}] from //key:value annotations
  line: number               // source line number
}
\`\`\`

#### tags
Deduplicated array of tag strings from:
1. The `tags` array field in YAML front-matter (if present)
2. All inline `#hashtag` nodes in the body

Tags are normalized to lowercase. Duplicates are removed preserving first
occurrence order.

#### links
\`\`\`
LinkReference {
  url: string
  title: string | null
  line: number
}
\`\`\`

Extracted from all link and image nodes in the body content.

### 7.3 Backlinks (Collection-level)

Backlinks are computed across a collection of documents. For each document,
backlinks is the set of other documents that contain a link to it.

Backlink computation is NOT a per-document computed field — it requires
knowledge of the full collection. Implementations MUST provide a separate
API for computing backlinks.
```

### Section 8: Computed Field Resolvers

**Spec content to write**:

```markdown
## 8. Computed Field Resolvers

Implementations MUST support user-defined computed fields via a resolver
interface.

### 8.1 Resolver Contract

A computed field resolver receives:
1. The document AST (root node)
2. The parsed front-matter data
3. The built-in computed field values

And returns a key-value map of custom computed fields.

### 8.2 Execution Order

1. Parse front-matter
2. Parse body into AST
3. Compute built-in fields (wordCount, readingTime, tasks, tags, links)
4. Execute custom resolvers in registration order
5. Later resolvers can access results from earlier resolvers

### 8.3 Interface

The specific interface is implementation-defined but MUST follow the
pattern established by `MentionResolver` and `EmbedResolver`:
- TypeScript: type/interface with a function signature
- PHP: interface with a method signature
```

### Section 9: Front-matter Merging

**Spec content to write**:

```markdown
## 9. Front-matter Merging

BFM defines a merge algorithm for combining multiple documents into one.

### 9.1 Merge Order

Documents are merged left-to-right. The first document is the base; each
subsequent document's data is merged into the accumulator.

### 9.2 Merge Rules

For each key encountered across all documents:

| Value Type | Rule |
|------------|------|
| Scalar (string, number, boolean, null) | Last-wins: later value replaces earlier |
| Array | Concatenation: later items appended to earlier |
| Object | Recursive: apply same rules at each nested level |
| Type mismatch | Later value replaces entirely (e.g., scalar replaces array) |

### 9.3 Body Merging

Body content from all documents is concatenated in order, separated by
two newlines (`\n\n`).

### 9.4 Post-merge Computation

After merging, computed fields (including custom resolvers) are recomputed
on the merged result. This ensures `wordCount`, `tasks`, `tags`, etc.
reflect the combined document.

### 9.5 Configurable Strategy

Implementations SHOULD support alternative merge strategies:
- `last-wins` (default): as described above
- `first-wins`: first value takes precedence for scalars
- `error`: throw/raise on scalar conflicts (arrays still concatenate)
- Custom resolver: user-provided function called for each conflict

### 9.6 Example

Given documents:
\`\`\`yaml
# a.md
---
key1: value1
tags:
  - arrValue1
  - arrValue2
---
Content from a.
\`\`\`

\`\`\`yaml
# b.md
---
keyA: valueB
tags:
  - arrValueA
  - arrValueB
---
Content from b.
\`\`\`

Merged result:
\`\`\`yaml
---
key1: value1
keyA: valueB
tags:
  - arrValue1
  - arrValue2
  - arrValueA
  - arrValueB
---
Content from a.

Content from b.
\`\`\`
```

### Fixtures

**Implementation steps**:

1. Create `fixtures/metadata/` directory
2. Create `fixtures/merge/` directory
3. Write front-matter fixtures in `fixtures/blocks/`
4. Write hashtag fixtures in `fixtures/inlines/`
5. Write metadata extraction fixtures in `fixtures/metadata/`
6. Write merge fixtures in `fixtures/merge/`

**Key fixture details**:

`frontmatter-basic.md`:
```markdown
---
title: Hello World
author: nick
tags:
  - bfm
  - markdown
---

This is body content.
```

`hashtags-basic.md`:
```markdown
This is a paragraph with #typescript and #react hashtags.

A #multi-word tag and #with_underscores work too.
```

`tags-extraction.md` (combines front-matter + inline):
```markdown
---
tags:
  - bfm
  - markdown
---

This document discusses #typescript and #bfm features.
```
Expected metadata tags: `["bfm", "markdown", "typescript"]` (deduplicated, `#bfm` matches front-matter `bfm`)

`merge/a.md` and `merge/b.md`: Use the exact example from the contract.

### Grammar Appendix Updates

Add to the grammar summary:

```
frontmatter     := "---" NL yaml_content "---" NL
yaml_content    := .+?  (valid YAML 1.2)

hashtag         := "#" hashtag_ident
hashtag_ident   := [a-zA-Z][a-zA-Z0-9_-]*
```

### Conformance Updates

Update section 5 (renumbered to 10):
- **BFM Core**: Add front-matter parsing and hashtag syntax
- **BFM Extended**: Document metadata model, computed fields, merging

## Testing Requirements

### Fixture Validation

All fixtures are validated by the implementations in Phase 2 and 3. Phase 1 deliverable is the fixture files themselves.

**Key test cases per fixture**:

- Front-matter basic: scalar, array, nested object
- Front-matter empty: `---\n---` produces empty data
- Front-matter complex: multi-level nesting, various YAML types (bool, int, float, null, date)
- Hashtags basic: inline position, preceded by space, preceded by punctuation
- Hashtags edge: start of line in paragraph, inside code span (not parsed), adjacent to mention
- Metadata tasks: all 7 states extracted, modifiers captured, line numbers correct
- Metadata tags: front-matter + inline merged, deduplication works, lowercase normalization
- Metadata computed: wordCount matches, readingTime computes correctly
- Merge basic: scalars, arrays, body concatenation
- Merge deep: nested object recursive merge

## Validation Commands

```bash
# No build step for spec — validation is manual review + implementation testing
# Check that all fixture .md files have corresponding .ast.json files
ls fixtures/blocks/frontmatter-*
ls fixtures/inlines/hashtags-*
ls fixtures/metadata/*
ls fixtures/merge/*
```

## Open Items

- [ ] Decide if `#123` (numeric-only hashtags) should be valid or require leading letter (current spec: leading letter required)
- [ ] Confirm hashtag `identifier` excludes dots (unlike mention identifiers which allow dots)
