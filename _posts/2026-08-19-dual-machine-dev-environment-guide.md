---
layout: post
title: "双机开发环境搭建指南：Mac mini 负责 Apple 开发，Ubuntu 服务器成为开发计算节点"
date: 2026-08-19 00:00:00 +0800
categories: [Development]
tags: [Codex, VS Code, Remote-SSH, Docker, Mac mini, Ubuntu, iOS, 双机开发, 开发环境]
---

如果你手里恰好有一台 **Mac mini**（Apple 平台开发）和一台 **i9-13900K / 64GB RAM / RTX 4090** 的高配 Linux 服务器，很容易陷入两种纠结：

- 把什么都塞在 Mac 上跑，结果内存、磁盘很快吃紧；
- 或者反过来，想让服务器"替代" Mac，却忘了 iOS 开发离不开 Xcode 与 Simulator。

这篇文章给出的是一个折中且务实的方案：**让两台机器各司其职**——Mac 只保留 Apple 平台不可替代的工作，后端、数据库、Docker、构建、测试和 GPU 工作负载全部迁到服务器上，日常统一通过 VS Code 的 Remote-SSH 与 Codex 插件开发前后端。

> 适用场景：
> - Mac mini M2（约 256GB SSD）承担 Apple 平台开发；
> - i9-13900K / 64GB RAM / 2.5TB SSD / RTX 4090 服务器承担后端、数据库、Docker、构建、测试和 GPU 工作负载；
> - 日常使用 VS Code 中的 Codex 插件开发前后端；
> - 使用 Codex 时需要 VPN / 代理网络；
> - iOS 开发需要 Xcode 与 iOS Simulator。

全文按实施顺序展开，大体分为六大部分：

1. 架构与目标（一 ~ 三）
2. 服务器基础：系统、SSH、防火墙（六 ~ 八）
3. VS Code + Codex Remote（九 ~ 十）
4. 网络方案：VPN / Proxy 的三种选择（十一 ~ 十五）
5. Docker 与服务迁移（十六 ~ 二十二）
6. iOS 联调、日常维护与排查（二十三 ~ 四十二）

---

## 一、目标

这套方案的目标不是让高配服务器"替代 Mac"，而是让两台机器各自承担最适合自己的工作：

- **Mac mini：Apple 开发终端**
  - Xcode
  - Swift / SwiftUI
  - iOS Simulator
  - 真机调试
  - 签名、Archive、发布
  - 本地 VS Code + Codex（iOS 项目）
  - VPN / 代理客户端

- **13900K + 4090 服务器：开发计算中心**
  - Linux
  - VS Code Server / Remote-SSH
  - Codex Remote 工作区
  - 后端源码
  - Web 前端源码
  - Docker Engine
  - PostgreSQL / MySQL / Redis / MinIO 等
  - Node / Python / Java / Go 等运行环境
  - 编译、测试、Lint、构建
  - RTX 4090 AI / CUDA 工作负载

最终希望把 Mac 上最消耗资源的部分迁走，只保留 Xcode、Simulator 和必要的客户端界面。

---

## 二、推荐最终架构

```text
                              Internet
                                  │
                         VPN / Proxy 出口
                                  │
             ┌────────────────────┴────────────────────┐
             │                                         │
             │                                         │
      ┌──────▼────────┐                        ┌───────▼─────────┐
      │   Mac mini M2 │                        │  13900K Server  │
      │               │     1G / 2.5G LAN     │                 │
      │ VS Code       │◄──────────────────────►│ Ubuntu Linux    │
      │ Codex Local   │         SSH            │ VS Code Server  │
      │ Xcode         │                        │ Codex Remote     │
      │ Simulator     │                        │ Docker           │
      │ Git           │                        │ Backend / Web    │
      │ VPN / Proxy   │                        │ DB / Redis       │
      │               │                        │ Build / Tests    │
      └───────────────┘                        │ RTX 4090         │
                                               └─────────────────┘
```

### 职责分配

| 工作 | Mac mini | 13900K Server |
|---|---:|---:|
| Xcode | ✅ | ❌ |
| iOS Simulator | ✅ | ❌ |
| Swift / SwiftUI | ✅ | 可存代码但不作为主运行环境 |
| Apple 签名 / Archive | ✅ | ❌ |
| VS Code 界面 | ✅ | VS Code Server |
| Codex iOS 开发 | ✅ 本地 | ❌ |
| Codex 后端 / Web 开发 | VS Code UI | ✅ Remote 工作区 |
| Docker | 尽量不跑 | ✅ |
| PostgreSQL / Redis | ❌ | ✅ |
| Node/Python/Java/Go 后端 | 尽量不跑 | ✅ |
| Web Build / Test | 尽量不跑 | ✅ |
| CUDA / AI | ❌ | ✅ RTX 4090 |
| Git | ✅ | ✅ |

---

## 三、一个关键事实：Codex 和 VPN 怎么处理

你使用的是 **VS Code 中的 Codex 插件**。

当 VS Code 使用 Remote-SSH 打开服务器目录时：

1. Mac 上仍然显示 VS Code 界面；
2. VS Code 会在 Linux 服务器安装 VS Code Server；
3. Workspace 类型扩展通常会在远程 Extension Host 中运行；
4. Codex IDE 扩展使用 Codex app-server；
5. 因此，实际部署时必须验证：**远程服务器侧是否能够完成 Codex 所需的网络访问和认证。**

