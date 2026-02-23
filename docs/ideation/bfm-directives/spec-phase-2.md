# Implementation Spec: BFM Directives Expansion - Phase 2 (Leaf Directives)

**Contract**: ./contract.md
**Estimated Effort**: M

## Technical Approach

Phase 2 adds five leaf directives to the BFM spec: `@include`, `@query`, `@toc`, `@math`, and `@endnotes`. All are leaf type — their body content is either literal text (not parsed as markdown) or empty (like `@embed`). Each gets a new subsection under §1, following the structure of §1.5 (`@embed`).

`@query` gets special treatment: a core set of required filter keys (`state`, `tag`) with an extensibility pattern for implementations.

Pattern to follow: §1.5 (`@embed`) in `bfm-spec.md` lines 114-143 for leaf directive spec sections.

## Feedback Strategy

**Inner-loop command**: N/A (spec-writing, not code)

**Playground**: Manual review — structural consistency with existing spec.

**Why this approach**: Documentation/spec task. Validation is review-based.

## File Changes

### Modified Files

| File Path | Changes |
|-----------|---------|
| `bfm-spec.md` | Add §1.10 (`@include`), §1.11 (`@query`), §1.12 (`@toc`), §1.13 (`@math`), §1.14 (`@endnotes`) |

### New Files

| File Path | Purpose |
|-----------|---------|
| `fixtures/blocks/include-basic.md` | Basic `@include` file transclusion |
| `fixtures/blocks/query-basic.md` | Basic `@query` with core filters |
| `fixtures/blocks/toc-basic.md` | Basic `@toc` placement |
| `fixtures/blocks/math-basic.md` | Basic `@math` with LaTeX content |
| `fixtures/blocks/endnotes-basic.md` | Basic `@endnotes` placement |

## Implementation Details

### §1.10 Built-in Directive: `@include`

**Pattern to follow**: §1.5 `@embed` in `bfm-spec.md`

**Overview**: File transclusion — includes the content of another file inline. Leaf type — the body is empty (or ignored). The included file's content is parsed as BFM at the point of inclusion.

**Spec section content**:

```markdown
### 1.10 Built-in Directive: `@include`

**Type:** Leaf

**Parameters:**

| Param | Required | Default | Values |
|-------|----------|---------|--------|
| (positional) path | Yes | — | Relative file path |
| `heading` | No | `""` | Heading text to extract a section |

The positional path is resolved relative to the including document's location.
Absolute paths are NOT supported — all includes are relative.

When `heading` is provided, only the section under that heading (and its
children, up to the next heading of equal or higher level) is included.

The included content is parsed as BFM markdown at the point of inclusion.
This means the included content participates in the including document's
AST, metadata extraction, and rendering.

**Circular includes** (A includes B which includes A) MUST be detected and
produce a parse error. Implementations SHOULD track the include stack and
reject cycles.

**Missing files** SHOULD produce a parse error. Implementations MAY support
a fallback rendering (e.g., an error callout in the output).

**Example:**

```markdown
@include ./shared/header.md
@endinclude
```

```markdown
@include ./api-docs.md heading="Authentication"
@endinclude
```

**AST:**

The `@include` directive is resolved during parsing. The AST contains
the included content directly — no `directive_block` node remains for
`@include` after resolution.

Pre-resolution (for tooling that needs to track includes):

```json
{
  "type": "directive_block",
  "name": "include",
  "params": { "path": "./shared/header.md" },
  "meta": { "resolved": true }
}
```

Post-resolution: the node is replaced by the parsed AST of the included file.

**HTML Output:**

No special HTML wrapper — the included content renders as if it were
written inline in the including document.

Implementations MAY add a wrapping element for debugging:

```html
<!-- @include ./shared/header.md -->
<p>Content from the included file.</p>
<!-- @endinclude -->
```
```

**Key decisions**:
- Relative paths only — prevents security issues with absolute path traversal
- `heading` param enables partial includes (common use case: include just one section from a larger doc)
- Include resolves at parse time — included content is part of the AST, not a runtime reference
- Circular include detection is required (MUST), not optional

