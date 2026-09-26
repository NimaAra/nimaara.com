# AGENTS.md

Guidance for AI coding agents working in this repository.

## What this is

Personal blog built with Jekyll 4.4 and the minima theme, deployed to GitHub Pages from `main` (custom domain `nimaara.com`, see `CNAME`). Articles are Markdown files in `_posts/`.

There is **no Ruby on the host machine** and none is needed: local preview runs in Docker, and GitHub Pages builds the site server-side on push to `main`.

## Commands

| Task | Command |
|------|---------|
| Preview with live reload at `http://localhost:4000` | `docker compose up` |
| Build/update after changing `Gemfile` or verify a build | `docker run --rm -v "$PWD:/site" -w /site ruby:3.3 sh -c "bundle install --quiet && bundle exec jekyll build"` |

- Run anything Ruby-related through the `ruby:3.3` Docker image as above. Do not assume Ruby, bundler, or gem tooling on the host.
- CI: `.github/workflows/build-check.yml` builds the site on every push; it must pass. It runs on all branches, so you can validate changes on a feature branch without deploying.
- Commit updated `Gemfile.lock` whenever the `Gemfile` changes. The lock is multi-platform (Windows + Linux entries); never regenerate it on a single platform.

## Writing posts

1. File name: `_posts/YYYY-MM-DD-title-slug.md`
2. Front matter:
   ```yaml
   ---
   title: "Post Title"
   tags: "DotNet C#"
   ---
   ```
   Free-form tags; they default to `Other` via `_config.yml`.
3. Do **not** add `layout:` to post front matter — `_config.yml` defaults posts to `layout: post`.
4. Content conventionally starts with `### Post Title`. The `jekyll-titles-from-headings` plugin strips that heading from the rendered page; without the plugin the title would appear twice.
5. Permalinks are the Jekyll default `/YYYY/MM/DD/title-slug.html`. When changing an existing post's URL, preserve the old one with `redirect_from:` (see `_posts/2019-04-10-tranquillity-in-csharp.md` for an example).
6. Syntax highlighting is client-side highlight.js (gruvbox theme, loaded in `_includes/head.html`). Kramdown's server-side highlighter is disabled in `_config.yml`. Use fenced code blocks with a language tag.

## Environment parity (important)

Production builds with the `github-pages` gem; this repo builds locally with plain Jekyll from the `Gemfile`. Pages' build historically relied on plugins that plain Jekyll lacks, which caused local/production drift. The following explicit pieces close that gap — do not remove them:

| Piece | Purpose |
|-------|---------|
| `_config.yml` `defaults` → `layout: post` | Replaces Pages-only `jekyll-default-layout` |
| `jekyll-titles-from-headings` in `Gemfile` + `plugins` | Strips leading `###` heading (Pages-only plugin made explicit) |
| `_config.yml` `url: "https://nimaara.com"` | Canonical URLs, share links, `og:url` (Pages normally injects this) |

Rules:

- Only add gems listed on the GitHub Pages whitelist (<https://pages.github.com/versions/>), because production still builds with the `github-pages` gem.
- A local build must match production HTML. Expected differences are **only**: the GA4 snippet (production-only via `JEKYLL_ENV=production`), Cloudflare email obfuscation (injected at the edge, not by Jekyll), and the `meta generator` version string. Anything else means drift — investigate before committing.
- `_config.yml` `exclude` keeps repo-tooling files (`AGENTS.md`, `docker-compose.yml`) out of the built site. Without it, the Pages-only `jekyll-optional-front-matter` plugin renders `AGENTS.md` as `/AGENTS.html` and minima's header nav shows it. Do not remove these entries; add new tooling-only files there too.
- `README.md` is not rendered as a page (production returns 404 for `/README.html`); locally it is copied as a static file — known, harmless difference.

## Verifying changes

1. Build in Docker (command above).
2. Spot-check output in `_site/` — every post and page must contain `<!DOCTYPE html>` (a fragment means the layout did not apply).
3. Optionally diff a page against production, e.g.:
   ```
   curl -s https://nimaara.com/<path> | diff - _site/<path>
   ```
   Expect only the differences listed above.

## Structure

See `README.md` for the layout of `_posts/`, `_layouts/` (custom `post`/`home` overrides of minima), `_includes/`, `css/override.css` (style tweaks), and `docker-compose.yml` (preview server; serves drafts from `_drafts/` and future-dated posts, which production hides).