所以不要把下面两句话等同：

```text
Mac 能使用 Codex
```

和：

```text
Remote-SSH 服务器里的 Codex 也一定能使用
```

Mac 有 VPN，并不自动意味着服务器继承 Mac 的 VPN。**这一条是整个方案里最容易被忽略、也最值得先验证的一点。**

---

## 四、推荐实施顺序

不要一次迁移所有东西，建议按以下阶段推进：

```text
Phase 1  Server 安装 Ubuntu
Phase 2  配好固定 IP / SSH
Phase 3  Mac VS Code Remote-SSH 成功
Phase 4  Server 解决 VPN / Proxy
Phase 5  Remote 工作区 Codex 验证
Phase 6  Docker / DB / Backend 迁移
Phase 7  iOS Simulator 联调 Server API
Phase 8  再考虑 RTX 4090 / AI
```

**在 Phase 5 完成以前，不要大规模迁移代码。**

---

## 五、准备变量

下面所有命令使用示例参数，请按自己的网络环境修改。

```bash
SERVER_USER=dev
SERVER_HOST=dev-server
SERVER_IP=192.168.10.20

MAC_IP=192.168.10.10

BACKEND_PORT=8080
POSTGRES_PORT=5432
REDIS_PORT=6379

# 如果让服务器通过 Mac 的代理上网：
MAC_PROXY_PORT=7890
```

建议在路由器上给两台机器做 DHCP Reservation：

```text
Mac mini:
192.168.10.10

13900K:
192.168.10.20
```

**优先推荐路由器 DHCP 固定地址，而不是一开始就手写 Linux 静态 IP。** 这样以后换网卡、改网关、改 DNS 时更省事。

---

## 六、服务器安装 Ubuntu

### 推荐系统

优先选择：

```text
Ubuntu Server 24.04 LTS
```

理由：

- 稳定；
- Docker 支持成熟；
- NVIDIA 驱动生态成熟；
- VS Code Remote SSH 支持良好；
- 不需要服务器桌面环境。

如果服务器同时还有其他虚拟化需求，再考虑 Proxmox。对于当前开发用途，**直接 Ubuntu Server 更简单。**

### 安装后的基础配置

登录服务器：

```bash
sudo apt update
sudo apt full-upgrade -y

sudo apt install -y \
  openssh-server \
  git \
  curl \
  wget \
  ca-certificates \
  gnupg \
  jq \
  htop \
  tmux \
  build-essential \
  ufw

sudo systemctl enable --now ssh
```

顺便把主机名和时区设置好（时区对日志、测试、数据库时间都很有影响）：

```bash
sudo hostnamectl set-hostname dev-server
sudo timedatectl set-timezone Asia/Shanghai
```

为服务器上的 Git 配置好身份（很多新手会漏掉，导致提交时报错）：

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

检查：

```bash
hostname
ip addr
timedatectl
```

---

## 七、配置 SSH

### Mac 生成独立 SSH Key

建议不要与 GitHub 共用同一把 key，为服务器单独生成一把：

```bash
ssh-keygen -t ed25519 \
  -f ~/.ssh/id_ed25519_dev_server \
  -C "macmini-to-dev-server"
```

复制公钥到服务器：

```bash
ssh-copy-id \
  -i ~/.ssh/id_ed25519_dev_server.pub \
  dev@192.168.10.20
```

如果 macOS 没有 `ssh-copy-id`，可以手动追加：

```bash
cat ~/.ssh/id_ed25519_dev_server.pub | \
ssh dev@192.168.10.20 \
'mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys'
```

验证登录（首次连接会提示确认 host key，确认无误后输入 yes）：

```bash
ssh -i ~/.ssh/id_ed25519_dev_server dev@192.168.10.20
```

### Mac 配置 ~/.ssh/config

编辑：

```bash
nano ~/.ssh/config
```

增加：

```sshconfig
Host dev-server
    HostName 192.168.10.20
    User dev
    IdentityFile ~/.ssh/id_ed25519_dev_server

    ServerAliveInterval 30
    ServerAliveCountMax 3

    ControlMaster auto
    ControlPersist 10m
    ControlPath ~/.ssh/cm-%C
```

配置完成后，直接：

```bash
ssh dev-server
```

即可连接。

> `ServerAliveInterval` 用于避免长时间无操作时连接被 NAT / 防火墙掐断；`ControlMaster / ControlPersist` 让后续连接复用已有通道，重连和 Remote-SSH 都会更快。如果遇到"连接被复用但通道已死"的怪问题，可用 `ssh -O exit dev-server` 清理复用通道。

### SSH 安全加固

**一定先确认 SSH Key 登录成功，再禁止密码登录。**

服务器：

```bash
sudo nano /etc/ssh/sshd_config.d/99-dev-server.conf
```

写入：

```text
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
```

检查配置：

```bash
sudo sshd -t
```

如果没有输出错误：

```bash
sudo systemctl restart ssh
```

此时先不要关闭原 SSH 窗口。另开一个 Mac Terminal：

```bash
ssh dev-server
```

确认能正常登录后，再关闭旧连接。

---

## 八、防火墙

基础规则：

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

只允许 Mac SSH：

```bash
sudo ufw allow from 192.168.10.10 to any port 22 proto tcp
```