### §1.11 Built-in Directive: `@query`

**Pattern to follow**: §1.5 `@embed` in `bfm-spec.md`

**Overview**: Dynamic content block driven by BFM's document metadata system. Renders a filtered/sorted list of items from the current document's metadata (tasks, tags, links). Leaf type — body is empty.

**Spec section content**:

```markdown
### 1.11 Built-in Directive: `@query`

**Type:** Leaf

**Parameters:**

| Param   | Required | Default  | Values |
|---------|----------|----------|--------|
| `type`  | Yes      | —        | `tasks`, `tags`, `links` |
| `state` | No       | (all)    | Task state: `open`, `done`, `scheduled`, `migrated`, `irrelevant`, `event`, `priority` |
| `tag`   | No       | (all)    | Filter by tag identifier |
| `limit` | No       | (all)    | Maximum number of results |
| `sort`  | No       | `line`   | `line` (source order), `alpha` (alphabetical), `due` (by due date) |

**Core filter keys** (`state`, `tag`) are REQUIRED for conforming implementations.
`limit` and `sort` are RECOMMENDED. Implementations MAY define additional filter
keys following the same `key=value` parameter syntax.

Unknown filter keys SHOULD be preserved in the AST and ignored during rendering
(not a parse error).

The `type` parameter determines which metadata collection is queried:

- `tasks`: Queries the `tasks` collection from §7.2. Supports `state` filter.
- `tags`: Queries the `tags` array from §7.2. Supports `tag` filter (for prefix matching).
- `links`: Queries the `links` array from §7.2.

All query types support `limit` and `sort`.

**Example:**

```markdown
@query type=tasks state=open sort=due limit=10
@endquery
```

```markdown
@query type=tasks tag=security state=priority
@endquery
```

**AST:**

```json
{
  "type": "directive_block",
  "name": "query",
  "params": {
    "type": "tasks",
    "state": "open",
    "sort": "due",
    "limit": "10"
  }
}
```

**HTML Output:**

HTML output is implementation-defined. The query is evaluated against
the document's computed metadata (§7) and rendered as a list.

Default recommended rendering for `type=tasks`:

```html
<div class="query" data-query-type="tasks">
  <ul class="query__results">
    <li class="query__item" data-task-state="open">
      <span class="task-marker task-marker--open">[ ]</span>
      Write integration tests
      <span class="task-modifier" data-key="due">2025-03-01</span>
    </li>
  </ul>
</div>
```

Default recommended rendering for `type=tags`:

```html
<div class="query" data-query-type="tags">
  <ul class="query__results">
    <li class="query__item"><span class="hashtag">#typescript</span></li>
    <li class="query__item"><span class="hashtag">#security</span></li>
  </ul>
</div>
```

When the query returns no results, implementations SHOULD render an empty
container (not omit the element entirely).

**Collection-level queries** (spanning multiple documents) are NOT part of
this specification. Implementations MAY extend `@query` with a `scope`
parameter for collection-level queries, but this is non-normative.
```

**Key decisions**:
- `type` is required — forces explicit declaration of what's being queried
- `state` and `tag` are the core required filter keys per user decision
- `limit` and `sort` are recommended but not required
- Unknown filter keys are preserved, not errors — enables implementation extensions
- Collection-level queries explicitly deferred (out of scope per contract)
- HTML output is "recommended" not "required" — different renderers may present differently

### §1.12 Built-in Directive: `@toc`

**Pattern to follow**: §1.5 `@embed` in `bfm-spec.md`

**Overview**: Auto-generated table of contents from document headings. Leaf type — body is empty. The `@toc` directive marks where the TOC should be rendered.

**Spec section content**:

