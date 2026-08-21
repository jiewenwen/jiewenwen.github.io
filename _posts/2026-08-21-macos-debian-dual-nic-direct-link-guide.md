---
layout: post
title: "macOS ↔ Debian 双网卡直连开发环境搭建"
date: 2026-08-21 00:00:00 +0800
categories: [Development]
tags: [macOS, Debian, 双网卡, 有线直连, mDNS, Avahi, SSH, 双机开发, 开发环境]
---

如果你在做双机开发——Mac 负责 Xcode / iOS Simulator，Debian 台式机负责后端、Docker、数据库和编译——那么 Mac 与 Debian 之间最值得优先投入的，往往不是更快的 Wi‑Fi，而是一根 **Cat6 网线直连**。

本文介绍一套 **双网卡有线直连** 方案：

- Mac 的 Ethernet 与 Debian 的一张网卡通过独立网线 **点对点直连**，SSH 与开发流量全程走网线，稳定、低延迟、不占用 Wi‑Fi 带宽；
- Debian 通过 **第二张网卡**（板载网口 / Wi‑Fi / USB 网卡）独立访问互联网，`apt update`、`docker pull`、`git clone` 都不依赖 Mac；
- Mac 用 `devbox.local` 访问 Debian，由 mDNS / Avahi 自动解析到直连网卡的 `10.10.10.2`，**不需要记 IP**，Debian 换个网络地址也不影响。

> 适用场景：
> - 一台 Mac（笔记本 / mini）承担 Xcode、iOS Simulator 等 Apple 平台开发；
> - 一台 Debian 台式机承担后端、Docker、数据库与编译任务；
> - 希望 SSH、后端 API、数据库连接走专用网线，稳定且不占 Wi‑Fi；
> - 希望 Debian 的 Internet 访问与开发链路相互独立，互不拖累。

> 如果你还没有搭建双机开发的整体框架，建议先阅读《{% post_url 2026-08-19-dual-machine-dev-environment-guide %}》；本文是其中「网络链路」部分的独立深化版，也可以单独使用。

全文按实施顺序展开，分为以下几个部分：

1. 方案概览与 IP 规划（1 ~ 3）
2. macOS 有线接口配置（4）
3. Debian 双网卡配置与路由验证（5 ~ 8）
4. 打通 `.local` 与 SSH（9 ~ 21）
5. 后端服务、Docker 与 iOS 联调（22 ~ 24）
6. Headless 使用、健康检查与故障排查（25 ~ 35）
7. 日常使用方式与配置模板（36 ~ 40）

---

## 1. 方案概览

推荐的最终网络结构：

```text
                         Internet
                            │
                    ┌───────┴───────┐
                    │               │
                Wi‑Fi/路由器     Wi‑Fi/路由器
                    │               │
                    ▼               ▼
               ┌─────────┐      ┌─────────────┐
               │ macOS   │      │   Debian    │
               │ Mac M2  │      │ 13900K PC   │
               └────┬────┘      └──────┬──────┘
                    │                  │
             Ethernet/USB-C       Ethernet NIC 1
             10.10.10.1/24        10.10.10.2/24
                    │                  │
                    └──── Cat6 ────────┘

                                      │
                                NIC 2 / Wi‑Fi
                                      │
                                      ▼
                                  Internet
```

Debian 的两个网络接口职责明确：

| 设备 | 接口 | 示例地址 | 默认网关 | 用途 |
|---|---|---:|---|---|
| Mac | Wi‑Fi | DHCP | 有 | Mac 自己上网 |
| Mac | Ethernet | `10.10.10.1/24` | **无** | 直连 Debian |
| Debian | Ethernet NIC 1 | `10.10.10.2/24` | **无** | 直连 Mac |
| Debian | Wi‑Fi / NIC 2 | DHCP | 有 | Debian 自己上网 |

最重要的设计原则：

> **Mac ↔ Debian 的直连网卡不配置默认网关，也不配置 DNS。**

这样：

- `ssh devbox.local` → 走 Cat6 网线；
- `http://devbox.local:3000` → 走 Cat6 网线；
- Debian 的 `apt update`、`docker pull`、`git clone` → 走 Debian 自己的第二张网卡；
- Mac 自己访问 Internet → 走 Mac 的 Wi‑Fi；
- 两台机器互不依赖对方进行 NAT/网络共享。

---

## 2. 硬件要求

### 2.1 Mac

Mac 需要一个可用的 Ethernet 接口。

如果 Mac 没有 RJ45：

```text
Mac USB-C / Thunderbolt
        │
        ▼
USB-C → Ethernet Adapter
        │
        ▼
      Cat6
```

