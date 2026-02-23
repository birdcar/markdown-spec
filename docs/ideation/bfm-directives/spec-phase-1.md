# Implementation Spec: BFM Directives Expansion - Phase 1 (Container Directives)

**Contract**: ./contract.md
**Estimated Effort**: M

## Technical Approach

Phase 1 adds four container directives to the BFM spec: `@details`, `@tabs`/`@tab`, `@figure`, and `@aside`. All are container type — their body content is parsed as full BFM markdown (like `@callout`). Each directive gets a new subsection under §1 (Directive Blocks), following the exact structure of §1.4 (`@callout`) and §1.5 (`@embed`): type declaration, parameters table, example, AST shape, and HTML output.

Pattern to follow: §1.4 (`@callout`) in `bfm-spec.md` lines 75-112 for container directive spec sections.

Fixture files follow the existing pattern in `fixtures/blocks/callout-basic.md` — minimal `.md` inputs demonstrating core usage and edge cases.

## Feedback Strategy

**Inner-loop command**: N/A (spec-writing, not code)

**Playground**: Manual review — read each new section and verify it's internally consistent with existing spec conventions.

**Why this approach**: This is a documentation/spec task. Validation is structural review, not automated tests.

## File Changes

### Modified Files

| File Path | Changes |
|-----------|---------|
| `bfm-spec.md` | Add §1.6 (`@details`), §1.7 (`@tabs`/`@tab`), §1.8 (`@figure`), §1.9 (`@aside`) |

### New Files

| File Path | Purpose |
|-----------|---------|
| `fixtures/blocks/details-basic.md` | Basic `@details` usage with summary and body |
| `fixtures/blocks/tabs-basic.md` | Basic `@tabs`/`@tab` with multiple tabs |
| `fixtures/blocks/figure-basic.md` | Basic `@figure` with image and caption |
| `fixtures/blocks/aside-basic.md` | Basic `@aside` with content |

## Implementation Details

### §1.6 Built-in Directive: `@details`

**Pattern to follow**: §1.4 `@callout` in `bfm-spec.md`

**Overview**: Collapsible/expandable section. Maps to HTML `<details>`/`<summary>`. Container type — body is parsed as BFM.

**Spec section content**:

```markdown
### 1.6 Built-in Directive: `@details`

**Type:** Container

**Parameters:**

| Param     | Required | Default | Values                            |
|-----------|----------|---------|-----------------------------------|
| `summary` | No       | `""`    | Any string (quote if contains spaces) |
| `open`    | No       | (flag)  | Present = initially expanded      |

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
```

**Key decisions**:
- `open` is a boolean flag param (no value), consistent with `//hard` and `//wait` modifier pattern
- `summary` maps to `<summary>` element, not a heading — keeps semantic HTML alignment
- When `summary` is empty, omit `<summary>` entirely rather than rendering an empty element

### §1.7 Built-in Directive: `@tabs` / `@tab`

**Pattern to follow**: §1.4 `@callout` in `bfm-spec.md`

**Overview**: Tabbed content groups. `@tabs` is a container that MUST contain one or more `@tab` children. `@tab` is a container that holds the content for a single tab pane. `@tab` is only valid as a direct child of `@tabs`.

**Spec section content**:

```markdown
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

| Param   | Required | Default | Values                                |
|---------|----------|---------|---------------------------------------|
| `label` | Yes      | —       | Any string (quote if contains spaces) |
| `active`| No       | (flag)  | Present = initially selected tab      |

If no `@tab` has `active`, the first tab is active by default.

**Example:**

```markdown
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
```

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
```

**Key decisions**:
- `@tab` is only valid inside `@tabs` — enforced at parse level, not just semantically
- `id` on `@tabs` enables cross-block synchronization (common in docs with language selectors)
- HTML uses ARIA roles for accessibility
- Tab switching is explicitly implementation-defined — spec doesn't mandate JS

