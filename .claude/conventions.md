# Conventions

- **Content lives in Markdown.** Author pages as Markdown under `content/` with front matter.
  Keep presentation concerns (layout, styling) in `layouts/` and `assets/`, not inline in content.
- **Design assets are source-controlled, generated output is not.** Commit `.svg` sources and
  original design files; the rendered site (`public/`, `resources/_gen/`) is gitignored and
  produced by `hugo`.
- **One source of truth for the logo.** `docs/images/logo.svg` is the canonical brand mark. The site
  references it (or a copy under `assets/`/`static/`); don't fork divergent copies.
- **Prefer relative links** between content pages so the site builds correctly under any base URL.
- **Accessibility is not optional.** Every image carries meaningful `alt` text; SVGs used as
  content carry a `<title>`; color choices meet WCAG AA contrast.
- **Sign every commit, in two ways.** Contributor commits must be (1) GPG-signed — set
  `git config commit.gpgsign true` (and `user.signingkey <KEY-ID>`) so `%G?` shows `G` on
  `git log --show-signature` — and (2) carry a DCO `Signed-off-by:` trailer, which the per-clone
  `prepare-commit-msg` hook documented in `.claude/git-hooks.md` auto-appends. Both apply to
  merge / squash subjects too. Don't bypass either with `--no-gpg-sign` or `--no-verify`.
- **Conventional commit subjects.** PR titles are validated by the PR Formatting workflow
  (`.github/workflows/200-flow-pull-request-formatting.yaml`); use conventional-commit prefixes
  (`docs:`, `feat:`, `fix:`, `chore:`, `style:`, `ci:`).
