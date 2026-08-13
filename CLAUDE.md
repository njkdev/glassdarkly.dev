# Glass Darkly

Hugo site deployed at glassdarkly.dev. Custom theme in `themes/glass/`.

Verified against the code on 24 July 2026; the Deploy section was rewritten on
13 August 2026 for the move from Pages to Workers.

## Build & preview
- `hugo server -D` — local dev **with drafts**. `content/posts/cold-read.md` is
  `draft = true` and invisible without `-D`.
- `hugo` — production build to `public/`.

## Structure
- `content/posts/` — the writing (6 files, 1 currently a draft)
- `layouts/` — project-level overrides: RSS and 404 only
- `themes/glass/` — layouts, `static/css/style.css`, the six woff2 fonts,
  favicon
- `static/` — the two standalone art pages, the shared p5.js, the RSS
  stylesheet

## Deploy
- **Cloudflare Workers** with static assets, building from `origin/main` —
  migrated off Cloudflare Pages on 13 August 2026. Workers Builds runs `hugo`
  on every push to main and serves `public/` per `wrangler.jsonc`. The Worker
  is named `glassdarkly`; that name must match the dashboard or the build warns
  and offers a PR to reconcile it.
- **`wrangler.jsonc` is assets-only** — no `main`, no script, no bindings. Its
  `not_found_handling: "404-page"` line is what serves `404.html`; Pages
  inferred that from the file's presence, Workers requires it stated. Drop the
  line and unknown URLs quietly return a bare 404 instead of the site's own.
- **No local Node toolchain is involved.** Workers Builds runs
  `npx wrangler deploy` inside Cloudflare's build container. This machine needs
  only Homebrew Hugo, exactly as it did under Pages.
- **`HUGO_VERSION` is pinned in the Workers dashboard**, currently **0.165.0**.
  The dashboard holds that value, not this repo — if a local build ever
  disagrees with the deployed site, check the pin first. As of 13 August 2026
  Homebrew also ships 0.165.0, so local matches CI; that alignment breaks
  silently whenever Homebrew moves, so re-check it rather than assume it.
- The **www→apex redirect is a zone-level Cloudflare rule, not a file.** A
  `_redirects` file cannot match on hostname — as true under Workers as it was
  under Pages — so re-adding one won't work. `www` is an **A record →
  `192.0.2.1`, proxied**: RFC 5737 TEST-NET-1, deliberately unroutable, there
  only to give the redirect rule a proxied hostname to attach to. It was a
  dangling `CNAME → glassdarkly.pages.dev` for a while after the Pages project
  was deleted; don't reintroduce that shape.
- Canonical URL is `https://glassdarkly.dev` (no www), enforced in two
  independent places: the zone redirect rule, and `baseURL` in `hugo.toml`
  which drives every canonical tag, feed URL and sitemap entry. They must
  agree — the 301 does not rescue a wrong `baseURL`.
- Nothing in the repo checks build status, and this site once served a stale
  build for a month — two curly quotes broke TOML front matter in June 2026 and
  the failure went unnoticed. Two guards, neither automatic: run `hugo` locally
  before pushing, and turn on **build notifications in the Workers dashboard**
  so a failed deploy emails you instead of passing in silence.

## Conventions
- **goldmark `unsafe` is off** (`hugo.toml:13`) — no raw HTML in markdown, and
  the content currently honours that with zero HTML tags.
- **p5.js is shared** at `static/js/p5.min.js`; both art pages reference it by
  absolute path. Don't vendor a second copy.
- **Favicons:** SVG for browsers (`themes/glass/static/favicon.svg`), PNG for
  RSS (`static/favicon.png`), ICO + apple-touch in `static/` for legacy agents.
- `static/rss.xsl` is referenced by a hardcoded `/rss.xsl` in
  `layouts/_default/rss.xml:7` — moving it breaks feed preview without breaking
  the build.
- The feed's `lastBuildDate` is wrapped in `{{- with $pages }}`. Without it, an
  empty `content/posts/` builds happily and emits
  `<lastBuildDate></lastBuildDate>` — valid XML, invalid RSS, no warning. Both
  date fields also need `| safeHTML`, or Hugo escapes the `+` of the timezone
  offset into `&#43;`.
- Three of the six fonts (Quattro Bold, Quattro BoldItalic, Plex Mono Italic)
  are declared but not matched by any current content — no `**bold**`, no
  blockquotes yet. They cost nothing at runtime and complete the families;
  dropping them would quietly substitute synthetic bold the first time a post
  needs it.
- The two art pages (`encounter-fields`, `nodes-without-edges`) are linked from
  the footer of every page, so they're structural navigation rather than hidden
  — they're simply never mentioned in prose.

## Language: use `.Site.Language.Locale`
`hugo.toml:2` sets `locale = 'en-us'`, and three templates read it —
`themes/glass/layouts/_default/baseof.html:2`, `layouts/404.html:2`,
`layouts/_default/rss.xml:14`.

They must use **`.Site.Language.Locale`**, which returns `en-us`. The
near-neighbours don't: `.Site.Language.Lang` and `.Site.Language.Name` both
return the language *key* (`en`), and `.Site.LanguageCode` returns `en-us` but
is deprecated as of 0.158 with Hugo pointing at `Locale` as its replacement.

These three sites previously used `.Site.Language.Lang`, so the site shipped
`lang="en"` and the config line did nothing. Fixed 24 July 2026 and verified
against a 0.164.0 build: all 11 output pages plus both feeds now carry `en-us`,
and the full output diff contained no other change. Nothing warns if this gets
reverted — the wrong accessor is silently valid.

## Editorial
njk's editorial stance — no schedule, no retrospective editing of published
posts — is archived at `~/Developer/review/glassdarkly.dev/EDITORIAL.md`. A
revised style book is in progress; ask rather than assuming the archived
version still governs.