如果需要 Mac / Simulator 访问后端：

```bash
sudo ufw allow from 192.168.10.10 to any port 8080 proto tcp
```

启用并查看状态：

```bash
sudo ufw enable
sudo ufw status verbose
```

> 注意：Docker 发布端口时存在绕过部分 UFW 规则的情况（Docker 直接操作 iptables）。因此后面 Docker Compose 中应尽量明确绑定服务器的 LAN IP，而不是简单写 `0.0.0.0:端口`。

---

## 九、VS Code Remote-SSH

### Mac 安装扩展

在 Mac 的 VS Code 安装：

```text
Remote - SSH
```

然后：

```text
Command Palette
→ Remote-SSH: Connect to Host...
→ dev-server
```

第一次连接时 VS Code 会自动在远端安装 VS Code Server。连接成功后，左下角应显示类似：

```text
SSH: dev-server
```

### 创建服务器项目目录

服务器：

```bash
mkdir -p ~/projects
cd ~/projects
```

例如：

```bash
git clone <YOUR_BACKEND_REPO>
cd your-backend
```

然后在 Remote VS Code：

```text
File
→ Open Folder
→ /home/dev/projects/your-backend
```

---

## 十、VS Code + Codex Remote 验证

这是整个方案**最重要的一步**，请务必在迁移任何代码之前完成验证。

### 在 Remote SSH 窗口安装 / 启用 Codex

打开扩展页时，注意 VS Code 会区分：

```text
LOCAL
SSH: dev-server
```

如果 Codex 提示：

```text
Install in SSH: dev-server
```

则安装到远程环境。安装后打开 Codex Sidebar。

> 如果远程安装失败，先确认 Mac 端 VS Code 与远端 VS Code Server 版本匹配，再在 Remote Terminal 里检查网络，最后重试安装。

### 认证

正常情况下直接：

```text
Sign in with ChatGPT
```

完成浏览器登录。

如果远程场景下登录出现问题，可把 Codex CLI 作为排障工具。在服务器安装当前官方 Codex CLI：

```bash
curl -fsSL https://chatgpt.com/codex/install.sh | sh
```

重新登录 Shell 后检查：

```bash
codex --version
```

检查认证状态：

```bash
codex login status
```

如果服务器是 headless 环境且浏览器回调有问题，可尝试：

```bash
codex login --device-auth
```

前提是对应账户 / Workspace 已允许 Device Code 登录。

Codex CLI 和 IDE Extension 会复用 Codex 的登录缓存，因此 CLI 很适合用来排查远程认证问题。

### 判断 Codex 到底运行在哪台机器

在 Remote VS Code 的 Codex 中输入：

```text
不要修改任何文件。

请告诉我当前工作目录、hostname，
并读取当前项目的 README 或 package.json，
然后告诉我项目名称。
```

期望结果：

```text
pwd:
 /home/dev/projects/your-backend

hostname:
 dev-server
```

而不是：

```text
/Users/你的Mac用户名/...
```

同时可以打开 Remote Terminal 确认：

```bash
pwd
hostname
uname -a
```

这一步确认后，才算 Remote Codex 真正成功。

---

## 十一、VPN / Proxy 网络方案

共有三种方案，推荐优先级按"长期稳定性"排序：

```text
方案 A：路由器 / 软路由统一处理
        ↓
方案 B：服务器自己运行 VPN / Proxy
        ↓
方案 C：服务器借 Mac 的代理出口
```

如果你想最快跑通，通常可以先用方案 C，跑通后再决定是否升级为 A / B。

---

## 十二、方案 A：路由器 / 软路由统一代理

长期最省事。

```text
Mac ─────┐
Server ──┼── Router ─── Internet
iPhone ──┘       │
                  └── VPN / Proxy
```

优势：

- Mac 不需要给 Server 当网关；
- Mac 睡眠不会影响服务器；
- Docker / Git / npm / pip / Codex 都可以统一处理；
- 手机真机调试也方便；
- 规则统一维护。

理想的分流规则：

```text
Apple / 国内服务 / LAN
→ DIRECT

需要代理的开发服务
→ PROXY
```

如果你已经有 OpenWrt / pfSense / OPNsense / 软路由环境，这通常是最终最佳方案。

---

## 十三、方案 B：Server 自己运行 VPN / Proxy

如果你的 VPN 服务提供 Linux 客户端，这是结构最清晰的方式。

```text
Mac VPN → Internet

Server VPN → Internet
```

两台机器通过局域网互访：

```text
Mac 192.168.10.10
      │
      └── 192.168.10.20 Server
```

局域网访问不需要绕 VPN。

服务器连接成功后测试：

```bash
curl -I https://chatgpt.com
```

以及：

```bash
curl -I https://github.com
```

然后测试：

```bash
codex login status
```

最后重新启动 VS Code Remote SSH 窗口，再测试 Codex。

---

## 十四、方案 C：Server 通过 Mac 的代理上网

这个方案特别适合：

- VPN 只有 macOS 客户端；
- 暂时不想动路由器；
- 想最快验证整个架构。

拓扑：

```text
13900K Server
      │
      │ HTTP / SOCKS Proxy
      ▼
Mac mini
      │
      │ VPN
      ▼
Internet
```

### Mac 代理软件配置

不同软件名称不同，但需要找到类似配置：

