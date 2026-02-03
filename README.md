# FLYNOA Blog

个人博客，记录 AI 研究、应用开发与持续学习。线上访问：https://blog.flynoa.cc （GitHub Pages + Jekyll Theme Chirpy 7.4）。

## 快速开始
- 依赖：Ruby 3.3+、Bundler；可选 html-proofer 用于链接校验。
- 安装：`bundle install`
- 本地预览：`bash tools/run.sh`（默认 http://127.0.0.1:4000，开启 LiveReload）；生产模式追加 `-p`。
- 手动命令：`bundle exec jekyll s -l -H 127.0.0.1`

## 写作
- 新文章放在 `_posts/`，命名 `YYYY-MM-DD-title.md`。
- 推荐 Front Matter：
```markdown
---
layout: post
title: "文章标题"
date: 2026-02-03 10:00:00 +0800
categories: [AI, Notes]
tags: [LLM, Tools]
---
```
- 导航页位于 `_tabs/`，站点配置在 `_config.yml`；图片与头像放在 `assets/`。

## 部署与测试
- Push 到 `main` 或 `master` 会触发 `.github/workflows/pages-deploy.yml`，使用 Ruby 3.3 构建 `JEKYLL_ENV=production` 并通过 `actions/deploy-pages` 发布。
- 本地生产构建：`JEKYLL_ENV=production bundle exec jekyll b`
- 内容与链接校验：`bash tools/test.sh`（基于 html-proofer，忽略外部链接）

## 许可
- 代码与内容遵循根目录 `LICENSE`（MIT）。
- 主题基于 [jekyll-theme-chirpy](https://github.com/cotes2020/jekyll-theme-chirpy)。
