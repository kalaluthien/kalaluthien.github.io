---
share: true
title: Adding an English/Korean Toggle to Blog Posts
categories:
  - Report
  - Meta
tags:
  - chirpy
  - jekyll
  - i18n
  - publishing-pipeline
author: claude
---
> To add en/ko toggle feature to the "blog" posts, what changes are required? Write a report for it.

An in-page toggle is the smallest change that fits the current pipeline. It
touches the **site repo** (add a button and a script to the post layout) and
needs **no vault schema change** if the toggle script auto-detects the two
language blocks. The human writes both languages in one post file; a button
swaps which one shows. Two heavier alternatives exist — paired posts and the
`jekyll-polyglot` plugin — but both fight the byte-for-byte mirror that makes
the current publish simple.

## The constraint the feature must fit

The vault publishes with `sync-site.py`: it copies each `share: true` note's
body **verbatim** into the site repo's flat `_posts/`, where the filename is
the post's identity and URL. Chirpy 7.6 runs as a **gem** — the site overrides
only `_layouts/home.html`, so every other layout and include comes from the
gem. The site builds with **full Jekyll** in GitHub Actions
(`bundle exec jekyll b`), not the restricted GitHub Pages gem, so custom
plugins are allowed. CI runs **htmlproofer**, so every internal link must
resolve or the deploy fails. `_config.yml` sets `lang: en` — one site-wide UI
language, with no per-post switching built in.

Any toggle design is scored against these five facts.

## Three approaches

| Approach | Content shape | Site-repo work | Vault work | Fit with pipeline |
|---|---|---|---|---|
| **A. In-page toggle** (recommended) | One file, two `<div lang>` blocks, JS shows one | Post-layout override + small JS/CSS | None (or one optional flag) | Best — all content stays in the body the copy already moves |
| **B. Paired posts** | Two files, cross-linked | None | None | Good, but two URLs and doubled category/tag entries |
| **C. `jekyll-polyglot`** | Per-language file with `lang` key | Plugin + config + Chirpy patches | `lang` on every post | Poor — parallel URL trees fight the flat mirror |

## Recommended: in-page toggle

The human writes both languages inside one post body, wrapped in language
blocks. A button on the rendered page switches which block is visible. Because
everything lives in the body, the publish pipeline needs no change at all — it
already copies bodies verbatim, and kramdown passes raw HTML through.

### Post body convention (vault side)

```markdown
<div class="post-lang" lang="en" markdown="1">

English text here. Normal Markdown works inside the block.

</div>

<div class="post-lang" lang="ko" markdown="1">

한국어 본문. 블록 안에서 마크다운 그대로 사용.

</div>
```

`markdown="1"` tells kramdown to render Markdown inside the raw HTML block.
The blank lines around the content are required for that.

### Site repo (`kalaluthien.github.io`) — the only code changes

1. **Override the post layout.** Copy the Chirpy gem's `_layouts/post.html`
   into the site repo (the site already overrides `home.html`, so this is the
   established pattern). Add, near the top of the article body, a toggle
   control and a script tag:

   ```html
   <div class="lang-toggle" hidden>
     <button data-lang="en">EN</button>
     <button data-lang="ko">한국어</button>
   </div>
   <script src="{{ '/assets/js/lang-toggle.js' | relative_url }}"></script>
   ```

2. **Add `assets/js/lang-toggle.js`.** On load it looks for
   `.post-lang` blocks. If it finds both `en` and `ko`, it un-hides the
   toggle, shows one language (default from `<html lang>` or `localStorage`),
   and hides the other. If a post has only one block, the button never
   appears — so the change is invisible on every existing post.

3. **Add one CSS rule** (in the site's existing custom stylesheet under
   `assets/`): `.post-lang[hidden] { display: none; }` plus light styling for
   the button row.

No plugin, no `_config.yml` change, no Gemfile change.

### Vault repo (`notes`) — documentation, optionally schema

- **No pipeline change needed.** The verbatim copy already moves the two
  `<div>` blocks. htmlproofer is unaffected because the toggle adds no
  cross-post links.
- **Optional front-matter flag.** If you want the layout to key off metadata
  instead of DOM auto-detection, add an optional `langs` key
  (e.g. `langs: [en, ko]`). That requires adding `langs` to
  `PUBLISHABLE_OPTIONAL` in
  `.claude/skills/filling-front-matter/scripts/lint-front-matter.py`,
  otherwise the linter rejects the unknown key and the publish refuses. The
  auto-detection version avoids this entirely — prefer it.
- **Document the convention** in three places so authors and the agent know
  it: the front-matter/authoring section of `README.md`, the
  `filling-front-matter` skill, and the `blog-publisher` agent (so it does not
  "fix" the raw HTML blocks and knows a bilingual post is normal).

### Trade-offs to accept

- **One URL carries both languages.** Search engines index the whole page, so
  both languages appear in the excerpt and meta description. Chirpy builds the
  excerpt from the first 200 characters of the body — the language that comes
  first in the file wins the preview.
- **`<html lang="en">` stays global.** The per-block `lang` attributes keep the
  markup honest for screen readers, but the page-level language does not
  change when the button is pressed.
- **No-JS readers see both blocks.** Acceptable degradation; both languages
  are simply shown stacked.

## Alternative A: paired posts

Write two files — for example `...-hello-world-en.md` and
`...-hello-world-ko.md` — each a normal post, cross-linked in the body with a
root-relative link (`[한국어](/posts/hello-world-ko/)`). Filename uniqueness is
already enforced by the publish, and standard Markdown links are already
required in shared bodies, so **no code changes** are needed anywhere.

Cost is structural, not code: two URLs to maintain, both languages listed on
every category and tag page, and the cross-links must stay valid or htmlproofer
fails the build. This suits a long post you would rather keep as two clean
single-language pages than one heavy bilingual one. It is a "translation pair,"
not a true in-place toggle.

## Alternative C: jekyll-polyglot

The plugin gives real i18n: a `/ko/...` URL tree, a site-wide language
selector, and per-language sitemaps. Because CI builds with full Jekyll, the
plugin can run. But the cost is high and works against the current design:

- **Config and Gemfile changes** — `languages`, `default_lang`,
  `exclude_from_localization`, plus the gem.
- **Chirpy does not ship a polyglot language selector.** The nav, sitemap, and
  feed includes need patching (copied from the gem and edited) — ongoing
  maintenance every time the theme updates.
- **The flat mirror breaks.** Polyglot expects a `lang` key per file and
  generates parallel trees; the vault's "filename = one post = one URL" model
  no longer holds, so `sync-site.py` and the linter would both need rework.

Choose this only if the site becomes broadly bilingual (UI, tags, multiple
posts) rather than a per-post convenience. For "a toggle on blog posts," it is
over-built.

## Recommendation and rollout order

Ship the **in-page toggle with DOM auto-detection** — it is the only option
with zero pipeline and zero schema change, and it degrades safely on posts that
stay monolingual.

1. Site repo: add `assets/js/lang-toggle.js`, the CSS rule, and the
   post-layout override. Verify locally that a single-language post shows no
   button.
2. Convert one post (`2026-07-19-hello-world.md`) to the two-block form as the
   pilot; publish; confirm the toggle and that htmlproofer still passes.
3. Vault repo: document the convention in `README.md`,
   `filling-front-matter`, and `blog-publisher`. Add the optional `langs` flag
   to the linter only if a metadata-driven layout is later preferred.
