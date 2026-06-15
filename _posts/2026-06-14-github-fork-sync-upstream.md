---
layout: post
title: "GitHub Fork 仓库如何自动同步上游更新"
date: 2026-06-14 00:00:00 +0800
categories: [Development]
tags: [GitHub Actions, Fork, Git, 自动同步]
---

很多人在 GitHub 上 fork 一个开源仓库之后，会以为自己的 fork 会自动跟随原仓库更新。实际上，**GitHub 的 fork 默认不会自动同步上游仓库**。原仓库更新后，你的 fork 不会自动变化，除非手动同步或配置自动同步任务。

本文介绍如何通过 **GitHub Actions 实现 fork 的自动定时同步**，这是目前最省心、最可靠的方案。

## **一、Fork、Origin、Upstream 是什么？**

假设原仓库是：

```text
https://github.com/original-owner/original-repo
```

你 fork 之后得到：

```text
https://github.com/your-name/original-repo
```

在 Git 里，一般有两个远程仓库概念：

- `origin`   = 你自己的 fork 仓库
- `upstream` = 原始仓库，也就是被 fork 的仓库

同步 fork 的本质就是：

```text
从 upstream 拉取更新 → 合并到你的 fork → 推送到 origin
```

## **二、使用 GitHub Actions 自动同步**

如果你想让 fork 自动跟随上游仓库更新，可以使用 GitHub Actions 定时任务。

在你的 fork 仓库中创建文件：

```text
.github/workflows/sync-fork.yml
```

填入以下内容：

{% raw %}
```yaml
name: Sync fork with upstream

on:
  schedule:
    - cron: "0 3 * * *"
  workflow_dispatch:

permissions:
  contents: write

env:
  UPSTREAM_REPOSITORY: original-owner/original-repo
  UPSTREAM_BRANCH: main
  TARGET_BRANCH: main

jobs:
  sync:
    name: Sync fork
    runs-on: ubuntu-latest

    steps:
      - name: Checkout target branch
        uses: actions/checkout@v4
        with:
          ref: ${{ env.TARGET_BRANCH }}
          fetch-depth: 0

      - name: Configure Git
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"

      - name: Add upstream remote
        run: |
          git remote add upstream https://github.com/${{ env.UPSTREAM_REPOSITORY }}.git
          git remote -v

      - name: Fetch upstream
        run: |
          git fetch upstream ${{ env.UPSTREAM_BRANCH }}

      - name: Merge upstream into fork
        run: |
          git merge upstream/${{ env.UPSTREAM_BRANCH }}

      - name: Push changes
        run: |
          git push origin ${{ env.TARGET_BRANCH }}
```
{% endraw %}

需要修改的地方主要有三个环境变量：

| 字段                  | 含义                                     |
| --------------------- | ---------------------------------------- |
| `UPSTREAM_REPOSITORY` | 原仓库地址，不要带 `https://github.com/` |
| `UPSTREAM_BRANCH`     | 原仓库要同步的分支                       |
| `TARGET_BRANCH`       | 你的 fork 要更新的分支                   |

例如，原仓库是：

```text
https://github.com/someone/demo-project
```

你的 fork 想同步 `main` 分支，那么就写：

```yaml
env:
  UPSTREAM_REPOSITORY: someone/demo-project
  UPSTREAM_BRANCH: main
  TARGET_BRANCH: main
```

文件提交后，进入仓库的 **Actions** 标签页，手动触发一次 `Sync fork with upstream`，看到绿色对勾就说明配置成功了。

> **注意**：如果你的上游仓库默认分支是 `master` 而不是 `main`，请把上面所有 `main` 替换为 `master`。不确定的话，打开上游仓库页面，看分支下拉菜单默认选中的是哪个。

## **三、如果 fork 有自己的修改怎么办？**

如果你的 fork 不只是上游的副本，你自己也在上面提交过代码，那么自动同步就要多留个心眼。

例如：

```text
上游仓库更新了 A 文件
你的 fork 也修改了 A 文件
```

这时候可能产生冲突。上面的 workflow 使用 `git merge`，它会尝试自动合并——但冲突严重时仍然可能失败，需要你手动介入。

更稳妥的做法是：

1. `main` 分支只用于同步上游；
2. 自己的改动放在单独分支；
3. 定期从同步后的 `main` rebase 或 merge 到自己的开发分支。

例如：

```bash
git checkout my-feature
git rebase main
```

或者：

```bash
git checkout my-feature
git merge main
```

这样结构更清晰。

## **四、定时任务时间怎么写？**

这段配置表示每天 UTC 03:00 运行一次：

```yaml
on:
  schedule:
    - cron: "0 3 * * *"
```

GitHub Actions 的 cron 默认按 UTC 时间计算。

常见写法如下：

```yaml
# 每天 UTC 03:00
- cron: "0 3 * * *"

# 每 12 小时一次
- cron: "0 */12 * * *"

# 每周一 UTC 03:00
- cron: "0 3 * * 1"

# 每月 1 号 UTC 03:00
- cron: "0 3 1 * *"
```