### §1.8 Built-in Directive: `@figure`

**Pattern to follow**: §1.4 `@callout` in `bfm-spec.md`

**Overview**: Rich figure container wrapping media (images, embeds) with structured captions. Maps to HTML `<figure>`/`<figcaption>`. Container type — body is the caption, parsed as BFM.

**Spec section content**:

```markdown
### 1.8 Built-in Directive: `@figure`

**Type:** Container

**Parameters:**

| Param  | Required | Default | Values                                  |
|--------|----------|---------|-----------------------------------------|
| `src`  | Yes      | —       | URL or file path to media               |
| `alt`  | No       | `""`    | Alt text for the media                  |
| `id`   | No       | `""`    | Referenceable identifier (e.g., `fig-1`)|

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
```

**Key decisions**:
- `src` is a named param (not positional like `@embed`'s URL) because `@figure` has multiple params and positional would be ambiguous
- `id` enables cross-references (e.g., "see Figure 1" links) — resolution is implementation-defined
- Body is caption (parsed as BFM), distinct from `alt` which is plain text for accessibility
- Media type detection from `src` extension is implementation-defined

### §1.9 Built-in Directive: `@aside`

**Pattern to follow**: §1.4 `@callout` in `bfm-spec.md`

**Overview**: Supplementary/tangential content. Semantically distinct from `@callout` — an aside is related but not essential ("by the way..."), while a callout demands attention ("warning!"). Maps to HTML `<aside>`.

**Spec section content**:

```markdown
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
```

**Key decisions**:
- Intentionally minimal params — `@aside` is simple supplementary content, not a categorized alert system like `@callout`
- No `type` param (unlike `@callout`) — asides don't have severity levels
- Title renders as a `<p>` with a class, not a heading — avoids polluting document outline

### Fixture Files

**`fixtures/blocks/details-basic.md`**:
```markdown
@details summary="Click to expand"
This is the **hidden content** inside the details block.

- Item one
- Item two
@enddetails
```

**`fixtures/blocks/tabs-basic.md`**:
```markdown
@tabs id=lang
@tab label="TypeScript"
```typescript
const x: number = 42;
```
@endtab
@tab label="Python"
```python
x: int = 42
```
@endtab
@endtabs
```

**`fixtures/blocks/figure-basic.md`**:
```markdown
@figure src="https://example.com/photo.jpg" alt="A sunset" id=fig-1
A beautiful **sunset** over the mountains.
@endfigure
```

**`fixtures/blocks/aside-basic.md`**:
```markdown
@aside title="Fun Fact"
The original implementation used a **linked list** for this data structure.
@endaside
```

## Implementation Steps

1. Add §1.6 (`@details`) to `bfm-spec.md` after §1.5
2. Add §1.7 (`@tabs`/`@tab`) to `bfm-spec.md` after §1.6
3. Add §1.8 (`@figure`) to `bfm-spec.md` after §1.7
4. Add §1.9 (`@aside`) to `bfm-spec.md` after §1.8
5. Create `fixtures/blocks/details-basic.md`
6. Create `fixtures/blocks/tabs-basic.md`
7. Create `fixtures/blocks/figure-basic.md`
8. Create `fixtures/blocks/aside-basic.md`

## Validation

- [ ] Each new section follows the exact structure of §1.4 (Type, Parameters table, Example, AST, HTML Output)
- [ ] All parameter names use lowercase alphanumeric (`[a-z][a-z0-9_]*`) per §1.2
- [ ] Container directive bodies are described as "parsed as full BFM markdown"
- [ ] AST nodes use `directive_block` type with `name` matching the directive
- [ ] HTML output uses semantic elements where applicable (`<details>`, `<figure>`, `<aside>`)
- [ ] Fixture files are minimal and demonstrate core usage
- [ ] `@tab` validity constraint (only inside `@tabs`) is explicitly stated

---

_This spec is ready for implementation. Follow the patterns and validate at each step._