```text
Allow LAN
允许局域网连接
Expose HTTP Proxy
Expose SOCKS Proxy
```

例如：

```text
Mac IP:
192.168.10.10

HTTP Proxy:
192.168.10.10:7890
```

**不要直接把代理端口暴露到公网。** 如果软件允许设置 Allowed Clients，最好只允许：

```text
192.168.10.20
```

即服务器 IP。

### 服务器临时测试代理

服务器：

```bash
export HTTP_PROXY=http://192.168.10.10:7890
export HTTPS_PROXY=http://192.168.10.10:7890

export http_proxy=$HTTP_PROXY
export https_proxy=$HTTPS_PROXY
```

测试：

```bash
curl -I https://chatgpt.com
```

测试 Git：

```bash
git ls-remote https://github.com/openai/codex.git HEAD
```

如果都能返回，说明基础代理路径正常。

### NO_PROXY 很重要

一定排除局域网流量：

```bash
export NO_PROXY="localhost,127.0.0.1,::1,192.168.10.0/24"
export no_proxy="$NO_PROXY"
```

否则可能出现这种错误结构：

```text
Server → Mac Proxy → VPN → 再绕回来访问 Server
```

导致：

- API 慢；
- Docker DB 连接异常；
- WebSocket 异常；
- 本地联调出现奇怪的超时。

### 持久化代理

先不要急着写全局配置，建议按以下顺序验证：

1. 临时 export 验证；
2. Codex 验证；
3. Git / npm / Docker 验证；
4. 最后再持久化。

如果决定持久化，可在服务器用户环境中配置，例如：

```bash
nano ~/.profile
```

加入：

```bash
export HTTP_PROXY=http://192.168.10.10:7890
export HTTPS_PROXY=http://192.168.10.10:7890

export http_proxy=$HTTP_PROXY
export https_proxy=$HTTPS_PROXY

export NO_PROXY="localhost,127.0.0.1,::1,192.168.10.0/24"
export no_proxy="$NO_PROXY"
```

重新登录：

```bash
exit
ssh dev-server
```

验证：

```bash
env | grep -i proxy
```

> 如果 VS Code Server 或某个扩展没有继承这些变量，不要盲目继续加代理层。先从 Remote Terminal 确认环境变量，再重新连接 Remote-SSH，并在 Codex 中重新验证网络。

---

## 十五、Codex 的两个"网络"不要混淆

Codex 涉及两类网络需求，排查问题时要先分清是哪一个出了问题。

### Codex 自己连接服务

这是：

```text
Codex Extension / app-server
→ OpenAI / ChatGPT
```

这决定 Codex 能不能回答和工作。

### Codex 执行的命令访问网络

例如 Codex 执行：

```bash
npm install
pip install
git fetch
curl
```

这是另外一层网络权限。Codex 本地 / IDE 工作流具有 Sandbox 和网络权限控制。

所以可能出现：

```text
Codex 聊天正常
但 npm install 被阻止
```

这不一定是 VPN 问题，也可能是 Codex 的 Sandbox / Network Permission。如果 Codex 执行命令需要联网，按实际需要开启网络权限，不要一上来就给整个 Agent 无限网络和系统权限。

---

## 十六、安装 Docker Engine

在服务器安装官方 Docker Engine。

先移除可能冲突的软件：

```bash
sudo apt remove -y \
  docker.io \
  docker-compose \
  docker-compose-v2 \
  docker-doc \
  podman-docker \
  containerd \
  runc || true
```

添加官方仓库：

```bash
sudo apt update
sudo apt install -y ca-certificates curl

sudo install -m 0755 -d /etc/apt/keyrings

sudo curl -fsSL \
  https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc

sudo chmod a+r /etc/apt/keyrings/docker.asc
```

增加 apt source（deb822 格式）：

```bash
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

安装：

```bash
sudo apt update

sudo apt install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

测试：

```bash
sudo docker run hello-world
```

---

## 十七、让普通用户使用 Docker

```bash
sudo usermod -aG docker $USER
```

退出 SSH：

```bash
exit
```

重新登录：

```bash
ssh dev-server
```

测试：

```bash
docker ps
docker compose version
```

> 注意：加入 `docker` 用户组等价于给该用户非常高的主机控制能力。开发机可以这样做，但不要随意给不可信用户加入 docker group。

---

## 十八、不要开放 Docker 2375

不推荐：

```text
tcp://0.0.0.0:2375
```

特别不要：

```text
Internet
→ 2375
→ Docker daemon
```

如果未来需要 Mac 上的 Docker CLI 管理 Server，优先走 SSH：

```bash
docker context create dev-server \
  --docker host=ssh://dev@192.168.10.20
```

使用：

```bash
docker context use dev-server
docker ps
```

切回：

```bash
docker context use default
```

不过对于当前架构，其实更简单：

```text
Remote VS Code Terminal
→ 直接运行 Server 上的 docker compose
```

这样 Mac 连 Docker CLI 都可以不装。

---

## 十九、推荐开发服务目录

服务器：

```text
/home/dev/
└── projects/
    ├── myapp-backend/
    ├── myapp-web/
    └── infra/
```

例如：

```bash
mkdir -p ~/projects/infra
```

---

## 二十、一个可直接改的 Docker Compose 示例

示例：

