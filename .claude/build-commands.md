<!--
  ~ SPDX-License-Identifier: Apache-2.0
-->

# Key Build Commands

This repository is intentionally minimal. `Taskfile.yaml` holds only repository checks; once the Hugo
site is scaffolded, drive it with the Hugo CLI directly, and add site tasks only if the workflow grows
enough to warrant them.

- `task lint:license` — Check SPDX license headers. Installs license-eye with `go install` at
  `LICENSE_EYE_VERSION`, pinned only in `Taskfile.yaml` (requires Go). CI runs this task.
- `task license:fix` — Add missing SPDX license headers.

- `hugo server -D` — Run the local dev server with drafts enabled; live-reloads on change
  (default <http://localhost:1313>).
- `hugo` — Build the production site into `public/` (gitignored).
- `hugo --minify` — Production build with minified HTML/CSS/JS output.
- `hugo --gc --minify` — Production build that also runs cache garbage collection; use this in CI.
- `hugo new content <section>/<name>.md` — Scaffold a new content page from the archetype.
- `hugo version` — Print the Hugo version; confirm the **extended** edition when the theme uses
  SCSS.
- `hugo mod get -u` / `hugo mod tidy` — Update and prune Hugo module dependencies (if the site uses
  Hugo Modules for its theme).

If the theme has a Node-based asset pipeline (PostCSS / Tailwind), install with `npm ci` and run
the theme's `npm run` scripts; Hugo invokes them via its asset pipeline during `hugo`.
