# Contributing

Thanks for your interest in `forge`. This repository holds the **documentation, design assets, and
website** for the Server Curio project family. Contributions are oriented toward clear, accessible
content and a maintainable site rather than expanding the repository's scope into application code.

## Before you start

- **Open an issue first** for anything beyond a typo or small copy fix. Discussing scope and
  approach saves rework.
- **Keep it in scope.** This repository is content and presentation. Application code, backend
  services, and persistence belong in the project repositories, not here.

## Local setup

```sh
git clone https://github.com/servercurio/forge.git
cd forge
hugo server -D    # once the site is scaffolded; serves at http://localhost:1313
```

The site is built with the **extended** edition of [Hugo](https://gohugo.io). See
[`.claude/build-commands.md`](.claude/build-commands.md) for the full command reference. This
repository has no build runner yet — drive Hugo with its CLI directly.

## Required local git hook

Every clone needs a per-clone `prepare-commit-msg` hook that auto-appends a DCO `Signed-off-by:`
line. The hook is **not** version-controlled; install it manually after cloning. Contents and
rationale live in [`.claude/git-hooks.md`](.claude/git-hooks.md).

```sh
# Verify it's in place
ls -l .git/hooks/prepare-commit-msg
```

If the hook is missing, copy the script from `.claude/git-hooks.md` and `chmod +x` it.

## Required commit signing

Every contributor commit must be **GPG-signed** in addition to carrying the DCO `Signed-off-by:`
trailer above, so the entire `main` history verifies.

Configure once per clone (or globally):

```sh
git config commit.gpgsign true
git config user.signingkey <YOUR-GPG-KEY-ID>
# Optional: also sign annotated tags
git config tag.gpgsign true
```

Verify a commit you just authored:

```sh
git log -1 --show-signature
# Look for "Good signature from ..." and a "G" in `git log --pretty='%G?'`
```

Don't bypass either requirement with `--no-gpg-sign` or `--no-verify`. If `gpg` prompts you for a
passphrase on every commit, configure `gpg-agent` rather than disabling signing.

## Commit-message conventions

This repo uses [Conventional Commits](https://www.conventionalcommits.org). The **PR Formatting**
workflow validates every pull-request title against this grammar. Common types for a
documentation/website repository:

| Type     | Example                                        |
|----------|------------------------------------------------|
| `docs`   | `docs: add getting-started guide`              |
| `feat`   | `feat: add project comparison page`            |
| `fix`    | `fix: correct broken link in footer`           |
| `style`  | `style: tighten heading spacing`               |
| `chore`  | `chore: bump theme submodule`                  |
| `ci`     | `ci: pin actions to release SHAs`              |

Subject line: imperative mood, no trailing period, ≤72 chars. The `prepare-commit-msg` hook appends
your DCO line automatically.

## Pull request workflow

1. Branch from `main`.
2. Build the site locally (`hugo --gc --minify`) and preview your change (`hugo server -D`) before
   pushing.
3. Open the PR. The **PR Formatting** workflow validates the title against Conventional Commits, and
   the DCO check confirms every commit is signed off.
4. Address review feedback in additional commits — don't force-push during review unless asked.
5. Squash on merge is fine; the squash subject should still be a valid Conventional Commit.

## Content & design conventions

See [`.claude/conventions.md`](.claude/conventions.md). Highlights:

- Author content as Markdown under `content/`; keep presentation in `layouts/`/`assets/`.
- Commit design **sources** (`.svg`, originals); generated output (`public/`, `resources/_gen/`) is
  gitignored.
- `docs/logo.svg` is the canonical brand mark — reference it rather than forking copies.
- Accessibility is required: meaningful `alt` text, `<title>` on content SVGs, WCAG AA contrast.

## Reporting security issues

Please **don't** file public issues for vulnerabilities. See [`SECURITY.md`](SECURITY.md) for the
private reporting flow.
