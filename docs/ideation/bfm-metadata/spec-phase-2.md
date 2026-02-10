# Implementation Spec: BFM Metadata - Phase 2

**PRD**: ./prd-phase-2.md
**Estimated Effort**: XL

## Technical Approach

This phase adds four feature modules to `markdown-js`: front-matter parsing, hashtag syntax, document metadata extraction, and front-matter merging. Each follows the established micromark/mdast extension pattern.

Front-matter and hashtags are syntax-level features (micromark tokenizer + mdast transform). Metadata extraction and merging are higher-level utilities that operate on the already-parsed AST — they don't need micromark extensions.

We add `yaml` as a runtime dependency for YAML parsing. All new features get independent subpath exports for tree-shaking. The `remarkBfm` plugin registers front-matter and hashtag extensions automatically.

## File Changes

### New Files

| File Path | Purpose |
|-----------|---------|
| `src/blocks/frontmatter/syntax.ts` | Micromark tokenizer for `---` delimited YAML blocks |
| `src/blocks/frontmatter/from-markdown.ts` | mdast transform: creates `yaml` node with parsed data |
| `src/blocks/frontmatter/to-markdown.ts` | mdast serializer: writes front-matter back to `---` blocks |
| `src/blocks/frontmatter/index.ts` | Export `remarkBfmFrontmatter` remark plugin |
| `src/inlines/hashtags/syntax.ts` | Micromark tokenizer for `#identifier` |
| `src/inlines/hashtags/from-markdown.ts` | mdast transform: creates `hashtag` nodes |
| `src/inlines/hashtags/to-markdown.ts` | mdast serializer: writes `#identifier` |
| `src/inlines/hashtags/index.ts` | Export `remarkBfmHashtags` remark plugin |
| `src/metadata/extract.ts` | `extractMetadata()` — walks AST, computes built-in fields |
| `src/metadata/computed-fields.ts` | Built-in computed field implementations |
| `src/metadata/types.ts` | `DocumentMetadata`, `TaskCollection`, `ExtractedTask`, `LinkReference` types |
| `src/metadata/index.ts` | Public exports for metadata module |
| `src/merge/merge.ts` | `mergeDocuments()` — deep merge front-matter + concatenate bodies |
| `src/merge/types.ts` | `MergeOptions`, `MergeStrategy`, `MergeResolver` types |
| `src/merge/index.ts` | Public exports for merge module |
| `src/contracts/computed-field-resolver.ts` | `ComputedFieldResolver` type definition |
| `test/frontmatter.test.ts` | Front-matter parsing tests |
| `test/hashtags.test.ts` | Hashtag parsing tests |
| `test/metadata.test.ts` | Metadata extraction tests |
| `test/merge.test.ts` | Document merging tests |

### Modified Files

| File Path | Changes |
|-----------|---------|
| `src/types.ts` | Add `HashtagNode`, `YamlNode` types. Extend token type map with frontmatter and hashtag tokens. |
| `src/plugin.ts` | Register `remarkBfmFrontmatter` and `remarkBfmHashtags` in `remarkBfm()` |
| `src/index.ts` | Re-export new modules |
| `src/contracts/index.ts` | Export `ComputedFieldResolver` |
| `package.json` | Add `yaml` dependency. Add subpath exports for `./frontmatter`, `./hashtags`, `./metadata`, `./merge`. |

## Implementation Details

### Front-matter (micromark extension)

**Pattern to follow**: `src/blocks/callout/syntax.ts` for block-level tokenization

**Overview**: Tokenize `---` delimited blocks at document start. This is a flow-level construct.

```typescript
// src/blocks/frontmatter/syntax.ts
import type { Extension, Tokenizer } from 'micromark-util-types'

const tokenizeFrontmatter: Tokenizer = function (effects, ok, nok) {
  // Only match at the very start of the document (self.now().line === 1, offset === 0)
  // Enter 'frontmatter' token
  // Consume opening '---' + line ending
  // Enter 'frontmatterContent' token
  // Consume lines until closing '---'
  // Exit tokens
}

export function frontmatterSyntax(): Extension {
  return {
    flow: {
      45: { // '-' char code
        name: 'frontmatter',
        tokenize: tokenizeFrontmatter,
      }
    }
  }
}
```

