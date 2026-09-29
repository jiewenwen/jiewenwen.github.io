---
layout: post
title: "MacOS + Debian 双机开发架构完整实施指南"
date: 2026-09-29 00:00:00 +0800
categories: [Development]
tags: [Mac mini, Debian, Codex, 双机开发, 长时间测试, VS Code, rsync, Docker, 架构设计]
---

**适用场景**：Mac mini 作为主开发机，Debian 台式机作为 Linux 运行时、后端服务与长时间测试节点；使用 VS Code + Codex 同时开发 SwiftUI/iOS 与完整后端；iOS 与 backend 为两个独立 Git 仓库；两台机器通过独立网线直连，同时各自可经 Wi‑Fi/VPN 上网；完整回归测试可持续 15 小时以上。  
**重要原则**：源码、Codex 上下文与跨前后端产品视图集中在 Mac；Linux 运行时、数据库、容器、后端测试与长时间计算尽量卸载到 Debian。  
**执行提示**：文中命令可直接复制，但示例 IP、用户名、路径、版本号与密钥均须按实际环境替换；启用直连网段前，先确认与现有 VPN/局域网无地址冲突。

---

## 1. 先读结论

这套架构只遵循一个核心原则：

> **源码、Codex 上下文和跨前后端产品视图集中在 Mac；Linux 运行时、数据库、容器、后端测试和长时间计算尽量卸载到 Debian。**

最终职责：

```text
Mac mini
=
源码 Source of Truth
+ VS Code / Codex
+ ios repo
+ backend repo
+ Xcode / Simulator
+ XCTest / XCUITest
+ 产品级编排与测试控制面

Debian PC
=
Backend Runtime
+ Docker / Compose
+ PostgreSQL / Redis / Queue / Object Storage
+ Backend Unit / Integration / API E2E
+ 长时间回归测试
+ Build/Test Cache
+ 测试快照与 Artifacts
+ Linux 计算节点
```

不要把它做成：

```text
Mac 上开发 iOS
Debian 上开发 backend
```

而应做成：

```text
Mac 一个完整产品工作区
        │
        ├── ios/
        └── backend/
              │
              │ rsync / SSH
              ▼
        Debian 执行副本
```

---


## 2. 适用范围与假设

本文默认：

- Mac mini 是日常坐在面前使用的主机。
- VS Code 和 Codex 主要运行在 Mac。
- iOS/SwiftUI 和 backend 是两个独立 Git 仓库。
- 后端可以在 Linux 上完整运行。
- Xcode、Simulator、签名和 Apple 平台测试必须留在 Mac。
- Debian 台式机可以长期在线。
- Debian 通过 Wi‑Fi 独立访问 Internet/VPN。
- Mac 与 Debian 之间有一根独立 Ethernet 直连线。
- 两台机器可能同时开启 VPN。
- 长时间测试期间仍希望继续开发下一项任务。
- Debian 是开发/测试机器，不承载生产密钥或不可替代数据。

示例名称统一使用：

```text
产品名:
MyProduct

Mac 产品根目录:
~/Developer/MyProduct

Debian:
devbox

示例直连网络:
10.77.0.0/30

Mac:
10.77.0.1

Debian:
10.77.0.2
```

**注意：`10.77.0.0/30` 只是示例。**

如果 VPN、公司网络或家庭网络已经使用这一段地址，必须换一个不冲突的 RFC1918 私网 `/30`。

---

## 3. 架构原则

### 3.1 Source of Truth 只有 Mac 一份

正常开发过程中：

```text
Mac ios/
Mac backend/
```

才是主源码。

Debian：

```text
/srv/myproduct/dev/backend
```

是可删除、可重建的执行 mirror。

禁止形成：

```text
Mac backend 有未提交修改
+
Debian backend 也有人为修改
```

否则很快出现双主问题。

---

### 3.2 Codex 看到的是“产品”，不是“两台电脑”

Codex 日常看到：

```text
MyProduct/
├── AGENTS.md
├── ios/
├── backend/
├── scripts/
└── docs/
```

它修改：

```text
backend migration
backend DTO
backend endpoint
ios DTO
ios networking
SwiftUI View
tests
```

都在同一个 VS Code 窗口中完成。

Debian 对 Codex 应表现为：

> 一个由脚本调用的远程执行后端，而不是另一个 IDE workspace。

---

### 3.3 Git 与执行同步是两件不同的事

Git：

```text
版本历史
PR
release
审计
CI
```

rsync：

```text
把当前 working tree 快速送到 Debian 执行
```

开发态不要求：

```text
commit
push
pull
```

未提交的代码也应能直接测试。

---

### 3.4 短测试可以使用 live mirror，长测试必须 immutable

规则建议：

```text
< 5 分钟:
可在 live mirror 运行

5–10 分钟:
视团队习惯决定

> 10 分钟:
建议 snapshot

Release / Full Regression:
必须 snapshot
```

15 小时的完整测试绝不能使用会被 rsync 持续更新的目录。

---

### 3.5 Full Regression 必须同时冻结 iOS 和 backend

这是一条非常关键的修订。

错误：

```text
Debian backend = snapshot
Mac ios = 当前活跃工作区
```

如果 iOS full test 的后续 suite 会触发重新构建，继续编辑代码仍可能改变测试输入。

正确：

```text
同一个 RUN_ID

Mac:
ios/input
ios/work

Debian:
backend/input
backend/work
```

Mac 的日常：

```text
~/Developer/MyProduct/ios
```

继续开发，不影响 `ios/work` 中正在运行的 full test。

---

## 4. 最终拓扑

```text
                                Internet
                           ┌────────┴────────┐
                           │                 │
                     Mac Wi‑Fi/VPN    Debian Wi‑Fi/VPN
                           │                 │
┌──────────────────────────▼─────────────────┐
│                  Mac mini                  │
│                                            │
│ VS Code + Codex                            │
│                                            │
│ ~/Developer/MyProduct                     │
│ ├── AGENTS.md                              │
│ ├── scripts/                               │
│ ├── docs/                                  │
│ ├── ios/.git       ← Repo A                │
│ └── backend/.git   ← Repo B                │
│                                            │
│ Xcode / Simulator                          │
│                                            │
│ Full run snapshot:                         │
│ ~/Library/Caches/MyProduct/test-runs/      │
│                                            │
│ 10.77.0.1/30                               │
└──────────────────────────┬─────────────────┘
                           │
                           │ Ethernet only
                           │ SSH / rsync / API
                           │
┌──────────────────────────▼─────────────────┐
│                 Debian PC                  │
│                                            │
│ 10.77.0.2/30                               │
│                                            │
│ /srv/myproduct/                            │
│ ├── dev/backend/       ← live mirror       │
│ ├── state/                                 │
│ ├── config/                                │
│ ├── cache/                                 │
│ ├── tools/                                 │
│ ├── artifacts/                             │
│ └── test-runs/<RUN_ID>/                    │
│       └── backend/                         │
│           ├── input/                       │
│           ├── work/                        │
│           ├── logs/                        │
│           └── artifacts/                   │
│                                            │
│ Docker / Compose                           │
│ PostgreSQL / Redis / Queue                 │
│ Unit / Integration / API E2E               │
└────────────────────────────────────────────┘
```

---

## 5. Mac 产品工作区

推荐目录：

```text
~/Developer/MyProduct/
├── .git/                      # 推荐但可选：轻量 workspace/ops repo
├── .gitignore
├── AGENTS.md
├── README.md
├── .vscode/
│   ├── settings.json
│   └── tasks.json
├── scripts/
│   ├── lib/
│   ├── rsync-backend.exclude
│   ├── rsync-ios-snapshot.exclude
│   ├── dev-status
│   ├── workspace-state
│   ├── sync-backend
│   ├── watch-backend
│   ├── dev-up
│   ├── dev-down
│   ├── backend-logs
│   ├── test-backend-fast
│   ├── test-ios
│   ├── test-all
│   ├── create-full-run
│   ├── snapshot-ios
│   ├── snapshot-backend
│   ├── run-ios-full-snapshot
│   ├── launch-backend-full
│   ├── run-status
│   └── fetch-artifacts
├── docs/
│   ├── architecture.md
│   ├── testing.md
│   └── operations.md
├── ios/
│   └── .git/
└── backend/
    └── .git/
```

---

## 6. 是否建立外层 `.git`

### 推荐做法

建立一个非常小的 workspace/ops repo：

```text
MyProduct/.git
```

只管理：

```text
AGENTS.md
.vscode/
scripts/
docs/
README.md
integration-lock.env
```

`.gitignore`：

```gitignore
ios/
backend/
artifacts/
.env.local
.DS_Store
```

这样：

```text
Workspace Repo
= 双机开发基础设施 + 产品级说明

ios Repo
= iOS 源码

backend Repo
= 后端源码
```

优点：

- 双机脚本有版本历史。
- Codex 规则可审计。
- 可以保存一对“已验证兼容”的 iOS/backend SHA。
- 不需要 Git submodule。

如果不想增加第三个 Git repo，也可以不建，但 `scripts/` 和顶层 `AGENTS.md` 最好仍有其它备份方式。

---

## 7. VS Code：默认使用单一父目录 Workspace

推荐：

```bash
cd ~/Developer/MyProduct
code .
```

而不是长期分别打开：

```bash
code ios
code backend
```

也不把 Remote-SSH 窗口作为主要编码窗口。

采用父目录单 workspace 的原因：

1. Codex 从同一目录树看到前后端。
2. 顶层 `AGENTS.md` 可以覆盖两个 repo。
3. 跨端 grep/search 更自然。
4. VS Code 可以同时管理多个 Git repository。
5. VS Code 本身支持 multi-root，但扩展必须正确适配 multi-root；为减少扩展差异，单父目录是这里更简单的默认方案。[R1]

---

## 8. VS Code 设置

创建：

```text
.vscode/settings.json
```

示例：

```json
{
  "git.autoRepositoryDetection": "subFolders",
  "git.repositoryScanMaxDepth": 3,
  "scm.alwaysShowRepositories": true,

  "files.watcherExclude": {
    "**/.git/objects/**": true,
    "**/.git/subtree-cache/**": true,
    "**/DerivedData/**": true,
    "**/node_modules/**": true,
    "**/.build/**": true,
    "**/.venv/**": true,
    "**/dist/**": true,
    "**/coverage/**": true
  },

  "search.exclude": {
    "**/DerivedData": true,
    "**/node_modules": true,
    "**/.build": true,
    "**/.venv": true,
    "**/dist": true,
    "**/coverage": true
  }
}
```

VS Code 官方支持在同一个 workspace 中管理多个 repository。[R2]

---

## 9. Codex 上下文策略

Codex IDE extension 可以利用 workspace、已打开文件和选中代码等本地上下文，因此本架构优先保持完整产品目录在同一个 IDE 工作区中。[R3]

---

### 9.1 顶层 `AGENTS.md`

示例：

```md
# MyProduct Engineering Instructions

This directory represents one product with two independent Git repositories.

## Repository layout

- `ios/`
  - SwiftUI/iOS client.
  - Source of truth is on this Mac.
  - Build with Xcode on macOS.
  - Use Simulator/XCTest/XCUITest on macOS.

- `backend/`
  - Backend source.
  - Source of truth is on this Mac.
  - Runtime and most backend tests execute on Debian.

## Cross-stack changes

For API/schema/model changes:

1. Inspect backend producer.
2. Inspect iOS consumer.
3. Update both repos when required.
4. Update tests on both sides.
5. Verify contract compatibility before considering the task complete.

## Remote execution

Do not manually edit files under `/srv/myproduct/dev/backend` on Debian.

Use only the workspace scripts:

- `./scripts/sync-backend`
- `./scripts/dev-up`
- `./scripts/test-backend-fast`
- `./scripts/test-all`
- `./scripts/create-full-run`

## Safety

Never:
- run `sudo` on Debian unless the user explicitly requests an administrative task;
- access production credentials;
- run `docker system prune -a`;
- delete `/srv/myproduct` recursively;
- modify firewall/routing configuration as part of an ordinary coding task;
- use Debian as a second Git source of truth.

## Testing

For ordinary backend changes:
`./scripts/test-backend-fast`

For iOS changes:
`./scripts/test-ios`

For cross-stack changes:
run both relevant test commands.

For long/full regression:
use immutable snapshots via:
`./scripts/create-full-run`
```

