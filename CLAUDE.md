<!-- MENO_DOCS_VERSION: 0.1.7 -->
# Meno — Visual CMS for Astro

This is an **Astro project** — standard `src/pages` and `src/components` `.astro` files plus Astro
content collections — authored with **Meno, a visual CMS / page builder for Astro**.

Meno reads and writes these `.astro` files in a constrained, fully round-trippable convention — the
**Meno format** (the spec calls it the *meno-astro dialect*): a small, well-defined subset of Astro that
the editor can parse back into its visual model. The result is a normal Astro codebase you can read,
hand-edit, and version — with a visual editor layered on top.

You can edit this project two ways, and they stay in sync:
- **Visually**, in the Meno editor (the primary surface).
- **By hand**, editing the `.astro` files directly.

The `meno-astro` codec (`emit` / `parse`) bridges the two and guarantees a lossless round-trip — so
**anything you hand-write must stay inside the Meno-format grammar below**, or it won't round-trip
(it'll be lost on the next visual save).

> **Read this first, then keep `.claude/docs/meno/meno-astro-dialect.md` open for the full grammar,
> and use the `/meno-astro` skill when authoring or editing `.astro` by hand.**

---

## Project layout

```
project.config.json        — project config (baseComponent, locales, format: "astro")
colors.json                — theme colors (var(--text), var(--bg), …)
variables.json             — CSS design tokens
src/
  pages/<slug>.astro       — pages  (the route tree)
  pages/<collection>/[slug].astro — CMS template pages (dynamic routes; schema in meta.cms)
  components/<Name>.astro  — reusable components
  content/<collection>/    — CMS items (JSON entries; .draft.json = unpublished)
  content.config.ts        — Astro content-collection config (build-readiness)
images/  fonts/  icons/    — assets, referenced with absolute paths (/images/hero.webp)
```

- A **page** is frontmatter (`const meta`) + the node tree wrapped in `<BaseLayout meta={meta}>`.
- A **component** is frontmatter (a single authoritative `resolveProps(Astro, {…})` prop block,
  optional `__meno` meta) + body + optional `<style>` / `<script>`.

---

## The golden rules (do / don't)

1. **Styles live in `style({...})`, never in a raw `class="..."`.**
   The argument is a Meno `StyleObject`: `{ base: {...}, tablet: {...}, mobile: {...} }` (or a flat
   object). Prop-bound values are `{ _mapping: true, prop: "name", values: {...} }`.
   - ✅ `<div class={style({ base: { display: "flex", gap: "12px" } })}>`
   - ❌ `<div class="flex gap-3">` — a raw Tailwind/CSS class string does **not** round-trip.

2. **i18n values live in `i18n({...})`** with the `{ _i18n: true, en, pl, … }` shape.
   - ✅ `<Heading text={i18n({ _i18n: true, en: "About", pl: "O nas" })} />`

3. **Meno templates are `{{expr}}` in the model → `{expr}` (or `` `…${expr}…` ``) in markup.**
   To put a template back by hand, write a JSX `{expr}` with a bare identifier / member / ternary; the
   parser turns it into `{{expr}}`. A whole-string template → bare expr; a mixed string → backtick
   literal.
   - ✅ `<span>{item.title}</span>` ⟶ model `"{{item.title}}"`

4. **Component props are JSX attributes.**
   - string → `text="Hi"` (or `text={"a \"quoted\" value"}` if it has quotes/newlines)
   - number → `size={1}` · boolean → `isMarginTop={true}`
   - object / link → `link={{ href: "/x", target: "_blank" }}`
   - i18n → `text={i18n({ _i18n: true, … })}`
   Component tags are **Capitalized** and need a matching local import in the frontmatter
   (`import Name from '../components/Name.astro'` for pages, `'./Name.astro'` for components).

5. **The `resolveProps(Astro, {…})` argument is authoritative for component props.**
   There is no separate `interface Props`/`__meno_props`: a component declares its props exactly once
   as `const { …names…, class: className } = resolveProps(Astro, {...})`. The `{...}` literal is
   authoritative (it's what the parser reads); the destructured names + their TS types are
   inferred/regenerated on save. If you change a prop, change that literal. Keep `class: className`
   in the destructure, and emit the call even for an empty interface
   (`const { class: className } = resolveProps(Astro, {});`). Component metadata (`category`,
   `acceptsStyles`, `libraries`) goes in `const __meno = {...}`
   (omit if empty). A component's JS that needs its props uses native Astro
   `<script define:vars={{ a, b }}>` (`true` = all props / `string[]` = a subset); otherwise
   a plain `<script is:inline>`. `defineVars` is **not** a `__meno` key.

6. **Conditionals are `{cond && ( … )}`.** A node's `if: "{{visible}}"` → `{visible && ( … )}`;
   `if: false` → `{false && ( … )}`; a `BooleanMapping` → `{when({...}) && ( … )}`.

7. **Lists:**
   - prop list → `{ list(items, { limit: 6 }).map((item, itemIndex) => ( … )) }` (loop var defaults to
     `item`; index is always `<var>Index`).
   - collection list → a frontmatter `const xList = await getCollectionList("blog", { … }, Astro)` then
     `{ xList.map((blog, blogIndex) => ( … )) }` (loop var defaults to `singularize(source)`).
   - ⚠ If you author a collection list, make the loop variable match the templates in the body
     (`(blog, blogIndex)` + `{{blog.title}}`). Set `itemAs` if you want a specific name.

8. **Other node forms (use the exact tags):**
   - `<Link href="/x">…</Link>` (link); `href={i18n({...})}` or `href={href({...})}` for i18n / mapping hrefs.
   - `<Embed html={`<svg>…</svg>`} />` (single-line) or hoist multi-line HTML to a frontmatter
     `const __embedN = \`…\`` and use `html={__embedN}`. `__embedN` is the **canonical** hoist
     name (0-based + sequential: `__embed0`, `__embed1`, …). A hand-authored hoist under a
     different name (`const __iconChat = …`) still round-trips — the HTML is recovered — but
     emit renames the const to `__embedN` on save, so don't rely on a semantic name surviving.
   - `<slot />` / `<slot>fallback</slot>`.
   - `<LocaleList … />` (locale switcher; style sub-props wrapped in `style(...)`, editor meta in a
     single `meta={{...}}`).
   - Dynamic tag (`h{{size}}`) → frontmatter `const Tag_0 = \`h${size}\`` + `<Tag_0>…</Tag_0>`.
   - **Optimized image** — a plain `<img data-meno-optimize="true" src=… alt=… width=… height=… />`
     emits the runtime `<MenoImage>` wrapper (Astro `astro:assets` `<Image>`). It stays a normal
     `img` node — the marker attr is the only difference. A **remote** `src` is only actually
     optimized if its host is allow-listed in `project.config.json` `image.domains` (below).
   - **Markdown** — `<Markdown source={\`# Title\n\nbody\`} />` (multi-line hoists to a frontmatter
     `const __mdN = \`…\``). The `source` is **verbatim and NEVER template-resolved** — a literal
     `{{x}}` or `${x}` stays literal, so don't use `{{templates}}` inside it.
   - **Island** (BYO framework) — `<Counter client:visible … />` with an
     `import Counter from '../islands/Counter.tsx'`. Drop the React/Preact/Vue/Svelte file in
     `src/islands/` (and its deps in `package.json`); `meno()` auto-wires the renderer —
     **never import a renderer (`@astrojs/react`/`react()`) or any non-`meno-astro` package in
     `astro.config`, or the Meno preview won't open the project** (it allow-lists only
     `astro/config` + `meno-astro`). Put the `client:*` directive on the tag (bare for
     `load`/`idle`/`visible`, valued for `media`/`only`; omit for a server-only, zero-JS island).
   - **Custom component** (opaque foreign `.astro`) — `<Fancy label="Hi">…</Fancy>` with an
     `import Fancy from '../custom/Fancy.astro'`. Author the full-power `.astro` file under
     `src/custom/` (any frontmatter, imports, helpers); Meno never models its internals — it's a
     **server-rendered black box**. Pass **explicit props only** (JSX attributes) + optional
     slotted children; an island's `client:*` does **not** apply (a custom component is
     server-only, zero JS). Reach for it only when a piece of UI can't be expressed in dialect and
     needs no client framework — see the escalation ladder below.
     - **Editor prop controls (islands + custom).** Meno can't model a foreign file's internals, so
       the PropsPanel discovers its props by reading the source — an island's `interface Props` /
       `defineProps` / `$props()` / `export let`, or a custom `.astro`'s frontmatter `interface
       Props` / `const { … } = Astro.props` — and renders one input per prop. The declared **type
       picks the control**: a **string-literal union** (`variant?: 'info' | 'warn' | 'success'`)
       becomes a **dropdown** of those values, `boolean`→toggle, `number`→number input, everything
       else→text box. So to give the editor a fixed set of choices for an island/custom prop, **type
       it as a union of string literals**. (A union with any non-literal member — `'a' | string`,
       `'sm' | 1` — stays free-text; and a `{{binding}}` or other off-list value falls back to the
       text input so it's never stranded.)

9. **When the dialect can't express it, escalate — don't give up.** Prefer the dialect, but Meno
   has two real escape hatches for things that genuinely can't be modeled (see the escalation
   ladder below): a **custom component** (`type:"custom"` — an opaque `.astro` under `src/custom/`,
   server-rendered, round-trips) and, for a whole bespoke route, a **hand-authored page** in
   `src/pages/`. The one thing still **not** preserved across a round-trip is a raw `class="…"`
   (static Tailwind/CSS) on a dialect node — always use `style({...})` for classes.

10. **Serialization is deterministic.** All literals (style / props / meta / i18n / list config) are
    printed with stable key order, JSON string escaping, and 80-col wrapping. Don't hand-tune
    formatting — a save re-emits canonically. Empties drop on normalization (empty `style`, empty
    `children`, empty `meta` / `interface`, and a lone array child collapses to a bare string).

---

## When Meno can't express it — the escalation ladder

Always reach for the **lowest** rung that works, and escalate only when the rung above genuinely
can't express what's needed. Never start at a custom file for something the dialect already models
— that throws away visual editing for nothing.

**1. Native Meno (the dialect) — the default.** Express the UI with dialect nodes (`node`,
`component`, `link`, `embed`, `list`, `slot`, `markdown`, …), `style({...})`, `i18n({...})`, and
`{{bindings}}`; factor anything repeated into a reusable `.astro` under `src/components/`. Fully
visual, the editor is the source of truth, lossless round-trip. **Stay here unless you hit a hard
wall.**

**2. Custom component** — `type:"custom"`, an opaque foreign `.astro` under `src/custom/`. Use when
a *piece of a page* can't be expressed in the dialect: a third-party Astro component, arbitrary
server-side `.astro`/JS markup, a complex widget. Author the file with full Astro power (any
frontmatter, `import`s, `getCollection`, helper functions) and reference it as a `custom` node:

```astro
---
import PriceTable from '../custom/PriceTable.astro';
---
<PriceTable plan="pro" seats={5}>
  <p>This child renders server-side into the component's default slot.</p>
