---
description: Load Meno documentation for a specific topic
allowed-tools: Read, Glob
---

# Meno Documentation Loader

Load Meno documentation for the specified topic to provide context for your task.

## Usage

```
/meno-docs [topic]
```

## Available Topics

| Topic | Description |
|-------|-------------|
| `examples` | Copy-ready patterns: Layout, Navigation, Hero, Card grid, CMS template, Footer, FAQ with .js + .css |
| `cms-schema` | CMS collections, field types, URL patterns |
| `javascript` | `defineVars`, vanilla JS patterns, component communication |
| `libraries` | External scripts and CSP configuration |
| `redirects` | URL redirects for static hosting |
| `meno-filter` | Client-side filtering with data attributes |
| `meno-filter-api` | MenoFilter JavaScript API |
| `website-convert` | Converting imported website analysis to Meno components |

For node types, interface prop types, template expressions, interactive styles, conditional rendering, locales/i18n, page meta, and project config — see the root `CLAUDE.md`.

## Instructions

$ARGUMENTS contains the topic requested by the user.

1. Parse the topic from $ARGUMENTS (e.g., `examples`, `cms-schema`)
2. If no topic provided or topic is `all`, list available topics
3. Read the documentation file from `.claude/docs/meno/{topic}.md`
4. Present the documentation content to help with the current task

### Topic Aliases
- `cms` → `cms-schema`
- `filter` → `meno-filter`
- `filter-api` → `meno-filter-api`
- `js` → `javascript`
- `convert` → `website-convert`
- `import` → `website-convert`
- `example` → `examples`

### Multi-topic Loading
If user requests multiple topics (comma-separated), load all of them:
- `/meno-docs examples,cms-schema` → Load both examples and cms-schema docs

## Example

User: `/meno-docs examples`

Response: Read and present the content of `.claude/docs/meno/examples.md`
