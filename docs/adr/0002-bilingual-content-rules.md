# Bilingual content: Chinese default, English on demand

博客采用中英双语，中文为默认语言（URL 无前缀），英文内容位于 `/en/` 并挂在 `content/en` 下。中英文章通过 `translationKey` 配对；不强制每篇都成对发布，英文可以按需补译。这样避免翻译成为发布阻塞，同时保留统一的 URL 结构与语言切换体验。
