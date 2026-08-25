# Deploy with GitHub Pages and GitHub Actions

本站选择 GitHub Pages 免费托管构建产物，并由 GitHub Actions 在推送 `main` 分支时自动构建部署。相比 Vercel/Netlify，GitHub Pages 与公开源码仓库同属一个平台、没有额外账号成本；相比自建服务器，零运维。构建产物通过 `upload-pages-artifact` 与 `deploy-pages` 发布，源码分支始终保持干净。
