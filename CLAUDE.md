# CLAUDE.md — project brief for The Sortis Vellum

This file orients any Claude Code session working on this project. Read it first.

## What this is

A digital worldbuilding engine inspired by Jerry's Map. A single self-contained web app: a randomized **card deck** that drives creation, a **panel template generator**, a **world-state atlas**, a **World Rules** tome, and **Obsidian vault** Markdown export. The guiding idea: *the deck proposes the chaos, the player ("the explorer") authors the meaning.*

## Working philosophy (carry this into every change)

- **Playable before pretty.** Prefer a working prototype the user can try over upfront polish. Iterate from real play-testing.
- **One self-contained file.** The whole app is `index.html` — vanilla HTML/CSS/JS, **no build step, no framework, no bundler**. Keep it that way unless the user explicitly asks to introduce tooling. New features go inline.
- **Degrade gracefully.** Optional capabilities (cloud sync, the Supabase lib, `config.js`) must be guarded so the app still runs fully **local-only** if they're absent. Never let a missing key or blocked CDN break the core app.
- **The red/black ink, coordinates, and lore-naming are real mechanics**, not decoration — preserve their meaning.

## File map

- `index.html` — the entire app (UI + engine + canvas panel generator + vault export + cloud sync).
- `config.js` — cloud-sync config (`window.SORTIS_CONFIG`). Blank = local-only. Filled with Supabase URL + anon key to enable sync. Must be served (no build step), so it's committed; the anon key is public-safe (security is via RLS).
- `vercel.json` — static hosting config.
- `docs/SUPABASE-SETUP.md` — exact steps + SQL to stand up cloud sync.
- `docs/MAP-VIEWER.md` — **next feature to build**: panel image upload + a coordinate map viewer with a generation slider. Designed, not yet implemented. Read it before starting that work.
- `README.md` — human-facing overview.

## Architecture of `index.html`

Everything hangs off a single state object `S`, persisted and rendered.

```
S = {
  meta:    { world, year, draws },
  settings:{ travelOdds },              // % chance a draw triggers travel
  deck:    [ card, ... ],               // active cards
  archive: [ card, ... ],               // retired cards (restorable)
  panels:  [ { coord, terrain, founded, gen, order } ],   // ordered registry; travel walks this order
                                                          // (planned: + images:{ "<gen>": {path,thumbPath,...} } — see docs/MAP-VIEWER.md)
  currentPanel: "N0 E0",                // the panel you're working
  places:  [ { name, kind, panel, founded, tier, pop } ], // settlements/roads/monuments
  heroes:  [ { name, alignment, status, born, panel } ],  // alignment 1–10, status alive|fallen
  rules:   [ { text, year } ],          // World Rules tome
  log:     [ { year, card, color, number, advanced, notes[] } ], // Chronicle
  loreSeq: { "Yr1": n, ... }            // running lore counter per year
}
```

### Card schema
`{ id, title, category, number(=steps), color('black'|'red'), instruction, traits[], roll('d10'…), space(bool), smin, smax, make(bool), cap(''|capture-type override), status, drawn }`

- `category`: foundation | terrain | develop | art | discovery | hero | event | lore | meta | force
- `number` = **Steps** (how many panels you move when travelling); `color` = direction (black clockwise, red counter).
- `space` cards roll **Sectors** in `[smin, smax]` (panel is an 80-sector grid).
- `make:true` (the "New Law" card) shows an inline card-forge with randomized kind/steps/ink.
- `cap` overrides the category's default capture type (e.g. a lore-category card with `cap:'rule'`).

### Key conventions (must stay consistent across deck, forge, and vault)
- **Coordinates** are canonical: `N{a} E{b}` with zero-normalized direction (origin = `N0 E0`). Used as panel titles, `[[links]]`, and filenames.
- **Lore entries** auto-name `Yr{year}{coordNoSpace}-{n}`, where `n` is a single running counter per year across all panels (see `S.loreSeq`).
- **Population tiers**: Camp 400 → Holdfast 800 → Reach 1200 → Bastion 2400.
- **Capture → vault**: `buildNote(type)` returns `{ title, md, summary, commit? }`. `commitToWorld(commit)` writes the same record into `S` (panel/place/hero/rule). Note shapes are frontmatter + `# title` + `> summary line` + body.
- **Eligibility**: heroes/events/terrain/art/etc. require ≥1 panel to exist (`eligible(card)`); only foundation/meta surface in an empty world.

### Storage tiers (in `Store` + cloud module)
1. `window.storage` — used only inside Claude's artifact runtime.
2. `localStorage` — used when self-hosted / opened in a normal browser.
3. **Supabase** (`worlds` table, one JSONB row per user) — when `config.js` has keys **and** the user is signed in. Strategy: last-write-wins; pull on sign-in, debounced upsert on change.

Binary panel images (when the map viewer is built) go in **Supabase Storage** (`panels` bucket), not in the JSON/database — only their object paths are stored on `panel.images`. See `docs/MAP-VIEWER.md`.

### Conventions for edits
- New starter cards: add to `seedDeck()`. Existing users pull them via Settings → **Load new starter cards** (`syncStarterCards()` adds by title, non-destructively).
- After any change, sanity-check the script with:
  `sed -n '/<script>/,/<\/script>/p' index.html | sed '1d;$d' > /tmp/x.js && node --check /tmp/x.js`
- Keep new UI within the existing token system (CSS variables at `:root`) and the "survivor's almanac" aesthetic (parchment, red/black ink, verdigris, gold for heroes).

## Deploy

Static site. `git push` to the GitHub repo connected to Vercel → auto-deploy (~60–90s). No build command needed. For cloud sync, complete `docs/SUPABASE-SETUP.md` and fill `config.js`.

## The workflow this project uses

- **Creative/design** (new cards, mechanics, features) is worked out in Claude chat.
- **Implementation/deploy** happens here in Claude Code: apply the agreed changes to `index.html`, sanity-check, commit, push. Vercel redeploys.
