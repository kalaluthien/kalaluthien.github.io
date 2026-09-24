# kalaluthien.github.io — working rules for coding agents

This repository is the blog: the posts are written here, and a push to `main`
publishes them.

## Read this first
- `_config.yml` — the site's own settings. The theme is `jekyll-theme-chirpy`.
- `.github/workflows/pages-deploy.yml` — what GitHub Pages runs on a push to
  `main`.

## What lives here
The posts (`_posts/`) and the about page (`_tabs/about.md`); the theme and its
overrides (`_includes/`, `_plugins/`, `assets/`); the site settings in
`_config.yml`; the Ruby and gem pins; and the deploy workflow.

## Working conventions
- Ruby is pinned to the version in `.ruby-version`, for rbenv and for CI.
- Serve the site locally with `bash tools/run.sh`; it runs `bundle exec jekyll s -l`.
- Run `bash tools/test.sh` before a push. It builds the site and runs
  html-proofer over `_site`.
- `_site/`, `.jekyll-cache/` and `Gemfile.lock` are build output and are
  git-ignored. Never commit them.
- A post `_posts/YYYY-MM-DD-<slug>.md` is served at `/posts/<slug>/`, without
  the date prefix; the date-prefixed path 404s like a failed deploy. Verify a
  publish at that URL or in `/sitemap.xml`, after the Pages run finishes.
