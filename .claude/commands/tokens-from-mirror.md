---
description: Foundation step for the mirror→scalable conversion. Derives design tokens — brand colors (colors.json) and typography/layout variables (variables.json) — from a mirror-imported site's LOCAL copied CSS (public/_mirror/css) and fonts/. No live fetch, no Studio server, no headless browser. Idempotent: only ADDS missing tokens, never overwrites keys you set by hand, never touches pages/components/images.
allowed-tools: Bash, Read, Write, Edit
argument-hint: "[--theme=light]"
---

# /tokens-from-mirror $ARGUMENTS

A site imported with the **mirror** importer renders via its own verbatim copied
CSS (`public/_mirror/css/*.css`), with class names kept as plain attributes — it
looks pixel-perfect but its design system is locked inside opaque Webflow/builder
CSS. This skill is the **first foundation step** of converting that faithful
mirror into a scalable Meno project: it lifts the site's **colors** and
**typography/layout scale** into Meno's token files (`colors.json`,
`variables.json`) so the project gains a real, globally-editable design system.

It reads the mirror **entirely offline** — no URL, no sidecar, no Studio port. It
writes only the two token files and leaves pages, components, images, and the
`_mirror/` CSS untouched (de-mirroring of individual components happens later, in
the per-component style migration step).

> **Non-interactive contract.** Never ask the user mid-run. Halt with a one-line
> error on an unrecoverable problem; never call AskUserQuestion. Uncertainty
> becomes a line in the final report, not a question.

## Preconditions

This skill operates on an already-mirrored project. Verify, and halt with a
one-line error if missing:

- `public/_mirror/css/` exists and contains at least one `*.css` file.
- `.meno/mirror-manifest.json` exists (proves this is a mirror import, not the
  legacy lossy import).

If there is no mirror yet, halt: `No mirror found — run the mirror importer first (public/_mirror/css is empty).`

## What to do

1. **Parse `$ARGUMENTS`.** Optional `--theme=<name>` selects which `colors.json`
   theme to write into (default `light`). Any other argument is ignored. Never halt on a bad flag — drop it and continue.

2. **Inventory the mirror.** One read-only batch:

   ```bash
   ls -la public/_mirror/css public/_mirror/fonts 2>/dev/null
   # Font families actually shipped (filenames carry the family + weight):
   ls public/_mirror/fonts 2>/dev/null
   # @font-face declarations (the authoritative family list):
   grep -ho 'font-family:[^;}]*' public/_mirror/css/*.css | sort | uniq -c | sort -rn | head -30
   ```

3. **Detect the design system — variables first.** Modern Webflow/Framer mirrors
   ship their design system as **named CSS custom properties** — that IS the token
   set, already structured; lift it directly. Only older class-only sites need raw
   aggregation. Check which case you're in:

   ```bash
   # Every custom-property definition (sorted, deduped). A named system shows up as
   # families like --typography--*, --color--*, --spacings--*, --main--*, --brand--*.
   grep -hoE '\-\-[a-z0-9-]+:[^;}{]*' public/_mirror/css/*.css | sort -u | head -120
   ```

   - **A named `--*` system is present (the common case)** → do 3a.
   - **No meaningful custom properties** → do 3b (fallback).

3a. **Lift the existing CSS variables (preferred).** These already are the design
   system — map them onto Meno's token files, keeping the source values:
   - **Resolve alias chains.** Semantic tokens often point at a base palette
     (`--color--text: var(--brand--brand-900)`, `--brand--brand-900: #14193d`).
     Follow the chain to the literal value before writing it.
   - **Capture responsive pairs into `scales`.** A token defined twice — once in
     `:root` and once inside an `@media` — is responsive (e.g. `--typography--h1`
     is `5rem` base, `3rem` on mobile). Write the base as `value` and the
     breakpoint override(s) into the CSSVariable `scales` map (keyed by the
     breakpoint name from `project.config.json`), with `type: "none"` so Meno's
     auto-scaling doesn't double-apply on top of the authored values.
   - **Map by family:**
     - `--typography--h1..h6`, `--typography--text*` → font-size vars (`type: "fontSize"` only if you want auto-scaling instead of authored `scales`; otherwise `type: "none"`), `group: "font-size"`.
     - `--main--*-font` / `--*--*-font` → font-family vars, `type: "none"`, `group: "font-family"`.
     - `--main--*-weight` → font-weight vars, `type: "none"`, `group: "font-weight"`.
     - `--spacings--*` → `type: "padding"`/`"margin"`/`"gap"` (or `"none"` to keep authored), `group: "gap"`/`"padding"`.
     - `--main-size` / container widths → `type: "size"`, `group: "size"`.
     - `--color--*` (resolved) → `colors.json` theme colors, NOT variables.json.
   - **Keep the source variable names** when they're already clear
     (`--typography--h1`, `--spacings--l`) — re-pointing every reference to a renamed
     token is out of scope for this step. Only normalize names if the source ones
     are opaque hashes.

