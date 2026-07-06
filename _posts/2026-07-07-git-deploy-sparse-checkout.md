---
layout: post
title: "Git 部署时如何避免服务器拉取不必要的文件"
date: 2026-07-07 00:00:00 +0800
categories: [Git]
tags: [git, sparse-checkout, deploy, ci-cd]
---

在开发项目时，我们经常会遇到一种情况：本地仓库里既有源码，也有文档、设计稿、开发笔记等内容；但服务器部署时，其实只需要运行代码、配置文件和构建脚本，并不希望把 `docs/`、设计文档等非运行文件也拉到服务器上。

很多人第一反应是使用 `.gitignore`，但这个场景下 `.gitignore` 通常解决不了问题——如果文档已经被 Git 跟踪并提交到了远程仓库，服务器执行 `git pull` 时仍然会把这些文件拉下来。

更合适的方案是使用 **Git sparse checkout**，或者通过部署分支、CI/CD、`rsync` 等方式控制服务器上的文件范围。

------

## 问题场景

假设项目目录结构如下：

```bash
my-project/
├── cmd/
├── internal/
├── pkg/
├── config/
├── scripts/
├── docs/
├── README.md
├── Dockerfile
└── docker-compose.yml
```

其中：

- `cmd/`、`internal/`、`pkg/` 是程序代码；
- `config/`、`scripts/` 是部署需要的配置和脚本；
- `Dockerfile`、`docker-compose.yml` 是部署入口；
- `docs/` 是开发文档、设计文档、Review 记录等内容。

本地开发时，`docs/` 很重要，应该保留在仓库中。但服务器部署时，`docs/` 并不需要出现——尤其是一些内部设计文档、Review 报告等，没必要放在生产环境目录中。

核心诉求就是：**本地仓库保持完整，但服务器工作区只保留部署需要的文件。**

------

### 为什么 `.gitignore` 不适合这个场景

假设你已经提交过 `docs/`：

```bash
git add docs/
git commit -m "add project docs"
git push
```

即使后来在 `.gitignore` 中加入：

```gitignore
docs/
```

服务器执行 `git pull` 时，`docs/` 仍然会被拉下来。

原因是 `.gitignore` 只负责告诉 Git：哪些**未被跟踪的新文件**不要自动纳入版本控制。它不会让 Git 忽略已经被跟踪的文件，也不会影响远程仓库里已经存在的内容。

所以，只要文件已经提交进仓库，`.gitignore` 就不是正确工具。

------

## 推荐方案：Git sparse checkout

Git sparse checkout 的作用是：**让工作区只显示仓库中的一部分文件。**

远程仓库仍然是完整的，本地开发环境也可以保持完整；但服务器上的工作区可以只 checkout 部署需要的路径。

Git 官方文档对 sparse checkout 的描述是：将工作树从"包含所有已跟踪文件"改成"只包含部分已跟踪文件"。它控制的是服务器工作区里实际出现哪些文件，而不是改变远程仓库内容。

下面介绍两种具体用法。

------

### 方案一：排除模式（`--no-cone`）

如果你的目标只是让服务器不要出现 `docs/`，可以在服务器仓库中配置 sparse checkout 排除规则。

进入服务器上的项目目录：

```bash
cd /path/to/your/repo
```

设置 sparse checkout 规则：

```bash
git sparse-checkout set --no-cone '/*' '!/docs/'
```

这条命令的含义是：

- `/*` — 默认包含仓库根目录下的所有内容；
- `!/docs/` — 排除 `docs/` 目录。

之后执行：

```bash
git sparse-checkout reapply
git pull
```

此后，服务器工作区中就不会出现 `docs/` 目录。

#### 为什么这里要使用 `--no-cone`

Git sparse checkout 默认使用 **cone mode**。cone mode 更推荐用于"只选择某些目录"的场景，例如：

```bash
git sparse-checkout set cmd internal pkg config scripts
```