```yaml
services:
  postgres:
    image: postgres:17-alpine
    restart: unless-stopped
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: myapp
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "${SERVER_BIND_IP}:5432:5432"

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    volumes:
      - redis_data:/data
    ports:
      - "${SERVER_BIND_IP}:6379:6379"

  api:
    build:
      context: ../myapp-backend
    restart: unless-stopped
    environment:
      DATABASE_URL: postgresql://myapp:${POSTGRES_PASSWORD}@postgres:5432/myapp
      REDIS_URL: redis://redis:6379
    depends_on:
      - postgres
      - redis
    ports:
      - "${SERVER_BIND_IP}:8080:8080"

volumes:
  postgres_data:
  redis_data:
```

`.env`：

```bash
SERVER_BIND_IP=192.168.10.20
POSTGRES_PASSWORD=PLEASE_CHANGE_ME
```

**不要提交 `.env`。**

`.gitignore`：

```gitignore
.env
.env.local
```

提交：

```text
.env.example
```

而不是实际 secret。

> 关于内存限制：老写法 `mem_limit: 4g` 仍可用，但 Compose Spec 推荐在 `deploy.resources.limits` 下声明。具体写法见第 34 节。

---

## 二十一、Docker 网络建议

数据库通常不需要给 Mac 直接访问，更安全的结构是：

```text
PostgreSQL
   ↑
Docker internal network
   │
API
   │
   ↓
192.168.10.20:8080
   │
   ↓
Mac / Simulator
```

因此生产化一点可以删除：

```yaml
postgres:
  ports:
```

以及：

```yaml
redis:
  ports:
```

只保留 API 对 LAN 暴露。

这样：

```text
Mac
只能直接访问 API

不能直接访问数据库
```

如果临时需要数据库 GUI，可使用 SSH Tunnel，而不是永久开放 DB：

```bash
ssh -L 15432:127.0.0.1:5432 dev-server
```

然后 Mac 数据库客户端连接：

```text
127.0.0.1:15432
```

---

## 二十二、后端迁移原则

Mac 上尽量停止：

```text
Docker Desktop
PostgreSQL
Redis
Elasticsearch
MinIO
后台 Node Server
后台 Python Server
大型测试
前端 production build
```

服务器承担：

```text
docker compose up -d
npm / pnpm
pytest
gradle
maven
go test
eslint
tsc
vite build
next build
```

Mac 保留：

```text
Xcode
Simulator
VS Code UI
浏览器
必要的轻量工具
```

---

## 二十三、iOS Simulator 连接 Server API

Server API：

```text
192.168.10.20:8080
```

iOS 开发配置：

```text
Development:
http://192.168.10.20:8080

Production:
https://api.example.com
```

不要把地址散落在代码中。例如 Swift 可以这样组织：

```swift
enum AppEnvironment {
    case development
    case production

    var baseURL: URL {
        switch self {
        case .development:
            return URL(string: "http://192.168.10.20:8080")!
        case .production:
            return URL(string: "https://api.example.com")!
        }
    }
}
```

更推荐使用：

```text
xcconfig
```

或：

```text
Build Configuration
```

管理不同环境，把地址集中在一处、并区分 Debug / Release。

---

## 二十四、HTTP / ATS 注意事项

iOS 对 HTTP 有 App Transport Security（ATS）限制。开发阶段有三种选择：

### 推荐

本地开发 API 也使用 HTTPS，例如：

```text
Caddy / nginx
→ HTTPS
→ API
```

### 次选

仅 Debug 配置中增加开发例外。

### 不推荐

生产 App 里长期：

```text
NSAllowsArbitraryLoads = YES
```

如果为了本地 HTTP 临时使用 ATS 例外，确保：

```text
Debug 有
Release 没有
```

上线前必须检查。

---

## 二十五、真机调试

如果不是 Simulator，而是 iPhone：

```text
iPhone
  │ Wi-Fi
  ▼
Router / AP
  │
  ▼
192.168.10.20:8080
```

需要满足：

- iPhone 与服务器在可互访的 LAN；
- Wi-Fi 不启用 Client Isolation；
- 后端端口允许 LAN 访问；
- App 根据 iOS 本地网络权限要求正确配置。

iOS 14+ 访问局域网设备会触发本地网络权限弹窗，需要在 `Info.plist` 中声明用途描述：

```xml
NSLocalNetworkUsageDescription
```

并建议开发网络不要使用访客 Wi-Fi。

---

## 二十六、推荐代码仓库结构

### 方案 1：分仓库（长期推荐）

```text
myapp-ios/
myapp-backend/
myapp-web/
myapp-api-contract/
```

Mac：

```text
myapp-ios
```

Server：

```text
myapp-backend
myapp-web
```

优点：

- 职责清晰；
- Codex 上下文更小；
- Xcode 不碰 Remote FS；
- 后端完全 Server 化；
- CI 容易拆分。

### 方案 2：保留 Monorepo

如果现在已经是：

```text
myapp/
├── ios/
├── backend/
├── web/
└── infra/
```

不需要为了迁移立刻重构。

可以：

Mac Clone：

```text
~/Projects/myapp
```

主要打开：

```text
ios/
```

Server Clone：

```text
~/projects/myapp
```

主要打开：

```text
backend/
web/
infra/
```

注意：**两台机器不是自动同步文件系统的。**

正确同步方式：

```text
Git commit
Git push
Git pull
```

不要：

