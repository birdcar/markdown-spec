# Tasks Manifest

**Created:** 2026-02-10
**Project:** bfm-metadata

## Phases

| Phase | Status | Spec File | Description |
|-------|--------|-----------|-------------|
| 1 | pending | spec-phase-1.md | BFM spec updates + shared test fixtures |
| 2 | blocked by Phase 1 | spec-phase-2.md | JavaScript (markdown-js) implementation |
| 3 | blocked by Phase 1 & 2 | spec-phase-3.md | PHP (markdown-php) implementation |

## Execution

For each phase, start a fresh Claude session and run:

```
/execute-spec docs/ideation/bfm-metadata/spec-phase-{n}.md
```

## Repositories

- Spec: `~/Code/birdcar/markdown-spec/`
- JS: `~/Code/birdcar/markdown-js/`
- PHP: `~/Code/birdcar/markdown-php/`