```markdown
### 1.12 Built-in Directive: `@toc`

**Type:** Leaf

**Parameters:**

| Param     | Required | Default | Values |
|-----------|----------|---------|--------|
| `depth`   | No       | `3`     | `1`-`6` — maximum heading level to include |
| `ordered` | No       | (flag)  | Present = use ordered list (`<ol>`) |

The TOC is generated from all ATX headings in the document body (excluding
the heading that contains or immediately precedes the `@toc` directive, if any).
Front-matter content is excluded.

Headings inside directive blocks (e.g., inside `@callout` or `@details`) are
included in the TOC by default. Implementations MAY provide an option to
exclude them.

A document MAY contain multiple `@toc` directives. Each renders independently
with its own parameters.

**Example:**

```markdown
@toc depth=2 ordered
@endtoc
```

**AST:**

```json
{
  "type": "directive_block",
  "name": "toc",
  "params": { "depth": "2", "ordered": true }
}
```

**HTML Output:**

```html
<nav class="toc" aria-label="Table of contents">
  <ol>
    <li><a href="#weekly-review">Weekly Review</a>
      <ol>
        <li><a href="#tasks">Tasks</a></li>
        <li><a href="#reference">Reference</a></li>
      </ol>
    </li>
  </ol>
</nav>
```

Heading IDs for `href` anchors are generated by slugifying the heading text
(lowercase, replace spaces with hyphens, strip non-alphanumeric characters
except hyphens). The specific slugification algorithm is implementation-defined
but SHOULD follow the GitHub-style convention.
```

**Key decisions**:
- `depth` defaults to 3 (h1-h3) — covers most common use cases
- `ordered` is a flag (like `open` on `@details`) — no value needed
- Uses `<nav>` with ARIA label for accessibility
- Slug algorithm is implementation-defined but GitHub-style recommended
- Headings inside directives are included — matches expectation that TOC covers the whole doc

### §1.13 Built-in Directive: `@math`

**Pattern to follow**: §1.5 `@embed` in `bfm-spec.md`

**Overview**: Display math block. Body is raw LaTeX (not parsed as markdown). Leaf type.

**Spec section content**:

```markdown
### 1.13 Built-in Directive: `@math`

**Type:** Leaf

**Parameters:**

| Param  | Required | Default | Values |
|--------|----------|---------|--------|
| `label` | No      | `""`    | Equation label for cross-references |

The body is treated as raw LaTeX math content. It is NOT parsed as BFM
markdown. Backslashes, braces, and other LaTeX syntax are preserved literally.

**Example:**

```markdown
@math label=eq-euler
e^{i\pi} + 1 = 0
@endmath
```

```markdown
@math
\int_{-\infty}^{\infty} e^{-x^2} dx = \sqrt{\pi}
@endmath
```

**AST:**

```json
{
  "type": "directive_block",
  "name": "math",
  "params": { "label": "eq-euler" },
  "meta": { "content": "e^{i\\pi} + 1 = 0" }
}
```

**HTML Output:**

Rendering of LaTeX is implementation-defined. The spec defines the output
wrapper only:

```html
<div class="math" id="eq-euler" role="math" aria-label="e^{i\pi} + 1 = 0">
  e^{i\pi} + 1 = 0
</div>
```

Implementations SHOULD integrate with a LaTeX rendering library (e.g.,
KaTeX, MathJax). When no renderer is available, the raw LaTeX MUST be
preserved in the output (not omitted).

**Interaction with `$$` syntax:**

Some markdown environments support `$$...$$` for display math. BFM does
not specify `$$` syntax. Implementations MAY support both `$$` and `@math`
but they are independent features — `@math` is the canonical BFM way to
express display math.
```

**Key decisions**:
- Body is raw LaTeX, not BFM — consistent with leaf directive semantics
- `label` enables equation numbering and cross-references (implementation-defined resolution)
- Raw LaTeX must be preserved when no renderer is available — never silently drop content
- Explicit non-stance on `$$` syntax — BFM doesn't own that, implementations can support both

### §1.14 Built-in Directive: `@endnotes`

**Pattern to follow**: §1.5 `@embed` in `bfm-spec.md`

**Overview**: Controls where collected footnotes are rendered. Leaf type — body is empty. Works in conjunction with the `[^label]` footnote inline syntax (Phase 3). Without an `@endnotes` directive, footnotes render at the end of the document.

**Spec section content**:

