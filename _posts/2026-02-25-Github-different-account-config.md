---
title: GitHub 多账号 SSH 配置（避坑指南）
date: 2026-02-25 13:00:00 +0800
categories: [Git]
tags: [github, ssh, git, includeif, permission-denied-publickey, key-is-already-in-use]
---

在同一台电脑上同时使用两个 GitHub 账号（例如个人账号和工作账号）时，常见问题包括：

- `Permission denied (publickey)`
- push 到了错误账号
- 明明加了新 key，另一个仓库却无法认证

这篇文章给出一套稳定、长期可维护的配置方式。

---

## 一、问题根源

默认情况下，仓库地址通常是：

```bash
git@github.com:USER/REPO.git
```

它实际连接的是同一个 SSH 主机：

```bash
ssh git@github.com
```

如果你有多把 key 但没有显式区分，SSH 会按默认规则尝试 key，最终可能命中错误身份。  
正确做法是：在 `~/.ssh/config` 里为不同账号配置不同 `Host` 别名。

---

## 二、为两个账号生成独立 SSH Key

```bash
ssh-keygen -t ed25519 -C "personal-github" -f ~/.ssh/id_ed25519_github_personal
ssh-keygen -t ed25519 -C "work-github" -f ~/.ssh/id_ed25519_github_work
```

会得到：

```text
~/.ssh/id_ed25519_github_personal
~/.ssh/id_ed25519_github_personal.pub
~/.ssh/id_ed25519_github_work
~/.ssh/id_ed25519_github_work.pub
```

---

## 三、把公钥分别添加到对应 GitHub 账号

查看公钥内容：

```bash
cat ~/.ssh/id_ed25519_github_personal.pub
cat ~/.ssh/id_ed25519_github_work.pub
```

登录对应账号后，在 `https://github.com/settings/keys` 分别添加。  
注意：同一把公钥不能同时绑定到两个 GitHub 账号。

### 常见误区：把同一把 key 加到两个账号

GitHub 不允许同一把公钥绑定到多个账号。  
如果尝试重复绑定，通常会看到：

```text
Key is already in use
```

正确做法是：为每个账号生成并使用独立 key。

---

## 四、加载 key 到 ssh-agent

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519_github_personal
ssh-add ~/.ssh/id_ed25519_github_work
```

macOS 可用：

```bash
ssh-add --apple-use-keychain ~/.ssh/id_ed25519_github_personal
ssh-add --apple-use-keychain ~/.ssh/id_ed25519_github_work
```

在较新 macOS（如 Ventura 及之后版本）上，key 往往不会自动保存到钥匙串。  
建议显式使用 `--apple-use-keychain`，或在 `~/.ssh/config` 中配置 `UseKeychain yes`。

---

## 五、配置 `~/.ssh/config`（关键步骤）

编辑文件：

```bash
nano ~/.ssh/config
```

写入：

```ini
Host github-personal
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_github_personal
  IdentitiesOnly yes
  AddKeysToAgent yes

Host github-work
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_github_work
  IdentitiesOnly yes
  AddKeysToAgent yes
```

如果你是 macOS，且希望 key 自动保存到钥匙串，建议追加：

```ini
Host *
  IgnoreUnknown UseKeychain

Host github-personal github-work
  UseKeychain yes
```

这样即使复用同一份 `~/.ssh/config` 到 Linux，也不会因为 `UseKeychain` 报错。

如果你所在网络封锁了 SSH 22 端口，再切换为 443 备用方案（SSH over HTTPS）。  
只替换上面两个 `Host` 段中的以下字段，其余配置保持不变：

```ini
Host github-personal
  HostName ssh.github.com
  Port 443

Host github-work
  HostName ssh.github.com
  Port 443
