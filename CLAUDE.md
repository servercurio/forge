# CLAUDE.md

Guidance for Claude Code working in this repository. This file covers project intent, conventions,
and procedural guidance that isn't captured in the agent reference docs:

- [`.claude/instructions.md`](.claude/instructions.md) — tech stack, personality, requirements
- [`.claude/build-commands.md`](.claude/build-commands.md) — Hugo commands and what they run
- [`.claude/module-structure.md`](.claude/module-structure.md) — directory-by-directory roles
- [`.claude/conventions.md`](.claude/conventions.md) — content and commit conventions for this repo
- [`.claude/git-hooks.md`](.claude/git-hooks.md) — required local git hooks (must be installed per clone)

@.claude/instructions.md
@.claude/build-commands.md
@.claude/module-structure.md
@.claude/conventions.md
@.claude/git-hooks.md

## What this project is

`forge` is the home for **project documentation, design assets, and the website** of the Server
Curio project family (the `go-*-starter` templates and related work). It is a content and
presentation repository built on [Hugo](https://gohugo.io) — not application code.

The repository is intentionally **minimal**: it currently ships only its configuration and meta
files (this file, the `.claude/` guidance, `.gitignore`, `CODEOWNERS`, the PR-formatting workflow,
`LICENSE`, `README.md`, and the brand mark at `docs/logo.svg`). The Hugo site is added on top of
this foundation.

When asked to add functionality, keep the repository focused on documentation, design, and the
site. Don't introduce application code, backend services, or persistence — those belong in the
project repositories, not here.

## When adding content

- **New page**: add a Markdown file under `content/` with appropriate front matter
  (`hugo new content <section>/<name>.md`). Keep presentation in `layouts/`/`assets/`, not inline.
- **New design asset**: commit the source (`.svg` preferred for marks and diagrams) under `docs/`
  or `assets/`. `docs/logo.svg` is the canonical brand mark — reference it rather than forking
  copies.
- **New CI workflow**: file under `.github/workflows/` following the numeric-prefix convention
  (200 = PR-triggered, 300 = main-branch push, 100 = operational/release, 800 = reusable).
- **Site tooling**: this repo has no build runner yet — drive Hugo with its CLI (see
  `.claude/build-commands.md`). Introduce a `Taskfile`/`Makefile` only if the workflow grows to
  need one.

## Things to leave alone unless asked

- Keep the repository minimal. Don't scaffold a full Hugo site, vendor a theme, or add tooling
  unless the task explicitly asks for it.
- `docs/logo.svg` is the shared Server Curio brand mark with the "Forge" wordmark. Don't restyle
  the mark itself; only the tagline text differs from sibling repositories.