```typescript
// src/blocks/frontmatter/from-markdown.ts
import YAML from 'yaml'
import type { CompileContext, Token } from 'mdast-util-from-markdown'

export function frontmatterFromMarkdown() {
  return {
    enter: {
      frontmatter: enterFrontmatter,
    },
    exit: {
      frontmatterContent: exitFrontmatterContent,
      frontmatter: exitFrontmatter,
    },
  }
}

function enterFrontmatter(this: CompileContext, token: Token) {
  this.enter({ type: 'yaml' as any, value: '', data: {} } as any, token)
}

function exitFrontmatterContent(this: CompileContext, token: Token) {
  const node = this.stack[this.stack.length - 1] as any
  const raw = this.sliceSerialize(token)
  node.value = raw
  node.data = YAML.parse(raw) ?? {}
}

function exitFrontmatter(this: CompileContext, token: Token) {
  this.exit(token)
}
```

**Key decisions**:
- Use `yaml` package (not `js-yaml`) — better YAML 1.2 compliance
- Front-matter is a flow construct, not text — registered under `flow` in micromark
- Position check: only tokenize if we're at line 1, column 1
- AST node type is `yaml` to match remark-frontmatter convention

**Implementation steps**:
1. Write `syntax.ts` with tokenizer that detects `---` at document start
2. Write `from-markdown.ts` to create `yaml` node and parse YAML content
3. Write `to-markdown.ts` to serialize `yaml` nodes back to `---` blocks
4. Write `index.ts` that exports `remarkBfmFrontmatter` plugin
5. Add token types to `types.ts`

### Hashtag Syntax (micromark extension)

**Pattern to follow**: `src/inlines/mentions/syntax.ts` — nearly identical structure

**Overview**: Tokenize `#identifier` in text. Same boundary rules as mentions but using `#` (code 35) instead of `@` (code 64).

```typescript
// src/inlines/hashtags/syntax.ts
import type { Extension, Tokenizer } from 'micromark-util-types'

const tokenizeHashtag: Tokenizer = function (effects, ok, nok) {
  const self = this
  return start

  function start(code: number | null) {
    if (code !== 35) return nok(code) // '#'

    // Must be preceded by whitespace, punctuation, or start of content
    const previous = self.previous
    if (previous !== null && !markdownSpace(previous) && !markdownLineEnding(previous)) {
      if (isAlphaNum(previous)) return nok(code)
    }

    effects.enter('hashtag')
    effects.enter('hashtagMarker')
    effects.consume(code)
    effects.exit('hashtagMarker')
    return identStart
  }

  function identStart(code: number | null) {
    if (code === null || !isAlpha(code)) return nok(code)
    effects.enter('hashtagIdentifier')
    effects.consume(code)
    return identCore
  }

  function identCore(code: number | null) {
    if (code !== null && isAlphaNum(code)) {
      effects.consume(code)
      return identCore
    }
    // '_' and '-' valid if followed by more identifier chars
    if (code !== null && (code === 95 || code === 45)) {
      return effects.check(
        { tokenize: tokenizeHashtagContinuation, partial: true },
        consumeConnector,
        endIdent,
      )(code)
    }
    return endIdent(code)
  }
  // ... endIdent exits tokens
}

export function hashtagSyntax(): Extension {
  return {
    text: {
      35: { // '#'
        name: 'hashtag',
        tokenize: tokenizeHashtag,
      },
    },
  }
}
```

**Key decisions**:
- Identifier pattern: `[a-zA-Z][a-zA-Z0-9_-]*` — no dots (unlike mentions)
- Disambiguation from ATX headings: micromark parses headings as flow constructs before text constructs, so `# Heading` is handled before our text-level tokenizer runs
- Same trailing-connector logic as mentions (strip trailing `_` or `-`)

**Implementation steps**:
1. Copy mention syntax structure, change `@` (64) to `#` (35)
2. Adjust identifier pattern (no dots)
3. Write from-markdown to create `hashtag` nodes
4. Write to-markdown to serialize
5. Add token types to `types.ts`

### Document Metadata Extraction

**Overview**: A utility function that walks a parsed AST and produces a `DocumentMetadata` object.

```typescript
// src/metadata/types.ts
export interface DocumentMetadata {
  frontmatter: Record<string, unknown>
  computed: BuiltinMetadata
  custom: Record<string, unknown>
}

export interface BuiltinMetadata {
  wordCount: number
  readingTime: number
  tasks: TaskCollection
  tags: string[]
  links: LinkReference[]
}

export interface TaskCollection {
  all: ExtractedTask[]
  open: ExtractedTask[]
  done: ExtractedTask[]
  scheduled: ExtractedTask[]
  migrated: ExtractedTask[]
  irrelevant: ExtractedTask[]
  event: ExtractedTask[]
  priority: ExtractedTask[]
}

export interface ExtractedTask {
  text: string
  state: TaskState
  modifiers: Array<{ key: string; value: string | null }>
  line: number
}

export interface LinkReference {
  url: string
  title: string | null
  line: number
}
```