```

设置权限：

```bash
chmod 600 ~/.ssh/config
chmod 600 ~/.ssh/id_ed25519_github_personal ~/.ssh/id_ed25519_github_work
chmod 644 ~/.ssh/id_ed25519_github_personal.pub ~/.ssh/id_ed25519_github_work.pub
```

---

## 六、测试连接

```bash
ssh -T git@github-personal
ssh -T git@github-work
```

如果配置正确，GitHub 会返回类似：

```text
Hi USERNAME! You've successfully authenticated, but GitHub does not provide shell access.
```

---

## 七、为不同仓库使用不同 remote

新克隆时：

```bash
# 个人账号
git clone git@github-personal:PERSONAL_USERNAME/REPO.git

# 工作账号
git clone git@github-work:WORK_USERNAME/REPO.git
```

已存在仓库可修改：

```bash
git remote set-url origin git@github-personal:PERSONAL_USERNAME/REPO.git
# 或
git remote set-url origin git@github-work:WORK_USERNAME/REPO.git
```

---

## 八、排查命令

```bash
# 查看当前仓库远端地址
git remote -v

# 查看实际使用了哪把 key
ssh -vT git@github-personal
ssh -vT git@github-work
```

---

## 九、补充：避免提交作者身份混淆

SSH 只决定“谁能 push”，不决定 commit 作者。  
建议在每个仓库单独设置：

```bash
git config user.name "Your Name"
git config user.email "your-email@example.com"
```

仓库数量较少时，这样逐仓库设置最直观。  
如果你有很多仓库，下一节 `includeIf` 更省心。

---

## 十、进阶：结合 includeIf 自动切换提交身份

如果你的目录结构固定（例如个人仓库放 `~/personal/`，工作仓库放 `~/work/`），可以让 Git 按目录自动切换提交身份。

在 `~/.gitconfig` 中添加：

```ini
[includeIf "gitdir:~/personal/"]
  path = ~/.gitconfig-personal

[includeIf "gitdir:~/work/"]
  path = ~/.gitconfig-work
```

然后创建 `~/.gitconfig-personal`：

```ini
[user]
  name = Your Personal Name
  email = your-personal-email@example.com
```

再创建 `~/.gitconfig-work`：

```ini
[user]
  name = Your Work Name
  email = your-work-email@example.com
```

`includeIf` 只会影响命中的目录。  
目录之外的仓库会使用全局 `~/.gitconfig` 里的 `user.name` / `user.email`（如果已设置）。

可在任意仓库里执行下面命令，确认当前实际命中的提交身份：

```bash
git config --show-origin --get-regexp '^user\.(name|email)$'
git var GIT_AUTHOR_IDENT
```

这样 SSH 身份和 commit 身份就能分别管理：  
- SSH 通过 remote host（`github-personal` / `github-work`）决定 push 账号
- commit 作者通过目录规则自动切换

---

## FAQ（常见问题）

### 1. 为什么我还是报 `Permission denied (publickey)`？

先检查三件事：

1. 公钥是否已添加到正确的 GitHub 账号
2. 仓库 remote 是否使用了正确 host（`github-personal` 或 `github-work`）
3. `ssh -vT git@github-personal` / `ssh -vT git@github-work` 是否命中预期 `IdentityFile`

### 2. 出现 `Key is already in use` 是什么原因？

同一把公钥已绑定到另一个 GitHub 账号。  
解决方式是为每个账号使用独立 key，而不是复用同一把 key。

### 3. 为什么 push 账号对了，但 commit 作者邮箱还是错的？

因为 SSH 只负责认证 push 权限，不决定 commit 作者。  
commit 作者由 `git config user.name` / `git config user.email` 决定，建议用第十节的 `includeIf` 自动切换。

---

## 总结

核心要点如下：

1. 每个 GitHub 账号使用独立 key
2. 每把公钥绑定到对应账号
3. `~/.ssh/config` 用不同 `Host` 显式区分身份
4. 仓库 remote 使用对应 `Host` 别名
5. 用 `includeIf` 让 commit 身份按目录自动切换
6. 仅在 22 端口受限时再启用 `ssh.github.com:443`

按这套方式配置后，两个账号可以长期稳定共存，不会互相冲突。
