# Hongwei's Blog

基于 **Hugo + Terminal** 主题的个人技术博客(终端绿 matrix 配色),托管在 GitHub Pages。

- 🌐 线上地址:<https://hongwei6.github.io>
- 📦 仓库:<https://github.com/Hongwei6/Hongwei6.github.io>
- 🎨 主题:[hugo-theme-terminal](https://github.com/panr/hugo-theme-terminal)

## ✨ 特性

- 复古终端风 + Matrix 终端绿配色(`static/style.css`)
- 代码高亮(Chroma,github-dark 风格)+ 一键复制
- 文章目录(TOC)、阅读时间、最后更新时间
- 标签、分类
- RSS 订阅:`https://hongwei6.github.io/index.xml`
- GitHub Actions 自动部署(推送 `main` 即上线)

---

## 📝 日常使用

### 写一篇新文章

```bash
cd ~/Hongwei6.github.io
hugo new posts/my-new-post.md
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
│   ├── _index.md         # 首页
│   └── posts/            # 博客文章放这里
├── themes/terminal/      # 主题(git submodule,勿手动改)
├── config.toml           # 站点配置(标题、菜单、语言、params)
├── static/style.css      # Matrix 终端绿配色覆盖
├── archetypes/           # 新建文章的模板
└── .github/workflows/    # 自动部署脚本
```

## ⚙️ 常见配置

- 改站点标题、副标题、菜单文案 → 编辑 `config.toml` 的 `[languages.zh-cn.params]`
- 改导航菜单 → 编辑 `config.toml` 的 `[[languages.zh-cn.menu.main]]`
- 改配色 → 编辑 `static/style.css`(`:root` 里的 `--background`/`--foreground`/`--accent`)
- 升级主题 → `git submodule update --remote themes/terminal`

## 🛠️ 环境要求

本地需要 **Hugo Extended**(本项目 + CI 均用 0.164.0;Terminal 要求 Extended ≥ 0.90):

```bash
brew install hugo
```

克隆后需初始化主题 submodule:

```bash
git submodule update --init --recursive
```