</PriceTable>
```

Meno **places** it, passes the **explicit props** you set (as JSX attributes), and slots children —
but treats its internals as a **black box** (the editor canvas shows a placeholder; it round-trips
intact). No `style()`, no instance-style merge, no cms/loop/ambient prop forwarding — only the props
you write are passed. **Server-rendered, zero client JS.**

> **Custom vs island — don't confuse them.** A **custom** component is a *server-only* native
> `.astro` file (`src/custom/`, no `client:*`). An **island** (rule 8) is a *client-hydrated
> framework* component (React/Preact/Vue/Svelte under `src/islands/`, carrying a `client:*`
> directive). Need browser interactivity from a framework → island. Need server-rendered markup the
> dialect can't model → custom.

**3. Entire custom page** — hand-author a complete `.astro` route in `src/pages/`. Use when even the
page *shell* can't be Meno: bespoke frontmatter logic, a user `getStaticPaths`/dynamic route, an
endpoint, a fully custom layout. Two outcomes, both of which build + deploy as normal Astro:

- **Dialect body + extra foreign frontmatter** (a stray `const`, a foreign `import`, a helper
  `function`) → the foreign frontmatter is captured as a verbatim `_frontmatter` passthrough block.
  The page **stays visually editable and round-trips** — you get dialect editing *plus* your
  hand-written setup code.
- **Fully non-dialect page** → Meno opens it **read-only**: it still lists and previews in the
  editor, but visual edits are disabled and a save is a no-op (a clobber-guard, so the editor never
  overwrites your hand-authored file). Keep editing it by hand.

---

## File skeletons

**Page** (`src/pages/<slug>.astro`):
```astro
---
import { i18n } from 'meno-astro';
import { BaseLayout } from 'meno-astro/components';
import Heading from '../components/Heading.astro';

