# Implementation Spec: BFM Metadata - Phase 3

**PRD**: ./prd-phase-3.md
**Estimated Effort**: XL

## Technical Approach

This phase mirrors the JS implementation in PHP using league/commonmark's extension system. Each feature follows the established PHP patterns: Extension class (registers components), Parser class (tokenizes input), Node class (AST node), Renderer class (HTML output).

Front-matter is a block-level construct using `BlockStartParserInterface`. Hashtags are inline using `InlineParserInterface`. Metadata extraction and merging are standalone service classes that operate on parsed documents.

We use `symfony/yaml` for YAML parsing — it's likely already present in PHP projects using league/commonmark and is the PHP ecosystem standard.

## File Changes

### New Files

| File Path | Purpose |
|-----------|---------|
| `src/Block/Frontmatter/FrontmatterExtension.php` | Registers block parser and renderer |
| `src/Block/Frontmatter/FrontmatterBlock.php` | AST node for front-matter block |
| `src/Block/Frontmatter/FrontmatterBlockStartParser.php` | Detects opening `---` at document start |
| `src/Block/Frontmatter/FrontmatterBlockContinueParser.php` | Continues until closing `---` |
| `src/Block/Frontmatter/FrontmatterRenderer.php` | Renders nothing (metadata, not content) |
| `src/Inline/Hashtag/HashtagExtension.php` | Registers inline parser and renderer |
| `src/Inline/Hashtag/Hashtag.php` | AST node for inline hashtag |
| `src/Inline/Hashtag/HashtagParser.php` | Parses `#identifier` in text |
| `src/Inline/Hashtag/HashtagRenderer.php` | Renders `<span class="hashtag">#tag</span>` |
| `src/Contracts/ComputedFieldResolverInterface.php` | Interface for custom computed fields |
| `src/Metadata/DocumentMetadata.php` | Value object for metadata result |
| `src/Metadata/TaskCollection.php` | Value object for tasks grouped by state |
| `src/Metadata/ExtractedTask.php` | Value object for a single extracted task |
| `src/Metadata/LinkReference.php` | Value object for an extracted link |
| `src/Metadata/MetadataExtractor.php` | Walks AST, computes built-in fields |
| `src/Metadata/ComputedFields/WordCountField.php` | Word count implementation |
| `src/Metadata/ComputedFields/ReadingTimeField.php` | Reading time implementation |
| `src/Metadata/ComputedFields/TaskExtractionField.php` | Task extraction implementation |
| `src/Metadata/ComputedFields/TagExtractionField.php` | Tag extraction implementation |
| `src/Metadata/ComputedFields/LinkExtractionField.php` | Link extraction implementation |
| `src/Merge/DocumentMerger.php` | Deep merge implementation |
| `src/Merge/MergeOptions.php` | Configuration for merge strategy |
| `src/Merge/MergeStrategy.php` | Enum for built-in strategies |
| `src/Merge/BfmDocument.php` | Simple value object (frontmatter + body) |
| `src/Merge/BacklinkComputer.php` | Collection-level backlink computation |
| `tests/Block/Frontmatter/FrontmatterExtensionTest.php` | Front-matter parsing tests |
| `tests/Inline/Hashtag/HashtagExtensionTest.php` | Hashtag parsing tests |
| `tests/Metadata/MetadataExtractorTest.php` | Metadata extraction tests |
| `tests/Merge/DocumentMergerTest.php` | Document merging tests |

### Modified Files

| File Path | Changes |
|-----------|---------|
| `src/Environment/BfmEnvironmentFactory.php` | Register `FrontmatterExtension` and `HashtagExtension` |
| `composer.json` | Add `symfony/yaml` dependency (if not already present) |

## Implementation Details

### Front-matter Block Parser

**Pattern to follow**: `src/Block/Callout/CalloutBlockStartParser.php` and `CalloutBlock.php`

**Overview**: Block parser that detects `---` at document start, captures YAML content, and produces a `FrontmatterBlock` AST node.

```php
// src/Block/Frontmatter/FrontmatterBlock.php
<?php

declare(strict_types=1);

namespace Birdcar\Markdown\Block\Frontmatter;

use League\CommonMark\Node\Block\AbstractBlock;

final class FrontmatterBlock extends AbstractBlock
{
    private string $rawYaml = '';
    private array $data = [];

    public function getRawYaml(): string { return $this->rawYaml; }
    public function setRawYaml(string $yaml): void { $this->rawYaml = $yaml; }
    public function getData(): array { return $this->data; }
    public function setData(array $data): void { $this->data = $data; }
}
```