1GbE 已经足够 SSH、后端 API、数据库和远程开发。

如果未来需要频繁传输大型 Docker 镜像、数据集或构建产物，可以使用 2.5GbE。

### 2.2 Debian 台式机

理想情况：

```text
NIC 1 → Mac 专用直连
NIC 2 → 路由器 / Internet
```

第二个接口可以是：

- 主板 Wi‑Fi；
- USB Wi‑Fi；
- 第二个板载 RJ45；
- PCIe Ethernet 网卡；
- USB Ethernet 网卡。

### 2.3 网线

推荐：

```text
Cat6
```

Cat6 可以正常工作在：

- 1GbE；
- 2.5GbE；
- 更高速率取决于距离和设备能力。

现代网卡通常支持 Auto MDI-X，因此 Mac 和 Debian 可以直接使用普通网线连接，不需要交叉线。

---

## 3. IP 地址规划

本文统一使用：

```text
Mac Ethernet:
10.10.10.1/24

Debian Direct Ethernet:
10.10.10.2/24
```

子网：

```text
10.10.10.0/24
```

对应子网掩码：

```text
255.255.255.0
```

**直连网络两端都不要配置 Router / Gateway。**

如果你的家庭网络、公司网络或 VPN 已经使用 `10.10.10.0/24`，请换一个不冲突的私有网段，例如：

```text
192.168.250.0/24

Mac:     192.168.250.1
Debian:  192.168.250.2
```

全文中的地址相应替换即可。

---

## 4. 配置 macOS 的有线接口

### 4.1 插入网线

连接：

```text
Mac Ethernet Adapter
        │
       Cat6
        │
Debian Direct NIC
```

不需要交换机，也不需要路由器。

---

### 4.2 macOS 设置静态 IPv4

打开：

```text
系统设置
→ 网络
→ Ethernet / USB LAN / 你的网卡名称
→ 详细信息
→ TCP/IP
```

设置：

```text
配置 IPv4：手动

IP 地址：
10.10.10.1

子网掩码：
255.255.255.0

路由器：
留空
```

DNS 不需要为这张直连网卡添加任何服务器。

最终应该是：

```text
Mac Ethernet
IP:       10.10.10.1
Mask:     255.255.255.0
Gateway:  none
DNS:      none
```

Apple 官方支持在 macOS 的 Network → TCP/IP 中手动设置 IPv4 地址、子网掩码和 Router。

> **同时开着 Wi‑Fi 和 Ethernet 也没关系：** macOS 会按「目标子网」选路。直连网段是独立的 `10.10.10.0/24`，只有网线两端在用，所以去 `10.10.10.2` 的流量必然走 Ethernet，其余流量仍走 Wi‑Fi，两者互不干扰；只要不给 Ethernet 填 Router，就不会影响 Mac 自己的默认路由。

---

### 4.3 查看 Mac 网络接口

Terminal：

```bash
ifconfig
```

也可以：

```bash
networksetup -listallnetworkservices
```

假设 USB Ethernet 最终对应：

```text
en7
```

后续排错时会用到这个接口名。

---

## 5. 配置 Debian 双网卡

下面主要使用 NetworkManager，因为 Debian Desktop 以及很多常见 Debian 安装默认使用它。

首先查看设备：

```bash
nmcli device status
```

也可以：

```bash
ip -br link
```

示例：

```text
DEVICE    TYPE      STATE
enp5s0    ethernet  connected
wlp4s0    wifi      connected
lo        loopback  connected
```

假设：

```text
enp5s0 = Mac 直连网卡
wlp4s0 = Internet Wi‑Fi
```

**请根据自己的机器修改接口名称。**

---

## 6. Debian：配置 Mac 专用直连网卡

创建一个 NetworkManager connection：

```bash
sudo nmcli connection add \
  type ethernet \
  ifname enp5s0 \
  con-name mac-direct \
  ipv4.method manual \
  ipv4.addresses 10.10.10.2/24 \
  ipv4.never-default yes \
  ipv6.method disabled
```

启用开机自动连接：

```bash
sudo nmcli connection modify mac-direct \
  connection.autoconnect yes
```

启动：

```bash
sudo nmcli connection up mac-direct
```

> **如果 `enp5s0` 之前插过路由器：** NetworkManager 可能已有一条旧的自动连接（例如 `Wired connection 1`）与 `mac-direct` 抢占这张网卡，导致 `mac-direct` 起不来或路由不对。可以先删除旧连接再重新启动：

```bash
sudo nmcli connection delete "Wired connection 1"
sudo nmcli connection up mac-direct
```

检查：