```typescript
// src/metadata/extract.ts
import type { Root } from 'mdast'
import { visit } from 'unist-util-visit'
import type { ComputedFieldResolver } from '../contracts/computed-field-resolver.js'
import type { DocumentMetadata, BuiltinMetadata } from './types.js'

export function extractMetadata(
  tree: Root,
  options?: { wpm?: number; computedFields?: ComputedFieldResolver[] }
): DocumentMetadata {
  const frontmatter = extractFrontmatter(tree)
  const computed = computeBuiltins(tree, frontmatter, options?.wpm ?? 200)
  const custom = runCustomResolvers(tree, frontmatter, computed, options?.computedFields ?? [])
  return { frontmatter, computed, custom }
}
```

**Key decisions**:
- `extractMetadata` is a standalone function, not a remark plugin — it operates on an already-parsed tree
- Users can call it after `remark.parse()` or as part of a unified pipeline
- Custom resolvers run after builtins so they can reference built-in values

**Implementation steps**:
1. Define types in `types.ts`
2. Implement `extractFrontmatter()` — find `yaml` node, return its `data`
3. Implement word count — visit text nodes, split on `\s+`, count
4. Implement reading time — `Math.ceil(wordCount / wpm)`
5. Implement task extraction — visit list items with `taskState`, collect marker + modifiers
6. Implement tag extraction — union front-matter `tags` + hashtag nodes, lowercase, dedupe
7. Implement link extraction — visit link/image nodes, extract url + title + line
8. Implement custom resolver execution loop

### Front-matter Merging

**Overview**: Merge N documents by deep-merging front-matter and concatenating bodies.

```typescript
// src/merge/types.ts
export type MergeStrategy = 'last-wins' | 'first-wins' | 'error'

export type MergeResolver = (
  key: string,
  existing: unknown,
  incoming: unknown,
) => unknown

export interface MergeOptions {
  strategy?: MergeStrategy | MergeResolver
  separator?: string  // default '\n\n'
}

// src/merge/merge.ts
export interface BfmDocument {
  frontmatter: Record<string, unknown>
  body: string
}

export function mergeDocuments(
  docs: BfmDocument[],
  options?: MergeOptions,
): BfmDocument {
  const strategy = options?.strategy ?? 'last-wins'
  const separator = options?.separator ?? '\n\n'

  return docs.reduce((acc, doc) => ({
    frontmatter: deepMerge(acc.frontmatter, doc.frontmatter, strategy),
    body: acc.body ? `${acc.body}${separator}${doc.body}` : doc.body,
  }))
}

function deepMerge(
  target: Record<string, unknown>,
  source: Record<string, unknown>,
  strategy: MergeStrategy | MergeResolver,
): Record<string, unknown> {
  const result = { ...target }
  for (const [key, value] of Object.entries(source)) {
    if (!(key in result)) {
      result[key] = value
      continue
    }
    const existing = result[key]

    if (Array.isArray(existing) && Array.isArray(value)) {
      result[key] = [...existing, ...value]
    } else if (isPlainObject(existing) && isPlainObject(value)) {
      result[key] = deepMerge(
        existing as Record<string, unknown>,
        value as Record<string, unknown>,
        strategy,
      )
    } else {
      // Scalar conflict
      if (typeof strategy === 'function') {
        result[key] = strategy(key, existing, value)
      } else if (strategy === 'last-wins') {
        result[key] = value
      } else if (strategy === 'first-wins') {
        // keep existing
      } else if (strategy === 'error') {
        throw new Error(`Merge conflict on key "${key}": ${JSON.stringify(existing)} vs ${JSON.stringify(value)}`)
      }
    }
  }
  return result
}
```

**Key decisions**:
- `BfmDocument` is a simple interface (frontmatter + body string), not an AST — merging operates on parsed front-matter and raw body text
- Deep merge is recursive with configurable conflict resolution
- Type mismatch (e.g., scalar replacing array) uses the scalar conflict strategy
- Separator is configurable but defaults to `\n\n`

**Implementation steps**:
1. Define types in `types.ts`
2. Implement `deepMerge()` with all three strategies + custom resolver
3. Implement `mergeDocuments()` that reduces over array of docs
4. Export from `index.ts`

### Computed Field Resolver Contract