```text
Mac 修改一半
Server 修改另一半
然后靠 rsync 强行互相覆盖
```

---

## 二十七、API 契约建议

iOS 与后端最好共享接口契约：

```text
OpenAPI
JSON Schema
Protobuf
```

例如：

```text
api/openapi.yaml
```

工作流：

```text
Server Codex
→ 修改 Backend API
→ 更新 openapi.yaml
→ commit

Mac pull
→ Codex 根据 OpenAPI 修改 Swift API Client
```

这样前后端不会靠"聊天记忆"同步接口。

---

## 二十八、建议写一个 AGENTS.md

Codex 很适合通过项目规则固定工作方式。

Server 项目：

```markdown
# Development Rules

- Backend runs on the Linux development server.
- Use Docker Compose for PostgreSQL and Redis.
- Never start PostgreSQL or Redis directly on the Mac.
- Run tests before finishing a backend task.
- Do not modify iOS files from this workspace.
- API contract lives in api/openapi.yaml.
- Do not commit .env or secrets.
```

iOS 项目：

```markdown
# iOS Development Rules

- This workspace is for iOS code only.
- Build and test using Xcode on macOS.
- Development backend is configured through xcconfig.
- Do not hardcode production API URLs.
- Do not enable unrestricted ATS settings in Release.
- Update the API client from the shared OpenAPI contract.
```

这样 Codex 每次工作更容易遵循你的双机边界。

---

## 二十九、日常开发工作流

### 开始工作

Mac：

```text
1. 启动 VPN
2. 打开 Xcode
3. 启动一个 Simulator
4. 打开 iOS VS Code / Codex
5. VS Code Remote-SSH → dev-server
```

Server Remote Terminal：

```bash
cd ~/projects/myapp
docker compose up -d
```

检查：

```bash
docker compose ps
```

后端日志：

```bash
docker compose logs -f api
```

### 开发后端

在：

```text
VS Code
SSH: dev-server
```

给 Codex：

```text
实现 refresh token。
修改数据库 migration。
运行相关测试。
最后给我列出修改文件、测试结果和 API 变化。
```

这些工作运行在：

```text
13900K / 64G
```

而不是 Mac。

### 开发 iOS

Mac 本地 Codex：

```text
根据最新 OpenAPI 更新 AuthService。
修改 LoginView。
不要改后端文件。
完成后告诉我需要在 Xcode 中运行哪些测试。
```

Xcode：

```text
Build
→ Simulator
→ 调用 192.168.10.20:8080
```

---

## 三十、推荐的双窗口布局

```text
┌─────────────────────┬──────────────────────┐
│ VS Code Remote      │ Xcode                │
│ SSH: dev-server     │                      │
│                     │ Swift / SwiftUI      │
│ Backend             │ Simulator            │
│ Web                 │                      │
│ Codex Remote        │                      │
├─────────────────────┼──────────────────────┤
│ Remote Terminal     │ Simulator            │
│ docker compose      │                      │
│ logs                │ App UI               │
└─────────────────────┴──────────────────────┘
```

如果需要 iOS Codex：

```text
第二个 VS Code 窗口
LOCAL
→ myapp-ios
```

---

## 三十一、Mac mini 资源优化

完成迁移后，Mac 上应该尽量不再运行：

```text
Docker Desktop
Postgres.app
本地 Redis
大型 Node Server
大型 Java 服务
Elasticsearch
本地 AI 模型
```

检查：

```text
Activity Monitor
→ Memory
→ Memory Pressure
```

理想状态：

```text
Xcode
Simulator
VS Code
Codex
浏览器
```

同时运行时 Memory Pressure 仍能维持可接受范围。

---

## 三十二、Mac 256GB SSD 优化

iOS 开发特别容易消耗 SSD，重点检查：

```text
Xcode
Simulator Runtime
DerivedData
Archives
DeviceSupport
项目 node_modules
Docker Desktop 数据
```

如果已经把 Docker 迁到服务器：**Mac 可直接卸载 / 停用 Docker Desktop。**

Xcode 可定期清理不可用的模拟器运行时：

```bash
xcrun simctl delete unavailable
```

清 DerivedData 前：**先关闭 Xcode**，再根据需要删除旧缓存。

不要把：

```text
CoreSimulator 系统目录
```

随意用软链接迁到 NAS / 网络盘，稳定性没有保障。

如果需要扩容，优先给 Mac 增加高速 USB4 / Thunderbolt NVMe SSD，用于：

- 项目；
- 大型素材；
- Archives；
- 可迁移缓存。

---

## 三十三、RTX 4090：第二阶段再启用

先把：

```text
Remote-SSH
Codex
Docker
Backend
iOS 联调
```

跑顺，然后再处理 GPU。

Linux 检查：

```bash
nvidia-smi
```

正常后再安装 NVIDIA Container Toolkit。示例流程：

```bash
sudo apt-get update
sudo apt-get install -y --no-install-recommends \
  ca-certificates \
  curl \
  gnupg2
```

增加 NVIDIA Container Toolkit 仓库后安装：

```bash
sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit
```

配置 Docker Runtime：

```bash
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

检查：

```bash
docker info | grep -i runtime
```

以后可以运行：

```text
Ollama
vLLM
Embedding
RAG
Whisper
图像模型
CUDA 开发
```

此时 4090 才真正参与开发体系。

---

## 三十四、推荐 Server 的资源分配原则

64GB RAM 可以大致按逻辑分：

```text
Host Linux:
4~8 GB