```php
// src/Block/Frontmatter/FrontmatterBlockStartParser.php
<?php

declare(strict_types=1);

namespace Birdcar\Markdown\Block\Frontmatter;

use League\CommonMark\Parser\Block\BlockStart;
use League\CommonMark\Parser\Block\BlockStartParserInterface;
use League\CommonMark\Parser\Cursor;
use League\CommonMark\Parser\MarkdownParserStateInterface;

final class FrontmatterBlockStartParser implements BlockStartParserInterface
{
    private bool $documentStarted = false;

    public function tryStart(Cursor $cursor, MarkdownParserStateInterface $parserState): ?BlockStart
    {
        // Only match at the very start of the document
        if ($this->documentStarted) {
            return BlockStart::none();
        }

        $this->documentStarted = true;

        if ($cursor->isIndented()) {
            return BlockStart::none();
        }

        $line = $cursor->getRemainder();
        if (trim($line) !== '---') {
            return BlockStart::none();
        }

        $cursor->advanceToEnd();

        return BlockStart::of(new FrontmatterBlockContinueParser())
            ->at($cursor);
    }
}
```

```php
// src/Block/Frontmatter/FrontmatterBlockContinueParser.php
<?php

declare(strict_types=1);

namespace Birdcar\Markdown\Block\Frontmatter;

use League\CommonMark\Node\Block\AbstractBlock;
use League\CommonMark\Parser\Block\AbstractBlockContinueParser;
use League\CommonMark\Parser\Block\BlockContinue;
use League\CommonMark\Parser\Cursor;
use Symfony\Component\Yaml\Yaml;

final class FrontmatterBlockContinueParser extends AbstractBlockContinueParser
{
    private FrontmatterBlock $block;
    private array $lines = [];

    public function __construct()
    {
        $this->block = new FrontmatterBlock();
    }

    public function getBlock(): AbstractBlock
    {
        return $this->block;
    }

    public function tryContinue(Cursor $cursor, ...$args): ?BlockContinue
    {
        $line = $cursor->getRemainder();

        if (trim($line) === '---') {
            $cursor->advanceToEnd();
            return BlockContinue::finished();
        }

        $this->lines[] = $line;
        $cursor->advanceToEnd();

        return BlockContinue::at($cursor);
    }

    public function closeBlock(): void
    {
        $raw = implode("\n", $this->lines);
        $this->block->setRawYaml($raw);
        $this->block->setData(Yaml::parse($raw) ?? []);
    }
}
```

**Key decisions**:
- `documentStarted` flag ensures front-matter only matches on first block attempt
- `FrontmatterRenderer` renders empty string — front-matter is metadata, not visible content
- YAML parsed in `closeBlock()` after all lines collected

**Implementation steps**:
1. Create `FrontmatterBlock` node class
2. Create `FrontmatterBlockStartParser` with document-start detection
3. Create `FrontmatterBlockContinueParser` to collect lines until closing `---`
4. Create `FrontmatterRenderer` that returns empty string
5. Create `FrontmatterExtension` to wire it all together
6. Register in `BfmEnvironmentFactory`

### Hashtag Inline Parser

**Pattern to follow**: `src/Inline/Mention/MentionParser.php` — nearly identical structure

```php
// src/Inline/Hashtag/HashtagParser.php
<?php

declare(strict_types=1);

namespace Birdcar\Markdown\Inline\Hashtag;

use League\CommonMark\Parser\Inline\InlineParserInterface;
use League\CommonMark\Parser\Inline\InlineParserMatch;
use League\CommonMark\Parser\InlineParserContext;

final class HashtagParser implements InlineParserInterface
{
    public function getMatchDefinition(): InlineParserMatch
    {
        return InlineParserMatch::string('#');
    }

    public function parse(InlineParserContext $inlineContext): bool
    {
        $cursor = $inlineContext->getCursor();
        $position = $cursor->getPosition();

        // Must not be preceded by alphanumeric
        if ($position > 0) {
            $prevChar = $cursor->getCharacter($position - 1);
            if ($prevChar !== null && preg_match('/[a-zA-Z0-9]/', $prevChar) === 1) {
                return false;
            }
        }

        $remainder = $cursor->getRemainder();

        // Match # followed by identifier: [a-zA-Z][a-zA-Z0-9_-]*
        if (preg_match('/^#([a-zA-Z][a-zA-Z0-9_-]*)/', $remainder, $matches) !== 1) {
            return false;
        }

        $identifier = $matches[1];

        // Strip trailing _ or - characters
        $identifier = rtrim($identifier, '_-');

        if ($identifier === '') {
            return false;
        }

        $cursor->advanceBy(1 + mb_strlen($identifier, 'UTF-8'));

        $inlineContext->getContainer()->appendChild(new Hashtag($identifier));

        return true;
    }
}
```