```typescript
// src/contracts/computed-field-resolver.ts
import type { Root } from 'mdast'
import type { BuiltinMetadata } from '../metadata/types.js'

export type ComputedFieldResolver = (
  tree: Root,
  frontmatter: Record<string, unknown>,
  builtins: BuiltinMetadata,
) => Record<string, unknown>
```

### Type Extensions

Add to `src/types.ts`:

```typescript
export type HashtagNode = Literal & {
  type: 'hashtag'
  identifier: string
}

// In PhrasingContentMap:
hashtag: HashtagNode

// In TokenTypeMap:
frontmatter: 'frontmatter'
frontmatterFence: 'frontmatterFence'
frontmatterContent: 'frontmatterContent'
hashtag: 'hashtag'
hashtagMarker: 'hashtagMarker'
hashtagIdentifier: 'hashtagIdentifier'
_hashtagPeek: '_hashtagPeek'
```

### Plugin Registration

Update `src/plugin.ts`:

```typescript
import { remarkBfmFrontmatter } from './blocks/frontmatter/index.js'
import { remarkBfmHashtags } from './inlines/hashtags/index.js'

export function remarkBfm(this: Processor<Root>) {
  remarkBfmFrontmatter.call(this)
  remarkBfmTasks.call(this)
  remarkBfmModifiers.call(this)
  remarkBfmMentions.call(this)
  remarkBfmHashtags.call(this)
  remarkBfmDirectives.call(this)
}
```

### Package Exports

Add to `package.json`:

```json
"./frontmatter": {
  "types": "./dist/blocks/frontmatter/index.d.ts",
  "import": "./dist/blocks/frontmatter/index.js"
},
"./hashtags": {
  "types": "./dist/inlines/hashtags/index.d.ts",
  "import": "./dist/inlines/hashtags/index.js"
},
"./metadata": {
  "types": "./dist/metadata/index.d.ts",
  "import": "./dist/metadata/index.js"
},
"./merge": {
  "types": "./dist/merge/index.d.ts",
  "import": "./dist/merge/index.js"
}
```

Add to `dependencies`:

```json
"yaml": "^2.4.0"
```

## Testing Requirements

### Unit Tests

| Test File | Coverage |
|-----------|----------|
| `test/frontmatter.test.ts` | Front-matter tokenization, AST output, empty/complex YAML, invalid YAML |
| `test/hashtags.test.ts` | Hashtag tokenization, boundary rules, edge cases, to-markdown roundtrip |
| `test/metadata.test.ts` | All 5 built-in computed fields, custom resolver execution |
| `test/merge.test.ts` | Deep merge, all strategies, type mismatches, body concatenation |

**Key test cases**:
- Front-matter: basic scalar/array, empty block, nested objects, invalid YAML throws, no front-matter returns empty
- Hashtags: inline position, preceded by whitespace, preceded by punctuation, not in code spans, not ATX headings, trailing connectors stripped
- Metadata wordCount: plain text, with code blocks, empty document
- Metadata tasks: all 7 states, tasks with modifiers, no tasks returns empty
- Metadata tags: front-matter only, inline only, both with dedup, case normalization
- Metadata links: markdown links, images, autolinks
- Merge: two docs, three docs, empty frontmatter, array concat, deep object merge, scalar conflict with all strategies, custom resolver
- Custom computed fields: resolver receives builtins, multiple resolvers chain

### Fixture Tests

All shared fixtures from Phase 1 should be loaded and validated:
- `fixtures/blocks/frontmatter-*.md` → parse and compare AST
- `fixtures/inlines/hashtags-*.md` → parse and compare AST + HTML
- `fixtures/metadata/*.md` → parse, extract metadata, compare `.metadata.json`
- `fixtures/merge/*.md` → merge pairs, compare output

## Error Handling

| Error Scenario | Handling Strategy |
|----------------|-------------------|
| Invalid YAML in front-matter | Throw `YAMLParseError` with line/column info from `yaml` package |
| Merge conflict with `error` strategy | Throw `MergeConflictError` with key name and both values |
| No front-matter in document | Return empty object for `frontmatter` field |
| Custom resolver throws | Let error propagate — don't swallow |

## Validation Commands

```bash
# Type checking
bun run typecheck

# Unit tests
bun run test

# Build
bun run build
```

## Open Items

- [ ] Consider whether `remark-frontmatter` (existing remark ecosystem package) should be used instead of custom tokenizer — tradeoff is dependency vs. control
- [ ] Decide if metadata extraction should be available as a remark plugin (`remarkBfmMetadata`) that attaches metadata to `file.data` in addition to standalone function
