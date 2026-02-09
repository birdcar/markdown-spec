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

## 5. Conformance

### 5.1 Levels

- **BFM Core**: CommonMark + Directive Blocks + Extended Task Lists +
  Task Modifiers + Mentions. All conforming implementations MUST support
  BFM Core.
- **BFM Full**: BFM Core + all built-in directive types (callout, embed) +
  all built-in modifier keys. Implementations SHOULD support BFM Full.

### 5.2 Testing

Conformance is verified against the shared fixture suite in the `bfm-spec`
repository. Each fixture consists of:

- `<name>.md` — Input markdown
- `<name>.ast.json` — Expected AST (normalized)
- `<name>.html` — Expected HTML output (default renderer)

An implementation is conforming if it produces matching AST structures and
HTML output for all fixtures in the suite.

### 5.3 Extension

Implementations MAY add additional directive types, task states, modifier
keys, and renderers beyond those specified here. Custom extensions MUST NOT
alter the parsing behavior of the core syntax defined above.

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
```