```php
// src/Inline/Hashtag/Hashtag.php
<?php

declare(strict_types=1);

namespace Birdcar\Markdown\Inline\Hashtag;

use League\CommonMark\Node\Inline\AbstractInline;

final class Hashtag extends AbstractInline
{
    public function __construct(
        private readonly string $identifier,
    ) {
    }

    public function getIdentifier(): string
    {
        return $this->identifier;
    }
}
```

**Key decisions**:
- No dots in hashtag identifiers (unlike mentions)
- Same boundary rules as mentions
- Default renderer produces `<span class="hashtag">#tag</span>`

**Implementation steps**:
1. Create `Hashtag` node class
2. Create `HashtagParser` (copy MentionParser, change `@` to `#`, adjust regex)
3. Create `HashtagRenderer`
4. Create `HashtagExtension`
5. Register in `BfmEnvironmentFactory`

### Metadata Extraction

**Overview**: Service class that walks a parsed document and produces metadata.

```php
// src/Metadata/MetadataExtractor.php
<?php

declare(strict_types=1);

namespace Birdcar\Markdown\Metadata;

use Birdcar\Markdown\Block\Frontmatter\FrontmatterBlock;
use Birdcar\Markdown\Contracts\ComputedFieldResolverInterface;
use Birdcar\Markdown\Inline\Hashtag\Hashtag;
use Birdcar\Markdown\Inline\Task\TaskMarker;
use League\CommonMark\Node\Block\Document;
use League\CommonMark\Node\NodeWalker;

final class MetadataExtractor
{
    /** @param ComputedFieldResolverInterface[] $resolvers */
    public function __construct(
        private readonly int $wordsPerMinute = 200,
        private readonly array $resolvers = [],
    ) {
    }

    public function extract(Document $document): DocumentMetadata
    {
        $frontmatter = $this->extractFrontmatter($document);
        $wordCount = $this->computeWordCount($document);
        $readingTime = (int) ceil($wordCount / $this->wordsPerMinute);
        $tasks = $this->extractTasks($document);
        $tags = $this->extractTags($document, $frontmatter);
        $links = $this->extractLinks($document);

        $builtins = new BuiltinMetadata($wordCount, $readingTime, $tasks, $tags, $links);

        $custom = [];
        foreach ($this->resolvers as $resolver) {
            $custom = array_merge($custom, $resolver->resolve($document, $frontmatter, $builtins));
        }

        return new DocumentMetadata($frontmatter, $builtins, $custom);
    }
}
```

```php
// src/Contracts/ComputedFieldResolverInterface.php
<?php

declare(strict_types=1);

namespace Birdcar\Markdown\Contracts;

use Birdcar\Markdown\Metadata\BuiltinMetadata;
use League\CommonMark\Node\Block\Document;

interface ComputedFieldResolverInterface
{
    /** @return array<string, mixed> */
    public function resolve(Document $document, array $frontmatter, BuiltinMetadata $builtins): array;
}
```

**Key decisions**:
- `MetadataExtractor` is a standalone service, not a league/commonmark extension — it operates on a parsed `Document`
- Value objects (`DocumentMetadata`, `TaskCollection`, `ExtractedTask`, `LinkReference`) are immutable
- Resolvers receive the parsed document, frontmatter, and builtins — matching the JS interface

**Implementation steps**:
1. Create value objects: `DocumentMetadata`, `TaskCollection`, `ExtractedTask`, `LinkReference`, `BuiltinMetadata`
2. Create `ComputedFieldResolverInterface`
3. Implement `MetadataExtractor` with individual computed field methods
4. Each computed field walks the AST using `NodeWalker`

### Document Merging

```php
// src/Merge/MergeStrategy.php
<?php

declare(strict_types=1);

namespace Birdcar\Markdown\Merge;

enum MergeStrategy: string
{
    case LastWins = 'last-wins';
    case FirstWins = 'first-wins';
    case Error = 'error';
}
```