但如果要写类似 `!/docs/` 这样的**排除规则**，就需要使用 **non-cone mode**（`--no-cone`）。Git 官方文档也说明，`--no-cone` 模式允许使用类似 `.gitignore` 的 pattern，但有一些性能和兼容性方面的缺点，官方不推荐大量复杂使用。

所以，如果只是简单排除一两个目录，`--no-cone` 可以用；但如果是生产部署，更推荐下面的白名单方式。

------

### 方案二：白名单模式（cone mode）

相比"拉取全部，再排除某些文件"，更安全的做法是：

> 服务器只 checkout 明确需要部署的文件和目录。

例如一个 Go 后端项目，服务器可能只需要：

```
cmd/  internal/  pkg/  config/  scripts/
go.mod  go.sum  Dockerfile  docker-compose.yml
```

可以这样配置：

```bash
cd /path/to/your/repo

git sparse-checkout set \
  cmd \
  internal \
  pkg \
  config \
  scripts \
  go.mod \
  go.sum \
  Dockerfile \
  docker-compose.yml
```

这时 Git 使用默认的 cone mode，更符合 sparse checkout 的推荐用法。之后服务器正常更新：

```bash
git pull
```

服务器工作区只会出现你明确列出的路径，不会出现 `docs/`、`design/`、`notes/`、`testdata/` 这类不需要部署的内容。

------

### 两种模式怎么选

| 场景 | 推荐模式 |
|------|---------|
| 服务器保留大部分仓库内容，只是不想要 `docs/` | 排除模式（`--no-cone`） |
| 服务器只保留部署必需文件，其他都不要 | **白名单模式（cone mode）** |

从生产部署角度看，白名单方式更干净，也更安全——它默认不把新文件带到服务器上，以后如果新增了某个部署必需目录，再手动 `git sparse-checkout add` 加入即可。

------

## 实操指南

### 新仓库初始化

如果服务器还没有 clone 仓库，可以先 clone 再设置 sparse checkout：

```bash
git clone git@github.com:your-name/your-repo.git
cd your-repo

git sparse-checkout set \
  cmd \
  internal \
  pkg \
  config \
  scripts \
  go.mod \
  go.sum \
  Dockerfile \
  docker-compose.yml
```

如果仓库很大，也可以使用 `--sparse` 参数一步到位：

```bash
git clone --sparse git@github.com:your-name/your-repo.git
cd your-repo

git sparse-checkout set \
  cmd \
  internal \
  pkg \
  config \
  scripts \
  go.mod \
  go.sum \
  Dockerfile \
  docker-compose.yml
```

这样从一开始就以 sparse checkout 的方式使用仓库。

### 已有仓库改造

如果服务器上已经有完整仓库，也可以直接改成 sparse checkout。

先确认当前没有未提交修改：

```bash
git status
```

如果工作区干净，再执行：

```bash
git sparse-checkout set \
  cmd \
  internal \
  pkg \
  config \
  scripts \
  go.mod \
  go.sum \
  Dockerfile \
  docker-compose.yml
```

然后检查结果：

```bash
ls
git sparse-checkout list
```

如果某些文件因为冲突、未提交修改或外部工具写入而没有被正确清理，可以执行：

```bash
git sparse-checkout reapply
```

### 恢复完整仓库

如果以后不想继续使用 sparse checkout，执行一条命令即可恢复：

```bash
git sparse-checkout disable
```

这会关闭 `core.sparseCheckout` 配置，让工作区恢复完整文件。

------

## 注意事项

### 1. sparse checkout 只影响工作区，不等于删除仓库内容

启用 sparse checkout 后，文件只是不会出现在服务器工作区中，它们仍然存在于远程仓库和 Git 历史中。如果曾经把敏感信息提交到仓库里，sparse checkout **不能**解决泄露问题——敏感信息应该从 Git 历史中彻底清理，并重新轮换密钥。

### 2. `.gitignore` 不能阻止已跟踪文件被 pull

