---
share: true
title: Bilingual blog publishing (English/Korean toggle)
categories:
  - Wiki
tags:
  - i18n
  - chirpy
  - jekyll
  - publishing-pipeline
author: claude
type: concept
created: 2026-07-19
updated: 2026-07-19
sources:
  - "[[2026-07-19-bilingual-en-ko-toggle-for-blog-posts]]"
aliases:
  - en-ko-toggle
  - language-toggle
  - bilingual-posts
  - jekyll-polyglot
---

How to serve one blog post in both English and Korean on this vault's
Jekyll + Chirpy site, and why an in-page toggle is the option that fits the
publish pipeline with zero schema and zero pipeline change.

## The constraint the feature must fit

Five facts about the pipeline score every design:

- **Verbatim copy.** `sync-site.py` copies each `share: true` note's body
  byte-for-byte into the site repo's flat `_posts/`, where the filename is the
  post's identity and URL. Anything a design adds to the body moves for free;
  anything it adds outside the body needs new pipeline code.
- **Chirpy is a gem.** The theme ships as a gem (7.6); the site overrides only
  `_layouts/home.html`, so every other layout and include comes from the gem.
  A new per-post behavior means overriding one more layout.
- **Full Jekyll in CI.** GitHub Actions builds with `bundle exec jekyll b`, not
  the restricted GitHub Pages gem, so custom plugins are allowed in principle.
- **htmlproofer gates the deploy.** Every internal link must resolve or CI
  fails — so any cross-language linking must stay valid.
- **One site-wide UI language.** `_config.yml` sets `lang: en`; there is no
  per-post language switch built in.

## Three approaches

| Approach | Content shape | Code changes | Fit |
|---|---|---|---|
| **In-page toggle** (recommended) | One file, two `<div lang>` blocks, JS shows one | Post-layout override + small JS/CSS in the site repo | Best — content stays in the body the copy already moves; zero vault change |
| **Paired posts** | Two files, cross-linked | None anywhere | Good, but two URLs and doubled category/tag entries |
| **`jekyll-polyglot`** | Per-language file with a `lang` key | Plugin + config + Chirpy patches + vault rework | Poor — parallel URL trees fight the flat mirror |

## Recommended: in-page toggle with DOM auto-detection

The human writes both languages inside one post body, each wrapped in a
language block; a button on the rendered page switches which block shows.

```markdown
<div class="post-lang" lang="en" markdown="1">

English text here. Normal Markdown works inside the block.

</div>

<div class="post-lang" lang="ko" markdown="1">

한국어 본문. 블록 안에서 마크다운 그대로 사용.

</div>
```

`markdown="1"` tells kramdown to render Markdown inside the raw HTML block;
the blank lines around the content are required for that. Raw HTML passes
through kramdown untouched, so the verbatim copy needs no help.

The only code lives in the **site repo** (`kalaluthien.github.io`):

1. **Override the post layout** — copy the Chirpy gem's `_layouts/post.html`
   into the site repo (same pattern as the existing `home.html` override) and
   add a toggle control plus a `<script>` tag near the top of the article body.
2. **Add `assets/js/lang-toggle.js`** — on load it finds `.post-lang` blocks.
   With both `en` and `ko` present it un-hides the toggle, shows one language
   (default from `<html lang>` or `localStorage`), and hides the other. A post
   with a single block never shows the button, so the change is invisible on
   every existing monolingual post.
3. **Add one CSS rule** — `.post-lang[hidden] { display: none; }` plus light
   button styling, in the site's existing custom stylesheet.

No plugin, no `_config.yml` change, no Gemfile change. The **vault repo** needs
no pipeline change: the verbatim copy already moves the two `<div>` blocks, and
htmlproofer is unaffected because the toggle adds no cross-post links.

### Trade-offs to accept

- **One URL carries both languages.** Search engines index the whole page, so
  both languages appear in the excerpt and meta description. Chirpy builds the
  excerpt from the first 200 characters of the body, so the language that comes
  first in the file wins the preview.
- **`<html lang="en">` stays global.** Per-block `lang` attributes keep the
  markup honest for screen readers, but the page-level language does not change
  when the button is pressed.
- **No-JS readers see both blocks** stacked — acceptable degradation.

### Optional metadata flag

If a layout keyed on metadata is later preferred over DOM auto-detection, add
an optional `langs` key (e.g. `langs: [en, ko]`). That requires adding `langs`
to `PUBLISHABLE_OPTIONAL` in the front-matter linter, or the unknown key is
rejected and the publish refuses (see obsidian-flavored-markdown for how
front-matter and body syntax are handled). Auto-detection avoids this entirely,
so prefer it.

## Alternative: paired posts

Write two files (`...-hello-world-en.md`, `...-hello-world-ko.md`), each a
normal post, cross-linked in the body with a root-relative link
(`[한국어](/posts/hello-world-ko/)`). Filename uniqueness is already enforced and
standard Markdown links are already required in shared bodies, so no code
changes are needed anywhere. The cost is structural: two URLs, both languages
on every category and tag page, and the cross-links must stay valid or
htmlproofer fails. Suits a long post better kept as two clean single-language
pages than one heavy bilingual one — a translation pair, not a true toggle.

## Alternative: jekyll-polyglot

The plugin gives real i18n — a `/ko/...` URL tree, a site-wide selector,
per-language sitemaps — and can run because CI uses full Jekyll. But the cost
is high and works against the current design: config and Gemfile changes;
Chirpy ships no polyglot selector, so the nav, sitemap, and feed includes need
gem-copied patches maintained on every theme update; and the flat mirror
breaks, because polyglot expects a `lang` key per file and generates parallel
trees, so `sync-site.py` and the linter would both need rework. Choose this
only if the whole site becomes bilingual (UI, tags, many posts), not for a
per-post convenience.

## Rollout order

1. Site repo: add `assets/js/lang-toggle.js`, the CSS rule, and the post-layout
   override; verify locally that a single-language post shows no button.
2. Convert one post as the pilot; publish; confirm the toggle works and
   htmlproofer still passes.
3. Vault repo: document the two-block convention where authors and the agent
   will see it. Add the optional `langs` flag to the linter only if a
   metadata-driven layout is later adopted.
