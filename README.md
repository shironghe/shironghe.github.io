# shironghe 的博客

一个开源、双语的个人技术博客，使用 Hugo 构建，主题为 PaperMod，部署在 GitHub Pages。

## 特性

- 中英双语：中文默认，英文按需翻译
- 文章、标签、分类、系列、归档
- RSS、站内搜索、代码高亮、暗色模式
- Giscus 评论（基于 GitHub Discussions）
- SEO 基础标签与 Open Graph
- GitHub Actions 自动构建部署

## 技术栈

- Hugo Extended 0.165.0
- PaperMod 8.0（git submodule）
- GitHub Pages + GitHub Actions

## 本地开发

需要安装 Hugo Extended。然后运行：

```bash
hugo server --buildDrafts
```

访问 `http://localhost:1313`。

## 新增文章

中文文章放在 `content/zh-cn/posts/`，英文文章放在 `content/en/posts/`。中英文章通过相同的 `translationKey` 配对，英文可以后补：

```bash
hugo new content zh-cn/posts/my-post.md
```

## 部署

推送 `main` 分支即可触发 `.github/workflows/deploy.yml`：

1. 在 GitHub 创建公开仓库 `shironghe/shironghe.github.io`
2. 推送本仓库到远程
3. 在 Settings → Pages 中把 Source 设为 GitHub Actions
4. 首次运行工作流后，站点发布到 `https://shironghe.github.io/`

## 开启评论

评论使用 Giscus，需要仓库开启 Discussions：

1. 在仓库 Settings → Discussions 启用 Discussions
2. 创建一个 Announcements 分类
3. 到 [giscus.app](https://giscus.app) 获取 `repoId` 和 `categoryId`
4. 填入 `hugo.toml` 的 `[params.giscus]` 后提交

## 许可证

- 代码：MIT，见 [LICENSE](LICENSE)
- 内容：CC BY-SA 4.0，见 [CONTENT-LICENSE.md](CONTENT-LICENSE.md)