这是最容易混淆的点。`.gitignore` 适合忽略 `.env`、`node_modules/`、`dist/`、`*.log` 这类本地生成、还没有被 Git 跟踪的文件。但如果文件已经被提交过，`.gitignore` 不会让服务器停止拉取它们。

### 3. 不要随便排除所有 Markdown 文件

直接写 `!/*.md` 并不安全——项目可能依赖 `README.md`、`LICENSE`、`CHANGELOG.md` 等文件，或者某些构建工具会读取 Markdown 文件生成页面或版本说明。

更稳妥的做法是只排除明确不需要部署的目录，例如 `!/docs/`、`!/design/`、`!/notes/`，或者直接使用白名单方式。

### 4. 新增部署目录后要记得同步规则

如果后续项目新增了 `migrations/` 目录，但服务器 sparse checkout 规则里没有它，那么服务器工作区不会出现这个目录。这时需要执行：

```bash
git sparse-checkout add migrations
git pull
git sparse-checkout reapply
```

### 5. sparse checkout 不是 CI/CD 的替代品

sparse checkout 适合"服务器通过 `git pull` 部署"的模式。但更规范的生产部署通常是：CI 拉取完整代码 → 执行测试和构建 → 只把构建产物发布到服务器，服务器不直接保存完整开发仓库。sparse checkout 是一种很实用的过渡方案，但不是唯一方案。

------

## 其他可选方案

### 部署分支

维护一个专门用于部署的分支（如 `deploy`），只包含部署需要的文件：

```bash
git checkout -b deploy
git rm -r docs
git commit -m "prepare deploy branch"
git push origin deploy
```

服务器只拉取部署分支：

```bash
git checkout deploy
git pull origin deploy
```

**优点**：直观，服务器永远只跟随部署分支。  
**缺点**：维护成本高，开发分支和部署分支之间需要持续同步，容易出现漏合并或冲突。

### rsync 部署

如果不想让服务器直接执行 `git pull`，也可以使用 `rsync`：

```bash
rsync -av --delete \
  --exclude 'docs/' \
  --exclude 'design/' \
  --exclude 'notes/' \
  ./ user@server:/srv/app/
```

**优点**：简单直接，适合个人项目或轻量部署。  
**缺点**：需要自己控制同步规则，`--delete` 使用不当可能误删服务器文件。

### CI/CD 部署

更正式的做法是使用 CI/CD：

1. GitHub Actions 或 GitLab CI 拉取完整仓库；
2. 执行测试；
3. 构建 Docker 镜像或二进制文件；
4. 推送镜像到镜像仓库，或上传构建产物；
5. 服务器只拉取镜像或构建产物。

这种模式下，服务器根本不需要保存完整源码仓库，更适合正式生产环境。

------

## 总结

服务器部署时不想拉取文档或其他非运行文件，不能简单依赖 `.gitignore`——它只能忽略未被 Git 跟踪的新文件，不能阻止服务器拉取已经提交进仓库的内容。

更合适的方式是使用 **Git sparse checkout**，它让远程仓库和本地开发环境保持完整的同时，服务器工作区只保留部署需要的文件。

选择建议：

- **简单排除个别目录** → `git sparse-checkout set --no-cone '/*' '!/docs/'`
- **生产部署，只保留必需文件** → 白名单模式（推荐）

```bash
git sparse-checkout set \
  cmd \
  internal \
  pkg \
  config \
  scripts \
  go.mod \
  go.sum \
  Dockerfile \
  docker-compose.yml
```

如果项目已进入正式生产环境，长期来看更推荐 **CI/CD + 构建产物部署**，让服务器彻底告别完整源码仓库。

------

## 参考资料

- [Git 官方文档：git-sparse-checkout](https://git-scm.com/docs/git-sparse-checkout)
- [Git 官方文档：sparse-checkout](https://git-scm.com/docs/git-sparse-checkout#_sparse_checkout)
