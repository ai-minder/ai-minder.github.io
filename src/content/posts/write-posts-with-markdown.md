---
title: 用 Markdown 发文章，流程到底有多顺
description: 新增一篇文章只要一个 Markdown 文件，这就是这套博客最爽的地方。
pubDate: 2026-03-18
updatedDate: 2026-03-19
tags:
  - Writing
  - Workflow
  - GitHub Pages
---

这套博客的核心不是“炫技”，而是 **省脑子**。

你以后新增文章，只需要在 `src/content/posts/` 下面创建一个新文件，比如：

```bash
src/content/posts/my-new-post.md
```

然后写上 frontmatter：

```md
---
title: 我的新文章
description: 这篇文章讲什么
pubDate: 2026-03-19
updatedDate: 2026-03-19
featured: false
tags:
  - Notes
  - AI
---

正文开始。
```

## 构建和预览

本地开发时常用这几个命令：

```bash
npm install
npm run dev
npm run build
```

- `npm run dev`：本地预览
- `npm run build`：生成生产环境静态文件

## 部署逻辑

仓库里已经加了 GitHub Actions 工作流。只要你往 `main` 分支推送代码，GitHub 就会：

1. 安装依赖
2. 构建站点
3. 发布到 GitHub Pages

## 适合写什么

这种站特别适合：

- 产品想法
- 技术笔记
- 周报月报
- AI 工作流总结
- SaaS 运营日志

别整太复杂，先把内容持续写出来。站点好不好，最终还是看内容，不是看你动画转得有多花。