const meta = {
  title: "About",
  description: "About this site",
  viewTransitions: true,                              // optional — see "Head, SEO & project config"
  sitemap: { priority: 0.8, changefreq: "weekly" }    // optional
};
---
<BaseLayout meta={meta}>
  <main>
    <Heading size={1} text={i18n({ _i18n: true, en: "About", pl: "O nas" })} />
  </main>
</BaseLayout>
```

> `meta` is a **plain `const meta = {…}`** — never `export const meta`, never `satisfies MenoPageMeta`,
> never `import type`. Those break the real `astro build`. New SEO fields just ride this same object.

**Component** (`src/components/<Name>.astro`):
```astro
---
import { resolveProps, style } from 'meno-astro';

const { text, class: className } = resolveProps(Astro, {
  text: { type: "string", default: "Heading" }
});

const __meno = { category: "ui" };
---
<h2 class={style({ base: { fontWeight: "500" } })}>{text}</h2>
```

---

## CMS (content collections)

- **Items** live as JSON under `src/content/<collection>/`. Each item file has a stable `_id` (matching
  its filename stem) and `_createdAt`. An unpublished edit is a sibling `<name>.draft.json`; the published
  file is the one without `.draft`.
- **Schemas** are defined by the collection's **template page** at `src/pages/<collection>/[slug].astro`
  — an idiomatic Astro dynamic route (with `getStaticPaths`) whose `meta.source === "cms"` carries the
  schema in `meta.cms`. That template is the source of truth for the collection's fields — **not**
  `src/content.config.ts` (which is generated with a permissive schema only so `astro build` can resolve
  the collection). The route directory comes from `meta.cms.urlPattern` (`/blog/{{slug}}` →
  `src/pages/blog/[slug].astro`). In the Meno editor the template is addressed as `/templates/<collection>`.
  The `import { getCollection }`, `getStaticPaths()`, and `const { cms } = Astro.props;` lines are derived
  boilerplate (regenerated from `meta.cms`; the codec skips them) — edit `meta.cms`, not those.
- A list backed by a collection is a frontmatter `getCollectionList("<collection>", { … }, Astro)` const
  mapped in the body (see rule 7).
- **Rendering fields:** plain fields render as `{i18n(cms.field)}`; **rich-text** fields render via
  `<Fragment set:html={richTextWithComponents(cms.field, cmsComponents)} />` — never a text
  interpolation (a plain `{i18n(cms.richField)}` would print `[object Object]` since a rich-text
  value is a structured object, not a string). `richTextWithComponents` (from `meno-astro`) also
  renders **components embedded in the rich text** (TipTap `menoComponent` nodes) against
  `cmsComponents` — the generated registry `src/cmsComponents.ts` (a constant `import.meta.glob`
  over `src/components/`; don't hand-edit it). Its import
  (`import { cmsComponents } from '../../cmsComponents'`) is derived boilerplate like
  `getStaticPaths`. An **embed node** bound to a rich-text field uses
  `<Embed html={i18n(cms.field)} />`.

When adding or changing a collection, edit the template `[slug].astro`'s `meta.cms` schema and keep the
item files in `src/content/<collection>/` consistent with it.

---

## Head, SEO & project config

These are real, shipped Astro features the `BaseLayout` and `meno()` integration wire up. Two homes:
**page `meta`** (per-page, in the page's plain `const meta`) and **`project.config.json`** (project-wide).

### Page `meta` fields (per page)

| Field | Shape | Effect |
|---|---|---|
| `viewTransitions` | `true` | Renders Astro's `<ClientRouter>` → SPA-style view-transition navigation. **Off by default.** |
| `noindex` | `true` | Emits `<meta name="robots" content="noindex">`. |
| `sitemap` | `{ priority?: 0..1, changefreq?: "always"\|"hourly"\|"daily"\|"weekly"\|"monthly"\|"yearly"\|"never", exclude?: true }` | Per-page `sitemap.xml` annotation. `exclude: true` drops the page (and every locale variant) from the sitemap. Invalid values are silently ignored. |
| `customCode` | `{ head?, bodyStart?, bodyEnd? }` | Raw HTML injected into `<head>` / right after `<body>` / before `</body>`. Merged **after** the project-wide `customCode`. |

```astro
const meta = {
  title: "Pricing",
  viewTransitions: true,
  noindex: true,
  sitemap: { priority: 1.0, changefreq: "daily" },
  customCode: { head: '<meta property="og:type" content="website" />' }
};
```

### `project.config.json` fields (project-wide)

| Key | Shape | Effect / gotcha |
|---|---|---|
| `customCode` | `{ head?, bodyStart?, bodyEnd? }` | Project-wide raw-HTML injection (merged **before** each page's `meta.customCode`). |
| `icons` | `{ favicon?, faviconDark?, appleTouchIcon? }` | Favicon `<link>`s. With both `favicon` + `faviconDark`, BaseLayout scopes them by `prefers-color-scheme`. Use absolute hrefs (`/icons/favicon.svg`). |
| `redirects` | `[{ from, to, status? }]` | Astro `redirects`. A `301` is the bare form; any other `status` uses Astro's object form. |
| `image` | `{ domains: string[] }` | Allow-list of remote hosts the optimizing `<MenoImage>` may process. **A remote `data-meno-optimize` image silently passes through (no optimization) unless its host is listed here.** |
| `prefetch` | `{ enabled: true, defaultStrategy: "hover"\|"tap"\|"viewport"\|"load" }` | Astro native link prefetch. Only applied when `enabled === true`. **Suppressed in the play/preview by design** (it would flood the single dev server) — it takes effect in real builds. |
| `devToolbar` | `true` | Shows Astro's dev toolbar in the play preview. **Off by default.** |
| `output` | `"server"` | Opts the project into **SSR** (on-demand rendering). Default is static. Pair it with `adapter`. |
| `adapter` | `{ name: "node"\|"cloudflare"\|"netlify"\|"vercel", mode?: "standalone"\|"middleware" }` | The SSR adapter (only with `output: "server"`). `meno()` registers `@astrojs/<name>` for you. **This is the ONLY way to add an adapter — NEVER `import '@astrojs/node'` (or any adapter) in `astro.config.mjs`; the preview allow-lists only `astro/config` + `meno-astro`, so a config adapter-import fails with _"...imports \"@astrojs/node\", which the shared Astro preview runtime doesn't include."_** |

> Changing a `project.config.json` field above (or i18n locales) is **frozen at the dev server's
> `config:setup`**, so the Meno play server auto-restarts to apply it; per-render fields (SEO, custom
> code, favicons) apply live. If a change doesn't show, use the editor's **Restart server** menu action.

---

## Selection

Read `.meno/selection.json` for the element currently selected in the editor. It's a **bare JSON array**
tracing the editing hierarchy from the outermost page through every component drill-in down to the
selected node — each entry is `<file>:<lineStart>-<lineEnd>` pointing into the real `.astro` source, e.g.
`["src/pages/index.astro:6-14", "src/components/section/Hero.astro:40-58", "src/components/ui/Button.astro:22-29"]`.
The **last** entry is the selected node; earlier entries are the drill-in instances (the next entry's
filename names that component, and the line range disambiguates which instance was entered when a page
uses several of the same component). Open the file at the given lines to see the element's tag,
`class={style(…)}`, props, and children in dialect source. The file contains `null` when nothing is
selected.

---

## Status & caveats (important)

This format is **new but functional end to end**. Be honest about what works today:

- ✅ **Editing & round-trip work.** Pages/components read, save, and round-trip through `meno-astro`'s
  `emit`/`parse`. Visual edits and hand-edits stay in sync as long as you stay in the grammar.
- ✅ **`astro build` works.** The runtime helpers the emitted markup imports (`style()`, `i18n()`,
  `href()`, `when()`, `list()`, `getCollectionList()`, `embedHtml()`, `richTextWithComponents()`,
  `renderMarkdown()`) and the `meno-astro/components` (`BaseLayout`, `Link`, `Embed`, `LocaleList`,
  `MenoImage`, `Markdown`) are **implemented and published** (`meno-astro` on npm). Emitted `.astro`
  runs under `astro dev`/`astro build` with the `meno()` integration (the converter scaffolds the
  config). i18n (locale routing, `meta.slugs`, hreflang), CMS content collections, client filtering,
  Astro islands, and the head/SEO/config features above are all wired and build-verified.
- ✅ **Verbatim JS *expressions* survive.** A `{expr}` the template engine can't evaluate (a
  function/method call) is preserved as `{ _code, expr }`, round-trips, and builds.
- ⚠ **Raw `class="…"` and arbitrary frontmatter do not round-trip yet.** Use `style({...})` for classes;
  don't hand-author ad-hoc Astro frontmatter logic expecting it to persist.

For the full grammar and the implemented/pending split, see:
- `.claude/docs/meno/meno-astro-dialect.md` — the dialect spec (grammar, normalization, round-trip contract).
- `.claude/docs/meno/meno-astro-api.md` — the `meno-astro` package API + status.
- the **`/meno-astro` skill** (`.claude/commands/meno-astro.md`) — copy-pasteable authoring cheat-sheet.
