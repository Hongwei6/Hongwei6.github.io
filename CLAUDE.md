# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A personal tech blog built with **Hugo** and the **Terminal** theme ([panr/hugo-theme-terminal](https://github.com/panr/hugo-theme-terminal)), styled with a **Matrix terminal-green** color scheme, deployed to GitHub Pages at https://hongwei6.github.io via GitHub Actions on every push to `main`.

## Commands

```bash
hugo server -D              # Local dev server at http://localhost:1313 (includes drafts)
hugo server                 # Local server, drafts hidden (production-like)
hugo --minify --gc          # Full production build into ./public
hugo new posts/my-post.md   # Scaffold a new post from archetypes/default.md
git submodule update --init --recursive     # First-time: fetch themes/terminal
git submodule update --remote themes/terminal   # Upgrade the theme
```

Local Hugo is **0.164.0+extended** (Homebrew). CI pins `HUGO_VERSION: 0.164.0`. The **extended** build is required — Terminal requires Hugo Extended ≥ 0.90 (it compiles SCSS via Hugo Pipes). A plain (non-extended) Hugo will fail the build.

## Architecture

### Configuration is a single root `config.toml`
Terminal uses Hugo's flat single-file config at the repo root — **not** a `config/` directory (that was the old Blowfish setup and is gone). Everything lives in `config.toml`:

| Section | Holds |
|---------|-------|
| top-level | `baseURL`, `theme = "terminal"`, `defaultContentLanguage = "zh-cn"`, `pagination`, `summaryLength` |
| `[markup.highlight]` | `noClasses = false` — **required** for Terminal's custom Chroma syntax highlighting |
| `[markup.goldmark.renderer]` | `unsafe = true` — raw HTML allowed in posts |
| `[params]` | Theme behavior: `contentTypeName = "posts"`, `showMenuItems`, `centerTheme`, `Toc`, `readingTime`, `showLastUpdated`, `dateFormat` |
| `[languages.zh-cn]` | Site title, and `[languages.zh-cn.params]` (subtitle, menu文案, logo text) |
| `[languages.zh-cn.menu.main]` | Top nav: `文章` / `标签` / `分类` |

To change appearance, navigation, or article features, edit `config.toml`. Taxonomies need no declaration — Hugo ships with `tags`/`categories` by default, and post front matter uses them directly.

### Theme is a git submodule
`themes/terminal` is tracked via `.gitmodules` (submodule `panr/hugo-theme-terminal`). **Never hand-edit files under `themes/terminal/`** — changes are lost on the next submodule update. CI uses `submodules: recursive` at checkout.

### Matrix color scheme via `static/style.css` (NOT `assets/css/colors.css`)
Terminal's `layouts/partials/head.html` loads theme CSS via `resources.Match "css/*.css"`, then appends `static/style.css` **last**, so it has the highest precedence. The matrix green scheme overrides the `:root` CSS variables (`--background`, `--foreground`, `--accent`) that the theme defines in `assets/css/main.css`.

> ⚠️ The theme's README mentions `assets/css/colors.css`, but this theme's `head.html` does **not** load it — that path is a leftover from an older theme version and won't apply. Use `static/style.css` for color overrides.

### Content
- Posts: `content/posts/*.md` — front matter uses `title`, `date`, `draft`, `description`, `tags`, `categories` (see `archetypes/default.md`).
- Homepage: `content/_index.md` — minimal front matter + a one-liner; Terminal renders the post list below it (driven by `contentTypeName = "posts"`). Do **not** use Blowfish-only shortcodes like `{{< typeit >}}` (this theme doesn't ship them).
- `draft = true` posts are excluded from production builds — flip to `false` before publishing.

### Deployment (`.github/workflows/hugo.yml`)
Push to `main` → builds with `hugo --minify --baseURL <pages-base-url>/` (production env, `TZ: America/Los_Angeles`) → uploads `./public` → deploys to GitHub Pages. `fetch-depth: 0` is required so `showLastUpdated`/`.Lastmod` resolve from full git history. Concurrency group `pages` serializes deploys.

## Conventions

- Default language is `zh-cn`; content and config comments are in Chinese. Match this when adding posts or editing config.
- For styling tweaks, prefer overriding CSS variables in `static/style.css` over editing the theme. The "matrix green" aesthetic lives there.
- `.omc/` is operational state (oh-my-claudecode) — gitignored tooling artifacts, not part of the site.
