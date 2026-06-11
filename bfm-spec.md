# Birdcar Flavored Markdown (BFM) Specification

**Version:** 0.1.0-draft
**Author:** Nick (birdcar)
**Status:** Draft

BFM is a superset of [CommonMark 0.31.2](https://spec.commonmark.org/0.31.2/)
and [GitHub Flavored Markdown 0.29](https://github.github.com/gfm/) (excluding
GFM Task List Items, which BFM replaces with an extended syntax).

All valid CommonMark and GFM (minus task lists) is valid BFM. BFM adds the
following extensions:

1. **Directive Blocks** — `@name` / `@endname` fenced containers
2. **Extended Task Lists** — `[x]`, `[>]`, `[!]`, etc. in list items
3. **Task Modifiers** — `//key:value` metadata annotations on tasks
4. **Mentions** — `@username` and `@platform:username` inline references
5. **YAML Front-matter** — `---` delimited metadata blocks
6. **Inline Hashtags** — `#topic` inline tag references
7. **Document Metadata** — Computed fields, task/tag extraction, and front-matter merging
8. **Footnotes** — `[^label]` inline references with collected endnotes

---

## 1. Directive Blocks

Directive blocks are fenced containers that begin with `@name` on its own line
and end with `@endname` on its own line. They may accept parameters on the
opening line.

### 1.1 General Syntax

```
@<name>[ <params>]
<body>
@end<name>
```

- The opening `@name` MUST start at the beginning of a line (no leading
  whitespace except up to 3 spaces of indentation, consistent with CommonMark
  block-level constructs).
- The `name` MUST be a lowercase alphanumeric identifier (`[a-z][a-z0-9]*`).
- Optional `params` follow the name, separated by whitespace.
- The closing `@endname` MUST appear on its own line.
- The `name` in the closing tag MUST match the opening tag exactly.
- Directive blocks can be nested (e.g., a callout inside a callout).

### 1.2 Parameters

Parameters are key-value pairs on the opening line:

```
@callout type=warning title="Watch Out"
```

- Keys are alphanumeric identifiers (`[a-z][a-z0-9_]*`).
- Values can be unquoted (`type=warning`) or double-quoted (`title="Watch Out"`).
- Unquoted values terminate at whitespace.
- Quoted values may contain spaces. A literal `"` inside a quoted value is
  escaped as `\"`.
- Positional (non-key-value) parameters are also permitted for specific
  directives (e.g., a URL for `@embed`).

### 1.3 Body Content

The body of a directive block is parsed differently depending on the directive
type:

- **Container directives** (e.g., `@callout`): Body is parsed as full BFM
  markdown, including other blocks, inlines, and nested directives.
- **Leaf directives** (e.g., `@embed`): Body is treated as literal text (not
  parsed as markdown). May be empty.

Each directive type's specification declares whether it is a container or leaf.

### 1.4 Built-in Directive: `@callout`

**Type:** Container

**Parameters:**

| Param   | Required | Default | Values                                    |
|---------|----------|---------|-------------------------------------------|
| `type`  | No       | `info`  | `info`, `warning`, `danger`, `success`, `note` |
| `title` | No       | `""`    | Any string (quote if contains spaces)     |

**Example:**

```markdown
@callout type=warning title="Breaking Change"
The API response format has changed. See the **migration guide** for details.
@endcallout
```

**AST:**

```json
{
  "type": "directive_block",
  "name": "callout",
  "params": { "type": "warning", "title": "Breaking Change" },
  "children": [
    {
      "type": "paragraph",
      "children": [
        { "type": "text", "value": "The API response format has changed. See the " },
        { "type": "strong", "children": [{ "type": "text", "value": "migration guide" }] },
        { "type": "text", "value": " for details." }
      ]
    }
  ]
}
```

### 1.5 Built-in Directive: `@embed`

**Type:** Leaf

**Parameters:**

| Param | Required | Default | Values    |
|-------|----------|---------|-----------|
| (positional) URL | Yes | — | Any `https?://` URL |

The body text, if present, is treated as a caption.

**Example:**

```markdown
@embed https://www.youtube.com/watch?v=dQw4w9WgXcQ
A classic internet moment.
@endembed
```

**AST:**

```json
{
  "type": "directive_block",
  "name": "embed",
  "params": { "url": "https://www.youtube.com/watch?v=dQw4w9WgXcQ" },
  "meta": { "caption": "A classic internet moment." }
}
```

### 1.6 Built-in Directive: `@details`

**Type:** Container

**Parameters:**

| Param     | Required | Default | Values                                    |
|-----------|----------|---------|-------------------------------------------|
| `summary` | No       | `""`    | Any string (quote if contains spaces)     |
| `open`    | No       | (flag)  | Present = initially expanded              |

When `summary` is omitted, implementations SHOULD render without a visible
summary label (the entire block is the toggle target).

**Example:**

```markdown
@details summary="Implementation Notes" open
This section contains **detailed notes** that are expanded by default.

- Point one
- Point two
@enddetails
```

**AST:**

```json
{
  "type": "directive_block",
  "name": "details",
  "params": { "summary": "Implementation Notes", "open": true },
  "children": [
    {
      "type": "paragraph",
      "children": [
        { "type": "text", "value": "This section contains " },
        { "type": "strong", "children": [{ "type": "text", "value": "detailed notes" }] },
        { "type": "text", "value": " that are expanded by default." }
      ]
    },
    {
      "type": "list",
      "children": [
        { "type": "list_item", "children": [{ "type": "paragraph", "children": [{ "type": "text", "value": "Point one" }] }] },
        { "type": "list_item", "children": [{ "type": "paragraph", "children": [{ "type": "text", "value": "Point two" }] }] }
      ]
    }
  ]
}
```

**HTML Output:**

```html
<details open>
  <summary>Implementation Notes</summary>
  <p>This section contains <strong>detailed notes</strong> that are expanded by default.</p>
  <ul>
    <li>Point one</li>
    <li>Point two</li>
  </ul>
</details>
```

When `summary` is empty, the `<summary>` element is omitted.

### 1.7 Built-in Directive: `@tabs` / `@tab`

**Type:** Container (both)

`@tabs` is a grouping container that MUST contain one or more `@tab` directives
as direct children. Content outside of `@tab` directives within a `@tabs` block
is a parse error.

`@tab` is only valid as a direct child of `@tabs`. A `@tab` appearing outside
a `@tabs` block is a parse error.

**Parameters (`@tabs`):**

| Param    | Required | Default | Values                           |
|----------|----------|---------|----------------------------------|
| `id`     | No       | `""`    | Alphanumeric identifier for sync |

The `id` parameter enables tab synchronization — multiple `@tabs` blocks with
the same `id` switch together when the user selects a tab. The selected tab is
matched by `label` across synced groups.

**Parameters (`@tab`):**

| Param    | Required | Default | Values                                |
|----------|----------|---------|---------------------------------------|
| `label`  | Yes      | —       | Any string (quote if contains spaces) |
| `active` | No       | (flag)  | Present = initially selected tab      |

If no `@tab` has `active`, the first tab is active by default.

**Example:**

````markdown
@tabs id=lang
@tab label="TypeScript"
```typescript
const greeting: string = "hello";
```
@endtab
@tab label="Python"
```python
greeting: str = "hello"
```
@endtab
@endtabs
````

**AST:**

```json
{
  "type": "directive_block",
  "name": "tabs",
  "params": { "id": "lang" },
  "children": [
    {
      "type": "directive_block",
      "name": "tab",
      "params": { "label": "TypeScript" },
      "children": [
        { "type": "code", "lang": "typescript", "value": "const greeting: string = \"hello\";" }
      ]
    },
    {
      "type": "directive_block",
      "name": "tab",
      "params": { "label": "Python" },
      "children": [
        { "type": "code", "lang": "python", "value": "greeting: str = \"hello\"" }
      ]
    }
  ]
}
```

**HTML Output:**

```html
<div class="tabs" data-sync-id="lang">
  <div class="tabs__nav" role="tablist">
    <button class="tabs__tab tabs__tab--active" role="tab" aria-selected="true">TypeScript</button>
    <button class="tabs__tab" role="tab" aria-selected="false">Python</button>
  </div>
  <div class="tabs__panel tabs__panel--active" role="tabpanel">
    <pre><code class="language-typescript">const greeting: string = "hello";</code></pre>
  </div>
  <div class="tabs__panel" role="tabpanel">
    <pre><code class="language-python">greeting: str = "hello"</code></pre>
  </div>
</div>
```

Tab switching behavior is implementation-defined (may require JavaScript).
Static renderers SHOULD show all panels with labels.

### 1.8 Built-in Directive: `@figure`

**Type:** Container

**Parameters:**

| Param  | Required | Default | Values                                   |
|--------|----------|---------|------------------------------------------|
| `src`  | Yes      | —       | URL or file path to media                |
| `alt`  | No       | `""`    | Alt text for the media                   |
| `id`   | No       | `""`    | Referenceable identifier (e.g., `fig-1`) |

The body content, if present, is parsed as BFM and rendered as the
`<figcaption>`. An empty body produces a figure without a caption.

**Example:**

```markdown
@figure src="./diagram.png" alt="System architecture" id=fig-arch
The **system architecture** showing data flow between services.
See @alice for the original design.
@endfigure
```

**AST:**

```json
{
  "type": "directive_block",
  "name": "figure",
  "params": { "src": "./diagram.png", "alt": "System architecture", "id": "fig-arch" },
  "children": [
    {
      "type": "paragraph",
      "children": [
        { "type": "text", "value": "The " },
        { "type": "strong", "children": [{ "type": "text", "value": "system architecture" }] },
        { "type": "text", "value": " showing data flow between services.\nSee " },
        { "type": "mention", "identifier": "alice" },
        { "type": "text", "value": " for the original design." }
      ]
    }
  ]
}
```

**HTML Output:**

```html
<figure id="fig-arch">
  <img src="./diagram.png" alt="System architecture" />
  <figcaption>
    <p>The <strong>system architecture</strong> showing data flow between services.
    See <span class="mention">@alice</span> for the original design.</p>
  </figcaption>
</figure>
```

When the body is empty, the `<figcaption>` element is omitted.

### 1.9 Built-in Directive: `@aside`

**Type:** Container

**Parameters:**

| Param   | Required | Default | Values                                |
|---------|----------|---------|---------------------------------------|
| `title` | No       | `""`    | Any string (quote if contains spaces) |

**Example:**

```markdown
@aside title="Historical Context"
This API was originally designed in 2019 for a different use case.
The `legacyMode` flag exists for **backwards compatibility** only.
@endaside
```

**AST:**

```json
{
  "type": "directive_block",
  "name": "aside",
  "params": { "title": "Historical Context" },
  "children": [
    {
      "type": "paragraph",
      "children": [
        { "type": "text", "value": "This API was originally designed in 2019 for a different use case.\nThe " },
        { "type": "code_span", "value": "legacyMode" },
        { "type": "text", "value": " flag exists for " },
        { "type": "strong", "children": [{ "type": "text", "value": "backwards compatibility" }] },
        { "type": "text", "value": " only." }
      ]
    }
  ]
}
```

**HTML Output:**

```html
<aside class="aside">
  <p class="aside__title">Historical Context</p>
  <p>This API was originally designed in 2019 for a different use case.
  The <code>legacyMode</code> flag exists for <strong>backwards compatibility</strong> only.</p>
</aside>
```

When `title` is empty, the title element is omitted.

### 1.10 Built-in Directive: `@include`

**Type:** Leaf

**Parameters:**

| Param              | Required | Default | Values             |
|--------------------|----------|---------|--------------------|
| (positional) path  | Yes      | —       | Relative file path |
| `heading`          | No       | `""`    | Heading text to extract a section |

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

### 1.11 Built-in Directive: `@query`

**Type:** Leaf

**Parameters:**

| Param   | Required | Default  | Values                                                                                     |
|---------|----------|----------|--------------------------------------------------------------------------------------------|
| `type`  | Yes      | —        | `tasks`, `tags`, `links`                                                                   |
| `state` | No       | (all)    | Task state: `open`, `done`, `scheduled`, `migrated`, `irrelevant`, `event`, `priority`     |
| `tag`   | No       | (all)    | Filter by tag identifier                                                                   |
| `limit` | No       | (all)    | Maximum number of results                                                                  |
| `sort`  | No       | `line`   | `line` (source order), `alpha` (alphabetical), `due` (by due date)                         |

**Core filter keys** (`state`, `tag`) are REQUIRED for conforming implementations.
`limit` and `sort` are RECOMMENDED. Implementations MAY define additional filter
keys following the same `key=value` parameter syntax.

Unknown filter keys SHOULD be preserved in the AST and ignored during rendering
(not a parse error).

The `type` parameter determines which metadata collection is queried:

- `tasks`: Queries the `tasks` collection from §7.2. Supports `state` filter.
- `tags`: Queries the `tags` array from §7.2. Supports `tag` filter (for prefix
  matching).
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

### 1.12 Built-in Directive: `@toc`

**Type:** Leaf

**Parameters:**

| Param     | Required | Default | Values                                       |
|-----------|----------|---------|----------------------------------------------|
| `depth`   | No       | `3`     | `1`-`6` — maximum heading level to include   |
| `ordered` | No       | (flag)  | Present = use ordered list (`<ol>`)           |

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

### 1.13 Built-in Directive: `@math`

**Type:** Leaf

**Parameters:**

| Param   | Required | Default | Values                              |
|---------|----------|---------|-------------------------------------|
| `label` | No       | `""`    | Equation label for cross-references |

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

### 1.14 Built-in Directive: `@endnotes`

**Type:** Leaf

**Parameters:**

| Param   | Required | Default | Values                                |
|---------|----------|---------|---------------------------------------|
| `title` | No       | `""`    | Heading text for the endnotes section |

This directive controls where footnotes (defined via `[^label]:` syntax,
see §10) are rendered in the document.

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

This directive is fully specified in conjunction with §10 (Footnotes).
See that section for the inline marker syntax and footnote definition syntax.

---

## 2. Extended Task Lists

BFM extends the GFM task list syntax with additional single-character state
markers inside the brackets of list items.

### 2.1 Syntax

A task list item is a list item whose first inline content matches the pattern:

```
[<marker>]<space>
```

Where `<marker>` is one of the defined state characters and `<space>` is a
literal space character (U+0020).

Task markers MUST appear at the very beginning of a list item's inline content.
They are only recognized inside list items (unordered or ordered).

### 2.2 State Markers

| Marker | State       | Semantics                          |
|--------|-------------|------------------------------------|
| ` `    | Open        | Incomplete task, to be done        |
| `x`    | Done        | Completed task                     |
| `>`    | Scheduled   | Deferred to a specific date/time   |
| `<`    | Migrated    | Moved to another list or project   |
| `-`    | Irrelevant  | Cancelled or no longer applicable  |
| `o`    | Event       | Calendar event or appointment      |
| `!`    | Priority    | Urgent or high-importance task     |

### 2.3 Examples

```markdown
- [ ] Open task
- [x] Completed task
- [>] Scheduled for later //due:2025-03-01
- [<] Migrated to backlog ->backlog
- [-] No longer relevant
- [o] Team standup at 10am
- [!] Critical production bug
```

### 2.4 AST

Each task marker produces a `task_marker` inline node as the first child of
the list item's paragraph:

```json
{
  "type": "list_item",
  "attributes": { "data-task": "scheduled" },
  "children": [
    {
      "type": "paragraph",
      "children": [
        { "type": "task_marker", "state": "scheduled" },
        { "type": "text", "value": "Call the dentist " },
        { "type": "task_modifier", "key": "due", "value": "2025-03-01" }
      ]
    }
  ]
}
```

### 2.5 Interaction with GFM

BFM's extended task list syntax is a **superset** of GFM's task list items.
`[ ]` and `[x]` produce identical AST structures to their GFM equivalents
but with the BFM `task_marker` node type instead of the GFM
`task_list_item_marker` type.

Implementations MUST NOT register both BFM task lists and GFM task lists
simultaneously, as they will conflict.

---

## 3. Task Modifiers

Task modifiers are inline annotations that attach temporal or categorical
metadata to task list items. They use the `//` prefix to avoid collision with
any standard markdown syntax or with `@mentions`.

### 3.1 Syntax

```
//<key>[:<value>]
```

- The `//` prefix MUST be preceded by whitespace or appear at the start of
  inline content.
- `key` is an alphanumeric identifier (`[a-z][a-z0-9]*`).
- The `:` separator and `value` are optional. When omitted, the modifier is
  a boolean flag (value is implicitly `true`).
- `value` extends from after the `:` until the next `//` modifier, the end
  of the line, or the end of the inline content — then is trimmed of trailing
  whitespace.
- Multiple modifiers can appear on the same line.

### 3.2 Built-in Modifier Keys

| Key       | Value Format           | Semantics                            |
|-----------|------------------------|--------------------------------------|
| `due`     | ISO 8601 date or partial: `YYYY-MM-DD`, `YYYY-MM`, `YYYY` | Hard deadline |
| `around`  | ISO 8601 partial date  | Soft / approximate target date       |
| `after`   | ISO 8601 date          | Don't surface until this date        |
| `every`   | Recurrence descriptor  | Repeating schedule (see §3.3)        |
| `cron`    | Cron expression        | Complex recurrence schedule          |
| `hard`    | (none — flag)          | Immovable deadline                   |
| `wait`    | (none — flag)          | Blocked / waiting on external input  |

Implementations MAY define additional modifier keys. Unknown keys SHOULD be
preserved in the AST and rendered as-is.

### 3.3 Recurrence Descriptors for `//every:`

The `//every:` modifier accepts human-readable interval strings:

```
daily, weekly, 2-weeks, monthly, quarterly, yearly
weekdays, weekends
mon, tue, wed, thu, fri, sat, sun
1st, 15th (day of month)
```

Numeric intervals use the format `<n>-<unit>` where unit is one of:
`days`, `weeks`, `months`, `years`.

### 3.4 Examples

```markdown
- [>] Call the dentist //due:2025-03-01
- [ ] Quarterly review //every:quarter
- [>] Follow up with Sarah //around:2025-03 //wait
- [o] Team retro //due:2025-02-07 //every:2-weeks
- [ ] Run backups //cron:0 9 * * 1
- [!] File taxes //due:2025-04-15 //hard
- [ ] Check in after launch //after:2025-06-01
```

### 3.5 AST

Each modifier produces a `task_modifier` inline node:

```json
{ "type": "task_modifier", "key": "due", "value": "2025-03-01" }
{ "type": "task_modifier", "key": "hard", "value": null }
{ "type": "task_modifier", "key": "cron", "value": "0 9 * * 1" }
```

### 3.6 Scope

Task modifiers are currently specified for use within task list items only.
Implementations MAY choose to recognize them in other inline contexts, but
this is not required for conformance.

---

## 4. Mentions

Mentions are inline references to users, entities, or platform accounts,
using the `@` prefix. BFM supports two forms: simple mentions and
platform-prefixed mentions.

### 4.1 Syntax

**Simple mention:**

```
@<identifier>
```

**Platform-prefixed mention:**

```
@<platform>:<platform-identifier>
```

- The `@` MUST be preceded by whitespace, punctuation, or appear at the start
  of inline content (not mid-word).
- For simple mentions, `identifier` matches `[a-zA-Z][a-zA-Z0-9._-]*` —
  alphanumeric with dots, underscores, and hyphens.
- For platform-prefixed mentions, `platform` matches `[a-z]+` and
  `platform-identifier` matches `[a-zA-Z0-9._@-]+` — alphanumeric with dots,
  underscores, hyphens, and `@` (to support federated handles like Mastodon).
- The identifier terminates at whitespace, punctuation (except `.`, `_`, `-`,
  `@` within the identifier), or end of input.
- When a mention contains a `:`, the portion before `:` is parsed as the
  `platform` and the portion after as the `platform-identifier`. This applies
  regardless of whether the platform is recognized — recognition only affects
  rendering behavior.

### 4.2 Disambiguation

`@` followed by a BFM directive name (e.g., `@callout`, `@embed`, `@end*`)
at the start of a line is parsed as a directive block, NOT a mention.

`@` appearing inline (within paragraph text) is always parsed as a mention.

### 4.3 Resolution

Simple mention resolution is **implementation-defined**. The parser produces
an AST node with the raw identifier. How that identifier maps to a person,
profile, URL, or other entity is determined by the consuming application.

Platform-prefixed mentions are resolved using platform-specific URL templates
(see §4.6). Unrecognized platforms are rendered as plain mentions without
links.

### 4.4 AST

**Simple mention:**

```json
{ "type": "mention", "identifier": "sarah" }
```

**Platform-prefixed mention:**

```json
{ "type": "mention", "identifier": "birdcar", "platform": "github" }
```

The `platform` field is present only for platform-prefixed mentions.

### 4.5 Built-in Platforms

| Platform   | Prefix     | URL Template                                                           |
|------------|------------|------------------------------------------------------------------------|
| GitHub     | `github`   | `https://github.com/{identifier}`                                      |
| Twitter/X  | `twitter`  | `https://twitter.com/{identifier}`                                     |
| Bluesky    | `bluesky`  | `https://bsky.app/profile/{identifier}`                                |
| Mastodon   | `mastodon` | Derived from handle: `user@instance` → `https://{instance}/@{user}`    |
| npm        | `npm`      | `https://www.npmjs.com/package/{identifier}`                           |
| LinkedIn   | `linkedin` | `https://www.linkedin.com/in/{identifier}`                             |

Implementations MAY define additional platforms. Unknown platforms MUST still
be parsed as platform mentions in the AST but SHOULD be rendered without
links (as plain mention spans).

### 4.6 HTML Output

**Simple mention (default):**

```html
<span class="mention">@sarah</span>
```

**Recognized platform mention:**

```html
<a href="{resolved-url}" class="mention mention--{platform}"
   title="{Platform}: {identifier}" rel="noopener noreferrer">@{platform}:{identifier}</a>
```

**Unrecognized platform mention:**

```html
<span class="mention">@{platform}:{identifier}</span>
```

Implementations MAY provide configurable rendering for simple mentions
(e.g., resolving `@sarah` to a user profile link).

---

## 5. YAML Front-matter

BFM supports YAML front-matter as an optional metadata block at the beginning
of a document.

### 5.1 Syntax

A front-matter block begins with `---` on the first line of the document and
ends with a closing `---` on its own line:

```
---
<yaml content>
---
```

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

```json
{
  "type": "yaml",
  "value": "key1: value1\ntags:\n  - tag1",
  "data": {
    "key1": "value1",
    "tags": ["tag1"]
  }
}
```

The `value` field contains the raw YAML string. The `data` field contains
the parsed key-value structure.

### 5.3 Interaction with CommonMark

The opening `---` of front-matter would normally be parsed as a thematic
break in CommonMark. BFM resolves this by only treating `---` as front-matter
when it appears on the very first line of the document. All other occurrences
of `---` on their own line are thematic breaks as usual.

---

## 6. Inline Hashtags

Hashtags are inline references to topics or categories, using the `#` prefix.

### 6.1 Syntax

```
#<identifier>
```

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

```json
{ "type": "hashtag", "identifier": "typescript" }
```

### 6.4 HTML Output

Default rendering:

```html
<span class="hashtag">#typescript</span>
```

Implementations MAY provide configurable rendering (e.g., linking to a
tag page).

---

## 7. Document Metadata

BFM defines a document metadata model that implementations extract from
parsed documents. This enables structured data access without consumers
needing to walk the AST directly.

### 7.1 Metadata Structure

```
DocumentMetadata {
  frontmatter: Record<string, unknown>   // parsed YAML front-matter
  computed: {
    wordCount: number                     // body word count
    readingTime: number                   // minutes, ceil(wordCount / wpm)
    tasks: TaskCollection                 // extracted tasks by state
    tags: string[]                        // unified deduplicated tags
    links: LinkReference[]                // extracted links
    footnotes: FootnoteReference[]        // extracted footnote references
  }
  custom: Record<string, unknown>         // user-defined computed fields
}
```

### 7.2 Built-in Computed Fields

#### wordCount

Count of words in body content (all text nodes). Code blocks are included.
Front-matter content is excluded.

#### readingTime

`ceil(wordCount / wordsPerMinute)` where default WPM is 200. Implementations
SHOULD allow configuring the WPM value.

#### tasks

```
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
```

#### tags

Deduplicated array of tag strings from:
1. The `tags` array field in YAML front-matter (if present)
2. All inline `#hashtag` nodes in the body

Tags are normalized to lowercase. Duplicates are removed preserving first
occurrence order.

#### links

```
LinkReference {
  url: string
  title: string | null
  line: number
}
```

Extracted from all link and image nodes in the body content.

#### footnotes

```
FootnoteReference {
  label: string
  index: number
  line: number       // line of the marker's first occurrence
}
```

Extracted from all `[^label]` inline markers in the body content. Ordered
by first appearance (matching the `index` assignment in §10.1).

### 7.3 Backlinks (Collection-level)

Backlinks are computed across a collection of documents. For each document,
backlinks is the set of other documents that contain a link to it.

Backlink computation is NOT a per-document computed field — it requires
knowledge of the full collection. Implementations MUST provide a separate
API for computing backlinks.

---

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

---

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

```markdown
---
key1: value1
tags:
  - arrValue1
  - arrValue2
---

Content from a.
```

```markdown
---
keyA: valueB
tags:
  - arrValueA
  - arrValueB
---

Content from b.
```

Merged result:

```markdown
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
```

---

## 10. Footnotes

Footnotes are inline references to endnote content, using the `[^label]`
syntax from Pandoc and PHP Markdown Extra.

### 10.1 Inline Markers

```
[^<label>]
```

- `label` matches `[a-zA-Z0-9_-]+` — alphanumeric with underscores and hyphens.
- The marker MUST appear within inline content (inside a paragraph, list item,
  or other inline context).
- The marker produces a superscript reference number in the output.
- Markers are auto-numbered in the order they first appear in the document.
- A marker referencing a label with no corresponding definition is a parse error.

### 10.2 Definitions

```
[^<label>]: <content>
```

- Definitions MUST appear at the start of a line (no leading whitespace
  except up to 3 spaces of indentation, consistent with CommonMark).
- The `label` MUST match a marker that appears in the document.
  An orphaned definition (no corresponding marker) SHOULD produce a warning
  but is NOT a parse error — the definition is silently ignored.
- Single-line content follows the `:` on the same line.
- Multi-line content is indicated by indentation (4 spaces or 1 tab) on
  subsequent lines:

```markdown
[^long]: This is a footnote with multiple paragraphs.

    The second paragraph is indented with 4 spaces.

    - Even lists work inside footnotes.
```

- Footnote content is parsed as BFM markdown (paragraphs, inline formatting,
  links, mentions, hashtags, etc.).
- Definitions may appear anywhere in the document body. Their position does
  NOT affect rendering order — footnotes are rendered in the order their
  markers first appear.

### 10.3 Rendering

Footnotes are collected and rendered as an ordered list at:
1. The location of the `@endnotes` directive (§1.14), if present.
2. The end of the document, if no `@endnotes` directive is present.

Each rendered footnote includes a backlink to the point of reference.

When a marker is referenced multiple times (same `[^label]` appears more than
once), each occurrence links to the same footnote. The footnote's backlinks
point to all referencing locations.

### 10.4 Disambiguation

- `[^` followed by a valid label and `]` is a footnote marker, not a link.
- `[^` in other contexts (e.g., `[^not a footnote` without closing `]`) is
  literal text.
- Footnote definitions at the start of a line take priority over paragraph
  content. A line starting with `[^label]:` is always a definition.
- Inside code spans and code blocks, `[^label]` is literal text.

### 10.5 AST

**Inline marker:**

```json
{ "type": "footnote_ref", "label": "1", "index": 1 }
```

`index` is the auto-assigned display number (1-based, in order of first
appearance).

**Definition:**

```json
{
  "type": "footnote_def",
  "label": "1",
  "index": 1,
  "children": [
    {
      "type": "paragraph",
      "children": [
        { "type": "text", "value": "This is the first footnote." }
      ]
    }
  ]
}
```

Footnote definitions are collected into a `footnotes` array at the root level
of the AST (not inline with body content):

```json
{
  "type": "root",
  "children": [ "..." ],
  "footnotes": [
    { "type": "footnote_def", "label": "1", "index": 1, "children": [ "..." ] },
    { "type": "footnote_def", "label": "note", "index": 2, "children": [ "..." ] }
  ]
}
```

### 10.6 HTML Output

**Inline marker:**

```html
<sup class="footnote-ref" id="fnref-1"><a href="#fn-1" role="doc-noteref">[1]</a></sup>
```

**Collected footnotes section:**

```html
<section class="endnotes" role="doc-endnotes">
  <ol>
    <li id="fn-1">
      <p>This is the first footnote. <a href="#fnref-1" class="endnote-backref" role="doc-backlink">↩</a></p>
    </li>
    <li id="fn-note">
      <p>This is a named footnote. <a href="#fnref-note" class="endnote-backref" role="doc-backlink">↩</a></p>
    </li>
  </ol>
</section>
```

IDs use the pattern `fn-{label}` for definitions and `fnref-{label}` for
references. When a label is used multiple times, subsequent reference IDs
use `fnref-{label}-{n}` (e.g., `fnref-1-2` for the second reference to
footnote 1).

---

## 11. Conformance

### 11.1 Levels

- **BFM Core**: CommonMark + Directive Blocks + Extended Task Lists +
  Task Modifiers + Mentions + YAML Front-matter + Inline Hashtags +
  Footnotes. All conforming implementations MUST support BFM Core.
- **BFM Extended**: BFM Core + Document Metadata model (§7) + Computed
  Field Resolvers (§8) + Front-matter Merging (§9). Implementations SHOULD
  support BFM Extended.
- **BFM Full**: BFM Extended + all built-in directive types (callout, embed,
  details, tabs/tab, figure, aside, include, query, toc, math, endnotes) +
  all built-in modifier keys + all built-in computed fields (wordCount,
  readingTime, tasks, tags, links, footnotes). Implementations SHOULD
  support BFM Full.

### 11.2 Testing

Conformance is verified against the shared fixture suite in the `bfm-spec`
repository. Each fixture consists of:

- `<name>.md` — Input markdown
- `<name>.ast.json` — Expected AST (normalized)
- `<name>.html` — Expected HTML output (default renderer)
- `<name>.metadata.json` — Expected metadata output (for BFM Extended fixtures)

An implementation is conforming if it produces matching AST structures and
HTML output for all fixtures in the suite. BFM Extended conformance
additionally requires matching metadata output.

### 11.3 Extension

Implementations MAY add additional directive types, task states, modifier
keys, computed fields, merge strategies, and renderers beyond those specified
here. Custom extensions MUST NOT alter the parsing behavior of the core
syntax defined above.

#### 11.3.1 Directive Registration (reference implementation)

A conforming implementation MAY expose a directive registration mechanism that
lets consumers declare custom directive types without modifying the parser. The
reference implementation does so via `remarkBfm({ directives })` (and its
lower-level `remarkBfmDirectives({ directives })`).

A **directive definition** has the following shape:

```
DirectiveDefinition ::= {
  kind:      "container" | "leaf"
  toHast?:   HastData | (DirectiveBlockNode -> HastData)
  transform?: (DirectiveBlockNode, DirectiveContext) -> void
}
```

- `kind` controls how the body is parsed (§1.3): `"container"` re-parses the
  body as BFM markdown; `"leaf"` stores it as raw text.
- `toHast` is a shorthand for attaching HTML serialization hints (compatible
  with the `remark-rehype` bridge) onto the node. It MAY be a static object
  or a function that receives the node and returns one.
- `transform` is an escape hatch that receives the node and a context object
  (which includes the full document tree). When `transform` is present,
  `toHast` is ignored.

Custom definitions supplied at call time are merged with the built-in registry;
a custom definition with the same name as a built-in MUST take precedence.

Directives that do not appear in the registry MUST still parse and appear in
the AST. The implementation SHOULD treat them as `"container"` by default and
MUST NOT attach render data to them.

---

## Appendix A: Grammar Summary

```
directive_block  := directive_open NL body directive_close NL
directive_open   := "@" name (WS params)?
directive_close  := "@end" name
name             := [a-z][a-z0-9]*
params           := param (WS param)*
param            := key "=" value | positional
key              := [a-z][a-z0-9_]*
value            := quoted_string | unquoted_string
quoted_string    := '"' ( [^"\\] | '\"' )* '"'
unquoted_string  := [^\s]+
positional       := [^\s]+

task_marker      := "[" state_char "]" " "
state_char       := " " | "x" | ">" | "<" | "-" | "o" | "!"

task_modifier    := "//" key (":" value)?
modifier_key     := [a-z][a-z0-9]*
modifier_value   := .+?  (until next "//" or end of inline)

mention          := "@" (platform ":" platform_ident | identifier)
platform         := [a-z]+
platform_ident   := [a-zA-Z0-9._@-]+
identifier       := [a-zA-Z][a-zA-Z0-9._-]*

frontmatter      := "---" NL yaml_content "---" NL
yaml_content     := .+?  (valid YAML 1.2)

hashtag          := "#" hashtag_ident
hashtag_ident    := [a-zA-Z][a-zA-Z0-9_-]*

footnote_ref     := "[^" fn_label "]"
footnote_def     := "[^" fn_label "]:" SP content NL
                    (INDENT content NL)*
fn_label         := [a-zA-Z0-9_-]+

; Built-in directive names (non-exhaustive):
; callout, embed, details, tab, tabs, figure, aside,
; include, query, toc, math, endnotes
```
