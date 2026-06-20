# Beyond Infra Blog

https://beyond-infra.github.io — 个人技术博客

## 技术栈

- **框架**: Jekyll + Chirpy 主题 v7.6
- **部署**: GitHub Actions (`.github/workflows/pages-deploy.yml`)
- **本地**: 不需要本地构建，所有 Jekyll 构建在 CI 完成
- **依赖**: 见 `Gemfile` (jekyll-theme-chirpy ~> 7.6)

## 添加新文章

1. 在 `_posts/` 创建 `YYYY-MM-DD-title.md`
2. 填写 front matter (title, date, categories, tags)
3. git add → git commit → git push
4. GitHub Actions 自动构建部署

## Front Matter 模板

```yaml
---
title: 文章标题
date: YYYY-MM-DD HH:MM:SS +0800
categories: [主分类]
tags: [标签1, 标签2]
---
```

## 目录结构

- `_posts/` — 博客文章 (Markdown)
- `_tabs/` — 导航页面 (关于、归档、分类、标签)
- `_data/` — 联系方式和分享配置
- `assets/img/` — 图片资源
- `assets/lib/` — 静态资源 (git submodule)
- `.github/workflows/` — CI/CD 工作流
