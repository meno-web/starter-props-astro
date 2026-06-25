---
description: Second foundation step for the mirror→scalable conversion. Carves the faithful, flat mirror pages (src/pages/*.astro) into a shared Layout + Header + Footer + per-section components, reusing the chrome across every page. Operates on the in-memory node tree via the Studio split endpoints — format-transparent, deterministic. Styling stays on the page's own _mirror CSS for now (de-mirroring is a later per-component step); this step is purely structural.
allowed-tools: Bash, Read
argument-hint: "[slug] [--port=N]"
---

# /componentize-mirror $ARGUMENTS

A mirror-imported site is one big flat page per route — every element a faithful
node, the chrome (nav/footer) duplicated in full on every page. This skill is the
**second foundation step** (after `/tokens-from-mirror`): it carves those flat
pages into a **shared** `Layout` + `Header` + `Footer` and one zero-prop component
per section, then rewrites each page as a thin Layout shell of section refs. The
server does the carve deterministically (single-child unwrap, `<main>` splice,
nav/footer detection, archetype naming, chrome build-or-reuse); this skill resolves
the port, picks the page order, guards against degenerate carves, and batches.

It changes **structure only** — classes and `_mirror` CSS are untouched, so the
render stays pixel-identical. Converting a component's opaque classes into
Meno-native editable styles ("de-mirror") is the **next** step, done per component.

> **Non-interactive contract.** Never ask the user mid-run — see
> `.claude/docs/meno/studio-port.md`. Validation halts with a one-line error;
> uncertainty (e.g. a page whose carve looks degenerate) becomes a line in the
> report and that page is skipped, not guessed.

> **Format-transparent.** The split endpoints operate on the in-memory node tree
> and the provider emits `.astro` underneath — you pass a route slug, not a file
> path.

## What to do

1. **Parse `$ARGUMENTS`.** Optional first token = a single page slug to carve just
   that page (e.g. `about`, or `integration/foo` for a nested route). Omitted =
   carve the whole site. Optional `--port=N` overrides port detection (drop it if
   not an integer 1024–65535; don't halt).

2. **Resolve the Studio port** per `.claude/docs/meno/studio-port.md` — the
   **editor** server (3000-range), NOT the 8080-range SSR preview. Substitute
   `<STUDIO_PORT>` below. If port resolution fails after all steps, halt with the
   one-line error from that doc. Do NOT ask the user.

3. **Enumerate the pages, homepage first.** Chrome (`Layout`/`Header`/`Footer`) is
   built on the **first** page that needs it and reused thereafter, so the homepage
   must go first to seed canonical chrome:

   ```bash
   # All mirror pages → route slugs (strip src/pages/ and .astro; index → home root)
   find src/pages -name '*.astro' | sed 's#^src/pages/##; s#\.astro$##' | sort
   ```

   Order: the `index` route first, then the rest in any order. If a single slug was
   passed in step 1, process only that one (it must already have chrome, or it
   becomes the chrome seed).

4. **Per page — inspect before carving (the mirror guard).** Faithful Webflow/
   builder trees are deeply nested and occasionally don't carve cleanly. Look at the
   outline first and only carve when it's healthy:

   ```bash
   curl -s "http://localhost:<STUDIO_PORT>/api/page-outline?slug=<slug>" | jq '{children: (.children|length), nav: .suggested.navIndex, footer: .suggested.footerIndex, names: .suggested.sectionNames}'
   ```

   - **Healthy** — roughly 2–40 section candidates, with a plausible nav/footer
     guess → proceed to carve.
   - **Degenerate** — `0`/`1` candidate (the whole body collapsed to one node) or a
     runaway count (e.g. >60, meaning the locator never found a section layer and is
     listing leaf nodes) → **skip this page**, record it in the report with the
     count, and move on. Do NOT carve a degenerate outline; it produces junk
     components that are worse than the flat page.
   - Already a thin Layout shell (`wrappedInLayout: true`) → skip (idempotent
     re-run); record as "already split".

5. **Carve the healthy page** in one shot:

   ```bash
   curl -s -X POST http://localhost:<STUDIO_PORT>/api/auto-split \
     -H 'content-type: application/json' -d '{"slug":"<slug>"}'
   ```

   The server builds `Layout`/`Header`/`Footer` only if not already in
   `src/components/` (so they're shared across all pages), creates one zero-prop
   section component per candidate, and rewrites the page as a thin shell. The
   response echoes the resolved plan + per-section result. On 4xx/5xx, surface
   `error`/`message` verbatim, record it for that page, and continue with the next
   page (one bad page must not abort the batch).

6. **Report** a per-page table and stop. Do not start de-mirroring, interactivity,
   or CMS — those are later skills.

   ```
   /componentize-mirror — 9 pages

   ✅ index           → shell · Layout built · Header built · Footer built · sections: Hero, LogoStrip, Features, CTA (4)
   ✅ about-us        → shell · chrome reused · sections: PageHero, Team, Values (3)
   ✅ careers         → shell · chrome reused · sections: Hero, Openings (2)
   ⏭️  integrations    → skipped (outline degenerate: 1 candidate)
   ⏭️  index           → already a Layout shell
   ❌ pricing         → 422 no section candidates

   Chrome: Layout · Header · Footer  (in src/components/, shared by all shells)
   Next: /verify against the mirror (screenshot diff — the _mirror render is ground truth),
         then de-mirror styles per component.
   ```

## Notes & guardrails

- **Why homepage-first:** `/api/auto-split` builds chrome only when it's missing,
  so whichever page runs first defines the shared `Header`/`Footer`. The homepage
  has the canonical nav/footer; seeding from a deep sub-page risks a stunted header.
- **The mirror render is ground truth.** Componentizing is pure structure, so the
  render must stay identical. After this skill, screenshot-compare each carved page
  against its pre-split mirror (the `public/_mirror` assets render offline) before
  trusting the carve — a drifted render means a section boundary cut through a
  positioned/overflow container.
- **Nested routes** (`integration/foo`) are valid slugs here — pass the full
  route-relative path. Do not strip the folder.
- **Do NOT hand-edit the carve.** Section refinement (merging/splitting sections,
  renaming) is a follow-up using `/split-node` and `/convert-children-to-components`
  on the already-carved page, not this skill.

## What you do NOT do

- Do NOT convert classes to Meno-native styles or remove `_mirror` CSS — structure
  only; de-mirroring is the next, per-component step.
- Do NOT touch `colors.json` / `variables.json` (that's `/tokens-from-mirror`),
  `images/`, or anything under `public/_mirror/`.
- Do NOT carve a degenerate outline, and do NOT ask the user what to do about one —
  skip it and report it.
- Do NOT add interactivity or CMS — later skills.

Re-running `/componentize-mirror` is safe: chrome is reused, already-split pages
report "already a Layout shell" and are skipped.