```markdown
### 1.14 Built-in Directive: `@endnotes`

**Type:** Leaf

**Parameters:**

| Param   | Required | Default | Values |
|---------|----------|---------|--------|
| `title` | No       | `""`    | Heading text for the endnotes section |

This directive controls where footnotes (defined via `[^label]:` syntax,
see §{footnote-section}) are rendered in the document.

- If `@endnotes` is present, footnotes render at the location of the directive.
- If `@endnotes` is absent, footnotes render at the end of the document.
- Multiple `@endnotes` directives in a document is a parse error.

The body MUST be empty.

**Example:**

```markdown
Some text with a footnote[^1].

## Notes

@endnotes title="References"
@endendnotes
```

**AST:**

```json
{
  "type": "directive_block",
  "name": "endnotes",
  "params": { "title": "References" }
}
```

**HTML Output:**

```html
<section class="endnotes" role="doc-endnotes">
  <h2>References</h2>
  <ol>
    <li id="fn-1">
      <p>Footnote content here. <a href="#fnref-1" class="endnote-backref" role="doc-backlink">↩</a></p>
    </li>
  </ol>
</section>
```

When `title` is empty, the heading element is omitted.

This directive is fully specified in conjunction with §{footnote-section}
(Footnotes). See that section for the inline marker syntax and footnote
definition syntax.
```

**Key decisions**:
- Only one `@endnotes` per document — prevents confusion about where footnotes render
- Default behavior (no directive) puts footnotes at end of doc — zero-config for simple cases
- `title` param adds an optional heading — useful for "References", "Notes", etc.
- Uses ARIA `doc-endnotes` and `doc-backlink` roles for accessibility

### Fixture Files

**`fixtures/blocks/include-basic.md`**:
```markdown
@include ./shared/header.md
@endinclude
```

**`fixtures/blocks/query-basic.md`**:
```markdown
## Open Tasks

@query type=tasks state=open sort=due
@endquery

## All Tags

@query type=tags sort=alpha
@endquery
```

**`fixtures/blocks/toc-basic.md`**:
```markdown
@toc depth=2
@endtoc

# Introduction

Some intro text.

## Background

Background details.

## Implementation

Implementation details.

### Sub-section

More details.
```

**`fixtures/blocks/math-basic.md`**:
```markdown
@math label=eq-quadratic
x = \frac{-b \pm \sqrt{b^2 - 4ac}}{2a}
@endmath
```

**`fixtures/blocks/endnotes-basic.md`**:
```markdown
First paragraph with a note[^1].

Second paragraph with another[^ref].

## Notes

@endnotes title="References"
@endendnotes

[^1]: This is the first footnote.
[^ref]: This is a named footnote.
```

## Implementation Steps

1. Add §1.10 (`@include`) to `bfm-spec.md` after §1.9
2. Add §1.11 (`@query`) to `bfm-spec.md` after §1.10
3. Add §1.12 (`@toc`) to `bfm-spec.md` after §1.11
4. Add §1.13 (`@math`) to `bfm-spec.md` after §1.12
5. Add §1.14 (`@endnotes`) to `bfm-spec.md` after §1.13
6. Create `fixtures/blocks/include-basic.md`
7. Create `fixtures/blocks/query-basic.md`
8. Create `fixtures/blocks/toc-basic.md`
9. Create `fixtures/blocks/math-basic.md`
10. Create `fixtures/blocks/endnotes-basic.md`

## Validation

- [ ] Each new section follows the structure of §1.5 (Type, Parameters table, Example, AST, HTML Output)
- [ ] Leaf directive bodies are described as "literal text" or "empty"
- [ ] `@query` core filter keys (`state`, `tag`) are marked as REQUIRED
- [ ] `@query` extensibility pattern is documented (unknown keys preserved)
- [ ] `@include` circular detection is MUST-level requirement
- [ ] `@include` path resolution is relative-only
- [ ] `@toc` slug algorithm is implementation-defined with recommendation
- [ ] `@math` body preservation requirement is stated
- [ ] `@endnotes` single-instance constraint is stated
- [ ] `@endnotes` cross-references §{footnote-section} from Phase 3

---

_This spec is ready for implementation. Follow the patterns and validate at each step._
