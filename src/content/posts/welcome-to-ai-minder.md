---
title: 欢迎来到 AI Minder
description: 一套为 Markdown 写作准备好的现代极简博客，现在已经能直接拿来发文章了。
pubDate: 2026-03-19
updatedDate: 2026-03-19
featured: false
tags:
  - Launch
  - Astro
  - Markdown
---

这个站现在已经不是空壳了。

它是一套基于 **Astro** 的静态博客，文章来源就是 Markdown 文件，构建后自动生成 HTML，再通过 GitHub Pages 发布出去。整个思路很简单：

1. 在 `src/content/posts/` 里新建一篇 `.md`
2. 写上 frontmatter
3. 推到 GitHub
4. 等 GitHub Actions 自动部署

## 为什么这样做

因为这套方案够省事：

- **写作体验干净**：专心写 Markdown，不用手搓页面
- **页面速度快**：静态站点，加载非常轻
- **部署链路稳**：推送即发布，少折腾
- **后续好扩展**：标签、归档、搜索、评论都能慢慢加

## 你接下来最常做的事

以后大部分时候，你只需要复制一篇现有文章，然后改这些字段：

| 字段 | 作用 |
| --- | --- |
| `title` | 文章标题 |
| `description` | 文章摘要 |
| `pubDate` | 发布时间 |
| `updatedDate` | 更新时间 |
| `featured` | 是否放到首页置顶 |
| `tags` | 文章标签 |

## 一个建议

如果你想把这个站长期用起来，下一步最值得加的是：

- 标签页
- 归档页
- 评论系统
- 自定义域名和 SEO 增强

先把内容发起来，比一开始就把功能堆满更重要。先写，别磨叽。
