---
title: "Building an Open-Source Blog with Hugo"
description: "How this blog is architected, built, and deployed"
date: 2026-08-25
draft: false
tags: ["Hugo", "GitHub Pages", "CI/CD"]
categories: ["engineering"]
series: ["blog setup"]
translationKey: "hugo-open-source-blog"
---

This post explains how this blog is built: Hugo + PaperMod + GitHub Pages + GitHub Actions.

## Tech stack

| Layer | Choice |
| --- | --- |
| Static site | Hugo Extended 0.165 |
| Theme | PaperMod 8.0 |
| Hosting | GitHub Pages |
| CI/CD | GitHub Actions |

## Local preview

```bash
hugo server --buildDrafts
```

## Deployment flow

After pushing to the `main` branch, GitHub Actions runs the build:

```yaml
- run: hugo --minify
```

The output is published through GitHub Pages while the source branch stays clean, containing only Markdown and configuration.