```bash
nmcli connection show mac-direct
```

以及：

```bash
ip -4 addr show dev enp5s0
```

应该看到：

```text
inet 10.10.10.2/24
```

---

### 6.1 为什么使用 `ipv4.never-default yes`

这是本方案非常重要的一项设置。

它表示：

> `mac-direct` 这张网卡永远不能成为 Debian 的默认 Internet 路由。

因此 Debian 不会错误地尝试通过 Mac 上网。

NetworkManager 官方定义中，`ipv4.never-default=yes` 表示该连接不会被分配 IPv4 默认路由。

---

## 7. Debian：配置第二张网卡访问 Internet

### 7.1 如果第二个接口是 Wi‑Fi

查看 Wi‑Fi：

```bash
nmcli device wifi list
```

建议使用交互方式输入密码，避免密码写进 shell history：

```bash
sudo nmcli --ask device wifi connect "你的 Wi-Fi SSID" ifname wlp4s0
```

检查：

```bash
nmcli device status
```

然后：

```bash
ip route
```

应该出现类似：

```text
default via 192.168.1.1 dev wlp4s0
10.10.10.0/24 dev enp5s0 proto kernel scope link src 10.10.10.2
192.168.1.0/24 dev wlp4s0 proto kernel scope link
```

---

### 7.2 如果第二个接口也是 Ethernet

例如：

```text
enp5s0 → Mac
enp6s0 → 路由器
```

如果 `enp6s0` 尚未配置：

```bash
sudo nmcli connection add \
  type ethernet \
  ifname enp6s0 \
  con-name internet \
  ipv4.method auto \
  ipv6.method auto
```

启动：

```bash
sudo nmcli connection up internet
```

路由器的 DHCP 会为该接口提供：

- IP；
- 默认网关；
- DNS。

---

## 8. 验证 Debian 路由是否正确

执行：

```bash
ip route
```

正确结构应该类似：

```text
default via 192.168.1.1 dev wlp4s0
10.10.10.0/24 dev enp5s0 scope link src 10.10.10.2
192.168.1.0/24 dev wlp4s0 scope link
```

### 验证去 Mac 的流量

```bash
ip route get 10.10.10.1
```

应该包含：

```text
dev enp5s0
src 10.10.10.2
```

### 验证去 Internet 的流量

```bash
ip route get 1.1.1.1
```

应该显示：

```text
dev wlp4s0
```

或者 Debian 的第二张有线网卡，例如：

```text
dev enp6s0
```

---

## 9. 先验证 Mac ↔ Debian 的基础网络

在 Debian：

```bash
ping -c 4 10.10.10.1
```

在 Mac：

```bash
ping -c 4 10.10.10.2
```

如果正常，通常会看到非常低的 RTT：

```text
< 1 ms
```

如果 IP 互 ping 都失败，请暂时不要继续 `.local` 和 SSH 配置，先检查：

- 网线；
- USB Ethernet Adapter；
- Debian 网卡是否 UP；
- 两边 IPv4 是否在同一 `/24`；
- 防火墙。

---

## 10. 配置 Debian 主机名

本文使用：

```text
devbox
```

设置：

```bash
sudo hostnamectl set-hostname devbox
```

检查：

```bash
hostname
```

应该输出：

```text
devbox
```

检查 `/etc/hosts`：

```bash
cat /etc/hosts
```

如果存在：

```text
127.0.1.1 old-hostname
```

建议改成：

```text
127.0.1.1 devbox
```

编辑：

```bash
sudoedit /etc/hosts
```

---

## 11. 配置 `.local`：Avahi / mDNS

`.local` 不是传统 DNS 的静态映射。

本方案使用：

```text
mDNS / Bonjour
```

Debian 上使用：

```text
Avahi
```

macOS 原生支持 Bonjour/mDNS。

目标是让：

```text
devbox.local
```

自动解析到 Debian 当前公布的地址。

---

### 11.1 安装 Avahi

Debian：

```bash
sudo apt update
sudo apt install avahi-daemon avahi-utils
```

启动并设置开机启动：

```bash
sudo systemctl enable --now avahi-daemon
```

检查：

```bash
systemctl status avahi-daemon
```

Debian 的 `avahi-daemon` 会发布本机地址；默认 mDNS 域就是 `.local`。

---

## 12. 强烈推荐：让 `.local` 只通过 Mac 直连网卡发布

这是双网卡环境里最重要的细节之一。

如果不限制 Avahi，Debian 可能同时在：

```text
enp5s0 → Mac direct
wlp4s0 → 家庭 Wi‑Fi
```

上发布：

```text
devbox.local
```

