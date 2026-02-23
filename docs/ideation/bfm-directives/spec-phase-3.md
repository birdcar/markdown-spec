# Implementation Spec: BFM Directives Expansion - Phase 3 (Footnotes + Housekeeping)

**Contract**: ./contract.md
**Estimated Effort**: M

## Technical Approach

Phase 3 adds footnote inline syntax (`[^label]` markers and `[^label]:` definitions) as a new BFM inline extension, then updates the grammar summary, conformance levels, and extension list to reflect all new directives from Phases 1-3.

Footnotes follow the Pandoc/PHP Markdown Extra convention. They're a new inline syntax (not a directive) — the first non-directive inline extension BFM adds beyond mentions and hashtags.

The `@endnotes` directive (Phase 2) controls footnote placement. This phase defines the footnote syntax that `@endnotes` collects.

## Feedback Strategy

**Inner-loop command**: N/A (spec-writing, not code)

**Playground**: Manual review — verify footnote syntax doesn't conflict with existing BFM inlines, and grammar summary covers all new productions.

**Why this approach**: Documentation/spec task. Validation is review-based.

## File Changes

### Modified Files

| File Path | Changes |
|-----------|---------|
| `bfm-spec.md` | Add §{N} (Footnotes), update §1 extension list, update §10 (Conformance), update Appendix A (Grammar), update top-of-file extension list |

### New Files

| File Path | Purpose |
|-----------|---------|
| `fixtures/inlines/footnotes-basic.md` | Basic footnote markers and definitions |
| `fixtures/inlines/footnotes-edge-cases.md` | Multi-paragraph footnotes, nested content, ordering |

## Implementation Details

### New Section: Footnotes

**Overview**: Footnotes are inline references to endnote content. They use two syntactic elements: an inline marker (`[^label]`) at the point of reference, and a definition (`[^label]: content`) that provides the footnote content. This is BFM's third inline extension (after mentions and hashtags).

**Spec section content** (new §, numbered after the last existing section before Conformance — the exact number depends on Phase 1 and 2 numbering):

