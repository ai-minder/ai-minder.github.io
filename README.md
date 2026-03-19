# AI Minder Blog

一个基于 **Astro** 的现代简约博客模板，支持：

- Markdown 文章内容
- 自动生成静态 HTML
- GitHub Pages 自动部署
- RSS 与 Sitemap
- 深色 / 浅色主题切换

## 本地开发

```bash
npm install
npm run dev
```

## 构建

```bash
npm run build
```

## 新增文章

在 `src/content/posts/` 下新增 Markdown 文件：

```md
---
title: 文章标题
description: 一句话摘要
pubDate: 2026-03-19
updatedDate: 2026-03-19
featured: false
tags:
  - AI
  - Notes
---

正文从这里开始。
```

## 部署到 GitHub Pages

仓库已经包含 `.github/workflows/deploy.yml`。

默认流程：

1. 推送到 `main`
2. GitHub Actions 自动构建
3. 构建产物发布到 GitHub Pages

如果仓库的 Pages 还没启用，在 GitHub 仓库设置中把 **Pages Source** 设为 **GitHub Actions** 即可。

## 目录结构

```text
.
├── public/
├── src/
│   ├── content/posts/
│   ├── layouts/
│   ├── pages/
│   └── styles/
└── .github/workflows/
```
