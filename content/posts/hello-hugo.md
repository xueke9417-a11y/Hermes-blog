---
title: 第一篇：这个博客怎么搭起来的
date: 2026-10-30T20:00:00+08:00
tags: [Hugo, GitHub Pages, 公告]
author: xueke9417-a11y
draft: false
---

这是本博客的第一篇文章，也是一次发布链路的自检。

## 技术选型

- **Hugo**：单二进制静态生成，构建在秒级完成，适合放在 CI 里跑
- **GitHub Pages**：仓库即站点，免费，绑定自定义域名也方便
- **GitHub Actions**：推送即构建，不用提交本地构建产物

## 发布链路

```
写 Markdown → 提 PR → 合并 main → Actions 构建 → Pages 上线
```

关键点是中间那道 PR：它让每篇文章在合并前都有一次可讨论、可回滚的机会。

## 链接写法

项目页部署在子路径下（`/Hermes-blog/`），所以站内链接一律用 `relURL` 或 `.RelPermalink`：

```go-html-template
<a href='{{ "posts/" | relURL }}'>文章列表</a>
```

写死 `/posts/` 会在子路径部署下 404。

## 下一步

- 补 RSS 与站点地图的入口
- 视觉细节微调（代码块、表格、暗色模式）
- 把「发布前检查清单」固化进 CI