OpenAI 官方说明 `AGENTS.md` 可用于向 Codex 提供持久的代码库与测试指令，并按目录层级作用于文件树。[R4]

---

### 9.2 Repo 内各自再放 `AGENTS.md`

```text
ios/AGENTS.md
backend/AGENTS.md
```

顶层写产品规则；子 repo 写技术栈规则。

#### `ios/AGENTS.md` 应包含

- Swift/SwiftUI 版本。
- 最低部署版本。
- 项目架构。
- State management。
- Networking 层。
- DTO/Domain model 规则。
- 测试 scheme/test plan。
- `xcodebuild` 统一入口。
- 哪些生成代码不能手改。
- 签名/证书不得由 Codex 擅自修改。
- Debug API base URL 如何注入。

#### `backend/AGENTS.md` 应包含

- 语言与框架。
- 格式化/lint。
- 单测。
- integration。
- API E2E。
- migration。
- fixture。
- Docker/Compose。
- OpenAPI/GraphQL/protobuf 等 contract 规则。
- 统一 `scripts/test-*` 命令。
- Debian mirror 禁止直接编辑。

---

## 10. 两个独立 Git Repo 的“产品状态”

只有同一 VS Code workspace 还不够。

因为：

```text
ios HEAD     = A
backend HEAD = B
```

是两个独立历史。

需要一个产品级状态概念。

---

### 10.1 `workspace-state`

创建：

```text
scripts/workspace-state
```

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
ROOT_DIR="$(cd "$SCRIPT_DIR/.." && pwd)"

print_repo() {
  local name="$1"
  local dir="$2"

  echo "[$name]"
  echo "path=$dir"
  echo "head=$(git -C "$dir" rev-parse HEAD)"
  echo "branch=$(git -C "$dir" symbolic-ref --quiet --short HEAD || echo DETACHED)"

  if git -C "$dir" diff --quiet &&
     git -C "$dir" diff --cached --quiet &&
     [[ -z "$(git -C "$dir" ls-files --others --exclude-standard)" ]]; then
    echo "dirty=false"
  else
    echo "dirty=true"
  fi

  echo "[status]"
  git -C "$dir" status --porcelain=v1 --untracked-files=all
  echo
}

echo "created_at=$(date -u '+%Y-%m-%dT%H:%M:%SZ')"
echo

print_repo "ios" "$ROOT_DIR/ios"
print_repo "backend" "$ROOT_DIR/backend"
```

授权：

```bash
chmod +x scripts/workspace-state
```

使用：

```bash
./scripts/workspace-state
```

---

### 10.2 已验证兼容版本：`integration-lock.env`

对于 release、稳定版本或 CI，可以记录：

```text
IOS_SHA=...
BACKEND_SHA=...
API_CONTRACT_SHA256=...
VERIFIED_AT=...
```

这不是开发态同步工具，而是：

> “这一对前后端 commit 已通过产品级验证”的记录。

它解决两个独立 repo 没有天然原子 commit 的问题。

---

## 11. API Contract 建议

如果 backend 已有：

- OpenAPI
- GraphQL schema
- protobuf
- JSON Schema

应把 contract 验证加入跨端 test。

推荐：

```text
Backend
   ↓
export schema
   ↓
schema hash
   ↓
iOS client/model generation or compatibility check
```

原则：

> API contract 变化必须在同一 Codex task 中检查 iOS 消费者。

如果暂时没有机器可读 contract，也至少保持：

```text
docs/api/
```

和跨端 integration tests。

---

## 12. 网络：先检查地址冲突，再设 `/30`

不要直接假定 `10.77.0.0/30` 一定可用。

Mac 开 VPN 后先看：

```bash
netstat -rn -f inet
```

Debian：

```bash
ip route
```

确认没有已有：

```text
10.77.0.0/30
10.77.0.0/16
10.0.0.0/8
```

以及其它会被 VPN 策略覆盖、进而影响目标地址的私网网段。

如果冲突，换另一段私网。

示例继续使用：

```text
Network:
10.77.0.0/30

Mac:
10.77.0.1

Debian:
10.77.0.2
```

---

## 13. 网络职责分离

```text
Mac Wi‑Fi
└── Internet / VPN

Mac Ethernet
└── Debian

Debian Wi‑Fi
└── Internet / VPN

Debian Ethernet
└── Mac
```

直连 Ethernet：

- 不设置默认网关。
- 不设置 DNS。
- 不承担 Internet。
- MTU 初期保持 1500。

按最长前缀匹配原则，路由应保证：

```text
10.77.0.0/30
```

直接走网线。

VPN 即使接管默认路由，也不应该改变这一点；但部分 VPN 的 kill switch / LAN blocking 仍可能主动阻断本地网络。

---

## 14. Mac Ethernet 配置

优先使用：

```text
系统设置
→ 网络
→ Ethernet
→ 详细信息
→ TCP/IP
```

设置：

```text
Configure IPv4:
Manually

IP Address:
10.77.0.1

Subnet Mask:
255.255.255.252

Router:
空
```

DNS：

```text
不要为该接口配置 DNS
```

检查：

```bash
networksetup -listallnetworkservices
networksetup -getinfo "Ethernet"
ifconfig
route -n get 10.77.0.2
```

目标：

```text
route -n get 10.77.0.2
```

输出应显示 Ethernet 对应的接口。

---

## 15. Debian Ethernet 配置

查看：

```bash
nmcli connection show
ip link
```

假设连接名：

```text
Wired connection 1
```

配置：

```bash
sudo nmcli connection modify "Wired connection 1" \
  ipv4.method manual \
  ipv4.addresses 10.77.0.2/30 \
  ipv4.gateway "" \
  ipv4.never-default yes
```

可选关闭该直连接口 IPv6：

```bash
sudo nmcli connection modify "Wired connection 1" \
  ipv6.method disabled
```

重新连接：

```bash
sudo nmcli connection down "Wired connection 1"
sudo nmcli connection up "Wired connection 1"
```

检查：

```bash
ip addr
ip route
ip route get 10.77.0.1
ip route get 1.1.1.1
```

期望：

```text
10.77.0.1 → Ethernet
1.1.1.1   → Wi‑Fi / VPN
```

---

## 16. 网络验收

Mac：

```bash
ping 10.77.0.2
```

Debian：

```bash
ping 10.77.0.1
```

Debian：

```bash
sudo apt install -y iperf3
iperf3 -s
```

Mac：

```bash
iperf3 -c 10.77.0.2
```

测试以下四种状态：

```text
1. 两边 VPN 都关
2. 只有 Mac VPN 开
3. 只有 Debian VPN 开
4. 两边 VPN 都开
```

以上四种组合下，下列命令均应成功：

```bash
ping 10.77.0.2
ssh devbox true
```

---

## 17. macOS Local Network Privacy

较新的 macOS 同样有本地网络隐私控制（Local Network Privacy）。

如果 Terminal、VS Code 或其它负责发起本地网络连接的应用被系统阻止，需要检查：

```text
System Settings
→ Privacy & Security
→ Local Network
```

Apple 当前文档说明 macOS 15 及以后也应用本地网络隐私控制。[R5]

排查时不要在路由正常的情况下只盯着 VPN；本地网络权限同样可能是问题来源。

---

## 18. VPN 设计

推荐：

```text
Mac VPN
→ Mac Internet

Debian VPN
→ Debian Internet
```

不要：

```text
Mac
→ Debian
→ VPN
→ Internet
```

否则：

- Debian VPN 故障会拖累 Mac。
- NAT/DNS/route 更复杂。
- Codex/SSH 故障诊断更困难。

如果 VPN 有以下设置：

```text
Allow LAN
Local Network Access
Bypass LAN
Split Tunnel
Exclude Routes
```

应保证直连网段被允许。

---

## 19. SSH：先建立可信连接

Debian：

```bash
sudo apt update
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
```

检查服务器 host key fingerprint：

```bash
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

在 Mac 上第一次连接时，对照该 fingerprint 确认。

不要为了省事设置：

```text
StrictHostKeyChecking no
```

---

## 20. SSH Key

Mac：

```bash
ssh-keygen \
  -t ed25519 \
  -f ~/.ssh/id_ed25519_devbox
```

建议设置 passphrase。

复制公钥：

```bash
cat ~/.ssh/id_ed25519_devbox.pub | \
ssh <DEV_USER>@10.77.0.2 \
'umask 077; mkdir -p ~/.ssh; cat >> ~/.ssh/authorized_keys'
```

---

## 21. SSH Config

Mac：

```text
~/.ssh/config
```

```sshconfig
Host devbox
    HostName 10.77.0.2
    User <DEV_USER>
    IdentityFile ~/.ssh/id_ed25519_devbox
    IdentitiesOnly yes

    ServerAliveInterval 30
    ServerAliveCountMax 3

    ControlMaster auto
    ControlPersist 10m
    ControlPath ~/.ssh/cm-%C
```

如果希望 macOS Keychain 管理 key，可按当前系统 OpenSSH 能力增加：

```sshconfig
    AddKeysToAgent yes
    UseKeychain yes
```

日常：

```bash
ssh devbox
```

自动化脚本建议使用：

```bash
ssh -o BatchMode=yes devbox ...
```

这样 key/网络出错会立即失败，而不是在后台任务里等待密码输入。

---

## 22. SSH 加固

**确认 key 登录成功后再改。**

Debian：

```text
/etc/ssh/sshd_config.d/10-devbox.conf
```

建议：

```text
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitEmptyPasswords no
```

可选：

```text
AllowUsers <DEV_USER>
```

验证：

```bash
sudo sshd -t
```

然后：

```bash
sudo systemctl restart ssh
```

保留原 SSH session，新开 Terminal 测试：

```bash
ssh devbox
```

新连接成功后再退出旧 session。

---

## 23. Debian 执行账号模型

如果 Codex 可以通过 Mac 脚本调用：

```text
ssh devbox
docker
tests
```

应明确权限边界。

推荐至少区分：

```text
admin user:
人工管理 Debian
有 sudo

dev user:
Mac/Codex 使用
日常测试/容器
默认不使用 sudo
```

但必须理解：

> **标准 Docker 的 `docker` group 本身具有接近 root 的权限。** Docker 官方也明确警告这一点。[R6]

因此有两种模式。

---

### 23.1 实用模式

Debian 是专用开发机：

```text
dev user
+ docker group
```

接受它具备高权限，但保证：

- 机器不存生产密钥。
- 不挂载私人重要目录到容器。
- 不把 SSH agent 转发进去。
- 不给它云生产管理员 token。
- AGENTS.md 中明确禁止 Codex 执行 sudo 或破坏性管理命令。

这是最简单、兼容性最好的模式。

---

### 23.2 更强隔离模式

使用：

```text
Rootless Docker
```

或：

```text
VM / disposable runner
```

适合：

- 需要运行来源不完全可信的代码。
- Agent 自动化程度很高。
- 想把 Debian 主系统当真正安全边界。

本文后续命令默认使用普通 Docker Engine。

---

## 24. Debian 基础包

```bash
sudo apt update

sudo apt install -y \
  openssh-server \
  rsync \
  git \
  curl \
  ca-certificates \
  tmux \
  jq \
  iperf3 \
  sysstat
```

检查版本：

```bash
cat /etc/os-release
rsync --version
git --version
```

---

## 25. Docker 安装

截至本文发布时，Docker 官方 Debian 安装文档支持 Debian 13 与 Debian 12，并推荐使用官方 apt repository。[R7]

先检查是否已有 Docker 环境，不要盲目删除：

