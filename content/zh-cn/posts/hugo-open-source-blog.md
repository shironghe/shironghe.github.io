---
title: "使用 Hugo 搭建开源博客"
description: "记录这个博客的架构、工具链与部署方式"
date: 2026-08-25
draft: false
tags: ["Hugo", "GitHub Pages", "CI/CD"]
categories: ["工程实践"]
series: ["博客搭建"]
translationKey: "hugo-open-source-blog"
---

这篇文章介绍本站的搭建方式：Hugo + PaperMod + GitHub Pages + GitHub Actions。

## 技术栈

| 层 | 选择 |
| --- | --- |
| 静态站点 | Hugo Extended 0.165 |
| 主题 | PaperMod 8.0 |
| 托管 | GitHub Pages |
| CI/CD | GitHub Actions |

## 本地预览

```bash
hugo server --buildDrafts
```

## 部署流程

推送 `main` 分支后，GitHub Actions 会执行构建：

```yaml
- run: hugo --minify
```

构建产物通过 GitHub Pages 发布，源码分支保持干净，只包含 Markdown 和配置。