VS Code / Codex / 编译:
8~16 GB

Docker 开发服务:
16~32 GB

其余:
文件缓存 / GPU 工作流
```

不需要人工硬切，Linux 会自动使用空闲内存作为 Page Cache。真正需要防止的是单个容器无限吃内存。

Docker Compose 可以按需要设置（Compose Spec 推荐写法）：

```yaml
services:
  api:
    deploy:
      resources:
        limits:
          memory: 4g

  postgres:
    deploy:
      resources:
        limits:
          memory: 8g
```

仅在确实遇到资源争抢后再加，不必一开始过度限制。

> 兼容提示：`docker compose`（v2）支持 `deploy.resources.limits.memory`；老式的 `mem_limit` 也仍然可用，但新项目建议用前者。

---

## 三十五、安全原则

必须遵守：

- 不把 SSH 22 直接暴露到公网，除非你明确做好额外安全控制；
- 不公开暴露 Docker 2375；
- 不把 Codex app-server transport 暴露到公网；
- Remote 访问优先使用 SSH / VPN / Mesh Network；
- `.env` 不进 Git；
- Codex 不需要时不要给无限网络 / 无限系统权限；
- Docker 数据库端口不要无脑 `0.0.0.0`；
- SSH 禁止 root 登录；
- SSH Key 成功验证后再关闭密码登录；
- Mac 代理若允许 LAN，只允许可信 LAN / Server IP；
- 服务器不要使用与你日常个人账户相同的弱密码。

如果服务器后续暴露到公网，可以再考虑加装 `fail2ban`、限制 SSH 来源 IP、以及定期 `apt upgrade`。

---

## 三十六、验收 Checklist

### Server 基础

- [ ] Ubuntu Server 安装完成
- [ ] Server IP 固定
- [ ] `hostname` 正确
- [ ] SSH 服务启动
- [ ] Mac SSH Key 登录成功
- [ ] SSH 密码登录已关闭
- [ ] UFW 基础规则完成

### VS Code

- [ ] Mac 安装 Remote-SSH
- [ ] `SSH: dev-server` 可连接
- [ ] Remote Folder 可打开
- [ ] VS Code Terminal 中 `hostname` 显示 Server
- [ ] VS Code Server 工作正常

### Codex

- [ ] Remote 环境 Codex 可打开
- [ ] ChatGPT 登录成功
- [ ] Codex 可读取 Remote 文件
- [ ] Codex 返回的 `pwd` 是 `/home/dev/...`
- [ ] Codex 执行的 `hostname` 是 `dev-server`
- [ ] Codex 能修改 Remote 文件
- [ ] Codex 能运行 Remote 测试

### Network / VPN

- [ ] Server DNS 正常
- [ ] Server 可通过目标代理路径联网
- [ ] `curl -I https://chatgpt.com` 成功
- [ ] `git fetch` 成功
- [ ] npm / pip 等按需求成功
- [ ] `NO_PROXY` 包含 LAN

### Docker

- [ ] Docker Engine 安装
- [ ] `docker ps` 无需 sudo
- [ ] `docker compose version` 正常
- [ ] Postgres 正常
- [ ] Redis 正常
- [ ] API 正常
- [ ] DB 端口没有不必要的公网暴露

### iOS

- [ ] Simulator 能访问 Server IP
- [ ] Development API Base URL 已独立配置
- [ ] Debug / Release 环境分离
- [ ] ATS 例外不会进入 Release
- [ ] 真机在需要时也可访问开发 API

### Mac 减负

- [ ] Docker Desktop 已停用或卸载
- [ ] 本地 PostgreSQL 已停用
- [ ] 本地 Redis 已停用
- [ ] Backend 默认不再运行在 Mac
- [ ] Web 大型构建默认在 Server
- [ ] Mac 主要资源留给 Xcode + Simulator

---

## 三十七、常见问题排查

### 1. Remote VS Code 能连接，但 Codex 一直转圈

先打开 Remote Terminal：

```bash
hostname
pwd
curl -I https://chatgpt.com
```

如果 curl 不通，说明是服务器网络问题，**先解决 Server VPN / Proxy**，不是 VS Code 的问题。

如果 curl 通：

```bash
codex login status
```

再：

```bash
codex --version
```

最后重新连接 Remote SSH。

### 2. Mac Codex 正常，Remote Codex 不正常

这是非常典型的现象：

```text
Mac VPN
≠
Server VPN
```

检查：

```bash
ssh dev-server

env | grep -i proxy
curl -I https://chatgpt.com
```

### 3. Server 走 Mac Proxy，但突然 Codex 断了

检查：

```text
Mac 是否睡眠
VPN 是否掉线
代理软件是否退出
Allow LAN 是否关闭
Mac IP 是否变化
代理端口是否变化
```

这也是为什么长期建议：

```text
Router Proxy
```

或：

```text
Server 自己的 VPN
```

### 4. Codex 能聊天，但 npm install 失败

可能是两类问题：

```text
A. Server 网络问题
B. Codex Sandbox 网络权限
```

先在 Remote Terminal 手工验证：

```bash
npm ping
```

如果 Terminal 正常、Codex 命令失败，再检查 Codex Permissions / Network Access。

