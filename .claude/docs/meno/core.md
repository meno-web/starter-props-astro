# Meno node model — reference

This Meno project is an **Astro project**: its source of truth on disk is `.astro`
files in the **meno-astro dialect**, not JSON.

- **On-disk format** → `src/pages/<slug>.astro` (pages) and `src/components/<Name>.astro`
  (components). For the on-disk grammar (how to read/hand-edit these `.astro` files so
  they round-trip), see `meno-astro-dialect.md` and the project `CLAUDE.md`.
- **The Studio API wire model** → the editor and the import/build tools speak a JSON
  **node tree** over the Studio dev-server HTTP API (`GET /api/pages/<slug>`,
  `POST /api/save-page`, `/api/component-data`, …). That JSON is *not* what lives on
  disk — the server's astro provider serializes it to/from `.astro` for you. This doc
  describes that node model: the shapes you send/receive over the API, and the node /
  style / token vocabulary they share with the dialect.

> Tools that build pages by hand (the import agents) work through the Studio API and
> therefore work in this node model. When you edit a file directly instead, write
> dialect `.astro` (see `meno-astro-dialect.md`).

## Page vs component (API wire model)

The API path argument for a page is the page's logical id (e.g. `index.json`,
`about.json`); the provider maps it to `src/pages/<slug>.astro`. The body is a node tree:

- Page: `{ "root": {...}, "meta": {...} }`
- Component: `{ "component": { "interface": {...}, "structure": {...} } }`

## Node types

| Type | Required | Children | Key Props |
|------|----------|----------|-----------|
| node | tag | nodes/text | style, attributes, interactiveStyles |
| component | component | nodes (if slot) | props |
| link | href | nodes/text | href (string or {href, target}) |
| embed | html | - | style |
| list | source | item template | sourceType ("prop"\|"collection"), itemAs, limit, sort, filter. Not an HTML element — works like `map()`, rendering children once per item. No style support. |
| locale-list | - | - | displayType, showFlag |
| slot | - | - | (in component structure only) |

## Style object

Only `base` is required. Add `tablet`/`mobile` only when overriding:
```json
"style": {
  "base": { "padding": "24px", "backgroundColor": "var(--background)" },
  "tablet": { "padding": "16px" },
  "mobile": { "padding": "12px" }
}
```
On disk this becomes a `style({ base: {...}, tablet: {...}, mobile: {...} })` call —
never a raw `class="..."` (a raw class string does not round-trip).

## Color format

Always use `var(--colorName)` (e.g., `var(--primary)`). Raw hex values do NOT work.
Colors are defined in `colors.json`.

## CSS property names

Use camelCase: `backgroundColor`, `fontSize`, `borderRadius` (not kebab-case).

## Text content

HTML nodes carry text in `children` (no `text` property) in the wire model:
```json
{ "type": "node", "tag": "span", "children": "Hello World" }
```
On disk this is `<span>Hello World</span>`.

## Enums

Project-level reusable option sets stored in `enums.json`:
```json
{
  "size": ["sm", "md", "lg", "xl"],
  "theme": ["light", "dark"]
}
```
Component select props reference enums via `enumName` instead of inline `options`:
```json
"size": { "type": "select", "enumName": "size", "default": "md" }
```
- **File**: `enums.json` in project root (separate from `project.config.json`)
- **API**: GET `/api/enums`, POST `/api/save-enums`
- **Service**: `EnumService` with caching and HMR via `hmr:enums-update`
- **Migration**: If `enums.json` doesn't exist, falls back to reading the `enums` key from `project.config.json` (read-only, no auto-write)

## Variables

CSS design tokens stored in `variables.json`:
```json
{
  "variables": [
    {
      "name": "Heading",
      "prop_name": "Size 1",
      "cssVar": "--h1-fs",
      "value": "48px",
      "type": "fontSize",
      "group": "font-size"
    }
  ]
}
```
Each variable has:
- `name` — display label (e.g., `"Heading"`)
- `prop_name` (optional) — secondary label for specificity (e.g., `"Size 1"`). Together with `name` forms the full label: "Heading — Size 1"
- `cssVar` — CSS custom property name (e.g., `--h1-fs`)
- `value` — base value (e.g., `48px`)
- `type` — responsive scaling category: `fontSize`, `padding`, `margin`, `gap`, or `none`
- `group` (optional) — UI filter group: `font-family`, `font-size`, `font-weight`, `line-height`, `letter-spacing`, `margin`, `padding`, `gap`, `size`, `border-radius`, `border-width`, `opacity`, `z-index`, `text-align`, `other`
- `scales` (optional) — per-variable breakpoint scale overrides, e.g., `{ "tablet": 0.88, "mobile": 0.75 }`

Variables with `type` other than `none` auto-scale at smaller breakpoints using `responsiveScales` from `project.config.json`.

Use in styles via `var()`: `{ "fontSize": "var(--h1-fs)" }`

- **File**: `variables.json` in project root
- **API**: GET `/api/variables-status`, GET `/api/variables-css`, POST `/api/save-variables`
- **Service**: `VariableService` with caching and HMR via `hmr:variables-update`

## Common mistakes (avoid these)

- NO `type: "image"` -> use `file` with `accept: "image/*"`
- NO `text` prop on nodes -> use `children` for text
- NO manual `data-component` attr -> system adds it automatically
- NO `children` in interface -> reserved, use `content` or `text`
- Colors: always `var(--name)` not hex values

## Working in this project

- **Read before editing.** When editing `.astro` files directly, read the file first
  and stay inside the dialect grammar (`meno-astro-dialect.md`).
- **Build pages via the Studio API** in the node model above when scripting (the
  import flow); the provider writes correct `.astro` for you.
- See `components.md` for component shape (interface, props, slots, interactiveStyles)
  and `meno-astro-api.md` for the full Studio dev-server API surface.