这样 Mac 有可能获得多个地址，无法保证你想要的 SSH 一定走 Cat6。

因此建议 Avahi **只监听 Mac 直连网卡**。

编辑：

```bash
sudoedit /etc/avahi/avahi-daemon.conf
```

找到现有的：

```ini
[server]
```

在这个 section 中加入：

```ini
[server]
allow-interfaces=enp5s0
use-ipv4=yes
use-ipv6=no
```

注意：

- 如果文件已经存在 `[server]`，不要再创建第二个 `[server]`；
- `enp5s0` 必须换成你实际连接 Mac 的网卡名；
- 本文为了让路径最确定，直连链路只使用 IPv4；
- Avahi 默认会使用系统 hostname，所以不需要额外设置 `host-name=devbox`。

重启：

```bash
sudo systemctl restart avahi-daemon
```

确认：

```bash
systemctl --no-pager --full status avahi-daemon
```

Debian 官方 `avahi-daemon.conf` 手册说明：

```text
allow-interfaces=
```

可以指定 Avahi 允许使用的网络接口，其他接口会被忽略。

---

## 13. 在 Mac 上测试 `.local`

首先：

```bash
ping -c 4 devbox.local
```

理想输出中应该看到：

```text
PING devbox.local (10.10.10.2)
```

重点是确认解析结果：

```text
10.10.10.2
```

而不是 Debian Internet 网卡上的 `192.168.x.x` 地址。

如果看到：

```text
devbox.local → 10.10.10.2
```

说明 mDNS 路径已经正确。

---

## 14. 安装和配置 SSH Server

Debian：

```bash
sudo apt update
sudo apt install openssh-server
```

启动：

```bash
sudo systemctl enable --now ssh
```

检查：

```bash
systemctl status ssh
```

检查监听：

```bash
ss -lntp | grep ':22'
```

---

## 15. Mac 使用 `.local` SSH

假设 Debian 用户名：

```text
developer
```

在 Mac：

```bash
ssh developer@devbox.local
```

正常情况下：

```text
devbox.local
    ↓ mDNS
10.10.10.2
    ↓ routing
Mac Ethernet
    ↓ Cat6
Debian enp5s0
```

这意味着即使 Debian 的 Internet 网卡 IP 发生变化，也不会影响：

```bash
ssh developer@devbox.local
```

> 可选：如果也想从 Debian SSH 到 Mac（例如反向排错），需要在 Mac 上开启 系统设置 → 通用 → 共享 → 远程登录。直连网卡本身不影响这一功能。

---

## 16. 验证 SSH 确实走网线

### 16.1 先看 `.local` 解析地址

Mac：

```bash
ping -c 1 devbox.local
```

应该解析：

```text
10.10.10.2
```

---

### 16.2 查看 macOS 对该地址选择的接口

Mac：

```bash
route -n get 10.10.10.2
```

查看：

```text
interface:
```

例如：

```text
interface: en7
```

这个接口应该就是你的 Ethernet Adapter。

可以通过：

```bash
ifconfig en7
```

确认其地址是：

```text
10.10.10.1
```

---

### 16.3 Debian 反向检查

Debian：

```bash
ip route get 10.10.10.1
```

应该看到：

```text
10.10.10.1 dev enp5s0 src 10.10.10.2
```

到这里，可以确定：

```text
SSH = Cat6
```

而不是 Wi‑Fi。

---

## 17. 推荐配置 SSH Key

### 17.1 Mac 生成密钥

如果还没有：

```bash
ssh-keygen -t ed25519 -a 100
```

默认：

```text
~/.ssh/id_ed25519
~/.ssh/id_ed25519.pub
```

建议为私钥设置密码。

---

### 17.2 将公钥加入 Debian

macOS 不一定预装 `ssh-copy-id`，可以直接使用：

```bash
ssh developer@devbox.local \
  'umask 077; mkdir -p ~/.ssh; cat >> ~/.ssh/authorized_keys' \
  < ~/.ssh/id_ed25519.pub
```

然后：

```bash
ssh developer@devbox.local \
  'chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys'
```

测试：

```bash
ssh developer@devbox.local
```

---

## 18. 配置 Mac 的 `~/.ssh/config`

编辑：

```bash
nano ~/.ssh/config
```

加入：

```sshconfig
Host dev
    HostName devbox.local
    User developer
    IdentityFile ~/.ssh/id_ed25519

    ServerAliveInterval 30
    ServerAliveCountMax 3

    AddKeysToAgent yes
    UseKeychain yes
```

设置权限：

```bash
chmod 600 ~/.ssh/config
```

以后：

```bash
ssh dev
```

即可。

