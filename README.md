# The Sortis Vellum

A digital worldbuilding engine inspired by **Jerry's Map** — a randomized card deck that decides what your world does next, a coordinate-based panel generator, a living atlas that remembers what you've made, and one-click export to an Obsidian vault.

> The deck proposes the chaos. You author the meaning.

## What it is

A single self-contained web app (no build step, no framework). Everything — the card engine, the panel template generator (HTML canvas), the world-state registry, the vault note exporter — lives in `index.html`.

- **The deck** — draw cards that create panels, shape terrain, found settlements, birth heroes, trigger events, add art, and write lore. Cards can be created, edited, retired, and restored. "Meta" cards rewrite the deck itself.
- **Travel & sectors** — each draw flips a stay/travel toss (move *Steps* panels in the ink's direction) and rolls *Sectors* (1–80) on cards that create space.
- **Panel forge** — generates print/Procreate-ready 8×10 @300DPI panel templates with a 1-inch grid, N/S/E/W orientation, compass rose, and a coordinate stamped into the title block and filename.
- **The Atlas** — a registry of every panel, settlement (with population tiers Camp → Holdfast → Reach → Bastion), and hero (with d10 alignment, alive/fallen).
- **World Rules** — a tome of the world's laws (magic exists, the year is 242 days, etc.).
- **Vault capture** — almost every draw emits a ready-to-paste Markdown note with `[[wikilinks]]` and `#tags` for Obsidian.

## Run it

Just open `index.html` in a browser. No install, no server.

- Local file → saves to that browser via `localStorage`.
- Deployed (HTTPS) → same, plus optional cloud sync (below).

## Deploy (Vercel)

It's a static site. Push this repo to GitHub, import it into Vercel, and every push auto-deploys. `vercel.json` is included.

## Cloud sync (optional)

Off by default; the app is fully usable without it. To sync one world across devices with a simple login (Google or email magic link), set up Supabase and fill `config.js`. See **[docs/SUPABASE-SETUP.md](docs/SUPABASE-SETUP.md)**.

## Map viewer

Upload your finished panel art (Atlas → **Upload** on any panel), and it attaches to that panel at its current generation. The **Map** tab assembles the whole world by coordinate, with a generation slider to see it as it looked at any point in its history — click any panel for the full-res image. Requires sign-in (cloud sync); without it the map still renders panel placeholders. Full spec in **[docs/MAP-VIEWER.md](docs/MAP-VIEWER.md)**.

## Working on this project

- **Design & ideas** (new cards, mechanics, features): worked out conversationally in Claude chat.
- **Build, iterate, deploy**: handled in **Claude Code** against this repo (git push → Vercel redeploys).

See **[CLAUDE.md](CLAUDE.md)** for architecture, the world-state shape, and conventions — it briefs Claude Code on how to work in this codebase.

## Backups

Settings → **Full backup** exports the entire world as JSON. **Export cards** shares just the deck. **Export Chronicle** writes the year-by-year log as Markdown.