```bash
docker version || true
dpkg -l | grep -E 'docker|containerd|runc' || true
```

新安装时参考 Docker 官方当前文档。

核心安装包为：

```text
docker-ce
docker-ce-cli
containerd.io
docker-buildx-plugin
docker-compose-plugin
```

安装后：

```bash
sudo systemctl status docker
sudo docker run --rm hello-world
docker compose version
```

如果把用户加入 `docker` group：

```bash
sudo usermod -aG docker "$USER"
```

必须完整注销并重新登录。

再次强调：

```text
docker group ≈ root-level privilege
```

---

## 26. Docker 日志必须限制

Docker 默认 `json-file` 日志如果未配置 rotation，长期高输出测试可能写满磁盘。Docker 官方推荐使用有轮转的 `local` driver，或显式给 `json-file` 设置大小限制。[R8]

推荐：

```text
/etc/docker/daemon.json
```

如果文件没有其它配置：

```json
{
  "log-driver": "local"
}
```

如果已有 `daemon.json`，合并配置，不要覆盖已有内容。

验证 JSON：

```bash
sudo jq . /etc/docker/daemon.json
```

重启：

```bash
sudo systemctl restart docker
```

检查：

```bash
docker info --format '{{.LoggingDriver}}'
```

注意：

> 修改默认 log driver 只影响之后新建的容器。

---

## 27. Debian 目录规划

统一：

```text
/srv/myproduct/
├── dev/
│   └── backend/
├── state/
├── config/
├── cache/
├── tools/
├── artifacts/
└── test-runs/
```

创建：

```bash
sudo mkdir -p \
  /srv/myproduct/dev/backend \
  /srv/myproduct/state \
  /srv/myproduct/config \
  /srv/myproduct/cache \
  /srv/myproduct/tools \
  /srv/myproduct/artifacts \
  /srv/myproduct/test-runs

sudo chown -R "$USER":"$USER" /srv/myproduct
```

职责：

```text
dev/
= 可删除 live mirror

state/
= sync/run 状态

config/
= Debian 本地配置和开发 secret

cache/
= build/package cache

tools/
= Debian runner scripts

artifacts/
= 长期需要保留的结果

test-runs/
= immutable/full test 工作目录
```

---

## 28. Secret 原则

不要从 Mac working tree rsync：

```text
.env
private key
production token
signing material
```

Debian：

```text
/srv/myproduct/config/backend.env
```

权限：

```bash
chmod 600 /srv/myproduct/config/backend.env
```

开发机只放：

- 本地开发 credential。
- sandbox credential。
- mock credential。

不要放生产管理员凭据。

测试 artifact 同样可能包含：

- access token。
- response body。
- 用户测试数据。

因此 artifact 目录也应：

```bash
chmod 700 /srv/myproduct/artifacts
chmod 700 /srv/myproduct/test-runs
```

---

## 29. rsync 版本

Mac 自带 rsync 的版本与能力可能和 Homebrew 当前版本不同。

先：

```bash
rsync --version
```

本文脚本使用：

```text
--delete-delay
--delay-updates
--checksum
--itemize-changes
```

如果本机 rsync 不支持所需参数：

```bash
brew install rsync
```

然后：

```bash
export RSYNC_BIN="$(brew --prefix)/bin/rsync"
```

脚本均允许：

```text
RSYNC_BIN
```

覆盖默认值。

---

## 30. Backend rsync 排除文件

```text
scripts/rsync-backend.exclude
```

示例：

```text
.git/
.DS_Store

node_modules/
.venv/
venv/
.build/
dist/
build/
coverage/
.pytest_cache/
.mypy_cache/
.next/
target/
tmp/
logs/

.env
.env.local
.env.development.local
.env.test.local

secrets/
*.p12
*.mobileprovision
```

注意：

**不要排除 dependency lock files。**

例如应该同步：

```text
package-lock.json
pnpm-lock.yaml
yarn.lock
uv.lock
poetry.lock
Cargo.lock
go.sum
Package.resolved
gradle.lockfile
```

如果项目中确实存在用于测试的 `.key` fixture，不要使用过宽的 `*.key` 排除规则。

---

## 31. 不同步 `.git`

默认不把：

```text
backend/.git
```

送到 Debian。

原因：

- 体积大。
- live mirror 不应成为 Git 工作副本。
- 避免 Debian 被误操作成第二个 Source of Truth。

如果 backend build 使用：

```bash
git describe
git rev-parse
```

不要依赖 Debian mirror 中存在 `.git`。

改为通过环境变量或生成文件注入：

```text
SOURCE_GIT_SHA
SOURCE_DIRTY
BUILD_VERSION
```

---

## 32. 安全版 `sync-backend`

```text
scripts/sync-backend
```

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
ROOT_DIR="$(cd "$SCRIPT_DIR/.." && pwd)"

SOURCE="$ROOT_DIR/backend/"
REMOTE_HOST="${DEVBOX_HOST:-devbox}"
REMOTE_DIR="/srv/myproduct/dev/backend"
REMOTE_STATE="/srv/myproduct/state"
EXCLUDE_FILE="$SCRIPT_DIR/rsync-backend.exclude"
RSYNC_BIN="${RSYNC_BIN:-rsync}"

LOCK_DIR="${TMPDIR:-/tmp}/myproduct-sync-backend.lock"

acquire_lock() {
  if mkdir "$LOCK_DIR" 2>/dev/null; then
    echo "$$" > "$LOCK_DIR/pid"
    return 0
  fi

  local old_pid=""
  old_pid="$(cat "$LOCK_DIR/pid" 2>/dev/null || true)"

  if [[ -n "$old_pid" ]] && kill -0 "$old_pid" 2>/dev/null; then
    echo "Another backend sync is running, pid=$old_pid" >&2
    exit 75
  fi

  rm -rf "$LOCK_DIR"
  mkdir "$LOCK_DIR"
  echo "$$" > "$LOCK_DIR/pid"
}

remote_marker_created=0

cleanup() {
  if [[ "$remote_marker_created" -eq 1 ]]; then
    ssh -o BatchMode=yes "$REMOTE_HOST" \
      "rm -f '$REMOTE_STATE/backend-sync.in-progress'" \
      >/dev/null 2>&1 || true
  fi

  rm -rf "$LOCK_DIR" 2>/dev/null || true
}

trap cleanup EXIT INT TERM

acquire_lock

if [[ ! -d "$SOURCE" ]]; then
  echo "Backend source not found: $SOURCE" >&2
  exit 2
fi

if [[ ! -f "$EXCLUDE_FILE" ]]; then
  echo "Exclude file not found: $EXCLUDE_FILE" >&2
  exit 2
fi

ssh -o BatchMode=yes "$REMOTE_HOST" "
  set -eu
  test \"\$(realpath -m '$REMOTE_DIR')\" = '$REMOTE_DIR'
  mkdir -p '$REMOTE_DIR' '$REMOTE_STATE'
  date -u '+%Y-%m-%dT%H:%M:%SZ' > '$REMOTE_STATE/backend-sync.in-progress'
"

remote_marker_created=1

echo "Syncing:"
echo "  $SOURCE"
echo "  -> $REMOTE_HOST:$REMOTE_DIR"

attempt=1
max_attempts=2

while true; do
  set +e

  "$RSYNC_BIN" \
    -a \
    --delete-delay \
    --delay-updates \
    --partial \
    --human-readable \
    --exclude-from="$EXCLUDE_FILE" \
    "$SOURCE" \
    "$REMOTE_HOST:$REMOTE_DIR/"

  rc=$?

  set -e

  if [[ "$rc" -eq 0 ]]; then
    break
  fi

  # rsync 24 常见于同步期间源文件变化/消失。
  if [[ "$rc" -eq 24 && "$attempt" -lt "$max_attempts" ]]; then
    echo "Source changed during sync; retrying once..."
    attempt=$((attempt + 1))
    sleep 1
    continue
  fi

  echo "rsync failed with exit code $rc" >&2
  exit "$rc"
done

BACKEND_HEAD="$(git -C "$ROOT_DIR/backend" rev-parse HEAD)"
BACKEND_BRANCH="$(git -C "$ROOT_DIR/backend" symbolic-ref --quiet --short HEAD || echo DETACHED)"

if git -C "$ROOT_DIR/backend" diff --quiet &&
   git -C "$ROOT_DIR/backend" diff --cached --quiet &&
   [[ -z "$(git -C "$ROOT_DIR/backend" ls-files --others --exclude-standard)" ]]; then
  BACKEND_DIRTY=false
else
  BACKEND_DIRTY=true
fi

{
  echo "synced_at=$(date -u '+%Y-%m-%dT%H:%M:%SZ')"
  echo "git_head=$BACKEND_HEAD"
  echo "git_branch=$BACKEND_BRANCH"
  echo "dirty=$BACKEND_DIRTY"
} | ssh -o BatchMode=yes "$REMOTE_HOST" \
  "cat > '$REMOTE_STATE/live-backend.meta'"

ssh -o BatchMode=yes "$REMOTE_HOST" \
  "rm -f '$REMOTE_STATE/backend-sync.in-progress'"

remote_marker_created=0

echo "Backend sync complete."
```

授权：

```bash
chmod +x scripts/sync-backend
```

---

## 33. 为什么加入 `--delay-updates`

开发态 rsync 无法做到真正“目录原子切换”，因为 backend 可能在 live mirror 上直接 hot reload。

`--delay-updates` 的目标是：

> 尽量把新文件内容的最终替换推迟到传输末尾，减少服务看到半写入文件的窗口。

它不是严格事务。

如果 backend hot reloader 对成批文件变化非常敏感，可以进一步改成：

```text
sync
→ 完成 marker
→ 明确调用 backend reload/restart
```

而不是依赖任意文件变化触发 reload。

---

## 34. 自动同步

初期：

```bash
./scripts/sync-backend
```

稳定后再加 watcher。

Mac：

```bash
brew install fswatch
```

```text
scripts/watch-backend
```

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
ROOT_DIR="$(cd "$SCRIPT_DIR/.." && pwd)"

echo "Watching backend..."

fswatch \
  -o \
  -l 0.5 \
  --exclude '/\.git/' \
  --exclude '/node_modules/' \
  --exclude '/\.venv/' \
  --exclude '/build/' \
  --exclude '/dist/' \
  "$ROOT_DIR/backend" |
while read -r _; do
  "$SCRIPT_DIR/sync-backend" || true
done
```

`sync-backend` 自身有锁，因此手工同步和 watcher 同时触发不会并发破坏 mirror。

---

## 35. Backend repo 内必须有统一命令

推荐：

```text
backend/scripts/
├── dev-up
├── dev-down
├── logs
├── wait-ready
├── test-fast
├── test-unit
├── test-integration
├── test-e2e
└── test-full
```

外层 workspace 脚本不应该知道：

```text
pytest 参数
npm 参数
Gradle 参数
Cargo 参数
```

外层只调用稳定 API：

```bash
./scripts/test-fast
./scripts/test-full
```

---

## 36. `dev-up`

Mac：

```text
scripts/dev-up
```

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

"$SCRIPT_DIR/sync-backend"

ssh -o BatchMode=yes devbox '
  set -eu
  cd /srv/myproduct/dev/backend
  ./scripts/dev-up
  ./scripts/wait-ready
'
```

原则：

> 不用 `sleep 10` 猜服务是否启动。

应该由 `wait-ready` 检查 health endpoint / DB readiness。

---

## 37. Docker Compose 开发环境

典型：

```text
backend
postgres
redis
worker
queue
object-storage
```

Mac 只需要访问 backend API。

Compose：

```yaml
services:
  backend:
    # ...
    ports:
      - "10.77.0.2:8080:8080"

  postgres:
    # 不发布 host port，backend 走 Compose network

  redis:
    # 不发布 host port
```

Docker 官方说明，如果不指定 Host IP，published port 默认会绑定所有主机接口；指定 Host IP 可以限制到特定主机地址。[R9]

因此不要默认：

```yaml
ports:
  - "8080:8080"
