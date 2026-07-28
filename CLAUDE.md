# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A personal tech blog built with **Hugo** and the **PaperMod** theme ([adityatelange/hugo-PaperMod](https://github.com/adityatelange/hugo-PaperMod)), deployed to GitHub Pages at https://hongwei6.github.io via GitHub Actions on every push to `main`.

## Commands

```bash
hugo server -D              # Local dev server at http://localhost:1313 (includes drafts)
hugo server                 # Local server, drafts hidden (production-like)
hugo --minify --gc          # Full production build into ./public
hugo new posts/my-post.md   # Scaffold a new post from archetypes/default.md
git submodule update --init --recursive         # First-time: fetch themes/PaperMod
git submodule update --remote themes/PaperMod   # Upgrade the theme
```

Local Hugo is **0.164.0+extended** (Homebrew). CI pins `HUGO_VERSION: 0.164.0`. PaperMod requires Hugo ≥ 0.146.0; the extended build is used (CI installs `dart-sass`).

## Architecture

### Configuration is a single root `hugo.yml`
PaperMod uses a single YAML config at the repo root. Key sections:

| Section | Holds |
|---------|-------|
| top-level | `baseURL`, `theme: ["PaperMod"]`, `defaultContentLanguage: "zh-cn"`, `enableGitInfo: true` |
| `markup` | `goldmark.renderer.unsafe: true` (raw HTML in posts); `highlight.noClasses: false` (follows light/dark theme) |
| `menu.main` | Top nav: `文章` / `标签` / `分类` / `GitHub` |
| `params` | **Most theme behavior** — see below |
| `outputs.home` | Includes `JSON` so PaperMod's fuse.js search has an index |

Important `params` to know:
- `defaultTheme: auto` + `disableThemeToggle: false` → follows system light/dark, with manual toggle.
- `profileMode.enabled: true` + `profileMode.buttons` → homepage is a Profile card with quick links, **not** a post list. `mainSections: [posts]` is still set so PaperMod can find post sections.
- `socialIcons` → icons shown on the Profile card (GitHub, RSS).
- Article-page toggles: `ShowToc`, `UseHugoToc`, `ShowReadingTime`, `ShowWordCount`, `ShowCodeCopyButtons`, `ShowPostNavLinks`, `ShowBreadCrumbs`.
- `assets.disableHLJS: true` → use Hugo's built-in highlighter (Chroma) instead of highlight.js.

To change appearance, navigation, or article features, edit `hugo.yml`. Taxonomies need no declaration — Hugo ships with `tags`/`categories` by default, and post front matter uses them directly.

> This repo has used three different themes (PaperMod → Blowfish → Terminal → back to PaperMod). Earlier `CLAUDE.md`/`README.md` revisions and a root `config.toml` / `static/style.css` / `config/_default/` directory belonged to those other themes and have been removed. If a config file or override exists that doesn't match `hugo.yml`, it is leftover — trust `hugo.yml`.

### Theme is a git submodule
`themes/PaperMod` is tracked via `.gitmodules`. **Never hand-edit files under `themes/PaperMod/`** — changes are lost on the next submodule update. CI uses `submodules: recursive` at checkout.

### Content
- Posts: `content/posts/*.md` — front matter uses `title`, `date`, `draft`, `description`, `tags`, `categories` (see `archetypes/default.md`).
- Homepage: `content/_index.md` — front-matter only (`title`). In `profileMode` the body is overridden by the Profile card built from `hugo.yml` `params.profileMode`, so don't put content here expecting it to render.
- `draft = true` posts are excluded from production builds — flip to `false` before publishing.

### Known warning (non-blocking)
Hugo ≥ 0.158 emits one deprecation warning from PaperMod's own templates: `.Language.LanguageDirection was deprecated ... Use .Language.Direction instead`. This is the theme's code, not our config, and does not block the build. It will disappear when PaperMod ships a fix.

### Deployment (`.github/workflows/hugo.yml`)
Push to `main` → builds with `hugo --minify --baseURL <pages-base-url>/` (production env, `TZ: America/Los_Angeles`) → uploads `./public` → deploys to GitHub Pages. `fetch-depth: 0` + `enableGitInfo` make post "last modified" timestamps resolve from full git history. Concurrency group `pages` serializes deploys.

## Conventions

- Default language is `zh-cn`; content and config comments are in Chinese. Match this when adding posts or editing config.
- Prefer PaperMod's built-in `params` knobs and shortcodes over custom templates. Light/dark theming is configured, not hand-coded.
- `.omc/` is operational state (oh-my-claudecode) — gitignored tooling artifacts, not part of the site.
