# kalaluthien.github.io — working rules for coding agents

This repository is the published Jekyll site. It is a build target, not a
place to write.

## Read this first
- `_config.yml` — the site's own settings. The theme is `jekyll-theme-chirpy`.
- `.github/workflows/pages-deploy.yml` — what GitHub Pages runs on a push to
  `main`.

## Never edit the writing here
`_posts/` and `_tabs/about.md` are copied in from the `notes` repository by its
publishing skill, `~/workspace/notes/.claude/skills/publishing-to-github-pages/`.
A post edited here is overwritten by the next sync, and the edit never reaches
the vault. Edit the note in `~/workspace/notes` and publish it from there. The
sync's own commits say `publish: sync ...`.

## What does belong here
The theme and its overrides (`_includes/`, `_plugins/`, `assets/`), the site
settings in `_config.yml`, the Ruby and gem pins, and the deploy workflow.

## Working conventions
- Ruby is pinned to the version in `.ruby-version`, for rbenv and for CI.
- Serve the site locally with `bash tools/run.sh`; it runs `bundle exec jekyll s -l`.
- Run `bash tools/test.sh` before a push. It builds the site and runs
  html-proofer over `_site`.
- `_site/`, `.jekyll-cache/` and `Gemfile.lock` are build output and are
  git-ignored. Never commit them.
