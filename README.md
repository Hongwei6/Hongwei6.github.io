# Hongwei's Blog

基于 **Hugo + PaperMod** 主题的个人技术博客,托管在 GitHub Pages。

- 🌐 线上地址:<https://hongwei6.github.io>
- 📦 仓库:<https://github.com/Hongwei6/Hongwei6.github.io>

## ✨ 特性

- 代码高亮 + 一键复制
- 文章目录(TOC)
- 全文搜索
- 暗色 / 亮色模式(自动跟随系统)
- 标签、分类、归档
- RSS 订阅:`https://hongwei6.github.io/index.xml`
- GitHub Actions 自动部署(推送 `main` 即上线)

---

## 📝 日常使用

### 写一篇新文章

```bash
cd ~/Hongwei6.github.io
hugo new content posts/my-new-post.md
```

然后用编辑器打开 `content/posts/my-new-post.md` 写正文。
写好后把头部的 `draft = true` 改成 `draft = false`(否则不会发布)。

### 本地预览

```bash
cd ~/Hongwei6.github.io
hugo server -D      # -D 表示包含草稿
```

浏览器打开 <http://localhost:1313/> 预览。按 `Ctrl+C` 停止。

### 发布上线

```bash
cd ~/Hongwei6.github.io
git add -A
git commit -m "post: 新文章标题"
git push
```

推送后,GitHub Actions 会自动构建并部署。约 1-2 分钟后线上即更新。
可在 <https://github.com/Hongwei6/Hongwei6.github.io/actions> 查看部署进度。

---

## 🗂️ 目录结构

```
Hongwei6.github.io/
├── content/              # 文章(Markdown)
│   └── posts/            # 博客文章放这里
├── themes/PaperMod/      # 主题(git submodule,勿手动改)
├── hugo.toml             # 站点配置(标题、菜单、作者等)
├── archetypes/           # 新建文章的模板
└── .github/workflows/    # 自动部署脚本
```

## ⚙️ 常见配置

- 改站点标题、作者、社交链接 → 编辑 `hugo.toml`
- 改导航菜单 → 编辑 `hugo.toml` 的 `[[menu.main]]`
- 升级主题 → `git submodule update --remote themes/PaperMod`

## 🛠️ 环境要求

本地需要 Hugo(本项目使用 0.164.0,CI 用 0.147.4):

```bash
brew install hugo
```