```

而应：

```yaml
ports:
  - "10.77.0.2:8080:8080"
```

---

## 38. Docker + UFW 的重要修订

**不要以为 UFW 能完整限制 Docker publish。**

Docker 官方明确说明：

> Docker published container traffic 可能在 UFW 的普通 INPUT/OUTPUT 规则之前被处理，因此单靠 UFW 可能无法阻止已 publish 的容器端口。[R10]

因此策略顺序是：

### 第一层

Compose 显式绑定：

```text
10.77.0.2
```

而不是：

```text
0.0.0.0
```

### 第二层

PostgreSQL、Redis 等不需要 Mac 直接访问的服务：

```text
完全不 publish
```

### 第三层

UFW 仍用于保护：

- SSH。
- Debian host 自己监听的服务。

### 第四层，可选

如果使用 Docker iptables backend，可在 `DOCKER-USER` 做 defense-in-depth。

先确认：

```bash
sudo iptables -S DOCKER-USER
```

再按实际 Wi‑Fi interface 制定规则。

如果 Docker 使用 nftables backend，不应照抄 `DOCKER-USER`；当前 Docker 文档对 nftables 有不同机制。[R10]

不要为了这套开发环境擅自关闭 Docker firewall：

```text
iptables=false
```

这容易破坏容器网络与隔离。

---

## 39. UFW：只处理 Host Service

假设 Debian Ethernet：

```text
enp5s0
```

```bash
sudo apt install -y ufw

sudo ufw default deny incoming
sudo ufw default allow outgoing

sudo ufw allow in on enp5s0 \
  from 10.77.0.1 \
  to any port 22 \
  proto tcp

sudo ufw enable
sudo ufw status verbose
```

如果 backend 是 host 进程而不是 Docker published port，也可：

```bash
sudo ufw allow in on enp5s0 \
  from 10.77.0.1 \
  to any port 8080 \
  proto tcp
```

对 Docker published port，不以这条规则作为唯一安全控制。

---

## 40. 不开放 Docker Remote API

不要开启：

```text
0.0.0.0:2375
```

Docker daemon 默认使用 Unix socket。

如未来确实需要从 Mac 操作 remote Docker，更安全的方式是 Docker over SSH / Docker context，而不是无 TLS TCP daemon。[R11]

当前架构甚至不需要 remote Docker API：

```text
Mac script
→ ssh devbox
→ docker compose ...
```

已经足够。

---

## 41. Mac Simulator → Debian Backend

开发 API：

```text
http://10.77.0.2:8080
```

iOS 中不要散落硬编码 IP。

推荐：

```text
Debug.xcconfig
Release.xcconfig
```

例如：

```text
Debug:
API_BASE_URL = http://10.77.0.2:8080

Release:
API_BASE_URL = https://api.example.com
```

---

## 42. iOS Local Network Privacy

iOS 对本地网络访问有隐私控制。

Apple 当前技术说明：

- 直接连接本地 IP 也属于 local network operation。
- App 使用本地网络时应提供 `NSLocalNetworkUsageDescription`。[R5]

Debug/开发配置须确保 `Info.plist` 中有合理说明，例如：

```text
Privacy - Local Network Usage Description
用于连接本机开发环境中的后端服务。
```

如果用户曾拒绝过权限：

```text
Settings
→ Privacy & Security
→ Local Network
```

重新检查。

---

## 43. HTTP / ATS

开发初期可以使用：

```text
http://10.77.0.2:8080
```

但不要把：

```text
NSAllowsArbitraryLoads = YES
```

作为长期通用方案。

Apple 提供 `NSAllowsLocalNetworking` 用于本地资源的 ATS 策略；具体行为还受部署版本和地址形式影响。[R12]

建议：

1. Debug configuration 只放必要的本地网络例外。
2. Release 不继承宽泛例外。
3. 稳定后迁移本地 HTTPS。

---

## 44. 本地域名：用 `.test`

不要发明未来可能冲突的内部 TLD。

推荐：

```text
api.myproduct.test
```

Mac `/etc/hosts`：

```text
10.77.0.2 api.myproduct.test
```

`.test` 是专门用于测试的保留命名空间。

之后：

```text
https://api.myproduct.test
```

比业务代码长期依赖裸 IP 更清晰。

---

## 45. 本地 HTTPS

第二阶段再做。

推荐路线：

```text
Mac trusted dev CA
         │
         ▼
Debian Caddy/nginx
         │
         ▼
backend
```

需要明确解决：

- Mac 信任 CA。
- iOS Simulator 信任 CA。
- hostname/SAN。
- Debug 与 Release 配置分离。

不要直接关闭对自签名证书的验证。

---

## 46. 真机测试

iOS Simulator 运行在 Mac，所以：

```text
Simulator
→ Mac route
→ 10.77.0.2
```

没有问题。

但真实 iPhone 通常连接 Wi‑Fi：

```text
iPhone
→ Wi‑Fi router
```

它**默认不知道**：

```text
10.77.0.0/30
```

这个仅存在于 Mac↔Debian 直连链路的网络。

因此真机测试不能默认使用：

```text
10.77.0.2
```

---

### 46.1 真机方案 A：Mac 做开发代理

例如：

```text
iPhone
  ↓ Wi‑Fi
Mac Wi‑Fi IP:18080
  ↓
Mac
  ↓ Ethernet
Debian:8080
```

可以使用：

- Caddy。
- nginx。
- 专用 proxy。
- SSH local forward。

只在受信任开发网络使用，并确保 Mac firewall 配置正确。

---

### 46.2 真机方案 B：Debian Wi‑Fi 也暴露一个受限开发端口

例如：

```text
Debian Wi‑Fi IP:18080
```

但必须：

- 只在可信 LAN。
- 明确防火墙来源。
- 不暴露 DB/Redis。
- 不暴露 Docker API。

这不应成为默认模式。

---

## 47. 快速 Backend 测试

Mac：

```text
scripts/test-backend-fast
```

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

"$SCRIPT_DIR/sync-backend"

ssh -o BatchMode=yes -t devbox '
  set -eu
  cd /srv/myproduct/dev/backend
  ./scripts/test-fast
'
```

Codex 日常只需知道：

```bash
./scripts/test-backend-fast
```

---

## 48. iOS Fast Test

Mac：

```text
scripts/test-ios
```

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
ROOT_DIR="$(cd "$SCRIPT_DIR/.." && pwd)"

cd "$ROOT_DIR/ios"
./scripts/test
```

`ios/scripts/test` 再统一封装：

- Scheme。
- Test Plan。
- Destination。
- DerivedData。
- `.xcresult`。
- 是否允许 parallel testing。

Codex 不应每次重新拼一整条 `xcodebuild`。

---

## 49. `test-all`：中短测试并行

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
ROOT_DIR="$(cd "$SCRIPT_DIR/.." && pwd)"

RUN_TS="$(date -u '+%Y%m%dT%H%M%SZ')"
ARTIFACT_DIR="$ROOT_DIR/artifacts/local-$RUN_TS"

mkdir -p "$ARTIFACT_DIR"

set +e

"$SCRIPT_DIR/test-ios" \
  >"$ARTIFACT_DIR/ios.log" 2>&1 &
IOS_PID=$!

"$SCRIPT_DIR/test-backend-fast" \
  >"$ARTIFACT_DIR/backend.log" 2>&1 &
BACKEND_PID=$!

wait "$IOS_PID"
IOS_CODE=$?

wait "$BACKEND_PID"
BACKEND_CODE=$?

set -e

echo "===== Test Summary ====="
echo "iOS:     $IOS_CODE"
echo "Backend: $BACKEND_CODE"
echo "Logs:    $ARTIFACT_DIR"

if [[ "$IOS_CODE" -ne 0 || "$BACKEND_CODE" -ne 0 ]]; then
  exit 1
fi
```

注意：

> 如果这个命令未来扩展到几十分钟甚至几小时，就应该改用 snapshot，而不是继续直接使用活跃源码。

---

## 50. Full Test：统一 RUN_ID

Full regression 必须先生成一个产品级：

```text
RUN_ID
```

格式：

```text
20260914T141530Z-a7d2c91f-3f82c1ab
```

包含：

- UTC 时间。
- backend short SHA。
- 随机后缀。

避免同一秒重复提交造成目录碰撞。

Mac：

```bash
BACKEND_SHA="$(git -C backend rev-parse --short=8 HEAD)"
RUN_ID="$(
  printf '%s-%s-%s\n' \
    "$(date -u '+%Y%m%dT%H%M%SZ')" \
    "$BACKEND_SHA" \
    "$(uuidgen | tr '[:upper:]' '[:lower:]' | cut -c1-8)"
)"
```

---

## 51. Full Run 目录

Mac：

```text
~/Library/Caches/MyProduct/test-runs/<RUN_ID>/
├── product-state.txt
└── ios/
    ├── input/
    ├── work/
    ├── logs/
    ├── artifacts/
    ├── pid
    └── exit_code
```

Debian：

```text
/srv/myproduct/test-runs/<RUN_ID>/
├── product-state.txt
└── backend/
    ├── input/
    ├── work/
    ├── logs/
    ├── artifacts/
    ├── source.sha256
    └── exit_code
```

---

## 52. 为什么保留 `input` 与 `work`

`input`：

```text
测试提交时的冻结源码证据
```

`work`：

```text
真正允许 build/test 写入的目录
```

防止：

- test 修改源码文件。
- code generation 改掉 snapshot。
- coverage/tooling 写入 source。
- 失败后无法确认原始测试输入。

---

## 53. Snapshot 的竞态问题

简单：

```bash
rsync source remote
```

并不严格保证源目录在整个复制期间不变化。

解决策略：

1. 第一次复制。
2. 进行 checksum 校验同步。
3. 再进行一次 checksum 校验。
4. 只有第二次完全无变化，才认为 snapshot 稳定。

对于 15 小时的长测，这点额外成本是值得的。

---

## 54. Backend Snapshot 脚本

```text
scripts/snapshot-backend
```

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

RUN_ID="${1:?Usage: snapshot-backend <RUN_ID>}"

if [[ ! "$RUN_ID" =~ ^[0-9]{8}T[0-9]{6}Z-[0-9a-fA-F]{8}-[0-9a-fA-F]{8}$ ]]; then
  echo "Invalid RUN_ID: $RUN_ID" >&2
  exit 2
fi

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
ROOT_DIR="$(cd "$SCRIPT_DIR/.." && pwd)"

SOURCE="$ROOT_DIR/backend/"
REMOTE_HOST="${DEVBOX_HOST:-devbox}"
REMOTE_BASE="/srv/myproduct/test-runs/$RUN_ID/backend"
REMOTE_INPUT="$REMOTE_BASE/input"
REMOTE_WORK="$REMOTE_BASE/work"
EXCLUDE_FILE="$SCRIPT_DIR/rsync-backend.exclude"
RSYNC_BIN="${RSYNC_BIN:-rsync}"

ssh -o BatchMode=yes "$REMOTE_HOST" "
  set -eu
  test ! -e '$REMOTE_BASE'
  mkdir -p \
    '$REMOTE_INPUT' \
    '$REMOTE_WORK' \
    '$REMOTE_BASE/logs' \
    '$REMOTE_BASE/artifacts'
"

"$RSYNC_BIN" \
  -a \
  --exclude-from="$EXCLUDE_FILE" \
  "$SOURCE" \
  "$REMOTE_HOST:$REMOTE_INPUT/"

stable=0

for attempt in 1 2 3; do
  changes="$(
    "$RSYNC_BIN" \
      -a \
      --delete \
      --checksum \
      --itemize-changes \
      --exclude-from="$EXCLUDE_FILE" \
      "$SOURCE" \
      "$REMOTE_HOST:$REMOTE_INPUT/"
  )"

  if [[ -z "$changes" ]]; then
    stable=1
    break
  fi

  echo "Source changed while snapshotting; stabilization pass $attempt/3..."
  sleep 1