IDE 也可以直接使用：

```text
Host: dev
```

例如：

- VS Code Remote SSH；
- Cursor Remote SSH；
- JetBrains Gateway；
- rsync；
- scp；
- SFTP。

---

## 19. 可选：关闭 SSH 密码登录

**必须确认 SSH Key 登录成功以后再做。**

Debian：

```bash
sudoedit /etc/ssh/sshd_config.d/99-devbox.conf
```

写入：

```text
PubkeyAuthentication yes
PasswordAuthentication no
PermitRootLogin no
```

检查配置：

```bash
sudo sshd -t
```

如果没有输出，表示语法正确。

重新加载：

```bash
sudo systemctl reload ssh
```

**修改期间不要关闭当前 SSH Terminal。**

新开一个 Mac Terminal：

```bash
ssh dev
```

确认还能登录后，再关闭旧连接。

---

## 20. 可选：只允许 Mac 直连地址 SSH

如果你希望 SSH 只面向：

```text
10.10.10.2
```

而不监听 Debian 的 Internet/Wi‑Fi 网卡，可以配置：

```bash
sudoedit /etc/ssh/sshd_config.d/20-listen-direct.conf
```

加入：

```text
ListenAddress 10.10.10.2
```

测试：

```bash
sudo sshd -t
```

然后：

```bash
sudo systemctl restart ssh
```

优点：

- Wi‑Fi/LAN 上看不到 SSH 服务；
- SSH 只允许从 Mac 专用网线进入。

缺点：

- 如果直连网卡配置出错，你无法通过第二张网卡 SSH 救援。

对于无显示器服务器，建议等整个环境运行稳定后再启用这一项。

---

## 21. 防火墙建议

如果 Debian 没有启用入站阻断防火墙，局部直连网络本身已经相对简单。

如果你使用 UFW，并且默认拒绝入站，可以只开放 Mac：

```bash
sudo ufw allow in on enp5s0 \
  from 10.10.10.1 \
  to any port 22 proto tcp
```

如果 Avahi 被防火墙阻断，还需要允许该直连接口上的 IPv4 mDNS：

```bash
sudo ufw allow in on enp5s0 \
  from 10.10.10.1 \
  to 224.0.0.251 port 5353 proto udp
```

检查：

```bash
sudo ufw status verbose
```

**在远程无显示器机器上修改防火墙时，一定先保留现有 SSH 会话，再从第二个 Terminal 测试新连接。**

---

## 22. 后端服务如何通过 `.local` 访问

例如 Debian 上的后端监听：

```text
10.10.10.2:3000
```

Mac 可以：

```text
http://devbox.local:3000
```

例如：

```bash
curl http://devbox.local:3000
```

---

### 22.1 后端监听地址

如果服务只需要给 Mac 使用，推荐绑定：

```text
10.10.10.2
```

比：

```text
0.0.0.0
```

更严格。

例如概念上：

```text
Backend:
listen = 10.10.10.2:3000
```

这样服务不会自动暴露在 Debian 的 Wi‑Fi / 家庭 LAN 接口上。

---

## 23. Docker 推荐绑定方式

如果 Docker Container 内部监听：

```text
0.0.0.0:3000
```

宿主机发布端口时可以只绑定直连地址。

`compose.yaml`：

```yaml
services:
  backend:
    image: your-backend
    ports:
      - "10.10.10.2:3000:3000"
```

Mac：

```bash
curl http://devbox.local:3000
```

这样：

```text
Mac → devbox.local → 10.10.10.2:3000
```

可以访问。

而 Debian 第二张网卡的：

```text
192.168.x.x:3000
```

不会因为这个 Docker port mapping 自动暴露服务。

---

## 24. iOS Simulator / macOS App 开发

如果 Xcode 和 iOS Simulator 在 Mac 上：

```text
Xcode App / Simulator
        │
        ▼
http://devbox.local:3000
        │
        ▼
Mac Ethernet
        │
       Cat6
        │
        ▼
Debian Backend
```

这是非常适合本方案的使用方式。

### 注意：真机 iPhone

真实 iPhone 如果只连接家庭 Wi‑Fi，它**并不在 Mac ↔ Debian 的 10.10.10.0/24 直连网络上**。

因此：

```text
iOS Simulator → 可以直接访问 devbox.local
真实 iPhone   → 不一定可以
```

如果需要真实 iPhone 访问后端，可选择：

1. 让 Debian 后端同时在家庭 LAN 上提供服务；
2. 由 Mac 做路由/NAT；
3. 使用开发隧道；
4. 单独设计开发 VLAN / LAN。

这与 Mac 本地 Simulator 的情况不同。

