# AGENTS.md

## What this is

Chinese-language personal tech blog built with **Hugo** + **github-style** theme (`MeiK2333/github-style`), deployed to GitHub Pages at https://hongwei6.github.io.

## Critical: CLAUDE.md is stale

`CLAUDE.md` was written for a previous PaperMod setup. The actual repo uses:
- **Config:** `hugo.toml` (not `hugo.yml`)
- **Theme:** `github-style` (not PaperMod), tracked as git submodule in `themes/github-style/`
- **Posts directory:** `content/post/` (not `content/posts/`)

Trust `hugo.toml` and this file over `CLAUDE.md` until it is updated.

## Commands

```bash
hugo server -D              # Dev server at http://localhost:1313 (includes drafts)
hugo server                 # Production-like preview (drafts hidden)
hugo --minify --gc          # Production build into ./public
hugo new post/my-post.md    # Scaffold new post (goes to content/post/)
git submodule update --init --recursive   # First-time: fetch theme submodule
```

Hugo version: 0.164.0+extended (Homebrew). Theme requires Hugo ≥ 0.146.0.

## Architecture

### Config (`hugo.toml`)

Single root TOML config. Key sections:
- Top-level: `baseURL`, `theme: 'github-style'`, `defaultContentLanguage = 'zh'`, `enableGitInfo = true`
- `[params]`: author, avatar, favicon, custom CSS, search, social links
- `[params.links]`: homepage link cards (GitHub, RSS)
- `[markup]`: goldmark with `unsafe = true` (raw HTML in posts), highlight style `github`
- `[outputs]` home includes JSON for local search (fuse.js)

### Theme submodule

`themes/github-style` is a git submodule. **Never hand-edit files under `themes/`** — they are lost on submodule update.

### Custom layouts

Repo overrides four partials in `layouts/partials/`:
- `footer.html`, `overview.html`, `comment-section.html`, `posts.html`

These extend/override the theme defaults.

### Content structure

- Posts: `content/post/*.md`
- About page: `content/about.md`
- Tags index: `content/tags/_index.md`
- Homepage: `content/readme.md` (github-style theme renders this as the profile/README section)

### Front matter

```yaml
title: "..."
date: YYYY-MM-DD
draft: false
description: "..."
tags: ["Spring AI", "Java"]
categories: ["后端"]
cover: "/hero/tt1.png"       # optional, used by github-style
```

Archetype at `archetypes/default.md` scaffolds with `draft: true`. Flip to `false` before publishing.

### Static assets

- Custom CSS: `static/css/custom.css`
- Images: `static/images/` (avatar, favicon), `static/hero/` (post cover images)

## 内容整理要求

- 整理本地文档时，去掉教学口吻，写成简洁的技术笔记风格，不要有"教程"味道。
- 文中涉及的图片必须下载到本地（`static/hero/` 或文章对应目录），用本地路径引用，禁止依赖外部图床链接。
- 文章整理完成后，执行 `git add` → `git commit` → `git push` 推送到 GitHub 触发部署。

## Conventions

- Language is `zh-cn`; content and config comments are in Chinese. Match this for new posts.
- No CI workflow currently exists — deployment is manual (`git push` to `main`).
- `.omc/` is operational state (oh-my-claudecode) — gitignored, not part of the site.
- `public/` and `resources/_gen/` are build artifacts — gitignored.

## Gotchas

- Post URLs derive from file path under `content/`, so `content/post/foo.md` → `/post/foo/`.
- `hugo.toml` uses `locale = 'zh-cn'` and `hasCJKLanguage = true` for proper Chinese word count and search tokenization.
- Math support is enabled via goldmark passthrough delimiters (`$$...$$` for block, `$...$` for inline).