done

if [[ "$stable" -ne 1 ]]; then
  echo "Backend source remained active; refusing to launch full test." >&2
  exit 75
fi

ssh -o BatchMode=yes "$REMOTE_HOST" "
  set -eu

  cd '$REMOTE_INPUT'

  find . -type f -print0 \
    | sort -z \
    | xargs -0 -r sha256sum -z \
    | sha256sum \
    | awk '{print \$1}' \
    > '$REMOTE_BASE/source.sha256'

  chmod -R a-w '$REMOTE_INPUT'

  cp -a --reflink=auto '$REMOTE_INPUT/.' '$REMOTE_WORK/'
  chmod -R u+w '$REMOTE_WORK'
"

echo "Backend snapshot ready:"
echo "  $REMOTE_BASE"
```

---

## 55. iOS Snapshot 排除文件

```text
scripts/rsync-ios-snapshot.exclude
```

示例：

```text
.git/
.DS_Store
DerivedData/
build/
.build/
coverage/
*.xcresult
```

如果使用：

```text
Pods/
Carthage/
SourcePackages/
```

是否排除要视项目情况而定：

- 如果它们是可重建 dependency cache，通常排除。
- 如果被 repo 正式管理，不能随便排除。

---

## 56. iOS Snapshot

```text
scripts/snapshot-ios
```

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

RUN_ID="${1:?Usage: snapshot-ios <RUN_ID>}"

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
ROOT_DIR="$(cd "$SCRIPT_DIR/.." && pwd)"

SOURCE="$ROOT_DIR/ios/"
RUN_ROOT="$HOME/Library/Caches/MyProduct/test-runs/$RUN_ID"
INPUT="$RUN_ROOT/ios/input"
WORK="$RUN_ROOT/ios/work"
EXCLUDE_FILE="$SCRIPT_DIR/rsync-ios-snapshot.exclude"
RSYNC_BIN="${RSYNC_BIN:-rsync}"

if [[ -e "$RUN_ROOT/ios" ]]; then
  echo "iOS run directory already exists: $RUN_ROOT/ios" >&2
  exit 2
fi

mkdir -p \
  "$INPUT" \
  "$WORK" \
  "$RUN_ROOT/ios/logs" \
  "$RUN_ROOT/ios/artifacts"

"$RSYNC_BIN" \
  -a \
  --exclude-from="$EXCLUDE_FILE" \
  "$SOURCE" \
  "$INPUT/"

stable=0

for attempt in 1 2 3; do
  changes="$(
    "$RSYNC_BIN" \
      -a \
      --delete \
      --checksum \
      --itemize-changes \
      --exclude-from="$EXCLUDE_FILE" \
      "$SOURCE" \
      "$INPUT/"
  )"

  if [[ -z "$changes" ]]; then
    stable=1
    break
  fi

  echo "iOS source changed while snapshotting; stabilization pass $attempt/3..."
  sleep 1
done

if [[ "$stable" -ne 1 ]]; then
  echo "iOS source remained active; refusing full test snapshot." >&2
  exit 75
fi

find "$INPUT" -type f -print0 \
  | sort -z \
  | xargs -0 shasum -a 256 \
  | shasum -a 256 \
  | awk '{print $1}' \
  > "$RUN_ROOT/ios/source.sha256"

chmod -R a-w "$INPUT"

# macOS cp -c uses clonefile/Copy-on-Write when the filesystem supports it.
# 不支持时会回退为普通复制。
cp -cR "$INPUT/." "$WORK/"
chmod -R u+w "$WORK"

echo "iOS snapshot ready:"
echo "  $RUN_ROOT/ios"
```

macOS 的 `cp -c` 会尝试使用 clonefile/Copy-on-Write，不支持时可回退到普通复制。[R13]

---

## 57. iOS Hash 的文件名边界

上面的 macOS Hash 示例对普通源码文件名足够。

如果你的 repo 允许极端文件名（例如文件名含换行），应使用专门的 manifest 工具，而不是依赖行分隔输出。

团队代码库通常应直接禁止这种文件名。

---

## 58. 创建产品级 Full Run

```text
scripts/create-full-run
```

逻辑：

```text
1. 生成 RUN_ID
2. 记录两个 Git repo 的状态
3. 创建 iOS snapshot
4. 创建 backend snapshot
5. 把同一 product-state.txt 放两边
6. 启动 Debian backend lane
7. 启动 Mac iOS lane
8. 输出 RUN_ID
```

示例：

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
ROOT_DIR="$(cd "$SCRIPT_DIR/.." && pwd)"

BACKEND_SHA="$(git -C "$ROOT_DIR/backend" rev-parse --short=8 HEAD)"
RAND="$(uuidgen | tr '[:upper:]' '[:lower:]' | cut -c1-8)"

RUN_ID="$(date -u '+%Y%m%dT%H%M%SZ')-${BACKEND_SHA}-${RAND}"

RUN_ROOT="$HOME/Library/Caches/MyProduct/test-runs/$RUN_ID"

mkdir -p "$RUN_ROOT"

"$SCRIPT_DIR/workspace-state" > "$RUN_ROOT/product-state.txt"

"$SCRIPT_DIR/snapshot-ios" "$RUN_ID"
"$SCRIPT_DIR/snapshot-backend" "$RUN_ID"

scp -q \
  "$RUN_ROOT/product-state.txt" \
  "devbox:/srv/myproduct/test-runs/$RUN_ID/product-state.txt"

"$SCRIPT_DIR/launch-backend-full" "$RUN_ID"

nohup \
  "$SCRIPT_DIR/run-ios-full-snapshot" "$RUN_ID" \
  >"$RUN_ROOT/ios/launcher.log" 2>&1 \
  </dev/null &

echo "$!" > "$RUN_ROOT/ios/pid"

echo
echo "Full run submitted:"
echo "  RUN_ID=$RUN_ID"
echo
echo "Check:"
echo "  ./scripts/run-status $RUN_ID"
```

如果你的 UI test 环境不适合从后台 `nohup` 启动，可把最后部分改成：

```text
在一个独立 Terminal 中运行 run-ios-full-snapshot
```

关键不在于是否后台运行，而在于：

> iOS full test 必须从 `ios/work` snapshot 跑。

---

## 59. Mac iOS Full Runner

```text
scripts/run-ios-full-snapshot
```

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

RUN_ID="${1:?Usage: run-ios-full-snapshot <RUN_ID>}"

RUN_ROOT="$HOME/Library/Caches/MyProduct/test-runs/$RUN_ID"
WORK="$RUN_ROOT/ios/work"
LOG="$RUN_ROOT/ios/logs/full-test.log"
EXIT_FILE="$RUN_ROOT/ios/exit_code"
STARTED="$RUN_ROOT/ios/started_at"
FINISHED="$RUN_ROOT/ios/finished_at"

if [[ ! -d "$WORK" ]]; then
  echo "Missing iOS work snapshot: $WORK" >&2
  exit 2
fi

date -u '+%Y-%m-%dT%H:%M:%SZ' > "$STARTED"

rc=0

set +e

(
  cd "$WORK"
  caffeinate -i -s ./scripts/test-full
) 2>&1 | tee "$LOG"

rc=${PIPESTATUS[0]}

set -e

printf '%s\n' "$rc" > "$EXIT_FILE"
date -u '+%Y-%m-%dT%H:%M:%SZ' > "$FINISHED"

exit "$rc"
```

这里不使用：

```text
caffeinate -d
```

因为长测试没有必要强制显示器一直亮。

`-i` / `-s` 已足以防止 Mac mini 在测试期间进入睡眠。

---

## 60. Debian Long Test 需要 systemd user

15 小时测试不能依赖当前 SSH session。

Debian：

```bash
sudo loginctl enable-linger "$USER"
```

官方 systemd 说明 linger 会让该用户的 user manager 在登出后继续存在，从而可以运行长时间用户服务。[R14]

验证：

```bash
loginctl show-user "$USER" -p Linger
systemctl --user status
```

---

## 61. systemd user 环境不是你的 interactive shell

不要假定：

```text
~/.bashrc
nvm
asdf
pyenv
某些自定义 PATH
```

一定会在 `systemd --user` 任务中加载。

因此 long test 最好：

```text
Docker 化
```

或使用：

```text
绝对路径
显式环境文件
```

先检查：

```bash
systemd-run --user --wait --pipe env | sort
```

确保 `backend/scripts/test-full` 不依赖只存在于交互 shell 的初始化。

---

## 62. Debian Full Runner

创建：

```text
/srv/myproduct/tools/run-backend-full
```

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

RUN_ID="${1:?RUN_ID required}"

if [[ ! "$RUN_ID" =~ ^[0-9]{8}T[0-9]{6}Z-[0-9a-fA-F]{8}-[0-9a-fA-F]{8}$ ]]; then
  echo "Invalid RUN_ID: $RUN_ID" >&2
  exit 2
fi

RUN_BASE="/srv/myproduct/test-runs/$RUN_ID/backend"
WORK="$RUN_BASE/work"
LOG="$RUN_BASE/logs/full-test.log"
EXIT_FILE="$RUN_BASE/exit_code"
STARTED="$RUN_BASE/started_at"
FINISHED="$RUN_BASE/finished_at"

GLOBAL_LOCK="/srv/myproduct/state/backend-full.lock"

mkdir -p "$RUN_BASE/logs" "$RUN_BASE/artifacts"

exec 9>"$GLOBAL_LOCK"

if ! flock -n 9; then
  echo "Another backend full test is already running." >&2
  exit 75
fi

if [[ ! -d "$WORK" ]]; then
  echo "Missing work directory: $WORK" >&2
  exit 2
fi

date -u '+%Y-%m-%dT%H:%M:%SZ' > "$STARTED"

finish() {
  local rc=$?
  printf '%s\n' "$rc" > "$EXIT_FILE.tmp" || true
  mv "$EXIT_FILE.tmp" "$EXIT_FILE" 2>/dev/null || true
  date -u '+%Y-%m-%dT%H:%M:%SZ' > "$FINISHED" || true
}

trap finish EXIT

cd "$WORK"

export RUN_ID
export COMPOSE_PROJECT_NAME="mp_${RUN_ID//[^a-zA-Z0-9]/_}"

./scripts/test-full \
  2>&1 \
  | tee "$LOG"

exit "${PIPESTATUS[0]}"
```

授权：

```bash
chmod 700 /srv/myproduct/tools/run-backend-full
```

默认只允许一个 backend full run：

```text
flock
```

这样不会因为误提交两个 15 小时任务把 Debian 完全打满。

以后确认硬件有余量后，再设计并发 semaphore。

---

## 63. Backend Test Compose 的端口原则

每个 full run 使用独立：

```text
COMPOSE_PROJECT_NAME
```

能隔离：

- container name。
- network。
- volume。

但是：

> **它不能自动解决你写死的 host port 冲突。**

因此 full test Compose 应优先：

```text
测试进程也运行在 Compose network 内
```

这样 PostgreSQL/backend/test runner 都只用容器内部端口：

```text
postgres:5432
backend:8080
```

不 publish 到 host。

如果确实要让 Mac 访问某个 test run：

- 动态分配 host port。
- 把端口写入该 RUN_ID metadata。
- 不固定所有 run 都抢 `8080`。

---

## 64. Launch Backend Full

Mac：

```text
scripts/launch-backend-full
```

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

RUN_ID="${1:?Usage: launch-backend-full <RUN_ID>}"

UNIT="myproduct-backend-${RUN_ID}"

ssh -o BatchMode=yes devbox "
  systemd-run \
    --user \
    --unit='$UNIT' \
    --collect \
    --property=Nice=10 \
    --property=CPUWeight=50 \
    --property=IOWeight=50 \
    --property=RuntimeMaxSec=20h \
    --property=KillMode=control-group \
    /srv/myproduct/tools/run-backend-full '$RUN_ID'
"
```