---

## 25. Debian 无显示器 Headless 使用

配置完成以后，Debian 台式机可以不连接显示器。

日常控制：

```bash
ssh dev
```

推荐在 Debian 上安装：

```bash
sudo apt install tmux
```

开始 Session：

```bash
tmux new -s dev
```

断线后重新连接：

```bash
tmux attach -t dev
```

这样 SSH 客户端异常退出也不会终止：

- 编译任务；
- 后端进程；
- shell 工作；
- 数据处理任务。

对于长期运行的服务，更推荐使用：

- systemd；
- Docker Compose；
- Kubernetes；
- supervisor 等正式服务管理方式。

> **电源与唤醒：** Mac 进入睡眠会断开直连链路，长时间开发可在终端用 `caffeinate -i` 保持唤醒。Debian 无显示器常开时，建议在 BIOS 里关闭网卡节能（如 PCIe ASPM），需要远程开机时开启主板 Wake-on-LAN。

---

## 26. 可选：Debian 图形桌面远程访问

如果 Debian 安装了桌面环境，也可以通过 RDP 使用。

安装：

```bash
sudo apt install xrdp xorgxrdp
```

启用：

```bash
sudo systemctl enable --now xrdp
```

Mac 远程桌面客户端填写：

```text
devbox.local
```

RDP 默认端口：

```text
3389
```

需要注意：

> xrdp 通常是远程图形登录 Session，不一定等同于“镜像台式机物理显示器当前 Session”。

如果只是后端开发，SSH + tmux 通常更加稳定、轻量。

---

## 27. 检查物理链路速度

Debian 安装：

```bash
sudo apt install ethtool
```

检查：

```bash
sudo ethtool enp5s0
```

重点：

```text
Speed: 1000Mb/s
Duplex: Full
Link detected: yes
```

如果使用 2.5GbE，则可能是：

```text
Speed: 2500Mb/s
```

如果发现：

```text
Speed: 100Mb/s
```

通常需要检查：

- 网线；
- RJ45 水晶头；
- USB Ethernet Adapter；
- 网卡协商；
- 驱动。

另外，链路速度由两端协商决定：如果一边是 1GbE、另一边是 2.5GbE，最终会协商到 1GbE。想跑满 2.5GbE，需要 Mac 的 USB 网卡、Debian 的网卡以及网线都支持 2.5GbE。

---

## 28. 一次性健康检查

### Debian

```bash
echo '=== interfaces ==='
ip -br addr

echo
echo '=== routes ==='
ip route

echo
echo '=== route to Mac ==='
ip route get 10.10.10.1

echo
echo '=== route to Internet ==='
ip route get 1.1.1.1

echo
echo '=== SSH ==='
systemctl is-active ssh

echo
echo '=== Avahi ==='
systemctl is-active avahi-daemon
```

理想结果：

```text
10.10.10.2     → enp5s0
default route  → wlp4s0 / Internet NIC
ssh            → active
avahi-daemon   → active
```

---

### Mac

```bash
ping -c 1 10.10.10.2
ping -c 1 devbox.local
route -n get 10.10.10.2
ssh dev
```

重点确认：

```text
devbox.local → 10.10.10.2
route interface → Ethernet
```

---

## 29. 重启测试

配置好以后一定执行一次完整重启测试。

先重启 Debian：

```bash
sudo reboot
```

等待机器启动后，在 Mac：

```bash
ping devbox.local
```

然后：

```bash
ssh dev
```

进入后检查：

```bash
ip route
```

确保：

```text
default        → Debian Internet NIC
10.10.10.0/24 → Debian Mac Direct NIC
```

再测试：

```bash
curl -I https://deb.debian.org/
```

确认 Debian 自己可以访问 Internet。

---

## 30. 常见故障排查

### 30.1 `ping 10.10.10.2` 不通

按顺序检查：

### Mac

```bash
ifconfig
```

确认 Ethernet：

```text
10.10.10.1
```

### Debian

```bash
ip -4 addr
```

确认 direct NIC：

```text
10.10.10.2/24
```

检查连接：

```bash
nmcli device status
```

检查链路：

```bash
sudo ethtool enp5s0
```

应该：

```text
Link detected: yes
```

---

### 30.2 IP 可以 ping，但 `devbox.local` 不通

Debian：

```bash
systemctl status avahi-daemon
```

检查：

```bash
hostname
```

应该：

```text
devbox
```

检查：

```bash
grep -E 'allow-interfaces|use-ipv4|use-ipv6' \
  /etc/avahi/avahi-daemon.conf
```

确认接口名没有写错。

重启：

```bash
sudo systemctl restart avahi-daemon
```