### 5. Simulator 调不到 Server API

Mac 先测试：

```bash
curl http://192.168.10.20:8080/health
```

如果 Mac 都不通，检查：

```text
API listen address
Docker port
Server firewall
```

后端必须监听：

```text
0.0.0.0
```

或服务器 LAN interface，不能只监听：

```text
127.0.0.1
```

如果 Mac curl 正常、Simulator 不正常，检查：

```text
ATS
App Environment URL
iOS Local Network 权限
```

### 6. Docker API 可访问，但数据库不应该暴露

检查监听端口：

```bash
ss -lntp
```

不需要从 LAN 访问 PostgreSQL 时，应避免：

```text
0.0.0.0:5432
```

更推荐：

```text
Postgres
仅 Docker internal network
```

需要 GUI 时用：

```bash
ssh -L
```

### 7. Remote-SSH 首次连接失败 / Permission denied

按顺序排查：

```text
1. host key 确认：首次连接是否接受了正确的指纹
2. 公钥是否写入 authorized_keys，且权限为 600
3. ~/.ssh 目录权限是否为 700
4. 是否启用了 PasswordAuthentication no 但 key 还没配好
5. UFW 是否只允许了 Mac IP 的 22 端口
```

---

## 三十八、最推荐的第一天实施范围

第一天不要折腾 GPU，只完成下面 8 件事：

```text
1. Ubuntu
2. 固定 IP
3. SSH Key
4. VS Code Remote SSH
5. Server VPN / Proxy
6. Remote Codex
7. Docker
8. 跑一个测试 Backend
```

测试项目甚至可以只是：

```text
hello-api
```

Server：

```text
GET /health
→ {"ok": true}
```

Mac：

```bash
curl http://192.168.10.20:8080/health
```

Simulator：请求同一个 API。

只要这一条链跑通：

```text
Codex Remote
      ↓
修改 Backend
      ↓
13900K Build / Test
      ↓
Docker
      ↓
192.168.10.20:8080
      ↓
Mac
      ↓
iOS Simulator
```

整个架构就成立了。

---

## 三十九、最终日常状态

理想状态应该是：

```text
Mac mini
CPU:
主要给 Xcode / Simulator

Memory:
主要给 Xcode / Simulator / VS Code

Disk:
不再被 Docker Images / DB 大量占用
```

Server：

```text
13900K
→ 编译 / 测试 / Backend / Web

64GB
→ Docker / DB / Cache / Codex

2.5TB SSD
→ Repo / Docker Volume / Build Cache

4090
→ AI / CUDA
```

---

## 四十、最终建议

对于你的硬件，不建议：

```text
把所有东西继续塞在 Mac
```

也不建议：

```text
强行让 13900K 替代 macOS / iOS Simulator
```

最合理的是：

```text
Mac = Apple Development Workstation

Server = Linux Development Compute Node
```

具体工作流：

```text
iOS:
Mac Local VS Code / Codex
+
Xcode
+
Simulator

Backend / Web:
Mac VS Code UI
+
Remote SSH
+
Server Codex
+
Server Docker
+
Server Tests
```

网络则保证：

```text
Mac 可以访问 Codex
Server Remote Codex 也可以访问 Codex
```

最终得到：

```text
             Mac mini
       Xcode / Simulator
             │
             │ LAN
             ▼
      13900K / 64G Server
    Codex / Docker / Backend
        DB / Build / Test
             │
             ▼
           RTX4090
```

这套架构的重点不是"远程控制一台服务器"，而是让服务器真正成为你的 **开发计算节点**，Mac 只做 Apple 平台不可替代的工作。

---

## 四十一、官方技术依据

本文档在 2026-08-19 根据以下官方资料核对了关键行为：

- OpenAI — Codex IDE extension
- OpenAI — Codex Authentication
- OpenAI — Codex App Server
- OpenAI — Codex Remote Connections
- OpenAI — Codex Agent Approvals & Security / Network
- OpenAI — Codex CLI
- Microsoft — Visual Studio Code Remote Development using SSH
- Microsoft — Supporting Remote Development and Remote Extension Host
- Docker — Install Docker Engine on Ubuntu
- Docker — Protect the Docker daemon socket / SSH Context
- NVIDIA — NVIDIA Container Toolkit Installation Guide

参考地址可从以下官方站点查找：

```text
https://developers.openai.com/
https://learn.chatgpt.com/docs/
https://code.visualstudio.com/docs/remote/ssh
https://docs.docker.com/
https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/
```

> 由于 Codex 与 VS Code 的版本迭代较快，实施时建议再查一遍上述官方文档的最新版本。

---

## 四十二、后续可继续细化的三个文档

在这份总方案跑通后，可以再单独整理：

```text
01-network.md
Mac VPN / Clash / Surge / 具体代理软件
如何安全共享给 Ubuntu

02-server-bootstrap.md
一键安装 Ubuntu 开发环境
Docker / Git / Node / Python / CUDA

03-myapp-dev.md
针对你实际项目的目录
Docker Compose
环境变量
API
数据库
Codex AGENTS.md
iOS xcconfig
```

其中最值得优先继续细化的是：

```text
01-network.md
```

因为你当前整个双机 Codex 架构能否成立，最关键的不是 13900K 或 4090，而是：

```text
Remote Codex 所在的 Server
是否拥有稳定、可控的网络出口。
```
