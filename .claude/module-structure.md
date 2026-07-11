# Repository Structure

This repo is currently **scaffolding only** — the config and meta files are present; the Hugo site
itself has not been created yet. The layout below is the intended target once content lands. Create
directories as they are first needed rather than adding empty placeholders.

## Present today

- `docs/` — Non-site documentation assets. The canonical brand mark lives at `images/logo.svg`. The README and
  the future site reference this logo.
- `.github/workflows/` — CI workflows. `200-flow-pull-request-formatting.yaml` validates PR titles
  against the conventional-commit grammar. Follow the numeric-prefix naming convention when adding
  workflows (200 = PR-triggered, 300 = main-branch push, 100 = operational/release, 800 = reusable).
- `.github/CODEOWNERS` — Review routing.
- `.claude/` — Agent guidance (this file and its siblings) plus `settings.json`.

## Intended Hugo layout (create as needed)

- `hugo.toml` (or `hugo.yaml`) — Site configuration: `baseURL`, title, params, menus.
- `content/` — Markdown pages and sections (the documentation and site copy).
- `layouts/` — Templates and partials that override or extend the theme.
- `assets/` — Pipeline-processed assets (SCSS, JS, images run through Hugo Pipes).
- `static/` — Files copied verbatim to the site root (favicons, robots.txt, `logo.svg`).
- `themes/` — Vendored or Hugo-Module theme(s).
- `archetypes/` — Front-matter templates for `hugo new content`.

## Generated (never committed)

- `public/`, `resources/_gen/`, `.hugo_build.lock`, `hugo_stats.json` — build output, gitignored.
