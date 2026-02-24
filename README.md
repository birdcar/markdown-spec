# Birdcar Flavored Markdown (BFM) Specification

Canonical specification and shared test fixture suite for BFM — a custom markdown dialect that extends CommonMark + GFM.

## What is BFM?

BFM is a superset of CommonMark and GFM (minus GFM task lists) that adds:

- **YAML front-matter** — `---` delimited metadata blocks parsed into structured data
- **Directive blocks** — 11 built-in directives:
  - *Container:* `@callout`, `@details`, `@tabs`/`@tab`, `@figure`, `@aside`
  - *Leaf:* `@embed`, `@include`, `@query`, `@toc`, `@math`, `@endnotes`
- **Footnotes** — Pandoc-style `[^label]` references and `[^label]: content` definitions
- **Extended task lists** — `[x]`, `[>]`, `[<]`, `[-]`, `[o]`, `[!]` state markers
- **Task modifiers** — `//due:2025-03-01`, `//every:quarter`, `//hard`
- **Mentions** — `@username` inline references
- **Hashtags** — `#project` inline tags
- **Metadata extraction** — computed fields (word count, reading time, tasks, tags, links, footnotes)
- **Document merging** — deep recursive merge of front-matter across files

See `bfm-spec.md` for the full specification.

## Implementations

| Repo | Language | Package |
|------|----------|---------|
| [markdown-js](https://github.com/birdcar/markdown-js) | TypeScript | `@birdcar/markdown` |
| [markdown-php](https://github.com/birdcar/markdown-php) | PHP | `birdcar/markdown-php` |

## Fixture Suite

Both implementations test against the shared fixture suite in `fixtures/`. Each fixture is a set of related files:

- `{name}.md` — Input BFM markdown
- `{name}.ast.json` — Expected normalized AST as JSON
- `{name}.html` — Expected default HTML output
- `{name}.metadata.json` — Expected metadata extraction result (metadata fixtures only)

### AST Format

- Root node type is `root` (mdast-compatible)
- Position information is omitted
- Properties are sorted alphabetically
- Text nodes: `{ "type": "text", "value": "..." }`

### Fixture Naming

- `{feature}-basic` — Happy path, simplest usage
- `{feature}-{variant}` — Specific scenarios
- `kitchen-sink` — Full document combining all features

### Directory Structure

```
fixtures/
  inlines/          # Inline-level features
    tasks-basic.*
    tasks-modifiers.*
    mentions-basic.*
    mentions-platform.*
    hashtags-basic.*
    footnotes-basic.*
    footnotes-edge-cases.*
    hashtags-edge-cases.*
  blocks/           # Block-level features
    aside-basic.*
    callout-basic.*
    details-basic.*
    endnotes-basic.*
    figure-basic.*
    frontmatter-basic.*
    frontmatter-empty.*
    frontmatter-complex.*
    include-basic.*
    kitchen-sink.*
    math-basic.*
    query-basic.*
    tabs-basic.*
    toc-basic.*
  metadata/         # Metadata extraction
    tasks-extraction.*
    tags-extraction.*
    computed-fields.*
  merge/            # Document merging
    a.md, b.md, ab-merged.*
    deep-a.md, deep-b.md, deep-ab-merged.md
```

## Adding Fixtures

When adding new features or edge cases:

1. Create the `.md` input file
2. Author the `.ast.json` with expected AST (use `root` as root type)
3. Author the `.html` with expected default HTML output
4. Ensure AST properties are sorted alphabetically
5. Verify JSON is valid: `jq . your-fixture.ast.json`