```php
// src/Merge/DocumentMerger.php
<?php

declare(strict_types=1);

namespace Birdcar\Markdown\Merge;

final class DocumentMerger
{
    public function merge(array $documents, ?MergeOptions $options = null): BfmDocument
    {
        $options ??= new MergeOptions();
        $result = array_shift($documents);

        foreach ($documents as $doc) {
            $result = new BfmDocument(
                frontmatter: $this->deepMerge($result->frontmatter, $doc->frontmatter, $options),
                body: $result->body . $options->separator . $doc->body,
            );
        }

        return $result;
    }

    private function deepMerge(array $target, array $source, MergeOptions $options): array
    {
        $result = $target;
        foreach ($source as $key => $value) {
            if (!array_key_exists($key, $result)) {
                $result[$key] = $value;
                continue;
            }

            $existing = $result[$key];

            if (is_array($existing) && is_array($value) && $this->isList($existing) && $this->isList($value)) {
                $result[$key] = array_merge($existing, $value);
            } elseif (is_array($existing) && is_array($value) && !$this->isList($existing) && !$this->isList($value)) {
                $result[$key] = $this->deepMerge($existing, $value, $options);
            } else {
                $result[$key] = match ($options->strategy) {
                    MergeStrategy::LastWins => $value,
                    MergeStrategy::FirstWins => $existing,
                    MergeStrategy::Error => throw new MergeConflictException($key, $existing, $value),
                };
            }
        }
        return $result;
    }

    private function isList(array $arr): bool
    {
        return array_is_list($arr);
    }
}
```

**Key decisions**:
- `MergeStrategy` is a PHP 8.1 enum
- Use `array_is_list()` (PHP 8.1+) to distinguish sequential arrays (lists) from associative arrays (objects)
- Custom resolver via closure in `MergeOptions` when more control needed
- `BfmDocument` is a simple readonly value object

**Implementation steps**:
1. Create `MergeStrategy` enum
2. Create `BfmDocument` value object
3. Create `MergeOptions` with strategy + separator
4. Implement `DocumentMerger` with `deepMerge()` recursive logic
5. Create `BacklinkComputer` for collection-level backlink computation

### Environment Factory Update

```php
// Update BfmEnvironmentFactory::create()
use Birdcar\Markdown\Block\Frontmatter\FrontmatterExtension;
use Birdcar\Markdown\Inline\Hashtag\HashtagExtension;

// In create() method, add before other extensions:
$environment->addExtension(new FrontmatterExtension());
// ... existing extensions ...
$environment->addExtension(new HashtagExtension());
```

Front-matter extension registered first so it captures `---` before CommonMark's thematic break parser.

## Testing Requirements

### Unit Tests

| Test File | Coverage |
|-----------|----------|
| `tests/Block/Frontmatter/FrontmatterExtensionTest.php` | Front-matter parsing, empty block, complex YAML, no front-matter |
| `tests/Inline/Hashtag/HashtagExtensionTest.php` | Hashtag parsing, boundaries, edge cases |
| `tests/Metadata/MetadataExtractorTest.php` | All built-in computed fields, custom resolvers |
| `tests/Merge/DocumentMergerTest.php` | Deep merge, all strategies, body concatenation |

**Key test cases**:
- Front-matter: basic, empty, complex, invalid YAML throws, thematic break not confused for front-matter
- Hashtags: inline, boundaries, not in code, not ATX heading, trailing connectors
- Tasks: all 7 states extracted, modifiers included, line numbers correct
- Tags: front-matter + inline, dedup, normalization
- Links: markdown links, images, line numbers
- Merge: scalars, arrays, nested objects, type mismatch, all strategies, custom resolver

### Fixture Tests

Load all Phase 1 fixtures and validate output matches expected JSON.

## Error Handling

| Error Scenario | Handling Strategy |
|----------------|-------------------|
| Invalid YAML | Throw `Symfony\Component\Yaml\Exception\ParseException` |
| Merge conflict with Error strategy | Throw `MergeConflictException` |
| No front-matter | Return empty array |
| Custom resolver throws | Let exception propagate |

## Validation Commands

```bash
# Tests
composer run test

# Static analysis (if configured)
composer run phpstan
```

## Open Items

- [ ] Determine if `symfony/yaml` needs to be added to `require` in `composer.json` or if it's already a transitive dependency
- [ ] Consider adding a `FrontmatterAwareTrait` for easy front-matter access in custom code