`--collect` 会在 transient unit 结束后回收 unit 状态；因此长期结果不要依赖 `systemctl status`，而应依赖以下文件：

```text
exit_code
started_at
finished_at
full-test.log
artifacts
```

systemd 对 `--collect` 的定义见官方文档。[R15]

`CPUWeight/IOWeight` 依赖系统的 cgroup 能力；如果你的 Debian 环境不支持对应属性，先去掉，再单独调优。

---

## 65. 长测试资源优先级

目标：

```text
完整回归在后台吃资源
但不能让当前开发完全卡住
```

建议初始：

```text
Nice=10
CPUWeight=50
IOWeight=50
```

再根据机器配置测。

不要一开始就：

```text
CPU 全核 100%
RAM 全吃满
DB worker 最大并发
```

因为你仍需要：

- live backend。
- Codex fast tests。
- Docker pull/build。
- SSH。
- DB 联调。

---

## 66. 磁盘预检

长测试前建议至少保留一定磁盘空间。

Debian：

```bash
df -h /srv/myproduct
docker system df
```

可以在 runner 前加：

```bash
MIN_FREE_GB="${MIN_FREE_GB:-50}"

AVAILABLE_KB="$(
  df -Pk /srv/myproduct |
  awk 'NR==2 {print $4}'
)"

REQUIRED_KB=$((MIN_FREE_GB * 1024 * 1024))

if (( AVAILABLE_KB < REQUIRED_KB )); then
  echo "Not enough free space for full test." >&2
  exit 75
fi
```

`50 GB` 只是示例，应根据：

- Docker image。
- DB fixture。
- coverage。
- screenshots。
- artifacts。

实际调整。

---

## 67. Full Test 数据隔离

每个 `RUN_ID` 要隔离：

```text
PostgreSQL database/schema
Redis prefix
Queue namespace
Object storage bucket/prefix
temporary filesystem
Compose volume
container network
test account
```

例如：

```text
TEST_RUN_ID=<RUN_ID>
```

所有 fixture 名称都带 RUN_ID。

否则多个 test suite 会相互污染。

---

## 68. 禁止 Full Test 访问生产依赖

完整测试环境必须确认：

```text
production DB
production Redis
production queue
production email
production payment
production push
```

均不可被误访问。

优先：

- fake service。
- sandbox account。
- local container。
- explicit allowlist。

如果 full test 需要 Internet，不要为了方便直接给它生产 token。

---

## 69. Backend `scripts/test-full`

推荐职责：

```text
preflight
→ test service up
→ migrations
→ unit
→ integration
→ API E2E
→ worker/queue
→ coverage
→ artifact export
→ cleanup
```

它自己应使用：

```bash
trap
```

保证失败或中断后也执行合理清理。

但不要在 cleanup 中删除：

```text
logs
artifacts
product-state
source hash
```

---

## 70. Xcode Full Test 建议

iOS snapshot 的：

```text
ios/scripts/test-full
```

应固定：

- Scheme。
- Test Plan。
- Simulator model。
- Simulator OS。
- DerivedData path。
- Result Bundle path。

建议每个 RUN_ID 独立：

```text
DerivedData
.xcresult
screenshots
simulator logs
```

不要共享一个可能被并发测试污染的 DerivedData。

示意：

```text
~/Library/Caches/MyProduct/test-runs/<RUN_ID>/ios/
├── derived-data/
└── artifacts/
    └── FullTests.xcresult
```

---

## 71. UI Test 并行不要盲目开满

XCTest/XCUITest 是否适合并行取决于：

- 是否共享账号。
- 是否共享 backend state。
- 是否有全局 device state。
- 是否依赖通知/Keychain。
- 测试是否会互相删除数据。

原则：

> 先做到 run 隔离，再逐步增加 `maximum-parallel-testing-workers`。

否则更多 worker 只会产生更多 flaky test。

---

## 72. Backend 并行也一样

可以使用：

```text
pytest-xdist
Jest workers
Vitest pool
cargo nextest
Go test parallelism
Gradle test workers
```

但瓶颈可能是：

```text
DB lock
IO
Docker
queue
test fixture creation
```

不是 CPU。

先测：

```bash
/usr/bin/time -v ...
iostat
pidstat
docker stats
```

再调整 worker 数。

---

## 73. 数据库测试优先拆分

优先级：

```text
每 worker 独立 DB
>
每 worker 独立 schema
>
共享数据库但独立事务
>
所有测试共用一个 testdb
```

Redis：

```text
prefix / namespace
```

Queue：

```text
RUN_ID namespace
```

Object Storage：

```text
RUN_ID bucket/prefix
```

---

## 74. 产品级 Run Status

Mac：

```text
scripts/run-status
```

示例：

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

RUN_ID="${1:?Usage: run-status <RUN_ID>}"

LOCAL_ROOT="$HOME/Library/Caches/MyProduct/test-runs/$RUN_ID"
IOS_EXIT="$LOCAL_ROOT/ios/exit_code"

echo "RUN_ID: $RUN_ID"
echo

echo "== iOS =="

if [[ -f "$IOS_EXIT" ]]; then
  echo "finished, exit=$(cat "$IOS_EXIT")"
elif [[ -f "$LOCAL_ROOT/ios/pid" ]] &&
     kill -0 "$(cat "$LOCAL_ROOT/ios/pid")" 2>/dev/null; then
  echo "running, pid=$(cat "$LOCAL_ROOT/ios/pid")"
else
  echo "unknown/not running"
fi

echo
echo "== Backend =="

ssh -o BatchMode=yes devbox "
  BASE='/srv/myproduct/test-runs/$RUN_ID/backend'

  if test -f \"\$BASE/exit_code\"; then
    echo \"finished, exit=\$(cat \"\$BASE/exit_code\")\"
  else
    UNIT='myproduct-backend-$RUN_ID'

    if systemctl --user is-active --quiet \"\$UNIT\"; then
      echo 'running'
    else
      echo 'unknown/not running'
    fi
  fi
"
```

---

## 75. Artifacts

Backend：

```text
/srv/myproduct/test-runs/<RUN_ID>/backend/artifacts/
```

iOS：

```text
~/Library/Caches/MyProduct/test-runs/<RUN_ID>/ios/artifacts/
```

建议收集：

```text
JUnit/XML
coverage
API traces
failed request/response
screenshots
video（如已有）
crash report
.xcresult
simulator logs
DB migration logs
random seed
failed test list
```

---

## 76. `fetch-artifacts`

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

RUN_ID="${1:?Usage: fetch-artifacts <RUN_ID>}"

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
ROOT_DIR="$(cd "$SCRIPT_DIR/.." && pwd)"

DEST="$ROOT_DIR/artifacts/full/$RUN_ID"

mkdir -p "$DEST/backend"

rsync -a \
  "devbox:/srv/myproduct/test-runs/$RUN_ID/backend/artifacts/" \
  "$DEST/backend/artifacts/"

rsync -a \
  "devbox:/srv/myproduct/test-runs/$RUN_ID/backend/logs/" \
  "$DEST/backend/logs/"

rsync -a \
  "$HOME/Library/Caches/MyProduct/test-runs/$RUN_ID/ios/artifacts/" \
  "$DEST/ios/artifacts/"

rsync -a \
  "$HOME/Library/Caches/MyProduct/test-runs/$RUN_ID/ios/logs/" \
  "$DEST/ios/logs/"

cp \
  "$HOME/Library/Caches/MyProduct/test-runs/$RUN_ID/product-state.txt" \
  "$DEST/product-state.txt"

echo "Artifacts:"
echo "  $DEST"
```

---

## 77. 环境版本也要存

结果不只需要源码 SHA。

Backend run 应保存：

```bash
uname -a
cat /etc/os-release
docker version
docker compose version
```

Mac/iOS run：

```bash
sw_vers
xcodebuild -version
xcrun simctl list runtimes
```

这样以后才能回答：

> 为什么同一份源码上个月通过、今天失败？

---

## 78. Random Seed

如果测试使用随机数据：

```text
RANDOM_SEED
```

必须写入 run metadata。

失败后可以：

```text
同 seed 重放
```

不要只写：

```text
flaky, rerun passed
```

而丢失第一次失败信息。

---

## 79. Retrying Policy

建议：

```text
首次失败
→ 保留完整原始日志
→ 明确标记是否属于 flaky 类测试
→ 最多有限次数 retry
→ 报告原始失败 + retry 结果
```

不要让：

```text
retry-until-green
```

掩盖真实问题。

---

## 80. `dev-status`

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

echo "== Route =="
route -n get 10.77.0.2 | grep -E 'interface|gateway|destination' || true

echo
echo "== Ping =="
ping -c 1 10.77.0.2

echo
echo "== SSH =="
ssh -o BatchMode=yes devbox '
  echo "host=$(hostname)"
  uptime
'

echo
echo "== Debian Network =="
ssh -o BatchMode=yes devbox '
  ip route get 10.77.0.1
  ip route get 1.1.1.1
'

echo
echo "== Docker =="
ssh -o BatchMode=yes devbox '
  docker version --format "server={{.Server.Version}}"
  docker compose version
  docker info --format "logging={{.LoggingDriver}}"
'

echo
echo "== Resources =="
ssh -o BatchMode=yes devbox '
  echo "-- memory --"
  free -h
  echo "-- disk --"
  df -h /srv/myproduct
  echo "-- docker disk --"
  docker system df
'

echo
echo "== Live Backend =="
ssh -o BatchMode=yes devbox '
  test -d /srv/myproduct/dev/backend && echo "mirror=OK"
  test ! -e /srv/myproduct/state/backend-sync.in-progress \
    && echo "sync=idle" \
    || echo "sync=in-progress/stale"
'
```

每天只需要：

```bash
./scripts/dev-status
```

---

## 81. 开发态工作流

日常：

```text
1. Mac 打开:
   ~/Developer/MyProduct

2. 查看状态:
   ./scripts/dev-status

3. 启动开发环境:
   ./scripts/dev-up

4. Codex 修改:
   ios/
   backend/

5. backend 快速测试:
   ./scripts/test-backend-fast

6. iOS 快速测试:
   ./scripts/test-ios

7. 联调:
   Simulator → Debian backend

8. 中等测试:
   ./scripts/test-all

9. 长测试:
   ./scripts/create-full-run

10. 继续开发下一项任务
```

重点：

> `create-full-run` 之后，测试输入已经冻结，所以继续编辑两个 repo 不会污染这一轮 full regression。

---

## 82. 理想 Codex 任务

用户：

```text
增加 avatarUrl。
修改 DB migration、backend DTO/API、
Swift networking model 和 ProfileView，
完成前后端测试。
```

Codex：

```text
读 backend
↓
读 ios
↓
修改 backend
↓
修改 ios
↓
./scripts/test-backend-fast
↓
./scripts/test-ios
↓
需要联调时 ./scripts/dev-up
↓
检查 Simulator / API
```

Codex 不需要：

```text
Remote-SSH 打开第二 workspace
```

也不需要理解：

```text
Debian IP
Docker 具体命令
PostgreSQL 启动顺序
```

这些都封装在脚本中。

---

## 83. VS Code Tasks

可选：

```text
.vscode/tasks.json
```

```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "Devbox: Status",
      "type": "shell",
      "command": "${workspaceFolder}/scripts/dev-status",
      "problemMatcher": []
    },
    {
      "label": "Backend: Sync",
      "type": "shell",
      "command": "${workspaceFolder}/scripts/sync-backend",
      "problemMatcher": []
    },
    {
      "label": "Backend: Fast Test",
      "type": "shell",
      "command": "${workspaceFolder}/scripts/test-backend-fast",
      "problemMatcher": []
    },
    {
      "label": "Product: Test All",
      "type": "shell",
      "command": "${workspaceFolder}/scripts/test-all",
      "problemMatcher": []
    },
    {
      "label": "Product: Create Full Run",
      "type": "shell",
      "command": "${workspaceFolder}/scripts/create-full-run",
      "problemMatcher": []
    }
  ]
}
```

---

## 84. Debian 电源设置

长测试机器不能自动 suspend。

首先通过桌面环境关闭自动 suspend。

如果未来 Debian 彻底变成 headless compute node，再考虑系统级：

```bash
sudo systemctl mask \
  sleep.target \
  suspend.target \
  hibernate.target \
  hybrid-sleep.target