Mac 再测试：

```bash
ping -c 1 devbox.local
```

---

### 30.3 `devbox.local` 解析到了 Wi‑Fi IP

例如：

```text
devbox.local → 192.168.1.55
```

而不是：

```text
10.10.10.2
```

说明 Avahi 很可能同时在多张网卡发布。

检查：

```bash
sudoedit /etc/avahi/avahi-daemon.conf
```

确保：

```ini
[server]
allow-interfaces=enp5s0
```

然后：

```bash
sudo systemctl restart avahi-daemon
```

再次测试：

```bash
ping -c 1 devbox.local
```

---

## 31. Debian 插入 Mac 网线后不能上 Internet

这是典型的默认路由错误。

检查：

```bash
ip route
```

如果看到：

```text
default ... dev enp5s0
```

说明 Mac 直连网卡错误获得了默认路由。

修复：

```bash
sudo nmcli connection modify mac-direct ipv4.never-default yes
sudo nmcli connection down mac-direct
sudo nmcli connection up mac-direct
```

再检查：

```bash
ip route
```

应该只有第二张网卡提供：

```text
default
```

---

## 32. Mac 插入网线后 Internet 出问题

检查 Mac Ethernet 配置。

Mac 的直连 Ethernet 应该：

```text
IP:       10.10.10.1
Mask:     255.255.255.0
Router:   空
DNS:      空
```

如果给 Ethernet 设置了 Router，它可能干扰 Mac 原本的 Internet 路由。

本方案不需要：

- Internet Sharing；
- NAT；
- DHCP Server；
- Bridge；
- 路由转发。

---

## 33. SSH 提示 Host Key Changed

如果重装过 Debian，SSH Host Key 会变化。

Mac 可能提示：

```text
WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!
```

**只有在你确认 Debian 确实刚刚重装或重新生成过 Host Key 时**，才删除旧记录：

```bash
ssh-keygen -R devbox.local
```

以及：

```bash
ssh-keygen -R 10.10.10.2
```

然后重新连接。

如果机器没有重装却突然出现 Host Key 改变，应先排查，不要直接忽略安全警告。

---

## 34. SSH 连接仍然偶尔掉线

先确认网络本身：

```bash
ping devbox.local
```

Debian 查看 NetworkManager：

```bash
journalctl -u NetworkManager --since "30 min ago"
```

查看 SSH：

```bash
journalctl -u ssh --since "30 min ago"
```

查看物理 link：

```bash
sudo ethtool enp5s0
```

如果出现 link up / link down，优先怀疑：

- 网线；
- USB Ethernet Adapter；
- USB Hub；
- 驱动；
- 网卡节能。

`ServerAliveInterval` 可以帮助 SSH 更快发现失效连接，但无法修复实际物理断链。

---

## 35. 如果 Debian 不使用 NetworkManager

不要让多个网络管理器同时管理同一张网卡。

检查：

```bash
systemctl is-active NetworkManager
```

如果 Debian 使用传统 `ifupdown`，可以在：

```text
/etc/network/interfaces.d/mac-direct
```

配置：

```text
auto enp5s0

iface enp5s0 inet static
    address 10.10.10.2/24
```

**不要添加 `gateway`。**

Internet 接口继续通过 DHCP。

配置完成后应确保该接口没有同时被 NetworkManager 或 systemd-networkd 接管。

---

## 36. 推荐的最终日常使用方式

Mac 开发：

```text
Xcode
iOS Simulator
Terminal
Browser
```

Debian：

```text
Docker
PostgreSQL
Redis
Node / Go / Java / Rust / Python
Backend API
编译任务
```

Mac 日常只记：

```bash
ssh dev
```

后端：

```text
http://devbox.local:3000
```

数据库例如：

```text
devbox.local:5432
```

其他服务：

```text
devbox.local:<port>
```

底层始终：

```text
devbox.local
      │
      ▼
10.10.10.2
      │
      ▼
Cat6 direct link
```

---

## 37. 最终配置模板

### macOS

```text
Internet:
Wi‑Fi → DHCP → Router

Mac Direct Ethernet:
IP      = 10.10.10.1
Mask    = 255.255.255.0
Gateway = none
DNS     = none
```

### Debian

```text
Hostname:
devbox

Direct NIC:
Interface = enp5s0
IP        = 10.10.10.2/24
Gateway   = none
DNS       = none
Default   = never

Internet NIC:
Interface = wlp4s0 / enp6s0
IP        = DHCP
Gateway   = DHCP
DNS       = DHCP
Default   = yes
```

### Avahi

