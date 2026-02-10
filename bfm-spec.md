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
4. **Mentions** — `@username` inline references (context-dependent resolution)
5. **YAML Front-matter** — `---` delimited metadata blocks
6. **Inline Hashtags** — `#topic` inline tag references
7. **Document Metadata** — Computed fields, task/tag extraction, and front-matter merging

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

Mentions are inline references to users or entities, using the `@` prefix.

### 4.1 Syntax

```
@<identifier>
```

- The `@` MUST be preceded by whitespace, punctuation, or appear at the start
  of inline content (not mid-word).
- `identifier` matches `[a-zA-Z][a-zA-Z0-9._-]*` — alphanumeric with dots,
  underscores, and hyphens.
- The identifier terminates at whitespace, punctuation (except `.`, `_`, `-`
  within the identifier), or end of input.

### 4.2 Disambiguation

`@` followed by a BFM directive name (e.g., `@callout`, `@embed`, `@end*`)
at the start of a line is parsed as a directive block, NOT a mention.

`@` appearing inline (within paragraph text) is always parsed as a mention.

### 4.3 Resolution

Mention resolution is **implementation-defined**. The parser produces an AST
node with the raw identifier. How that identifier maps to a person, profile,
URL, or other entity is determined by the consuming application.

### 4.4 AST

```json
{ "type": "mention", "identifier": "sarah" }
```

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

## 10. Conformance

### 10.1 Levels

- **BFM Core**: CommonMark + Directive Blocks + Extended Task Lists +
  Task Modifiers + Mentions + YAML Front-matter + Inline Hashtags. All
  conforming implementations MUST support BFM Core.
- **BFM Extended**: BFM Core + Document Metadata model (§7) + Computed
  Field Resolvers (§8) + Front-matter Merging (§9). Implementations SHOULD
  support BFM Extended.
- **BFM Full**: BFM Extended + all built-in directive types (callout, embed) +
  all built-in modifier keys + all built-in computed fields (wordCount,
  readingTime, tasks, tags, links). Implementations SHOULD support BFM Full.

### 10.2 Testing

Conformance is verified against the shared fixture suite in the `bfm-spec`
repository. Each fixture consists of:

- `<name>.md` — Input markdown
- `<name>.ast.json` — Expected AST (normalized)
- `<name>.html` — Expected HTML output (default renderer)
- `<name>.metadata.json` — Expected metadata output (for BFM Extended fixtures)

An implementation is conforming if it produces matching AST structures and
HTML output for all fixtures in the suite. BFM Extended conformance
additionally requires matching metadata output.

### 10.3 Extension

Implementations MAY add additional directive types, task states, modifier
keys, computed fields, merge strategies, and renderers beyond those specified
here. Custom extensions MUST NOT alter the parsing behavior of the core
syntax defined above.

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

mention          := "@" identifier
identifier       := [a-zA-Z][a-zA-Z0-9._-]*

frontmatter      := "---" NL yaml_content "---" NL
yaml_content     := .+?  (valid YAML 1.2)

hashtag          := "#" hashtag_ident
hashtag_ident    := [a-zA-Z][a-zA-Z0-9_-]*
```