```

这是比较强的系统级行为，不建议在仍作为普通桌面使用时盲目执行。

---

## 85. 时间同步

Debian：

```bash
timedatectl status
```

确认：

```text
System clock synchronized: yes
```

Mac：

```text
系统设置 → 日期与时间 → 自动设置
```

测试日志统一建议写 UTC：

```text
2026-09-14T14:15:30Z
```

---

## 86. Debian Cache

长期保留：

```text
Docker layers
package manager cache
compiler cache
language cache
test fixture cache
```

可按技术栈使用：

```text
pnpm/npm cache
uv/pip cache
Gradle cache
Cargo cache
Go build cache
BuildKit
ccache
sccache
```

不要从 Mac 同步：

```text
node_modules
.venv
target
.build
```

---

## 87. Mac ARM 与 Debian x86 的区别

如果 Mac mini 是 Apple Silicon，而 Debian 台式机是 x86_64：

```text
Mac:
arm64

Debian:
amd64
```

这正是“不复制构建产物”的另一个原因。

需要同步：

```text
源码
lock file
配置
```

不要同步：

```text
native binary
venv
node native addon
compiled target
```

如果 production 的 CPU 架构与 Debian 不同，release 阶段再补：

```text
multi-arch build/test
```

不必把日常开发全部复杂化。

---

## 88. Git 文件名大小写

macOS 默认 APFS 常是 case-insensitive。

Linux 一般 case-sensitive。

因此：

```text
UserService.swift
userservice.swift
```

可能产生跨平台差异。

建议：

- CI/Linux 定期验证。
- 不允许仅大小写不同的路径。
- rename 大小写时认真检查 Git diff。

---

## 89. Shell 文件权限与换行

项目脚本：

```bash
chmod +x scripts/test-full
git add scripts/test-full
```

建议 `.gitattributes`：

```gitattributes
*.sh text eol=lf
```

避免跨平台脚本换行和 executable bit 问题。

---

## 90. 测试分层

### L0：编辑级

```text
秒级
```

- format。
- syntax。
- lint。
- type check。
- affected unit。

---

### L1：Fast

```text
1–5 min
```

Backend：

```text
lint
fast unit
changed tests
```

iOS：

```text
compile
unit subset
```

---

### L2：Commit / PR

```text
5–30 min
```

Debian：

```text
unit
migration smoke
integration subset
API smoke
```

Mac：

```text
Swift unit
UI smoke
```

并行执行。

---

### L3：Full

```text
小时级
```

必须 immutable snapshot。

Debian：

```text
full unit
integration
API E2E
migration
worker
queue
coverage
regression
```

Mac：

```text
full XCTest
XCUITest
UI matrix
Simulator full regression
```

---

## 91. 并行时间模型

原来：

```text
Backend 7 h
+
iOS/UI 8 h
=
15 h
```

两台机器并行：

```text
max(7h, 8h)
≈ 8h
```

之后再优化每个 lane 内部并行度。

第一目标不是：

```text
把每个测试线程开到最大
```

而是：

```text
先把串行 lane 拆开
```

---

## 92. 长测期间的交互开发保护

Debian：

```text
dev backend
```

与：

```text
full test backend
```

必须使用不同：

```text
source directory
Compose project
DB
Redis namespace
volume
```

长测试应降低 CPU/IO 权重。

这样：

```text
Full Run A
正在跑第 6 小时

与此同时

Codex
继续开发 Feature B

./scripts/test-backend-fast
仍能得到合理响应
```

这才是双机方案真正的价值。

---

## 93. 清理策略

不要无限增长：

```text
/srv/myproduct/test-runs
~/Library/Caches/MyProduct/test-runs
```

建议：

```text
成功 full run:
7–14 天

失败 run:
30 天