不建议设置得太频繁，对于同步 fork 来说，每天一次通常已经足够。

## **五、手动触发同步**

workflow 里有这一段：

```yaml
workflow_dispatch:
```

它的作用是允许你手动运行同步任务。

添加之后，你可以进入：

```text
GitHub 仓库 → Actions → Sync fork with upstream → Run workflow
```

手动触发一次同步，在第一次测试时非常有用。

## **六、常见问题排查**

### 1. 提示没有权限 push

如果日志里出现类似：

```text
Permission denied
```

或者：

```text
403
```

一般是 GitHub Actions 没有写入权限。

检查 workflow 里是否有：

```yaml
permissions:
  contents: write
```

同时检查仓库设置：

```text
Settings → Actions → General → Workflow permissions
```

建议选择：

```text
Read and write permissions
```

如果仓库开启了分支保护规则，还要确认 GitHub Actions 是否允许推送到目标分支。

### 2. 定时任务没有运行

首先确认 workflow 文件在默认分支上。GitHub Actions 的 `schedule` 只会在默认分支上触发。

如果你的默认分支是 `main`，那么这个文件必须存在于：

```text
main 分支的 .github/workflows/sync-fork.yml
```

其次，公共仓库长时间没有活动时，GitHub 可能会自动停用定时 workflow。如果发现定时任务不运行，可以进入 Actions 页面重新启用 workflow。

### 3. merge 失败

如果日志里出现合并冲突，说明你的 fork 和上游仓库在同一处代码上有不同的修改。

常见原因是：

1. 你在 fork 的 `main` 分支上提交过自己的改动；
2. 上游仓库也更新了同一处代码；
3. 两边无法自动合并。

解决方法：

手动在本地解决冲突：

```bash
git fetch upstream
git checkout main
git merge upstream/main
```

如果有冲突，解决冲突后：

```bash
git add .
git commit
git push origin main
```

如果你完全不需要保留 fork 的改动，可以强制重置：

```bash
git fetch upstream
git checkout main
git reset --hard upstream/main
git push origin main --force
```

注意：`reset --hard` 和 `--force` 会覆盖你的 fork 分支历史。只有在你确定 fork 只是镜像仓库、不需要保留自己的提交时，才应该这样做。

## **七、如果上游是私有仓库怎么办？**

如果上游仓库是公开仓库，前面的配置可以直接使用。

如果上游仓库是私有仓库，普通的 HTTPS 地址可能无法拉取，这时需要使用 Personal Access Token（PAT）。

可以在 fork 仓库里添加 secret：

```text
Settings → Secrets and variables → Actions → New repository secret
```

例如添加：

```text
GH_PAT
```

然后把 checkout 和 fetch 相关配置改成使用 token。

示例：

{% raw %}
```yaml
- name: Checkout target branch
  uses: actions/checkout@v4
  with:
    ref: ${{ env.TARGET_BRANCH }}
    fetch-depth: 0
    token: ${{ secrets.GH_PAT }}
```
{% endraw %}

添加 upstream 时也可以写成：

{% raw %}
```bash
git remote add upstream https://x-access-token:${{ secrets.GH_PAT }}@github.com/${{ env.UPSTREAM_REPOSITORY }}.git
```
{% endraw %}

注意：不要把 token 直接写进 workflow 文件，必须放进 GitHub Secrets。

## **八、推荐实践**

如果你只是想让 fork 保持和原仓库一致，推荐：

```text
使用 GitHub Actions + git merge
```

优点是：

1. 不会因为 fork 轻微分叉就停止同步；
2. 出现真正冲突时会失败并提醒你处理；
3. 纯镜像 fork 场景下 merge 等价于 fast-forward，不会产生多余 commit。

如果你 fork 后还要自己开发，推荐：

```text
main 分支同步上游
自己的修改放到 feature 分支
```

例如：

```text
main        用来跟随 upstream/main
my-feature  你的自定义开发分支
```

这样以后维护会轻松很多。

## **九、总结**

GitHub fork 仓库默认不会自动同步上游更新。

如果想长期保持更新，推荐使用 GitHub Actions 自动同步：

{% raw %}
```yaml
name: Sync fork with upstream

on:
  schedule:
    - cron: "0 3 * * *"
  workflow_dispatch:

permissions:
  contents: write

env:
  UPSTREAM_REPOSITORY: original-owner/original-repo
  UPSTREAM_BRANCH: main
  TARGET_BRANCH: main

jobs:
  sync:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ env.TARGET_BRANCH }}
          fetch-depth: 0

      - run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"

          git remote add upstream https://github.com/${{ env.UPSTREAM_REPOSITORY }}.git
          git fetch upstream ${{ env.UPSTREAM_BRANCH }}

          git merge upstream/${{ env.UPSTREAM_BRANCH }}

          git push origin ${{ env.TARGET_BRANCH }}
```
{% endraw %}

这套方案适合大多数只想同步上游更新的 fork 仓库。

如果你的 fork 有大量自定义修改，就不要盲目自动合并，最好保留一个干净的 `main` 分支同步上游，把自己的开发放在单独分支里。
