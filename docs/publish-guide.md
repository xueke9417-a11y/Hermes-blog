# 发布规范

本仓库是 Hugo 静态站，部署到 GitHub Pages。任何文章变更都必须经过 PR，合并到 `main` 后由 `.github/workflows/hugo.yml` 自动构建并部署。

## 文章位置

```
content/posts/<slug>.md
```

文件名用小写英文 + 连字符（如 `hugo-pages-deploy.md`），它就是 URL 的一部分。

## frontmatter

固定四个字段，缺一不可：

```yaml
---
title: 文章标题
date: 2026-10-30T20:00:00+08:00
tags: [Hugo, GitHub Pages]
author: xueke9417-a11y
---
```

- `title`：字符串，会用于 `<title>` 和列表页
- `date`：必须带时区（`+08:00`），否则按 UTC 解析会差 8 小时
- `tags`：数组，写法保持一致，用于 `/tags/` 分类页
- `author`：字符串

**唯一的例外字段：`draft`**。Hugo 默认 `draft: true`，正文不写 `draft: false` 时生产构建会跳过该文，页面不会出现。因此定稿文章必须显式写 `draft: false`；草稿 PR 保留 `true`，可以本地预览但不会发布。

## 发布流程

1. 从最新 `main` 切分支：`git switch -c post/<slug> origin/main`
2. 把文章写入 `content/posts/<slug>.md`
3. 自检：`hugo server -D` 看渲染，`hugo --gc --minify` 看构建无报错
4. 提交并推送，开 PR 到 `main`，PR 描述里写清标题、slug、tags 和预览情况
5. 合并到 `main` → Actions 触发 `Deploy Hugo site to Pages` → 站点更新

## 部署说明

- 部署由 GitHub Actions 完成（Pages 的 Source 必须是 **GitHub Actions**，不是 `gh-pages` 分支）
- 站点根路径是 `/Hermes-blog/`，站内链接统一用 `relURL` / `RelPermalink`，不要写死根路径 `/`
- 推送成功 ≠ 部署成功：以 Actions 运行结论和线上 URL 实际可访问为准