Release:
显式保留
```

删除前必须：

```bash
du -sh
```

并使用白名单根目录。

任何自动清理脚本必须先检查：

```text
realpath
RUN_ID 格式
目标位于固定 test-runs 根目录
```

再删除。

不要让 Codex 日常使用：

```bash
rm -rf /srv/myproduct/*
```

---

## 94. Docker 清理

不要把：

```bash
docker system prune -a
```

作为日常自动任务。

它可能删除：

- 有价值的 warm cache。
- 暂停使用的 image。
- 调试环境。

更安全：

```bash
docker system df
```

按需：

```bash
docker image prune
docker builder prune
```

并人工确认。

---

## 95. Full Run 中断恢复

如果 Debian 重启：

1. `systemd-run` transient test 会停止。
2. `exit_code` 可能不存在。
3. `logs` 和 snapshot 仍在磁盘。

`run-status` 应显示：

```text
unknown/not running
```

而不是假装成功。

此时：

```text
重新提交同一 snapshot
```

或：

```text
标记 interrupted
```

不要自动把缺少 `exit_code` 当成失败 1 或成功 0。

---

## 96. Sync 中断恢复

如果：

```text
backend-sync.in-progress
```

长时间遗留：

先：

```bash
ps
ssh devbox
```

确认没有同步。

然后重新运行：

```bash
./scripts/sync-backend
```

新同步会重新覆盖 mirror。

live mirror 必须始终满足：

> 可以丢弃再建，不含唯一数据。

---

## 97. 后端服务健康检查

`backend/scripts/wait-ready` 示例思想：

```text
循环 health endpoint
→ 有最大 timeout
→ 成功返回 0
→ timeout 返回非 0
```

不要：

```bash
sleep 30
```

固定等待会：

- 服务启动快时，只是在浪费时间。
- 服务启动慢时，仍然不够。

---

## 98. 测试 Artifacts 不应只存在 Debian

重要失败结果：

```text
.xcresult
coverage
crash
API trace
```

应至少拉回 Mac，必要时再上传 GitHub/GitLab artifact storage。

Debian 本地磁盘不应成为唯一历史记录。

---

## 99. Backup 策略

需要备份：

```text
Git repos
workspace scripts/docs
重要 artifacts
必要的开发配置说明
```

不需要备份：

```text
/srv/myproduct/dev/backend
Docker layer cache
临时 test work
普通 dependency cache
```

`/srv/myproduct/config` 如果含 credential：

- 不放普通 Git。
- 使用密码管理器或可重建机制。
- 不把生产 secret 复制进开发节点。

---

## 100. CI 接入时机

只有下面这些稳定后再接 CI：

```text
sync
dev-up
test-fast
snapshot
test-full
artifact
cleanup
```

CI 应调用现有脚本，而不是再维护第二套测试逻辑。

理想：

```text
Developer:
./scripts/test-backend-fast

CI:
./scripts/test-backend-fast
```

---

## 101. GitHub Self-hosted Runner 安全

如果以后 Debian 变成 GitHub Actions self-hosted runner：

GitHub 官方明确指出：

- self-hosted runner 不是每次 clean/ephemeral。
- 不可信 workflow 可持久破坏节点。
- public repo 尤其不应让任意 PR 进入持久自托管 runner。[R16]

因此：

```text
个人私有 repo
可信代码
```

可以实用。

但：

```text
public PR
外部贡献者
不可信分支
```

不应直接跑在这台长期保存开发环境与 credential 的 Debian 上。

更强方案：

```text
ephemeral runner
VM
container/isolated runner
专用 runner 机器
```

---

## 102. CI 也不要破坏开发 Node

如果 Debian 同时是：

```text
interactive dev node
+
CI runner
```

应确保：

- CI 不清理 dev volume。
- CI 独立 workspace。
- CI 不用 `/srv/myproduct/dev/backend`。
- CI 有并发限制。
- CI 有 resource priority。
- CI 的 Docker project name 与 dev 分开。

---

## 103. 参考的最终脚本 API

人和 Codex 只需要理解：

```bash
./scripts/dev-status
./scripts/workspace-state

./scripts/sync-backend
./scripts/watch-backend

./scripts/dev-up
./scripts/dev-down
./scripts/backend-logs

./scripts/test-backend-fast
./scripts/test-ios
./scripts/test-all

./scripts/create-full-run
./scripts/run-status <RUN_ID>
./scripts/fetch-artifacts <RUN_ID>
```

这套脚本就是：

> 双机基础设施的稳定 API。

---

## 104. 实施阶段

### Phase 0：建立 baseline

先记录现在：

```text
backend unit duration
integration duration
E2E duration
iOS unit duration
UI duration
CPU
RAM
disk
```

否则改造完成后无法判断收益。

---

### Phase 1：网络

完成：

- [ ] 找一个不与 VPN 冲突的 `/30`
- [ ] Mac Ethernet static IP
- [ ] Debian Ethernet static IP
- [ ] 无 default gateway
- [ ] Wi‑Fi 各自 Internet
- [ ] VPN 四种组合都验证
- [ ] `iperf3`

验收：

```bash
route -n get 10.77.0.2
ping 10.77.0.2
```

---

### Phase 2：SSH

- [ ] SSH key
- [ ] fingerprint 验证
- [ ] `ssh devbox`
- [ ] `BatchMode`
- [ ] 密码登录关闭
- [ ] SSH 只从直连链路可访问，除非有明确 recovery 需求

验收：

```bash
ssh -o BatchMode=yes devbox 'hostname'
```

---

### Phase 3：统一 Codex Workspace

- [ ] `~/Developer/MyProduct`
- [ ] `ios/`
- [ ] `backend/`
- [ ] 顶层 AGENTS
- [ ] repo AGENTS
- [ ] VS Code nested Git
- [ ] Codex 可以一次搜索两个 repo

验收任务：

```text
让 Codex 找出某个 backend endpoint
以及所有 iOS consumer。
```

无需切换 IDE 窗口。

---

### Phase 4：Backend Remote Runtime

- [ ] `/srv/myproduct`
- [ ] Docker
- [ ] log rotation
- [ ] rsync
- [ ] dev-up
- [ ] healthcheck
- [ ] Simulator 可以访问 backend

验收：

```bash
./scripts/dev-up
curl http://10.77.0.2:8080/health
```

---

### Phase 5：Fast Test

- [ ] `backend/scripts/test-fast`
- [ ] Mac wrapper
- [ ] exit code 正确
- [ ] Codex 能读取失败日志

故意制造测试失败验证：

```bash
./scripts/test-backend-fast
echo $?
```

必须非 0。

---

### Phase 6：产品级 Snapshot

- [ ] RUN_ID
- [ ] workspace-state
- [ ] iOS input/work
- [ ] backend input/work
- [ ] source hash
- [ ] 同一 product-state
- [ ] snapshot 稳定性检查

关键验收：

1. 开始 full test。
2. 等 5 分钟。
3. 修改 Mac iOS/backend 源码。
4. `sync-backend`。
5. 验证 full run 两边的 `work` 内容不变。

---

### Phase 7：双机 Full Regression

- [ ] Debian systemd user task
- [ ] Mac snapshot runner
- [ ] 同一 RUN_ID
- [ ] run-status
- [ ] artifacts
- [ ] cleanup
- [ ] resource priority

---

### Phase 8：性能优化

之后再：

- [ ] test sharding
- [ ] DB worker isolation
- [ ] XCUITest parallelism
- [ ] BuildKit cache
- [ ] sccache/ccache
- [ ] NVMe
- [ ] 2.5/10GbE
- [ ] CI runner

---

## 105. 最终验收标准

### Codex

- [ ] 一个 VS Code 窗口同时包含 ios/backend。
- [ ] Codex 可在同一任务修改两个 repo。
- [ ] 不依赖 Remote-SSH workspace 做日常 backend 开发。
- [ ] Codex 只通过稳定脚本操作 Debian。

### Source of Truth

- [ ] Mac 是唯一人工源码编辑源。
- [ ] Debian live mirror 可直接删除重建。
- [ ] Debian 不维护独立开发 branch。

### 网络

- [ ] 直连 subnet 不与 VPN 冲突。
- [ ] `10.77.0.2` 明确走 Ethernet。
- [ ] Debian Internet 明确走 Wi‑Fi/VPN。
- [ ] 两边 VPN 开启仍可 SSH。

### Docker

- [ ] Backend 只 publish 到直连 IP。
- [ ] DB/Redis 默认不 publish。
- [ ] Docker 日志有 rotation。
- [ ] 没有开放 2375。
- [ ] 要明白 UFW 不是 Docker published port 的唯一保护层。

### iOS

- [ ] Simulator 可以访问 Debian backend。
- [ ] Local Network permission 配置正确。
- [ ] Debug ATS 配置与 Release 分离。
- [ ] 真机测试有单独网络方案。

### Fast Test

- [ ] backend fast test 在 Debian。
- [ ] iOS fast test 在 Mac。
- [ ] 两边可并行。
- [ ] exit code 正确回到 Codex。

### Full Test

- [ ] iOS/backend 同一 RUN_ID。
- [ ] 两边均运行 immutable work snapshot。
- [ ] 继续开发不会污染 full run。
- [ ] 每个 run 有源码 hash。
- [ ] 有 product-state。
- [ ] Backend long test 脱离 SSH session。
- [ ] 长测试资源不会完全压死 live dev。
- [ ] 测试数据彼此隔离。
- [ ] 固定 host port 不冲突。

### Artifacts

- [ ] `.xcresult` 保留。
- [ ] backend logs 保留。
- [ ] coverage 保留。
- [ ] 失败证据可拉回 Mac。
- [ ] artifact 不泄漏生产 secret。

### 运维

- [ ] 有 disk preflight。
- [ ] 有 retention policy。
- [ ] Docker log 不无限增长。
- [ ] Debian 不自动 suspend。
- [ ] 时间同步正常。
- [ ] interrupted run 不会误判为通过。

---

## 106. 常见问题排查

### VPN 开启后 SSH 失败

Mac：

```bash
route -n get 10.77.0.2
```

如果 interface 正确：

检查：

- VPN Local LAN Access。
- VPN kill switch。
- macOS Local Network Privacy。
- Debian firewall。
- VPN 是否安装更强的 packet filter。

---

### route 走错

查看：

```bash
netstat -rn -f inet
```

如果 VPN 本身已经拥有目标网段：

> 换直连 `/30`，不要长期和 VPN 路由硬抢。

---

### rsync 非常慢

先：

```bash
iperf3 -c 10.77.0.2
```

再检查排除项：

```text
node_modules
.venv
DerivedData
target
build
.git
```

如果网速正常但大量小文件很慢：

- 检查是否误同步 cache。
- 观察文件数量。
- 不要误以为 10GbE 会自动解决百万小文件 metadata 成本。

---

### rsync 退出 24

通常意味着：

```text
源文件在同步中发生变化/消失
```

`sync-backend` 会重试一次。

Full snapshot 会做 checksum 稳定性校验，持续编辑时会拒绝启动。

---

### Backend 是旧代码

检查：

```bash
cat /srv/myproduct/state/live-backend.meta
```

再：

```bash
stat /srv/myproduct/dev/backend/<FILE>
```

如果 mirror 已更新：

检查：

- 容器是否 bind mount 正确。
- hot reload 是否工作。
- 是否需要 rebuild。
- 是否跑错了 dev Compose project。

---

### Docker 端口在 Wi‑Fi 上可访问

Debian：

```bash
ss -lntp
docker ps
```

Compose 应：

```text
10.77.0.2:8080
```

而不是：

```text
0.0.0.0:8080
```

不要只看 UFW。

---

### Simulator 无法连接

依次：

```text
Mac curl backend
→ Simulator Local Network permission
→ Debug ATS
→ API base URL
→ backend health
```

先在 Mac：

```bash
curl -v http://10.77.0.2:8080/health
```

Mac 都不通时，不要先去查 iOS。

---

### 真机无法连接

检查你是否还在使用：

```text
10.77.0.2
```

真实 iPhone 通常没有到这个 `/30` 的 route。

改用：

- Mac proxy。
- 受限 LAN endpoint。
- 其它明确的真机网络方案。

---

### Full backend 跑着跑着消失

检查：

```bash
./scripts/run-status <RUN_ID>
```

Debian：

```bash
journalctl --user \
  -u myproduct-backend-<RUN_ID>
```

然后：

```bash
cat /srv/myproduct/test-runs/<RUN_ID>/backend/exit_code
```

如果 exit_code 不存在：

```text
标记 interrupted
```

不要当通过。

---

### Debian 磁盘突然满

检查：

```bash
df -h
docker system df
du -sh /srv/myproduct/test-runs/*
```

并确认：

```bash
docker info --format '{{.LoggingDriver}}'
```

没有继续使用无 rotation 的无限 `json-file`。

---

## 107. 不要做的事情

- 不把 ios/backend 分别作为两台机器上的主开发副本。
- 不让 Debian mirror 成为人工编辑区。
- 不用 SMB/NFS 让 Xcode 直接开发网络源码。
- 不同步 Mac 的 `node_modules` 到 Linux。
- 不同步 Mac `.venv` 到 Linux。
- 不同步 Xcode DerivedData 到 Debian。
- 不在 long test 中使用 live mirror。
- 不只冻结 backend 而让 iOS full test 使用活跃源码。
- 不把 `.git` mirror 到 Debian 作为日常方案。
- 不为了运行一次测试制造垃圾 commit。
- 不把生产 secret 放到 Debian 开发节点。
- 不把 Docker 2375 暴露出来。
- 不认为 UFW 一定会挡住 Docker published port。
- 不自动 `docker system prune -a`。
- 不自动 `rm -rf /srv/myproduct`。
- 不让 Codex 普通编码任务执行 sudo。
- 不一开始就上 Kubernetes。
- 不先搭复杂 CI 再补本地脚本。
- 不盲目把所有 test worker 数量开到 CPU thread 数。
- 不把缺少 exit code 的 interrupted run 当 success。
- 不假设真实 iPhone 能直接访问 Mac↔Debian `/30`。

---

## 108. 一套完成后的日常体验

```text
Mac VS Code + Codex
        │
        ├── ios/
        ├── backend/
        └── scripts/
              │
              ├── Fast iOS ───────────────► Mac
              │
              ├── Backend Fast ──SSH──────► Debian live mirror
              │
              └── Full Run
                    │
                    ├── iOS snapshot ─────► Mac full lane
                    │
                    └── Backend snapshot ─► Debian full lane
```

完整任务：

```text
Codex 修改前后端
        ↓
Fast Test
        ↓
Simulator 联调
        ↓
create-full-run
        ↓
同一 RUN_ID
   ┌────┴────┐
   ▼         ▼
Mac iOS    Debian backend
snapshot    snapshot
   │         │
   └────┬────┘
        ▼
统一结果
```

这时你可以继续：

```text
Feature B
```

而：

```text
Feature A Full Regression
```

仍然稳定地跑上一轮冻结源码。

---

## 109. 最终推荐

对于这个具体场景，优先级应是：

```text
1. Codex 上下文完整
2. Source of Truth 单一
3. 远程执行透明
4. 测试输入可复现
5. 长测试不阻塞开发
6. 双机并行
7. 缓存和性能
8. 最后再 CI 化
```

最终最重要的设计不是：

> “Debian 上放多少代码？”

而是：

> **Codex 始终在一个完整产品上下文里工作，而 Debian 对开发者和 Codex 来说只是一个可靠、可重复调用的 Linux 执行引擎。**

---

## 附录 A：推荐目录总览

### Mac

```text
~/Developer/MyProduct/
├── .git/
├── .gitignore
├── AGENTS.md
├── README.md
├── integration-lock.env
├── .vscode/
│   ├── settings.json
│   └── tasks.json
├── docs/
├── scripts/
├── ios/
│   ├── .git/
│   ├── AGENTS.md
│   └── scripts/
└── backend/
    ├── .git/
    ├── AGENTS.md
    └── scripts/
```

Full run：

```text
~/Library/Caches/MyProduct/test-runs/<RUN_ID>/
├── product-state.txt
└── ios/
    ├── input/
    ├── work/
    ├── source.sha256
    ├── logs/
    ├── artifacts/
    └── exit_code
```

---

### Debian

```text
/srv/myproduct/
├── dev/
│   └── backend/
├── state/
│   ├── live-backend.meta
│   ├── backend-sync.in-progress
│   └── backend-full.lock
├── config/
│   └── backend.env
├── cache/
├── tools/
│   └── run-backend-full
├── artifacts/
└── test-runs/
    └── <RUN_ID>/
        ├── product-state.txt
        └── backend/
            ├── input/
            ├── work/
            ├── source.sha256
            ├── logs/
            ├── artifacts/
            ├── started_at
            ├── finished_at
            └── exit_code
```

---

## 附录 B：最小落地顺序

如果只想最快开始，不要一次实施全文。

先做：

```text
1. 网线 /30
2. SSH
3. Mac 父目录统一 VS Code
4. AGENTS.md
5. Debian Docker
6. sync-backend
7. dev-up
8. test-backend-fast
```

完成并稳定使用一周。

然后做：

```text
9. RUN_ID
10. backend snapshot
11. iOS snapshot
12. create-full-run
13. systemd user
14. artifacts
15. retention
```

最后：

```text
16. parallel tuning
17. HTTPS
18. self-hosted CI
```

---

## 附录 C：官方参考资料

> 以下链接用于核验本文关键设计点。文档和软件会继续变化，实施时涉及安全/安装的命令应优先查看对应官方最新页面。

**[R1] Visual Studio Code — Multi-root Workspaces**  
https://code.visualstudio.com/docs/editing/workspaces/multi-root-workspaces

**[R2] Visual Studio Code — Working with repositories and remotes**  
https://code.visualstudio.com/docs/sourcecontrol/repos-remotes

**[R3] OpenAI — Codex IDE extension / Codex upgrades**  
https://openai.com/index/introducing-upgrades-to-codex/  
https://marketplace.visualstudio.com/items?itemName=openai.chatgpt

**[R4] OpenAI — Codex / AGENTS.md**  
https://openai.com/index/introducing-codex/  
https://github.com/openai/codex/blob/main/docs/agents_md.md

**[R5] Apple — TN3179 Understanding Local Network Privacy**  
https://developer.apple.com/documentation/technotes/tn3179-understanding-local-network-privacy

**[R6] Docker — Linux post-installation; docker group privileges**  
https://docs.docker.com/engine/install/linux-postinstall/

**[R7] Docker — Install Docker Engine on Debian**  
https://docs.docker.com/engine/install/debian/

**[R8] Docker — Configure logging drivers**  
https://docs.docker.com/engine/logging/configure/

**[R9] Docker — Port publishing and mapping**  
https://docs.docker.com/engine/network/port-publishing/

**[R10] Docker — Packet filtering and firewalls / Docker and UFW**  
https://docs.docker.com/engine/network/packet-filtering-firewalls/

**[R11] Docker — Protect the Docker daemon socket**  
https://docs.docker.com/engine/security/protect-access/

**[R12] Apple — NSAllowsLocalNetworking**  
https://developer.apple.com/documentation/bundleresources/information-property-list/nsapptransportsecurity/nsallowslocalnetworking

**[R13] macOS `cp(1)` / clonefile semantics**  
Run locally:
```bash
man cp
man clonefile
```

**[R14] systemd — `loginctl enable-linger`**  
https://www.freedesktop.org/software/systemd/man/latest/loginctl.html

**[R15] systemd — `systemd-run`**  
https://www.freedesktop.org/software/systemd/man/latest/systemd-run.html

**[R16] GitHub Actions — Secure use of self-hosted runners**  
https://docs.github.com/en/actions/reference/security/secure-use
