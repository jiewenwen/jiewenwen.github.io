---
title: 在同一台电脑上配置两个 GitHub 账号（SSH Key）
date: 2026-02-25 13:00:00 +0800
categories: [Git]
tags: [github, ssh, git]
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

---

## 五、配置 `~/.ssh/config`（关键步骤）

编辑文件：

```bash
nano ~/.ssh/config
```

写入：

```sshconfig
Host github-personal
  HostName ssh.github.com
  Port 443
  User git
  IdentityFile ~/.ssh/id_ed25519_github_personal
  IdentitiesOnly yes
  AddKeysToAgent yes

Host github-work
  HostName ssh.github.com
  Port 443
  User git
  IdentityFile ~/.ssh/id_ed25519_github_work
  IdentitiesOnly yes
  AddKeysToAgent yes
```

如果你是 macOS，且希望 key 自动保存到钥匙串，可在两个 `Host` 段里额外加上：

```sshconfig
UseKeychain yes
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

---

## 总结

核心只有四点：

1. 每个 GitHub 账号使用独立 key
2. 每把公钥绑定到对应账号
3. `~/.ssh/config` 用不同 `Host` 显式区分身份
4. 仓库 remote 使用对应 `Host` 别名

按这套方式配置后，两个账号可以长期稳定共存，不会互相冲突。