```markdown
## {N}. Footnotes

Footnotes are inline references to endnote content, using the `[^label]`
syntax from Pandoc and PHP Markdown Extra.

### {N}.1 Inline Markers

```
[^<label>]
```

- `label` matches `[a-zA-Z0-9_-]+` — alphanumeric with underscores and hyphens.
- The marker MUST appear within inline content (inside a paragraph, list item,
  or other inline context).
- The marker produces a superscript reference number in the output.
- Markers are auto-numbered in the order they first appear in the document.
- A marker referencing a label with no corresponding definition is a parse error.

### {N}.2 Definitions

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

### {N}.3 Rendering

Footnotes are collected and rendered as an ordered list at:
1. The location of the `@endnotes` directive (§1.14), if present.
2. The end of the document, if no `@endnotes` directive is present.

Each rendered footnote includes a backlink to the point of reference.

When a marker is referenced multiple times (same `[^label]` appears more than
once), each occurrence links to the same footnote. The footnote's backlinks
point to all referencing locations.

### {N}.4 Disambiguation

- `[^` followed by a valid label and `]` is a footnote marker, not a link.
- `[^` in other contexts (e.g., `[^not a footnote` without closing `]`) is
  literal text.
- Footnote definitions at the start of a line take priority over paragraph
  content. A line starting with `[^label]:` is always a definition.
- Inside code spans and code blocks, `[^label]` is literal text.

### {N}.5 AST

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
  "children": [ ... ],
  "footnotes": [
    { "type": "footnote_def", "label": "1", "index": 1, "children": [ ... ] },
    { "type": "footnote_def", "label": "note", "index": 2, "children": [ ... ] }
  ]
}
```

### {N}.6 HTML Output

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
```

**Key decisions**:
- Follows Pandoc convention exactly — broad ecosystem familiarity
- Labels are flexible (alphanumeric + underscore + hyphen), not restricted to numbers
- Multi-line footnotes use 4-space indentation (consistent with CommonMark continuation)
- Definitions can appear anywhere — position doesn't affect order (order follows first marker appearance)
- Orphaned definitions are warnings, not errors — lenient parsing
- Missing definitions ARE errors — strict on references to prevent broken links
- Footnotes collected at root AST level, not inline — clean separation of concerns
- ARIA roles (`doc-noteref`, `doc-endnotes`, `doc-backlink`) for accessibility

### Update: Top-of-file Extension List

Add new items to the numbered extension list at lines 14-20:

```markdown
1. **Directive Blocks** — `@name` / `@endname` fenced containers
2. **Extended Task Lists** — `[x]`, `[>]`, `[!]`, etc. in list items
3. **Task Modifiers** — `//key:value` metadata annotations on tasks
4. **Mentions** — `@username` and `@platform:username` inline references
5. **YAML Front-matter** — `---` delimited metadata blocks
6. **Inline Hashtags** — `#topic` inline tag references
7. **Document Metadata** — Computed fields, task/tag extraction, and front-matter merging
8. **Footnotes** — `[^label]` inline references with collected endnotes
```

### Update: §10 Conformance Levels

Update the conformance levels to include new directives:

```markdown
- **BFM Core**: CommonMark + Directive Blocks + Extended Task Lists +
  Task Modifiers + Mentions + YAML Front-matter + Inline Hashtags +
  Footnotes. All conforming implementations MUST support BFM Core.
- **BFM Extended**: BFM Core + Document Metadata model (§7) + Computed
  Field Resolvers (§8) + Front-matter Merging (§9). Implementations SHOULD
  support BFM Extended.
- **BFM Full**: BFM Extended + all built-in directive types (callout, embed,
  details, tabs/tab, figure, aside, include, query, toc, math, endnotes) +
  all built-in modifier keys + all built-in computed fields (wordCount,
  readingTime, tasks, tags, links). Implementations SHOULD support BFM Full.
```

**Key decision**: Footnotes go in BFM Core (they're fundamental inline syntax). New directives go in BFM Full (they're optional richness). `@include` could arguably be Core, but keeping all directives in Full maintains a clean separation.

### Update: Appendix A Grammar Summary

Add new productions:

```
footnote_ref     := "[^" fn_label "]"
footnote_def     := "[^" fn_label "]:" SP content NL
                    (INDENT content NL)*
fn_label         := [a-zA-Z0-9_-]+
```

Add to the directive name list (informative, not normative):

```
; Built-in directive names (non-exhaustive):
; callout, embed, details, tab, tabs, figure, aside,
; include, query, toc, math, endnotes
```

### Update: Document Metadata (§7)

Add footnotes to the metadata structure if they should be extractable:

```
DocumentMetadata {
  ...
  computed: {
    ...
    footnotes: FootnoteReference[]   // extracted footnote references
  }
}

FootnoteReference {
  label: string
  index: number
  line: number       // line of the marker's first occurrence
}
```

**Key decision**: Include footnotes in metadata — enables tooling to count/list footnotes, check for orphaned references, etc. This is lightweight and consistent with how tasks, tags, and links are already extracted.

### Fixture Files

**`fixtures/inlines/footnotes-basic.md`**:
```markdown
This sentence has a footnote[^1] and another[^note].

[^1]: This is the first footnote.
[^note]: This is a named footnote with **bold** content.
```

**`fixtures/inlines/footnotes-edge-cases.md`**:
```markdown
Multiple references to the same footnote[^shared] appear here
and also here[^shared].

A footnote with multi-line content[^long].

[^shared]: This footnote is referenced twice.
[^long]: This is the first paragraph of a long footnote.

    This is the second paragraph, indented with 4 spaces.

    - And a list item inside the footnote.
```

## Implementation Steps

1. Add the Footnotes section to `bfm-spec.md` (new §, after Document Metadata or Computed Field Resolvers — exact number TBD based on Phase 1/2 section numbering)
2. Update the top-of-file extension list (add item 8: Footnotes)
3. Update §7.1 metadata structure to include `footnotes` in computed fields
4. Update §10.1 conformance levels to include Footnotes in Core and new directives in Full
5. Update Appendix A grammar summary with `footnote_ref`, `footnote_def`, `fn_label` productions and built-in directive names
6. Create `fixtures/inlines/footnotes-basic.md`
7. Create `fixtures/inlines/footnotes-edge-cases.md`
8. Final review: read the full updated spec for internal consistency (section numbering, cross-references, grammar completeness)

## Validation

- [ ] Footnote section follows the structure of existing inline extensions (§4 Mentions, §6 Hashtags)
- [ ] Disambiguation rules cover all `[^` parsing ambiguities
- [ ] AST includes both `footnote_ref` (inline) and `footnote_def` (root-level) nodes
- [ ] HTML output includes ARIA roles for accessibility
- [ ] Multi-line footnote indentation rule is consistent with CommonMark
- [ ] `@endnotes` directive (Phase 2) is properly cross-referenced
- [ ] Extension list at top of file includes all new items
- [ ] Conformance levels correctly classify Core vs Full
- [ ] Grammar summary includes all new productions from Phases 1-3
- [ ] All section numbers are consistent and cross-references resolve
- [ ] Document metadata structure includes footnotes
- [ ] Fixture files cover basic usage and edge cases (multi-ref, multi-line)

---

_This spec is ready for implementation. Follow the patterns and validate at each step._
