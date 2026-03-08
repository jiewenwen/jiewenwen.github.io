# FLYNOA Blog

个人博客，记录 AI 研究、应用开发与持续学习。  
线上地址：https://blog.flynoa.cc

本仓库已从 **GitHub Pages** 迁移到 **Cloudflare Pages**（Jekyll + Chirpy）。

## 技术栈

- Jekyll
- [jekyll-theme-chirpy](https://github.com/cotes2020/jekyll-theme-chirpy)
- Ruby 3.3+ / Bundler

## 本地开发

1. 安装依赖：

   ```bash
   bundle install
   ```

2. 启动开发服务（默认 `http://127.0.0.1:4000`，含 LiveReload）：

   ```bash
   bash tools/run.sh
   ```

3. 生产模式预览：

   ```bash
   bash tools/run.sh -p
   ```

4. 等价手动命令：

   ```bash
   bundle exec jekyll s -l -H 127.0.0.1
   ```

## 内容维护

- 文章目录：`_posts/`，文件名格式：`YYYY-MM-DD-title.md`
- 页面目录：`_tabs/`
- 站点配置：`_config.yml`
- 静态资源：`assets/`

推荐 Front Matter：

```markdown
---
layout: post
title: "文章标题"
date: 2026-02-03 10:00:00 +0800
categories: [AI, Notes]
tags: [LLM, Tools]
---
```

## 测试与构建

- 本地生产构建：

  ```bash
  JEKYLL_ENV=production bundle exec jekyll b
  ```

- 内容与链接校验（忽略外链）：

  ```bash
  bash tools/test.sh
  ```

## Cloudflare Pages 部署

> 本仓库不再使用 GitHub Pages / GitHub Actions 进行站点发布。

在 Cloudflare Pages 中建议使用以下配置：

- Production branch：`main`（如你实际使用其他分支，请对应调整）
- Build command：

  ```bash
  bundle install && JEKYLL_ENV=production bundle exec jekyll b
  ```

- Build output directory：`_site`

部署行为：

- 推送到生产分支会触发 Production 部署
- Pull Request 或非生产分支可触发 Preview 部署

迁移后建议检查：

- GitHub 仓库的 Pages 功能是否已关闭（避免双重部署）
- Cloudflare Pages 项目中是否已绑定自定义域名 `blog.flynoa.cc`

## 许可

- 代码与内容遵循根目录 `LICENSE`（MIT）。
