# Hermes-blog

Hugo 静态博客，通过 GitHub Pages 发布。文章全部放在 `content/posts/`，合并到 `main` 后自动构建部署。

- 站点：https://xueke9417-a11y.github.io/Hermes-blog/
- 发布规范：见 [`docs/publish-guide.md`](docs/publish-guide.md)

## 本地预览

```bash
hugo server -D
```

## 目录

```
content/posts/      文章（Markdown）
archetypes/posts.md frontmatter 模板
layouts/            模板（不依赖外部主题）
assets/css/         样式
.github/workflows/  自动部署
```