3b. **Aggregate raw declarations (fallback, class-only sites).** No variable system —
   derive from the most-frequent declarations:

   ```bash
   grep -hoE 'font-family:[^;}]*' public/_mirror/css/*.css | sort | uniq -c | sort -rn | head -8
   grep -hoE 'font-size:[^;}]*'   public/_mirror/css/*.css | sort | uniq -c | sort -rn | head -20
   grep -hoE 'font-weight:[^;}]*' public/_mirror/css/*.css | sort | uniq -c | sort -rn | head -10
   grep -hoE '#[0-9a-fA-F]{3,8}\b|rgba?\([^)]*\)' public/_mirror/css/*.css | tr 'A-F' 'a-f' | sort | uniq -c | sort -rn | head -30
   ```

   Materialize with canonical Meno names (only the levels the CSS surfaced — never
   fabricate): families → `--font-family-sans/serif/mono` (`none`/`font-family`);
   heading sizes → `--font-heading-xl/lg/md/sm` (`fontSize`/`font-size`); body →
   `--font-body-lg/md/sm`; line heights → `--line-height-tight/snug/normal/relaxed`
   (`none`/`line-height`); weights → `--font-weight-regular/medium/semibold/bold`
   (`none`/`font-weight`).

4. **Palette → colors.json.** From the resolved `--color--*` (3a) or the
   frequency-ranked literals (3b), map the named ones to the chosen theme: `text`,
   `bg`, `muted`, `border`, plus brand tokens you can name (`primary`, `accent`,
   `surface`). Normalize to 6-digit hex where lossless; keep `rgba()` with alpha
   as-is. Leave ambiguous one-offs alone (report as skipped).

6. **Merge — never clobber.** Both files may carry hand-added keys. Read each with
   the Read tool, mutate the in-memory object, Write it back. **Existing values
   win** — only ADD missing tokens; do not overwrite a token the user already set.
   Track added vs preserved for the report.

7. **Report and stop** (see format below). Do not kick off componentization or any
   follow-up — that's the next skill.

## Exact schemas

`colors.json` — themes map, named colors are plain keys:

```json
{
  "default": "light",
  "themes": {
    "light": {
      "label": "Light",
      "colors": { "text": "#1f2937", "bg": "#ffffff", "muted": "#6b7280", "border": "#e5e7eb", "primary": "#4f46e5" }
    }
  }
}
```

`variables.json` — `{ "variables": CSSVariable[] }`, each:

```json
{
  "name": "Heading XL",        // display name
  "cssVar": "--font-heading-xl", // CSS custom property
  "value": "3.5rem",            // base value
  "type": "fontSize",           // responsive-scaling category: fontSize|padding|margin|gap|borderRadius|size|none
  "group": "font-size"          // UI picker group: font-family|font-size|font-weight|line-height|margin|padding|gap|size|border-radius|...
}
```

`type` controls responsive scaling (per `project.config.json` → `responsiveScales`); use `"none"` for families / weights / line-heights / one-off values that must NOT auto-scale. `group` only affects which picker the variable shows up in.

## Report

```
/tokens-from-mirror  (theme: light)

🎨 colors.json
    + primary       #4f46e5
    + text          #1f2937
    = bg            (already present, preserved)
📝 variables.json
    + --font-family-sans   "Manrope, sans-serif"   none / font-family
    + --font-heading-xl    "3.5rem"                 fontSize / font-size
    = --font-weight-bold   (already present, preserved)

📊 Added: 11   Preserved: 3   Skipped: 4 (ambiguous colors)
Next: /componentize-mirror <slug>  — carve the mirror pages into Layout + sections.
```

Use `+` added, `=` present-and-kept, `~` only if the user explicitly asked you to overwrite.

## Edge cases

- **Mixed system** — a site may define some tokens as `--*` variables and hardcode
  others. Lift the variables (3a) and backfill only the missing slots from raw
  aggregation (3b); never duplicate a token that already has a variable.
- **Alias-only variables** — if `--color--text` resolves through two or three
  `var()` hops to a literal, write the literal. If a chain dead-ends with no
  literal, skip it and report it.
- **Minified single-line CSS** — never `cat` it; always grep for the declaration
  you want. The greps above handle minified files.
- **Greyscale / no brand color** — write `text`/`bg`/`muted`/`border` only and say
  so in the report.
- **Multiple visual themes on the source** (light/dark) — only populate the
  `--theme` you were given; note the other exists.

## What you do NOT do

- Do NOT touch `src/pages/`, `src/components/`, `src/content/`, `images/`, or
  anything under `public/_mirror/` — this step is tokens only.
- Do NOT delete or rewrite the mirror CSS. De-mirroring styles is a later,
  per-component step; tokens coexist with the mirror CSS until then.
- Do NOT fetch the live URL or start the Studio/sidecar — everything is read from
  the local mirror.
- Do NOT overwrite whole files or hand-added keys.

Re-running `/tokens-from-mirror` is safe and idempotent — it always preserves keys
you've added or changed by hand.
