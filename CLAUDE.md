# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A personal tech blog built with **Hugo** and the **Blowfish** theme (v2.105.0, cyberpunk "neon" color scheme), deployed to GitHub Pages at https://hongwei6.github.io via GitHub Actions on every push to `main`.

> ⚠️ `README.md` is **stale** — it describes the previous PaperMod setup (`hugo.toml`, `hugo new content posts/...`, `themes/PaperMod`). The repo has since migrated to Blowfish. Trust this file and the `config/_default/` directory, not the README, for how the site actually works.

## Commands

```bash
hugo server -D              # Local dev server at http://localhost:1313 (includes drafts)
hugo server                 # Local server, drafts hidden (production-like)
hugo --minify               # Full production build into ./public
hugo new posts/my-post.md   # Scaffold a new post from archetypes/default.md
git submodule update --remote themes/blowfish   # Upgrade the theme
```

Local Hugo is **0.164.0+extended** (Homebrew). CI pins `HUGO_VERSION: 0.164.0`. The **extended** build is required — Blowfish compiles Tailwind via Dart Sass, and the CI workflow installs `dart-sass` separately. A plain (non-extended) Hugo will fail the build.

## Architecture

### Configuration lives in `config/_default/` (NOT `hugo.toml`)
Hugo's config-splitting is used here — a flat `hugo.toml` at the repo root does **not** exist. Settings are split across language-scoped files in `config/_default/`, all loaded together:

| File | Holds |
|------|-------|
| `hugo.toml` | `baseURL`, `theme = "blowfish"`, `defaultContentLanguage`, taxonomies (`tag/category/author/series`), pagination, imaging |
| `languages.zh-cn.toml` | Site title, `params.author` (name/headline/bio/social links) for the `zh-cn` language |
| `params.toml` | **Most theme behavior**: `colorScheme = "neon"`, `homepage.layout = "hero"`, list/article/footer/header settings, `enableSearch`, `enableCodeCopy` |
| `markup.toml` | Code highlight (`style = "github-dark"`, `noClasses = false`), Goldmark `unsafe = true` (raw HTML allowed in posts), block attributes |
| `menus.zh-cn.toml` | Top nav (`文章`/`标签`/`分类`/GitHub) and footer nav |

To change site appearance, navigation, or article features, edit the matching file here — do not create a root `hugo.toml`.

### Theme is a git submodule, not vendored source
`themes/blowfish` is tracked via `.gitmodules` pointing at `nunocorcao/blowfish` branch `main`. **Never hand-edit files under `themes/blowfish/`** — changes will be lost on the next submodule update. Site-level overrides go in `layouts/` and `assets/` (currently empty; Blowfish supports standard Hugo template/scss overrides there).

CI uses `submodules: recursive` at checkout, so the theme is fetched fresh each build. Local clones need `git submodule update --init --recursive` once.

### Content
- Posts: `content/posts/*.md` — front matter uses `title`, `date`, `draft`, `description`, `tags`, `categories` (see `archetypes/default.md`).
- Homepage: `content/_index.md` uses Blowfish's `{{< typeit >}}` shortcode for a typewriter hero (works because `homepage.layout = "hero"` in `params.toml`).
- `draft = true` posts are excluded from production builds — flip to `false` before publishing.

### Deployment (`.github/workflows/hugo.yml`)
Push to `main` → builds with `hugo --minify --baseURL <pages-base-url>/` (production env, `TZ: America/Los_Angeles`) → uploads `./public` → deploys to GitHub Pages. `fetch-depth: 0` is required so Hugo's `.Lastmod` resolves from full git history. Concurrency group `pages` serializes deploys.

## Conventions

- Default language is `zh-cn`; content and config comments are in Chinese. Match this when adding posts or editing config comments.
- Prefer Blowfish's built-in shortcodes and `params.toml` knobs over adding custom templates. The "neon/hero/card" aesthetic is configured, not coded.
- `.omc/` is operational state (oh-my-claudecode) — gitignored tooling artifacts, not part of the site.