```ini
[server]
allow-interfaces=enp5s0
use-ipv4=yes
use-ipv6=no
```

### Mac SSH

```sshconfig
Host dev
    HostName devbox.local
    User developer
    IdentityFile ~/.ssh/id_ed25519
    ServerAliveInterval 30
    ServerAliveCountMax 3
    AddKeysToAgent yes
    UseKeychain yes
```

使用：

```bash
ssh dev
```

---

## 38. 配置完成后的预期行为

| 操作 | 路径 |
|---|---|
| `ssh dev` | Mac Ethernet → Cat6 → Debian |
| `ping devbox.local` | Mac Ethernet → Debian |
| `curl http://devbox.local:3000` | Mac Ethernet → Debian |
| Mac 浏览网页 | Mac Wi‑Fi → Router → Internet |
| Debian `apt update` | Debian NIC 2 → Router → Internet |
| Debian `docker pull` | Debian NIC 2 → Router → Internet |
| Debian Internet NIC IP 改变 | **不影响 Mac SSH** |
| 家庭 Wi‑Fi 波动 | **不影响 Mac ↔ Debian 有线 SSH** |
| 路由器 DHCP 更换 Debian LAN IP | **不影响 `devbox.local` 的直连路径** |

---

## 39. 核心结论

这套方案可以理解为建立两张完全不同职责的网络：

```text
开发控制平面：
Mac ← Cat6 → Debian
10.10.10.1   10.10.10.2
      devbox.local

Internet 平面：
Mac    → 自己的 Wi‑Fi → Internet
Debian → 自己的 NIC 2 → Internet
```

最关键的四项配置是：

```text
1. Mac direct Ethernet = 10.10.10.1/24，无 Gateway
2. Debian direct NIC    = 10.10.10.2/24，无 Gateway
3. Debian Internet NIC  = 唯一默认路由
4. Avahi 只发布 direct NIC
```

最终日常使用不需要记 IP：

```bash
ssh dev
```

或者：

```bash
ssh developer@devbox.local
```

这是一个稳定、简单、低延迟，并且非常适合 Mac App 开发 + Debian 后端开发的双机架构。

---

## 40. 官方参考资料

- Apple Support — Use DHCP or a manual IP address on Mac  
  https://support.apple.com/guide/mac-help/use-dhcp-or-a-manual-ip-address-on-mac-mchlp2718/mac

- Apple Support — Find your computer's name and local hostname / Bonjour `.local`  
  https://support.apple.com/guide/mac-help/find-your-computers-name-and-network-address-mchlp1177/mac

- NetworkManager Reference Manual — `ipv4.never-default`  
  https://www.networkmanager.dev/docs/api/latest/nm-settings-nmcli.html

- Debian Manpages — `avahi-daemon.conf(5)`  
  https://manpages.debian.org/trixie/avahi-daemon/avahi-daemon.conf.5.en.html

- Debian Packages — `avahi-daemon`  
  https://packages.debian.org/trixie/avahi-daemon

- Debian Packages — `openssh-server`  
  https://packages.debian.org/stable/openssh-server

---

## 附：最短部署清单

如果已经理解所有原理，实际部署可以简化成以下流程。

### Debian

```bash
# 查看接口
nmcli device status

# 示例：enp5s0 专门连接 Mac
sudo nmcli connection add \
  type ethernet \
  ifname enp5s0 \
  con-name mac-direct \
  ipv4.method manual \
  ipv4.addresses 10.10.10.2/24 \
  ipv4.never-default yes \
  ipv6.method disabled

sudo nmcli connection modify mac-direct connection.autoconnect yes
sudo nmcli connection up mac-direct

# SSH + mDNS
sudo apt update
sudo apt install openssh-server avahi-daemon avahi-utils

sudo hostnamectl set-hostname devbox

sudo systemctl enable --now ssh
sudo systemctl enable --now avahi-daemon
```

编辑：

```bash
sudoedit /etc/avahi/avahi-daemon.conf
```

在已有 `[server]` 中：

```ini
allow-interfaces=enp5s0
use-ipv4=yes
use-ipv6=no
```

然后：

```bash
sudo systemctl restart avahi-daemon
```

### macOS

Ethernet：

```text
IP       10.10.10.1
Mask     255.255.255.0
Router   empty
DNS      empty
```

测试：

```bash
ping -c 1 10.10.10.2
ping -c 1 devbox.local
ssh developer@devbox.local
```

最后配置：

```text
~/.ssh/config
```

```sshconfig
Host dev
    HostName devbox.local
    User developer
    ServerAliveInterval 30
    ServerAliveCountMax 3
```

之后日常只需：

```bash
ssh dev
```
