---
layout: post
title: "社交 App 生产部署实施手册：多区域架构、数据库容灾与故障切换"
date: 2026-09-18 00:00:00 +0800
categories: [Development]
tags: [vps, PostgreSQL, Cloudflare, WireGuard, Docker, Caddy, 高可用, 容灾, 生产部署, failover]
---

**适用场景**：你已经在 Frankfurt（法兰克福）、San Jose（圣何塞）、Singapore（新加坡）各有 1 台 VPS，希望正式部署一个社交 App，让用户就近连接，同时具备基本的高可用、数据安全、故障切换、备份恢复和可运维能力。  
**示例技术栈**：Ubuntu LTS + Cloudflare + Caddy + Docker Compose + PostgreSQL 18 + WireGuard + Valkey/Redis-compatible cache；NATS 为可选实时消息总线。  
**重要原则**：三地应用 Active-Active，数据库先坚持 Single-Primary；不要在只有三台跨洲 VPS 的条件下做跨洲 Kubernetes/etcd 或三地数据库多主。  
**执行提示**：本文包含可复制命令，但所有 `REPLACE_ME`、`<...>`、域名、IP、密钥、版本号和容量参数都必须先替换并在 Staging 验证；不要把示例参数未经评估直接用于生产。

---

## 1. 目标与非目标

### 1.1 目标

本方案希望实现：

- 欧洲用户优先访问 Frankfurt。
- 北美用户优先访问 San Jose。
- 亚洲用户优先访问 Singapore。
- 任意一台应用 VPS 宕机后，用户能够被自动切到其它健康区域。
- 三个区域运行同一份应用镜像。
- HTTP API 与 WebSocket 都可以工作。
- 用户登录状态不依赖某台 VPS 的本地内存。
- 头像、图片、视频不依赖 VPS 本地磁盘。
- 数据库有跨区域副本。
- 数据库具有离线备份、连续 WAL 归档、Point-in-Time Recovery 能力。
- 任何一次发布都可以明确回滚。
- 发生 VPS 故障、数据库故障、网络分区时有明确操作手册。

### 1.2 当前阶段明确不做

三台跨洲 VPS 阶段**不要**直接做：

- 一个横跨 Frankfurt / San Jose / Singapore 的 Kubernetes 或 k3s control plane。
- 跨洲 etcd / Consul quorum。
- PostgreSQL 三地主主复制。
- MySQL 三地多主。
- Redis 三地强一致集群。
- 3 个跨洲节点组成需要共识的 JetStream / Kafka 持久化集群。
- 把用户上传文件写到 VPS 本地目录并用 rsync 同步。
- 所有 GET 请求无脑读本地数据库副本。
- 仅因为有 replica 就认为“已经有备份”。

原因很简单：**应用层无状态可以天然横向扩展；强一致状态层则会把跨洲 RTT 直接带进共识与事务路径。**

---

## 2. 最终推荐架构

```text
                              Internet Users
                                     │
                                     ▼
                      ┌────────────────────────┐
                      │       Cloudflare       │
                      │ DNS / TLS / WAF / LB   │
                      │ Geo or Dynamic Steering│
                      └───────────┬────────────┘
                                  │
               ┌──────────────────┼──────────────────┐
               │                  │                  │
               ▼                  ▼                  ▼
       Frankfurt / FRA      San Jose / SJC     Singapore / SIN
       Public VPS #1        Public VPS #2       Public VPS #3
               │                  │                  │
             Caddy              Caddy              Caddy
               │                  │                  │
         ┌─────┴─────┐      ┌─────┴─────┐      ┌─────┴─────┐
         │ App A/B   │      │ App A/B   │      │ App A/B   │
         └─────┬─────┘      └─────┬─────┘      └─────┬─────┘
               │                  │                  │
         Local Valkey       Local Valkey       Local Valkey
               │                  │                  │
               └──────────── WireGuard Mesh ─────────┘
                                  │
                         ┌────────┴────────┐
                         │ PostgreSQL     │
                         │ Single Primary │
                         └────────┬────────┘
                                  │ async WAL
                         ┌────────┴────────┐
                         ▼                 ▼
                    Replica #1        Replica #2

        App ───────────────► Object Storage ─────────────► CDN
                                  │
                             images/video

        Optional:
        App/WebSocket ─────► Local NATS ── Gateway ──► Remote NATS
                               │
                        real-time notification only
                        DB remains source of truth
```

### 2.1 本文示例中的数据库 Primary

为了让命令可直接照着做，后文示例暂定：

```text
Singapore = PostgreSQL Primary
Frankfurt = PostgreSQL Replica
San Jose  = PostgreSQL Replica
```

对应：

```text
SIN WireGuard IP = 10.77.0.13
```

**你上线前必须根据真实用户分布重新确认 Primary 放在哪里。**

原则：

- 亚洲用户占绝大多数：Singapore。
- 欧洲用户占绝大多数：Frankfurt。
- 北美用户占绝大多数：San Jose。
- 有 EU 数据驻留要求：优先评估 Frankfurt，并确认跨境数据合规要求。
- 用户非常平均：第一阶段仍保持一个 Primary；不要仅为了“看起来全球化”做三主。

---

## 3. 关键设计决策

### 3.1 应用层：Active-Active

三地都运行应用。

任何请求理论上都可以由任意 Region 接收。

因此应用必须尽量：

```text
stateless
```

以下内容不能只存在某台 VPS 内存：

- 登录 Session。
- Refresh Token。
- 用户上传文件索引。
- 重要业务任务。
- WebSocket 消息历史。
- 支付状态。
- 密码重置状态。
- 关键限流状态。

### 3.2 数据库：Single-Primary + Async Replica

只允许一个节点接受普通写事务：

```text
INSERT
UPDATE
DELETE
DDL
```

另外两个作为物理异步副本。

好处：

- 架构清晰。
- 不存在三主冲突解决。
- PostgreSQL 原生能力成熟。
- 故障恢复流程可理解、可验证。
- 用户量不大时总体维护成本最低。

代价：

- 远端 Region 的写请求必须跨洲到 Primary。
- 异步副本存在 replication lag。
- Primary 故障需要提升副本。
- 在仅有三台服务器时，数据库自动 failover 很难做到既安全又简单。

### 3.3 RPO / RTO 目标

建议第一阶段定义：

| 故障 | 建议目标 |
|---|---|
| 单个 App 容器故障 | 本机几秒级恢复/切换 |
| 单个 Region VPS 故障 | LB 健康检查发现后切换 |
| PostgreSQL Primary 故障 | 人工 Failover，目标 RTO 10~30 分钟 |
| PostgreSQL Primary 数据损坏 | Replica 或 PITR |
| 整个数据库集群逻辑误删 | PITR |
| 单 Region 永久丢失 | 其它 Region + 对象存储 + DB 备份重建 |

异步复制下：

```text
RPO ≠ 绝对 0
```

RPO 取决于：

- WAL 是否已发送。
- Standby 是否已接收。
- Standby 是否已 flush。
- 网络是否中断。
- WAL archive 是否成功。

如果业务未来要求：

```text
RPO = 0
RTO < 1 分钟
```

不要继续硬堆这三台跨洲 VPS。

应升级为：

- Primary Region 内至少 3 个数据库/仲裁节点。
- Patroni / managed PostgreSQL / 云数据库 HA。
- Region 间继续异步 DR。

---

## 4. 命名、地址与端口规划

### 4.1 Region 编码

```text
Frankfurt = fra
San Jose  = sjc
Singapore = sin
```

### 4.2 WireGuard 网段

本文使用：

```text
10.77.0.0/24
```

分配：

| Region | WG IP |
|---|---:|
| FRA | 10.77.0.11 |
| SJC | 10.77.0.12 |
| SIN | 10.77.0.13 |

不要与：

- Docker 网段。
- VPS Provider 私网。
- 办公网。
- 家庭网。
- VPN 网段。

发生冲突。

### 4.3 域名

建议：

```text
api.example.com       API + 可选 WebSocket
ws.example.com        可选：独立 WebSocket 域名
media.example.com     图片 / 视频 CDN
status.example.com    状态页
```

内部服务不应该开放公网 DNS 端口。

### 4.4 端口

建议：

| 端口 | 用途 | 公网开放 |
|---|---|---|
| 22/TCP | SSH | 仅管理 IP / VPN |
| 443/TCP | Caddy Origin HTTPS | 仅 Cloudflare |
| 51820/UDP | WireGuard | 对其它 peer 公网 IP |
| 5432/TCP | PostgreSQL | **仅 WireGuard** |
| 6379/TCP | Valkey | **不开放** |
| 4222/TCP | NATS Client | **不开放** |
| 7222/TCP | NATS Gateway | **仅 WireGuard** |
| 8222/TCP | NATS Monitor | **仅本机/监控网** |
| 18081/TCP | App A | 127.0.0.1 |
| 18082/TCP | App B | 127.0.0.1 |

---

## 5. 上线前准备

准备以下信息：

```text
FRA_PUBLIC_IP=
SJC_PUBLIC_IP=
SIN_PUBLIC_IP=

DOMAIN=example.com

GHCR_OR_OTHER_REGISTRY=
APP_IMAGE=

DB_PRIMARY_REGION=sin
DB_NAME=social
DB_APP_USER=app_user
DB_REPLICATION_USER=replicator

OBJECT_STORAGE_BUCKET=
OBJECT_STORAGE_ENDPOINT=
OBJECT_STORAGE_REGION=

CLOUDFLARE_ZONE_ID=
```

建议建立密码管理器条目：

- VPS root/bootstrap 凭证。
- deploy SSH key。
- PostgreSQL app_user。
- PostgreSQL replicator。
- backup repository credential。
- object storage credential。
- Cloudflare scoped API token。
- registry deploy token。
- JWT signing key。
- refresh-token encryption/pepper。
- NATS credential（如果使用）。

任何 Secret 都不要写入 Git。

---

## 6. 三台 VPS 基础初始化与加固

以下步骤三台机器都执行。

示例系统：

```text
Ubuntu 24.04 LTS / Ubuntu 26.04 LTS
```

生产环境尽量统一 OS 版本。

### 6.1 更新系统

```bash
sudo apt update
sudo apt full-upgrade -y
sudo apt install -y \
  ca-certificates curl jq git vim htop unzip \
  ufw fail2ban chrony wireguard
```

统一 UTC：

```bash
sudo timedatectl set-timezone UTC
timedatectl
```

时间同步：

```bash
sudo systemctl enable --now chrony
chronyc tracking
```

数据库、JWT、日志、TLS 都依赖正确时钟。

---

### 6.2 创建 deploy 用户

```bash
sudo adduser deploy
sudo usermod -aG sudo deploy
```

从本地电脑安装 SSH 公钥：

```bash
ssh-copy-id deploy@VPS_PUBLIC_IP
```

打开一个**新的终端**测试：

```bash
ssh deploy@VPS_PUBLIC_IP
```

确认正常后再关闭原 root 会话。

---

### 6.3 SSH 加固

创建：

```bash
sudo nano /etc/ssh/sshd_config.d/99-hardening.conf
```

内容：

```text
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
AllowUsers deploy
```

验证：

```bash
sudo sshd -t
```

重载：

```bash
sudo systemctl reload ssh
```

> 不要在没有验证 deploy key 可登录之前禁掉 root/password。

---


### 6.4 防火墙原则

最终状态应尽量收敛为：

```text
Internet
  ├─ Cloudflare egress / monitor ─► 443/TCP
  ├─ 两个 WireGuard Peer 公网 IP ─► 51820/UDP
  └─ Admin 固定 IP / 管理 VPN ───► 22/TCP

WireGuard wg0
  ├─ 三个数据库节点之间 ─────────► 5432/TCP
  ├─ 可选 NATS Gateway ──────────► 7222/TCP
  └─ 其它内部端口按需精确开放
```

绝对不要：

```text
0.0.0.0/0 → 5432
0.0.0.0/0 → 6379
0.0.0.0/0 → 4222
```

初始基线：

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing

sudo ufw allow from YOUR_ADMIN_PUBLIC_IP/32 to any port 22 proto tcp

# Bootstrap 阶段暂时允许 WireGuard；
# 第 7.5 节确认三地互通后再收紧到两个 Peer 的公网 IP。
sudo ufw allow 51820/udp

# Bootstrap 测试 Caddy 时临时允许管理 IP。
sudo ufw allow from YOUR_ADMIN_PUBLIC_IP/32 to any port 443 proto tcp

sudo ufw enable
sudo ufw status verbose
```

#### WireGuard 内层流量也必须显式考虑

`ufw default deny incoming` 不只会影响公网接口，也会影响到达 `wg0` 的入站连接。

因此三台 PostgreSQL 节点都要允许来自 WireGuard 网段的 5432：

```bash
sudo ufw allow in on wg0 from 10.77.0.0/24 to any port 5432 proto tcp
```

如果启用 NATS Gateway：

```bash
sudo ufw allow in on wg0 from 10.77.0.0/24 to any port 7222 proto tcp
```

不要为了方便直接：

```bash
sudo ufw allow in on wg0
```

除非你明确接受“一台 VPS 被攻陷后可以访问另外两台所有监听在 wg0 上的服务”的横向移动风险。

#### Docker + UFW 特别警告

Docker 官方文档明确指出：Linux 上发布容器端口时，Docker 的 NAT/iptables 规则可能让流量在 UFW 常规 INPUT/OUTPUT 规则之前被处理，因此**不要以为 UFW deny 就一定保护了 `docker -p` 暴露的端口**。

本文因此采用：

```text
Caddy       → host 上监听 443
App A/B     → 仅 127.0.0.1:18081/18082
Valkey      → 不 publish host port
NATS Client → 不 publish 公网
PostgreSQL  → host 上只监听 127.0.0.1 + wg0 IP
```

如果未来必须把容器端口发布到公网，必须单独设计 Docker `DOCKER-USER`/firewall 规则并验证，不能只依赖 UFW。

### 6.5 Fail2ban

如果 SSH 仍可从公网管理网段访问：

```bash
sudo systemctl enable --now fail2ban
sudo fail2ban-client status
```

如果以后 SSH 完全只走专用 VPN/WireGuard，可以降低 Fail2ban 的重要性。

---

## 7. 建立三地 WireGuard 私网

### 7.1 生成 key

每台 VPS：

```bash
sudo -i

umask 077
wg genkey | tee /etc/wireguard/privatekey | wg pubkey > /etc/wireguard/publickey

cat /etc/wireguard/publickey
```

记录三台 public key：

```text
FRA_WG_PUBLIC_KEY=
SJC_WG_PUBLIC_KEY=
SIN_WG_PUBLIC_KEY=
```

Private Key 绝对不要离开对应服务器。

---

### 7.2 FRA 配置

`/etc/wireguard/wg0.conf`

```ini
[Interface]
Address = 10.77.0.11/24
ListenPort = 51820
PrivateKey = <FRA_PRIVATE_KEY>

[Peer]
PublicKey = <SJC_PUBLIC_KEY>
AllowedIPs = 10.77.0.12/32
Endpoint = <SJC_PUBLIC_IP>:51820

[Peer]
PublicKey = <SIN_PUBLIC_KEY>
AllowedIPs = 10.77.0.13/32
Endpoint = <SIN_PUBLIC_IP>:51820
```

### 7.3 SJC 配置

```ini
[Interface]
Address = 10.77.0.12/24
ListenPort = 51820
PrivateKey = <SJC_PRIVATE_KEY>

[Peer]
PublicKey = <FRA_PUBLIC_KEY>
AllowedIPs = 10.77.0.11/32
Endpoint = <FRA_PUBLIC_IP>:51820

[Peer]
PublicKey = <SIN_PUBLIC_KEY>
AllowedIPs = 10.77.0.13/32
Endpoint = <SIN_PUBLIC_IP>:51820
```

### 7.4 SIN 配置

```ini
[Interface]
Address = 10.77.0.13/24
ListenPort = 51820
PrivateKey = <SIN_PRIVATE_KEY>

[Peer]
PublicKey = <FRA_PUBLIC_KEY>
AllowedIPs = 10.77.0.11/32
Endpoint = <FRA_PUBLIC_IP>:51820

[Peer]
PublicKey = <SJC_PUBLIC_KEY>
AllowedIPs = 10.77.0.12/32
Endpoint = <SJC_PUBLIC_IP>:51820
```

权限：

```bash
sudo chmod 600 /etc/wireguard/wg0.conf
```

启动：

```bash
sudo systemctl enable --now wg-quick@wg0
```

检查：

```bash
sudo wg show
ip addr show wg0
```

从 FRA：

```bash
ping -c 3 10.77.0.12
ping -c 3 10.77.0.13
```

三地互相测试。

如果某台 VPS 在 NAT 后面，再考虑：

```ini
PersistentKeepalive = 25
```

普通拥有公网 IP 的 VPS 通常不必强制配置。

---


### 7.5 WireGuard 防火墙收紧

确认三台已经互相握手：

```bash
sudo wg show
```

至少检查：

```text
latest handshake
transfer
endpoint
```

之后将公网 `51820/udp` 从“任意来源”收紧为仅两个 Peer 的公网 IP。

例如 FRA：

```bash
sudo ufw delete allow 51820/udp

sudo ufw allow from <SJC_PUBLIC_IP> to any port 51820 proto udp
sudo ufw allow from <SIN_PUBLIC_IP> to any port 51820 proto udp
```

SJC、SIN 以同样方式只允许其它两个 Peer。

三台都允许 PostgreSQL 的 WireGuard 内层连接：

```bash
sudo ufw allow in on wg0 from 10.77.0.0/24 to any port 5432 proto tcp
```

启用 NATS Gateway 时再加：

```bash
sudo ufw allow in on wg0 from 10.77.0.0/24 to any port 7222 proto tcp
```

检查：

```bash
sudo ufw status numbered
sudo wg show
```

#### 关于 Ping

如果你发现：

```bash
ping 10.77.0.13
```

被主机防火墙挡住，不代表 WireGuard 一定断了。

更可靠的服务级测试：

```bash
wg show
nc -vz 10.77.0.13 5432
```

数据库部署完成后再用 `psql` 做端到端验证。

#### 关于 `PersistentKeepalive`

拥有稳定公网 IP、双方都能直接互访的普通 VPS 一般不需要。

只有 NAT / stateful firewall 使连接长时间空闲后失效时，才在位于 NAT 后面的 Peer 配置：

```ini
PersistentKeepalive = 25
```


## 8. 部署应用运行环境

### 8.1 安装 Docker

推荐使用 Docker 官方 APT repository，并安装：

```text
docker-ce
docker-ce-cli
containerd.io
docker-buildx-plugin
docker-compose-plugin
```

安装后：

```bash
sudo systemctl enable --now docker
sudo docker version
sudo docker compose version
```

让 deploy 可以操作 Docker：

```bash
sudo usermod -aG docker deploy
```

重新登录后：

```bash
docker ps
```

> `docker` group 等价于较高系统权限。只给可信运维账号。

---

### 8.2 生产目录

```bash
sudo mkdir -p /opt/socialapp
sudo chown -R deploy:deploy /opt/socialapp
cd /opt/socialapp
```

建议：

```text
/opt/socialapp/
├── compose.yml
├── .env.deploy
├── .env.app
├── nats/
│   └── nats.conf
└── scripts/
    ├── deploy.sh
    └── verify.sh
```

权限：

```bash
chmod 600 .env.deploy .env.app
```

---

### 8.3 App 必须提供的健康接口

至少实现：

```text
GET /livez
GET /health/edge
GET /health/deps
GET /version
```

#### `/livez`

只回答：

> 进程是否活着？

不要访问：

- PostgreSQL。
- Redis。
- Object Storage。
- NATS。
- 外部 API。

正常：

```json
{"status":"ok"}
```

HTTP 200。

#### `/health/edge`

给 Cloudflare LB 使用。

它应判断：

- App event loop / worker 正常。
- 本 Region 能否访问**核心数据库**。
- 数据库 ping 应来自已存在连接池，或使用非常短 timeout。
- 不要串行检查十几个外部依赖。
- NATS/邮件/第三方支付等非核心依赖失败，不应该把整个 Region 判死。

推荐缓存依赖健康状态几秒，避免健康检查本身制造数据库压力。

#### `/health/deps`

仅内部/鉴权访问。

可报告：

```text
db
valkey
nats
object-storage
outbox
```

不要返回密码、连接字符串或内部 Secret。

#### `/version`

返回：

```json
{
  "version": "2026.09.14",
  "git_sha": "abcdef1",
  "region": "fra"
}
```

方便定位版本漂移。

---

### 8.4 Docker Compose 示例

`compose.yml`

```yaml
services:
  app_a:
    image: ${APP_IMAGE}
    restart: unless-stopped
    env_file:
      - .env.app
    environment:
      APP_INSTANCE: app_a
    ports:
      - "127.0.0.1:18081:8080"
    networks:
      - app_net
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://127.0.0.1:8080/livez >/dev/null || exit 1"]
      interval: 10s
      timeout: 3s
      retries: 3
      start_period: 20s

  app_b:
    image: ${APP_IMAGE}
    restart: unless-stopped
    env_file:
      - .env.app
    environment:
      APP_INSTANCE: app_b
    ports:
      - "127.0.0.1:18082:8080"
    networks:
      - app_net
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://127.0.0.1:8080/livez >/dev/null || exit 1"]
      interval: 10s
      timeout: 3s
      retries: 3
      start_period: 20s

  valkey:
    image: valkey/valkey:<PINNED_VERSION>
    restart: unless-stopped
    command:
      - valkey-server
      - --save
      - ""
      - --appendonly
      - "no"
    networks:
      - app_net

networks:
  app_net:
    driver: bridge
```

> 请把 `<PINNED_VERSION>` 替换为你验证过的固定版本，不要长期运行 `latest`。

如果你的应用镜像没有 `wget`：

- 换成镜像中存在的 `curl`。
- 或在应用自身提供 healthcheck 命令。
- 不要为了 healthcheck 给生产镜像随便塞一堆不需要的软件。

---

### 8.5 `.env.deploy`

```bash
APP_IMAGE=ghcr.io/YOUR_ORG/YOUR_APP@sha256:REPLACE_ME
```

生产环境推荐使用：

```text
image digest
```

而不是：

```text
latest
```

---

### 8.6 `.env.app`

FRA 示例：

```dotenv
APP_ENV=production
APP_REGION=fra
PORT=8080

DATABASE_URL=postgresql://app_user:URL_ENCODED_PASSWORD@10.77.0.13:5432/social?sslmode=disable

VALKEY_URL=redis://valkey:6379

JWT_ISSUER=https://api.example.com
JWT_SIGNING_KEY=REPLACE_ME

OBJECT_STORAGE_ENDPOINT=REPLACE_ME
OBJECT_STORAGE_BUCKET=REPLACE_ME
OBJECT_STORAGE_REGION=REPLACE_ME
```

SJC：

```dotenv
APP_REGION=sjc
```

SIN：

```dotenv
APP_REGION=sin
```

其它保持一致。

> `sslmode=disable` 这里只建立在“PostgreSQL 连接只走 WireGuard 加密隧道”的前提上。如果你的数据库流量可能离开 WireGuard，必须启用 PostgreSQL TLS。

---

### 8.7 DB 连接池

不要每个 HTTP 请求都新建数据库 TCP/TLS 连接。

应用内部使用连接池。

第一阶段可从每个 App 实例：

```text
5~10 connections
```

开始。

当前：

```text
3 regions × 2 app instances × 10
= 60 database connections
```

再加 migration / admin / monitor，通常仍然可控。

不要盲目：

```text
pool_size=100
```

否则 6 个实例就是 600 个连接。

---

## 9. Caddy 反向代理与源站 TLS

本文让 Caddy 运行在 host，而不是 Docker。

原因：

1. 防火墙行为更容易理解。
2. 443 不需要 Docker port publishing。
3. App 只绑定 127.0.0.1。
4. Caddy 可以在本机两个 App 实例间健康检查。

---

### 9.1 安装 Caddy

按照 Caddy 官方 Debian/Ubuntu repository 安装。

验证：

```bash
caddy version
systemctl status caddy
```

---

### 9.2 Cloudflare Origin CA

Cloudflare Dashboard：

```text
SSL/TLS
→ Origin Server
→ Create Certificate
```

建议覆盖：

```text
api.example.com
ws.example.com
```

或合理的：

```text
*.example.com
```

保存：

```text
/etc/caddy/certs/origin.pem
/etc/caddy/certs/origin.key
```

权限：

```bash
sudo mkdir -p /etc/caddy/certs

sudo chown root:caddy /etc/caddy/certs/origin.pem
sudo chown root:caddy /etc/caddy/certs/origin.key

sudo chmod 640 /etc/caddy/certs/origin.pem
sudo chmod 640 /etc/caddy/certs/origin.key
```

Cloudflare 设置：

```text
SSL/TLS mode = Full (strict)
```

Origin CA 只用于：

```text
Cloudflare ↔ Origin
```

如果绕过 Cloudflare 直接浏览源站，浏览器不一定信任该证书，这是正常现象。

---

### 9.3 Caddyfile

`/etc/caddy/Caddyfile`

```caddy
{
    admin 127.0.0.1:2019
}

api.example.com {
    tls /etc/caddy/certs/origin.pem /etc/caddy/certs/origin.key

    encode zstd gzip

    header {
        -Server
        X-Content-Type-Options "nosniff"
        Referrer-Policy "strict-origin-when-cross-origin"
    }

    reverse_proxy 127.0.0.1:18081 127.0.0.1:18082 {
        lb_policy round_robin

        health_uri /livez
        health_interval 10s
        health_timeout 2s
        health_fails 3
        health_passes 2

        fail_duration 30s
        max_fails 2

        header_up X-Real-IP {http.request.header.CF-Connecting-IP}
    }

    log {
        format json
    }
}
```

验证：

```bash
sudo caddy validate --config /etc/caddy/Caddyfile
```

格式化：

```bash
sudo caddy fmt --overwrite /etc/caddy/Caddyfile
```

重载：

```bash
sudo systemctl reload caddy
```

本机测试：

```bash
curl -k \
  --resolve api.example.com:443:127.0.0.1 \
  https://api.example.com/livez
```

---

### 9.4 WebSocket

Caddy `reverse_proxy` 可以处理 WebSocket Upgrade。

Cloudflare 也支持 proxied WebSocket。

因此你的应用可以：

```text
wss://api.example.com/ws
```

或：

```text
wss://ws.example.com/
```

客户端必须实现：

- heartbeat。
- reconnect。
- exponential backoff。
- jitter。
- last_event_id / last_message_id。
- reconnect 后补拉漏掉的消息。

不要假设一条 WebSocket 可以跨 Region failover 后“无缝迁移”。

---


## 10. PostgreSQL 单主双副本

本文示例：

```text
Initial Primary = SIN = 10.77.0.13
Replica         = FRA = 10.77.0.11
Replica         = SJC = 10.77.0.12
```

示例采用 PostgreSQL 18。

核心原则：

```text
任意时刻只允许一个可写 Primary
+
另外两个节点随时具备被 Promote 的网络与配置条件
```

这意味着**不能只配置当前 SIN 能当 Primary**；FRA、SJC 也必须提前具备：

- 正确的 `listen_addresses`。
- `wal_level=replica`。
- 足够的 `max_wal_senders`。
- 足够的 `max_replication_slots`。
- 允许另外两个节点进行 replication 的 `pg_hba.conf`。
- pgBackRest 配置和 WAL archive 配置。
- WireGuard 5432 防火墙规则。

否则真正故障时虽然 `pg_promote()` 成功，新 Primary 却可能因为没有监听 WG IP 或 `pg_hba.conf` 不允许而无法接收应用和副本连接。

---

### 10.1 三台都安装 PostgreSQL 18

三台执行：

```bash
sudo apt install -y postgresql-common ca-certificates
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh

sudo apt update
sudo apt install -y postgresql-18 postgresql-client-18
```

检查：

```bash
psql --version
pg_lsclusters
```

记录实际 data directory：

```bash
sudo -u postgres psql -Atqc "show data_directory;"
```

本文后续假设：

```text
/var/lib/postgresql/18/main
```

---

### 10.2 三台预配置可 Promote 的 PostgreSQL 参数

#### FRA

`/etc/postgresql/18/main/postgresql.conf`

```conf
listen_addresses = '127.0.0.1,10.77.0.11'

password_encryption = 'scram-sha-256'

wal_level = replica
max_wal_senders = 10
max_replication_slots = 10

wal_keep_size = '1024MB'

# 必须按磁盘和 WAL 速率估算；8GB 只是示例起点。
max_slot_wal_keep_size = '8192MB'

hot_standby = on

# pg_rewind 需要 wal_log_hints=on 或数据校验和满足其前提。
wal_log_hints = on
```

#### SJC

只把监听 IP 改成：

```conf
listen_addresses = '127.0.0.1,10.77.0.12'
```

其余 replication 参数一致。

#### SIN

```conf
listen_addresses = '127.0.0.1,10.77.0.13'
```

其余一致。

#### 为什么 Standby 也要设置这些参数？

因为 Failover 后：

```text
Standby
  ↓ pg_promote()
New Primary
```

它必须立刻能够：

- 接收 App 的 5432。
- 创建 replication slots。
- 给另一个 Standby 发送 WAL。
- 产生并归档自己的 WAL。

PostgreSQL 也要求 Standby 的一些 replication 参数至少不低于 Primary，否则恢复/查询可能出现问题。

> `max_slot_wal_keep_size=-1` 默认允许 slot 无限保留 WAL。生产环境一定要结合磁盘余量设置上限并监控；slot 一旦落后超过上限可能进入 `lost`，此时需要重建该 Standby。

---

### 10.3 三台都预配置 `pg_hba.conf`

三台 `/etc/postgresql/18/main/pg_hba.conf` 都加入：

```conf
# Runtime application traffic over WireGuard.
host    social         app_user       10.77.0.0/24       scram-sha-256

# Migration account: 建议只允许受控来源；这里先限制 WireGuard。
host    social         migration_user 10.77.0.0/24       scram-sha-256

# Physical replication: 任意节点 Promote 后都能让其它两个节点跟随。
host    replication    replicator     10.77.0.11/32      scram-sha-256
host    replication    replicator     10.77.0.12/32      scram-sha-256
host    replication    replicator     10.77.0.13/32      scram-sha-256
```

不要：

```conf
host all all 0.0.0.0/0 scram-sha-256
```

三台 UFW：

```bash
sudo ufw allow in on wg0 from 10.77.0.0/24 to any port 5432 proto tcp
```

然后三台重启：

```bash
sudo systemctl restart postgresql
sudo systemctl status postgresql
```

分别确认监听地址。

FRA：

```bash
sudo ss -lntp | grep 5432
```

预期包含：

```text
127.0.0.1:5432
10.77.0.11:5432
```

SJC/SIN 对应各自 WG IP。

---

### 10.4 只在初始 Primary 创建业务与复制角色

SIN：

```bash
sudo -u postgres psql
```

执行：

```sql
CREATE ROLE app_user
LOGIN
PASSWORD 'REPLACE_WITH_LONG_RANDOM_PASSWORD';

CREATE DATABASE social
OWNER app_user;

CREATE ROLE migration_user
LOGIN
PASSWORD 'REPLACE_WITH_DIFFERENT_LONG_RANDOM_PASSWORD';

GRANT CONNECT ON DATABASE social TO migration_user;

CREATE ROLE replicator
WITH REPLICATION LOGIN
PASSWORD 'REPLACE_WITH_LONG_RANDOM_REPLICATION_PASSWORD';
```

Migration 权限必须按你的 migration 工具进一步授予，不建议简单给 superuser。

运行时原则：

```text
app_user       → 业务 DML
migration_user → 受控 DDL
replicator     → physical replication only
postgres       → 运维，不给 App
```

这些角色与数据库对象会通过物理复制进入两个 Standby。

---

### 10.5 初始 Primary 验证

SIN：

```bash
sudo ss -lntp | grep 5432

sudo -u postgres psql -c \
  "select inet_server_addr(), pg_is_in_recovery();"
```

预期：

```text
10.77.0.13
f
```

从 FRA：

```bash
psql \
  -h 10.77.0.13 \
  -U app_user \
  -d social \
  -c 'SELECT now();'
```

从 SJC 同样测试。

如果连接失败，按顺序检查：

```bash
wg show
ip route get 10.77.0.13
nc -vz 10.77.0.13 5432
sudo ufw status
sudo ss -lntp | grep 5432
```

---

### 10.6 创建初始 Physical Replication Slots

SIN：

```bash
sudo -u postgres psql
```

```sql
SELECT * FROM pg_create_physical_replication_slot('fra_slot');
SELECT * FROM pg_create_physical_replication_slot('sjc_slot');
```

查看：

```sql
SELECT
    slot_name,
    active,
    restart_lsn,
    wal_status,
    safe_wal_size
FROM pg_replication_slots;
```

必须告警：

```text
slot inactive
+
safe_wal_size 持续下降
+
Primary disk 持续增长
```

如果 slot 进入 `lost`，不要假装它还能继续复制；通常应修复网络后重新做 base backup / 重建 standby。

---

### 10.7 三台都准备 replication `.pgpass`

为了 Failover 后不临时找密码，三台都给 postgres 用户准备：

```bash
sudo -u postgres bash
```

```bash
cat > ~/.pgpass <<'EOF'
10.77.0.11:5432:replication:replicator:REPLICATION_PASSWORD
10.77.0.12:5432:replication:replicator:REPLICATION_PASSWORD
10.77.0.13:5432:replication:replicator:REPLICATION_PASSWORD
EOF

chmod 600 ~/.pgpass
exit
```

`REPLICATION_PASSWORD` 替换成同一个真实密码。

注意：

- `.pgpass` 文件所有者应为 `postgres`。
- 权限必须是 `0600`。
- 不要提交 Git。
- 定期 rotate 时要同步更新三台。

---

### 10.8 FRA 初始化为 Standby

FRA：

```bash
sudo systemctl stop postgresql
```

**再次确认 hostname/IP，避免清错机器：**

```bash
hostname
ip addr show wg0
```

应看到：

```text
10.77.0.11
```

然后：

```bash
sudo rm -rf /var/lib/postgresql/18/main/*
```

执行：

```bash
sudo -u postgres pg_basebackup \
  -h 10.77.0.13 \
  -U replicator \
  -D /var/lib/postgresql/18/main \
  -R \
  -X stream \
  -S fra_slot \
  --progress
```

`-R` 会创建 Standby 所需恢复设置并生成 `standby.signal`。

启动：

```bash
sudo systemctl start postgresql
```

验证：

```bash
sudo -u postgres psql -Atqc "select pg_is_in_recovery();"
```

预期：

```text
t
```

确认 slot：

```bash
sudo -u postgres psql -Atqc "show primary_slot_name;"
```

预期：

```text
fra_slot
```

确认上游：

```bash
sudo -u postgres psql -Atqc "show primary_conninfo;"
```

应指向：

```text
10.77.0.13
```

---

### 10.9 SJC 初始化为 Standby

流程相同。

SJC 先确认：

```text
wg0 = 10.77.0.12
```

然后：

```bash
sudo systemctl stop postgresql
sudo rm -rf /var/lib/postgresql/18/main/*

sudo -u postgres pg_basebackup \
  -h 10.77.0.13 \
  -U replicator \
  -D /var/lib/postgresql/18/main \
  -R \
  -X stream \
  -S sjc_slot \
  --progress

sudo systemctl start postgresql
```

验证：

```bash
sudo -u postgres psql -Atqc "select pg_is_in_recovery();"
sudo -u postgres psql -Atqc "show primary_slot_name;"
sudo -u postgres psql -Atqc "show primary_conninfo;"
```

---

### 10.10 Primary 检查复制状态

SIN：

```sql
SELECT
    application_name,
    client_addr,
    state,
    sync_state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn
FROM pg_stat_replication;
```

应看到 FRA 与 SJC 两个连接。

本文默认：

```text
sync_state = async
```

不要轻率设置：

```text
synchronous_commit = remote_apply
```

并让 commit 等待跨洲 Standby，否则业务写延迟会直接被 WAN RTT 拉高。

---

### 10.11 Replica Lag 与 Slot 监控

Standby：

```sql
SELECT
    now() - pg_last_xact_replay_timestamp() AS replay_delay,
    pg_last_wal_receive_lsn(),
    pg_last_wal_replay_lsn();
```

Primary：

```sql
SELECT
    application_name,
    client_addr,
    state,
    pg_wal_lsn_diff(sent_lsn, replay_lsn) AS replay_lag_bytes
FROM pg_stat_replication;
```

Slots：

```sql
SELECT
    slot_name,
    active,
    wal_status,
    safe_wal_size,
    restart_lsn
FROM pg_replication_slots;
```

告警至少包含：

- `state != streaming`。
- time lag > 60s warning。
- time lag > 300s critical。
- slot inactive。
- `safe_wal_size` 逼近 0。
- `wal_status` 变为 `unreserved`/`lost`。
- `pg_wal` 所在磁盘 > 80% / 90%。

---

### 10.12 初期读流量策略

第一版推荐：

```text
所有核心业务读写 → Primary
```

先保证行为简单正确。

以后仅把明确允许 eventual consistency 的请求发送本地 Replica，例如：

- Public Feed。
- 热门内容。
- 公开主页。
- 搜索展示。
- 非关键统计。
- 后台报表。

以下请求继续读 Primary：

```text
写后立即读
账户权限
封禁状态
安全设置
关键计数
支付/余额
需要强 read-after-write 的页面
```

否则可能：

```text
写已 commit 到 Primary
→ 本地 Replica 尚未 replay
→ 用户立刻看到旧状态
```

如果未来引入读写分离，在代码层显式提供：

```text
db.primary
db.replica
ConsistencyStrong
ConsistencyEventual
```

不要让 ORM 自己随机挑读库。



## 11. PostgreSQL 备份、WAL 归档与 PITR

**Replica 不是 Backup。**

例如误执行：

```sql
DELETE FROM users;
```

两个 Replica 会把错误忠实复制过去。

所以必须具备：

```text
Base Backup
+
Continuous WAL Archive
+
Offsite Object Storage
+
Point-in-Time Recovery
+
Restore Drill
```

深度 Review 后，本方案把 pgBackRest **预装并配置到三台数据库节点**，而不是只装在初始 SIN Primary。

原因：

```text
SIN Primary 故障
→ FRA Promote
→ FRA 应立即具备 archive/backup 能力
```

不应该等事故发生后才安装备份工具。

---

### 11.1 三台都安装与配置 pgBackRest

三台：

```bash
sudo apt install -y pgbackrest
sudo mkdir -p /etc/pgbackrest
sudo chmod 750 /etc/pgbackrest
```

三台使用相同 repository 配置：

`/etc/pgbackrest/pgbackrest.conf`

```ini
[social]
pg1-path=/var/lib/postgresql/18/main

[global]
repo1-type=s3
repo1-path=/postgresql/social

repo1-s3-bucket=REPLACE_BUCKET
repo1-s3-endpoint=REPLACE_ENDPOINT
repo1-s3-region=REPLACE_REGION

repo1-s3-key=REPLACE_ACCESS_KEY
repo1-s3-key-secret=REPLACE_SECRET_KEY

repo1-retention-full=4
repo1-retention-diff=14

start-fast=y
process-max=2
log-level-console=info
```

权限：

```bash
sudo chown postgres:postgres /etc/pgbackrest/pgbackrest.conf
sudo chmod 600 /etc/pgbackrest/pgbackrest.conf
```

如果 repository 支持：

- Object Lock / immutability。
- Versioning。
- 独立账号。
- 独立区域。

优先开启。

如果启用 pgBackRest repository encryption：

- cipher pass 独立备份。
- 不只存当前 VPS。
- 不只存同一个 S3 bucket。
- 丢失 cipher pass 等于备份不可用。

---

### 11.2 三台 PostgreSQL 都配置 WAL Archive

三台 `/etc/postgresql/18/main/postgresql.conf` 都设置：

```conf
archive_mode = on
archive_command = 'pgbackrest --stanza=social archive-push %p'
archive_timeout = '60s'
```

然后：

```bash
sudo systemctl restart postgresql
```

这里使用：

```text
archive_mode = on
```

而不是：

```text
archive_mode = always
```

原因是：

- `on` 时 Standby 恢复期间不会重复归档收到的 WAL。
- Standby Promote 成 Primary 后会开始归档它自己新产生的 WAL。
- 可以避免三个节点同时向同一个 repository archive-push 造成冲突风险。

这也意味着：Failover 前必须确保旧 Primary 已 **fence**，避免旧 Primary 和新 Primary 同时产生并推送分叉 timeline 的 WAL。

---

### 11.3 初始 Primary 创建并检查 stanza

SIN：

```bash
sudo -u postgres pgbackrest \
  --stanza=social \
  stanza-create
```

检查：

```bash
sudo -u postgres pgbackrest \
  --stanza=social \
  check
```

首次 Full：

```bash
sudo -u postgres pgbackrest \
  --stanza=social \
  --type=full \
  backup
```

查看：

```bash
sudo -u postgres pgbackrest \
  --stanza=social \
  info
```

确认 PostgreSQL archive 状态：

```sql
SELECT
    archived_count,
    failed_count,
    last_archived_wal,
    last_archived_time,
    last_failed_wal,
    last_failed_time
FROM pg_stat_archiver;
```

---

### 11.4 三台都部署“只在当前 Primary 执行备份”的脚本

创建：

```bash
sudo nano /usr/local/sbin/pgbackrest-backup-if-primary
```

内容：

```bash
#!/usr/bin/env bash
set -euo pipefail

TYPE="${1:-incr}"

case "$TYPE" in
  full|diff|incr) ;;
  *)
    echo "invalid backup type: $TYPE" >&2
    exit 2
    ;;
esac

ROLE="$(sudo -u postgres psql -XAtqc \
  'select pg_is_in_recovery();' postgres 2>/dev/null || true)"

if [[ "$ROLE" != "f" ]]; then
  # Standby 或 PostgreSQL 不可用时不执行备份。
  exit 0
fi

exec sudo -u postgres \
  pgbackrest --stanza=social --type="$TYPE" backup
```

权限：

```bash
sudo chmod 750 /usr/local/sbin/pgbackrest-backup-if-primary
```

三台都配置同样 cron：

`/etc/cron.d/pgbackrest-social`

```cron
0 2 * * 0 root /usr/local/sbin/pgbackrest-backup-if-primary full
0 2 * * 1-6 root /usr/local/sbin/pgbackrest-backup-if-primary diff
20 */6 * * * root /usr/local/sbin/pgbackrest-backup-if-primary incr
```

这样正常状态：

```text
SIN → pg_is_in_recovery() = false → 执行
FRA → true → 跳过
SJC → true → 跳过
```

Failover 后如果 FRA Promote：

```text
FRA → false → 下一周期自动承担 backup
SJC → true → 跳过
```

事故处理时仍应**立即手工验证**新 Primary 的 `pgbackrest check` 与 WAL archive，不要等下一次 cron 才发现 Secret/网络有问题。

---

### 11.5 备份监控

至少告警：

```text
最近 Full backup > 8 days
最近任意 backup > 26 hours
WAL archive 持续失败
last_archived_time 异常陈旧
Object Storage 不可达
repository quota/空间不足
restore drill 失败
```

另外监控：

```sql
SELECT
    archived_count,
    failed_count,
    last_archived_time,
    last_failed_time
FROM pg_stat_archiver;
```

Failover 后重点观察：

```text
新 timeline 的 WAL 是否开始进入 repository
```

---

### 11.6 PITR 恢复演练

每月至少一次；最低也应每季度完整演练。

**不要第一次 restore 就发生在真正事故当天。**

推荐使用：

```text
临时 VPS / 临时磁盘 / 隔离网络
```

流程：

1. 安装相同 PostgreSQL major version。
2. 安装 pgBackRest。
3. 配置 repository credential。
4. 停 PostgreSQL。
5. 确认目标机不是生产节点。
6. 清空测试 data directory。
7. 执行 time-target restore。
8. 启动 PostgreSQL。
9. 检查业务数据。
10. 记录实际 RTO。
11. 销毁临时恢复环境。

示例：

```bash
sudo systemctl stop postgresql

hostname
sudo -u postgres psql -Atqc "show data_directory;" 2>/dev/null || true
```

确认无误后：

```bash
sudo rm -rf /var/lib/postgresql/18/main/*
```

恢复：

```bash
sudo -u postgres pgbackrest \
  --stanza=social \
  --type=time \
  --target="2026-09-14 12:00:00+00" \
  --target-action=promote \
  restore
```

然后：

```bash
sudo systemctl start postgresql
```

检查：

```bash
sudo -u postgres psql -c \
  "select pg_is_in_recovery(), now();"
```

业务验证至少包含：

- users 数量。
- posts 数量。
- messages 最新 ID/时间。
- migration schema version。
- 关键索引。
- 关键唯一约束。
- 随机抽样用户与帖子。
- 恢复目标时间之后的数据**不应出现**。

记录：

```text
restore start
restore complete
PostgreSQL start
business validation complete
Total RTO
Recovered-to timestamp
```

---

### 11.7 备份与副本解决的是不同问题

| 机制 | 主要解决 |
|---|---|
| Standby | Primary 主机/Region 故障 |
| Physical replication | 快速切换到接近最新的数据 |
| WAL Archive | 连续恢复链 |
| Full/Diff/Incr backup | 基础恢复点 |
| PITR | 误删、错误 migration、逻辑事故 |
| Object immutability | 勒索/凭据泄露后降低备份被删风险 |
| Restore Drill | 证明“备份真的能恢复” |

任何一项都不能单独替代其它项。


## 12. Cloudflare 全球入口与就近路由

### 12.1 基础 DNS

不要直接给用户暴露：

```text
fra-api.example.com → FRA IP
sjc-api.example.com → SJC IP
sin-api.example.com → SIN IP
```

App 统一使用：

```text
https://api.example.com
```

客户端不负责 Region 选择。

---


### 12.2 创建 3 个 Pool

建议：

```text
pool-fra
pool-sjc
pool-sin
```

每个 pool 第一阶段只有对应的一台 VPS endpoint：

```text
pool-fra → FRA_PUBLIC_IP
pool-sjc → SJC_PUBLIC_IP
pool-sin → SIN_PUBLIC_IP
```

Endpoint 使用公网 IP 没问题，但 HTTPS monitor 和 Caddy virtual host 必须知道真正 hostname。

因此：

```text
Monitor Host Header = api.example.com
```

不要让 health monitor 默认使用：

```text
Host: <raw-ip-address>
```

否则：

- Caddy 可能匹配不到 `api.example.com` site。
- TLS SNI/证书名可能不匹配。
- Endpoint 会被 Cloudflare 判 unhealthy。

---

### 12.3 Health Monitor：必须同时验证 HTTPS、Host/SNI 和核心 DB

推荐：

```text
Type:            HTTPS
Port:            443
Path:            /health/edge
Method:          GET
Host Header:     api.example.com
Expected Codes:  200 / 2xx
Expected Body:   可选，例如 "ok"
Timeout:         3~5s
Retries:         0~1
Interval:        按 Cloudflare 套餐允许的最小值与源站负载选择
```

Cloudflare 当前文档说明：

- Pro 的 monitor 最小 interval 可能为 60s。
- Business 可低至约 15s。
- Enterprise 可低至约 10s。
- Endpoint 与 Monitor 都设置 Host header 时，Endpoint 的 override 优先。
- 对 HTTPS monitor，正确 Host header 也关系到 TLS ServerName/SNI。

因此不要在文档里把：

```text
15 秒 failover
```

当作所有套餐都一定能做到的承诺。

#### `/health/edge` 的判定

应该快速检查：

```text
App 可以服务
+
核心 PostgreSQL 可在短超时内访问
```

不要检查：

```text
邮件服务
分析平台
非核心第三方 API
NATS 非必要通知
大型 SELECT
对象存储深度读写
```

否则一个非核心依赖会把整个 Region 摘掉。

建议 body：

```json
{"status":"ok"}
```

Cloudflare 可选 `Expected Body = "ok"`。

#### Health Monitor Regions

不要无脑选择所有地区。

Cloudflare 会从所选 Region 的多个数据中心发 probe；地区越多、interval 越短，请求越多。

第一版选择能代表：

```text
Europe
North America
Asia
```

的必要健康检查区域即可。

如果使用 Authenticated Origin Pulls、BYO CA、Argo/特定 egress 路径等功能，要按 Cloudflare 当前文档评估 `Simulate Zone`，否则 monitor 的路径与真实用户流量路径可能不同。

---

### 12.4 Geo Steering 与“跨区域 fallback”必须显式配置

简单规则可以：

```text
Europe:
  1. FRA
  2. SJC
  3. SIN

North America:
  1. SJC
  2. FRA
  3. SIN

Asia:
  1. SIN
  2. FRA
  3. SJC
```

关键点：

> **不要只给每个 Geo 分配一个 Pool，然后假设它 unhealthy 后 Cloudflare 一定会自动选择你心目中的“第二近 Region”。**

Cloudflare Geo steering 会尊重每个 location 的 pool failover order；未显式定义的路径会落到 default/fallback 逻辑。Cloudflare 的 Adaptive Routing 文档也特别说明，Geo 场景的 cross-pool failover 有分组边界。

所以第一阶段最容易理解和演练的方式就是：

```text
每个 Region 显式写完整的 ordered pool list
```

并配置全局 fallback pool 作为最后兜底。

例如 Europe：

```text
[FRA, SJC, SIN]
```

然后实际做：

```text
disable FRA endpoint
→ 从欧洲测试客户端请求
→ 验证 /version 返回 sjc
```

恢复 FRA：

```text
→ 验证自动 failback 行为是否符合配置
```

#### Dynamic Steering

如果 Cloudflare 套餐支持且成本可以接受，可以评估 Dynamic Latency / Pool Sets。

但无论算法多智能：

```text
真实压测与跨洲 RTT baseline
>
“地理位置看起来更近”
```

一定要从真实欧洲、北美、亚洲网络验证 `/version` 与请求延迟。


### 12.5 Session Affinity

如果 App 真正 stateless：

```text
HTTP session affinity = OFF
```

WebSocket 本身建立后就是长连接，不需要通过普通 HTTP sticky session 维持。

重连后可能到另一个 Region。

所以 WS 服务必须支持：

```text
reconnect to any region
```

---

### 12.6 SSL/TLS

Cloudflare：

```text
Client
  ↓ HTTPS
Cloudflare
  ↓ HTTPS + strict validation
Caddy
```

设置：

```text
Full (strict)
```

不要用：

```text
Flexible
```

---

### 12.7 防止绕过 Cloudflare

Cloudflare proxy 不能代替 origin firewall。

最终 VPS 443 应：

```text
ALLOW Cloudflare IP ranges
DENY  other Internet sources
```

Cloudflare 当前 IP ranges 需要从官方地址获取并定期同步。

Bootstrap 示例：

```bash
for ip in $(curl -fsSL https://www.cloudflare.com/ips-v4); do
  sudo ufw allow from "$ip" to any port 443 proto tcp
done

for ip in $(curl -fsSL https://www.cloudflare.com/ips-v6); do
  sudo ufw allow from "$ip" to any port 443 proto tcp
done
```

然后删除之前给 Admin IP 临时开的 443。

检查：

```bash
sudo ufw status numbered
```

生产上应把 Cloudflare IP 更新做成：

- version-controlled script。
- 原子更新。
- 定期 timer。
- 更新失败告警。

更高安全级别可以再启用 Cloudflare Authenticated Origin Pulls / 自有 client certificate mTLS。

---

### 12.8 获取真实客户端 IP

Origin 看到的 TCP peer 是 Cloudflare。

业务日志、风控需要使用可信的：

```text
CF-Connecting-IP
```

前提：

```text
Origin 443 只允许 Cloudflare
```

否则攻击者直接访问源站时可以伪造该 Header。

---

### 12.9 缓存策略

对：

```text
/api/me
/api/messages
/api/settings
/api/admin
```

不要边缘缓存。

可以考虑缓存：

```text
公开用户头像
公开媒体
公共静态 JSON
公共 Trending Feed 的短 TTL
```

Authenticated API 默认：

```text
Cache-Control: no-store / private
```

除非你明确理解缓存 key。

---

## 13. 对象存储与社交媒体文件

用户内容不要放：

```text
/opt/socialapp/uploads
```

正确模式：

```text
Mobile App
   │
   ├─ 1. 向 API 请求 upload authorization
   ▼
API
   │
   ├─ 2. 生成短期 presigned upload
   ▼
Object Storage
   ▲
   └─ 3. Client direct upload
```

数据库只记录：

```text
object_key
owner_id
mime_type
size
width
height
status
created_at
```

例如：

```text
users/01J.../avatar/01J....webp
posts/01J.../media/01J....jpg
```

不要把数据库主键顺序直接暴露成可猜 URL。

---

### 13.1 上传安全

后端生成 presigned URL 时限制：

- object prefix。
- Content-Type。
- 最大尺寸。
- 有效时间。
- 上传用户。
- 文件数量。
- 必要时 checksum。

客户端上传成功后：

```text
status = uploaded
```

后端异步处理：

```text
virus scan
image decode validation
metadata stripping
thumbnail
video transcode
moderation
```

之后：

```text
status = published
```

不要仅相信文件扩展名：

```text
evil.exe → photo.jpg
```

---

### 13.2 私密媒体

如果你的社交 App 有：

- 私聊图片。
- 私密账号内容。
- 仅好友可见视频。

不要使用永不过期的公开 Bucket URL。

使用：

- private bucket。
- signed URL。
- signed cookie。
- authorized media proxy。

并在授权时检查：

```text
viewer → resource → permission
```

---

## 14. 缓存、Session 与限流

### 14.1 Valkey 定位

每个 Region 一套 local Valkey：

```text
FRA Valkey
SJC Valkey
SIN Valkey
```

主要用于：

- cache。
- 本地 rate limiting。
- 短期 dedup。
- ephemeral presence。
- 本地任务协调。

必须满足：

```text
删除 Valkey 数据后
业务主数据不会永久丢失
```

---

### 14.2 Session

推荐：

```text
short-lived access token
+
server-managed refresh token
```

Access Token：

```text
5~15 min
```

Refresh Token：

- 数据库保存 hash / session record。
- 可 revoke。
- 可按 device 管理。
- 支持 rotation。
- 被盗重放时可以检测。

不要使用：

```text
JWT 有效期 180 天 + 无法撤销
```

---

### 14.3 Rate Limit

Local Valkey 的限流存在一个问题：

用户可以：

```text
FRA 10次
SJC 10次
SIN 10次
```

因此：

#### 可以使用 local limit

- 普通 feed refresh。
- 非安全关键 API。
- 防误操作。

#### 安全关键 limit 要全局化

例如：

- Login。
- OTP。
- Password reset。
- 注册。
- 邀请码。
- 敏感管理 API。

应组合：

```text
Cloudflare edge rate limit
+
Database/global durable counter
+
device/account/IP risk model
```

---

## 15. WebSocket 与实时消息

如果 App 没有实时聊天，可以先不部署 NATS。

如果有：

- Chat。
- Online presence。
- Typing。
- Push-like realtime events。

推荐使用：

```text
DB = source of truth
NATS = delivery/fanout bus
```

而不是：

```text
NATS 收到 = 数据就算永久保存
```

---

### 15.1 消息发送可靠路径

推荐：

```text
Alice
  │
  ▼
FRA API
  │
  ▼
PostgreSQL Transaction
  ├─ INSERT messages
  └─ INSERT outbox
          │
          ▼
     Outbox Worker
          │
          ▼
         NATS
          │
          ▼
    SIN WebSocket
          │
          ▼
         Bob
```

数据库事务中同时写：

```text
message
outbox_event
```

这样即使：

```text
DB commit 成功
NATS 暂时宕机
```

Outbox Worker 仍可稍后重发。

---

### 15.2 消费端幂等

Event 必须有：

```text
event_id
message_id
```

消费者：

```text
if event_id already processed:
    ignore
```

避免：

```text
at-least-once delivery
→ 重复推送
→ 重复业务操作
```

---

### 15.3 三地 NATS

小规模可以每 Region 一个 NATS server。

Region 之间用：

```text
NATS Gateways
```

走：

```text
WireGuard IP
```

不要把：

```text
4222
7222
```

暴露到公网。

更重要的是：

> 不要因为有三个 Region 就把 3 台跨洲 NATS 直接当作一个低延迟持久化 quorum。

如果未来需要真正持久的消息流：

- 在一个 Region 内至少 3 个低延迟节点组成 JetStream/Kafka quorum。
- 或使用 managed messaging。
- Region 间做 federation / gateway / DR。

---

### 15.4 WebSocket 重连协议

客户端建议：

```text
connect
→ authenticate
→ subscribe
→ heartbeat
```

断线：

```text
1s
2s
4s
8s
...
max 30s
+ random jitter
```

重连后：

```text
last_received_message_id
```

服务端：

```text
SELECT missed messages
FROM DB
WHERE id > last_received_message_id
```

实时总线只负责加速，不负责唯一真相。

---

## 16. CI/CD 与安全发布

核心原则：

```text
Build Once
Deploy Same Artifact Everywhere
```

---

### 16.1 推荐流程

```text
Git Push
   │
   ▼
CI
   ├─ unit test
   ├─ integration test
   ├─ lint
   ├─ dependency audit
   ├─ docker build
   ├─ vulnerability scan
   └─ push immutable image
          │
          ▼
     image@sha256:...
          │
          ▼
     Region-by-region deploy
```

禁止三台机器分别：

```bash
git pull
npm install
npm run build
```

否则环境会逐渐漂移。

---

### 16.2 镜像版本

同时打：

```text
app:2026.09.14-abcdef1
app@sha256:xxxxxxxx
```

生产 deploy 文件最终记录 digest。

回滚时：

```text
old digest
```

明确可追溯。

---

### 16.3 Region-by-Region 发布

推荐顺序：

```text
1. 从 Cloudflare 暂时 drain 一个 Region
2. 部署新镜像
3. 本机 health check
4. 外部 health check
5. 恢复该 Region
6. 观察 5xx / latency / logs
7. 再部署下一 Region
```

这样即使：

```text
新版本完全坏掉
```

也不会同时毁掉三地。

---

### 16.4 Deploy Script

`/opt/socialapp/scripts/deploy.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

cd /opt/socialapp

: "${APP_IMAGE:?APP_IMAGE is required}"

echo "[1/5] Pull image"
docker compose pull

echo "[2/5] Recreate services"
docker compose up -d --remove-orphans

echo "[3/5] Wait"
sleep 5

echo "[4/5] Local health"
curl -fsS http://127.0.0.1:18081/livez >/dev/null
curl -fsS http://127.0.0.1:18082/livez >/dev/null

echo "[5/5] Version"
curl -fsS http://127.0.0.1:18081/version
echo

docker compose ps
```

权限：

```bash
chmod 750 /opt/socialapp/scripts/deploy.sh
```

---

### 16.5 Rollback

不要：

```text
紧急修代码
→ 重新 build
→ 希望修好
```

优先：

```text
恢复上一份已知健康 digest
```

例如：

```bash
export APP_IMAGE='ghcr.io/org/app@sha256:OLD_DIGEST'

docker compose pull
docker compose up -d
```

然后验证。

---

### 16.6 GitHub Actions 思路

至少分两阶段：

```text
build
deploy
```

生产 deploy 最好：

- GitHub Environment approval。
- Production branch/tag。
- SSH host key pinning。
- Scoped registry token。
- Scoped Cloudflare token。
- 不使用 root SSH。
- CI Secret 不落日志。

Deploy matrix 不要三地同时跑。

使用：

```text
max-parallel: 1
```

或显式：

```text
fra → sjc → sin
```

如果数据库 Primary 在 SIN，可以考虑最后部署 SIN。

---

## 17. 数据库迁移规范

数据库 migration 是多区域发布最容易出事故的地方之一。

不要让 6 个 App 实例启动时同时：

```text
run migrations
```

推荐 CI/CD 中只有一个专门 migration job。

---

### 17.1 Expand / Migrate / Contract

例如把：

```text
users.name
```

改成：

```text
users.display_name
```

错误方法：

```text
DROP COLUMN name
ADD display_name
Deploy new code
```

旧 Region 还没部署时会直接坏。

正确：

#### Phase 1 — Expand

```text
ADD display_name
保留 name
```

应用同时兼容新旧字段。

#### Phase 2 — Backfill

后台迁移数据。

#### Phase 3 — Deploy

三地全部升级。

#### Phase 4 — Contract

确认没有旧代码后，再删除旧字段。

---

### 17.2 大表 DDL

避免高峰期：

```text
ALTER TABLE huge_table ...
```

直接持有长锁。

使用：

- `CREATE INDEX CONCURRENTLY`。
- 分批 backfill。
- lock timeout。
- statement timeout。
- 先在 staging 用生产量级数据测试。

---

## 18. 监控、日志与告警

只有三台 VPS 时，不建议唯一的监控系统也只放在这三台里。

否则：

```text
VPS 挂了
+
监控一起挂了
```

建议：

- 外部 SaaS monitoring。
- 或额外一台非常小的独立监控节点。
- 至少 external uptime probes 不依赖三台生产机。

---

### 18.1 最低监控指标

#### VPS

```text
CPU
load
memory
swap
disk usage
disk inode
disk IO
network
reboot
OOM
```

#### Caddy

```text
request rate
2xx/4xx/5xx
p50/p95/p99
active connections
upstream errors
```

#### App

```text
request duration
error rate
DB pool active/idle/wait
event loop lag
worker backlog
WebSocket connections
login failure
rate-limit hit
```

#### PostgreSQL

```text
connections
transactions/s
slow query
deadlock
lock wait
buffer/cache
DB size
WAL generation
checkpoint
replication state
replication lag
slot WAL retention
disk free
```

#### Backup

```text
last backup time
last full backup time
WAL archive success
restore drill status
```

#### NATS（如果使用）

```text
connections
gateway status
subscriptions
in/out msgs
slow consumers
reconnects
```

---

### 18.2 建议告警

初始阈值示例：

```text
Disk > 80%                  warning
Disk > 90%                  critical

5xx > 2% for 5 min          warning
5xx > 5% for 5 min          critical

DB replication lag > 60s    warning
DB replication lag > 300s   critical

Replica disconnected        critical

Replication slot inactive
+ retained WAL growing      critical

No successful backup > 26h  critical

TLS cert < 30 days          warning

WireGuard peer no handshake
during expected traffic     warning
```

实际根据业务 baseline 调整。

---

### 18.3 日志

统一 JSON：

```json
{
  "timestamp": "...",
  "level": "info",
  "request_id": "...",
  "region": "fra",
  "instance": "app_a",
  "route": "/v1/posts",
  "status": 200,
  "duration_ms": 17
}
```

不要日志：

```text
Authorization: Bearer ...
password
refresh_token
OTP
cookie
private message body
full credit card data
secret key
```

用户邮箱/手机号/IP 等 PII 也应该根据用途和保留政策最小化。

---

### 18.4 Request ID

Cloudflare → Caddy → App → DB task → NATS event。

建议每个请求有：

```text
request_id
trace_id
```

故障排查时非常有价值。

---

## 19. 故障处理 Runbook

### 19.1 App A 容器挂掉

检查：

```bash
docker compose ps
docker compose logs --tail=200 app_a
```

Caddy 应自动把流量送到 App B。

恢复：

```bash
docker compose up -d app_a
```

验证：

```bash
curl -fsS http://127.0.0.1:18081/livez
```

---

### 19.2 整个 FRA VPS 挂掉

Cloudflare：

```text
FRA health fail
→ pool-fra unhealthy
→ users routed to fallback
```

运维：

1. 查看 Provider Console。
2. 检查网络。
3. 检查 VPS 是否被关机/OOM。
4. 不急着改 DNS。
5. Cloudflare 已经承担入口 failover。
6. 恢复后先验证本机。
7. 再恢复 pool。

---

### 19.3 某 Region 与 DB Primary 的 WireGuard 路由断开

表现：

```text
Cloudflare → FRA
FRA App → DB timeout
```

如果 `/health/edge` 包含核心 DB connectivity：

```text
FRA 应被标 unhealthy
```

流量转其它 Region。

检查：

```bash
wg show
ping 10.77.0.13
nc -vz 10.77.0.13 5432
```

不要把 DB timeout 设为几十秒。

对用户请求：

```text
connect timeout 1~3 sec
statement timeout 按接口定义
```

并做好 retry policy。

> PostgreSQL 写事务是否可重试必须由业务层根据幂等性判断，不能遇到 timeout 就无脑重放。

---


### 19.4 PostgreSQL Primary 故障

这是全篇最重要的 Runbook。

示例事故：

```text
旧 Primary = SIN 10.77.0.13
FRA/SJC    = Standby
```

假设最终选择 FRA 作为新 Primary。

---

#### Step 0：冻结变更与确认事故

事故期间先停止：

- 正在进行的数据库 migration。
- 自动 deployment。
- 批量任务。
- 会显著写数据库的后台 job。

记录：

```text
incident start UTC
旧 Primary
两个 Standby 当前 LSN
最后一次成功 backup
最后 WAL archive 时间
```

---

#### Step 1：必须先 Fence 旧 Primary

PostgreSQL 官方 failover 文档强调：旧 Primary 恢复后必须有机制保证它知道自己不再是 Primary，否则两个节点都接受写入会导致数据分叉和损失。

因此在 Promote 前，尽可能做到可证明的：

```text
Old Primary cannot accept writes
```

优先级：

1. Provider Console → Power Off / Stop VM。
2. Provider firewall → 阻断它的用户入口和 WireGuard。
3. 已确认还能 SSH 时 → `systemctl stop postgresql`。
4. 如果有存储层 fencing → detach/disable。

不要只因为：

```text
FRA ping 不到 SIN
```

就 Promote。

可能只是：

```text
FRA ↔ SIN 网络分区
```

而 SIN 仍然对 SJC/Internet 正常。

如果无法 Fence，宁可扩大只读/维护窗口，也不要盲目制造 Split-Brain。

---

#### Step 2：比较两个 Standby 谁最新

FRA：

```bash
sudo -u postgres psql -Atqc \
  "select pg_last_wal_replay_lsn();"
```

SJC：

```bash
sudo -u postgres psql -Atqc \
  "select pg_last_wal_replay_lsn();"
```

同时检查：

```sql
SELECT
    pg_is_in_recovery(),
    now() - pg_last_xact_replay_timestamp() AS replay_delay,
    pg_last_wal_receive_lsn(),
    pg_last_wal_replay_lsn();
```

在两个节点都来自同一个旧 Primary timeline 的前提下，优先选择：

```text
replay LSN 更新
+
服务器健康
+
网络正常
```

的节点。

不要只按地理位置选择。

---

#### Step 3：Promote FRA

FRA：

```bash
sudo -u postgres psql \
  -c "SELECT pg_promote(wait => true, wait_seconds => 60);"
```

验证：

```bash
sudo -u postgres psql -Atqc \
  "select pg_is_in_recovery();"
```

预期：

```text
f
```

检查监听：

```bash
sudo ss -lntp | grep 5432
```

必须包含：

```text
10.77.0.11:5432
```

这就是为什么第 10 章要求 Standby 提前设置自己的 `listen_addresses` 和 `pg_hba.conf`。

---

#### Step 4：立即验证新 Primary WAL Archive

FRA：

```bash
sudo -u postgres pgbackrest \
  --stanza=social \
  check
```

检查：

```sql
SELECT
    archived_count,
    failed_count,
    last_archived_wal,
    last_archived_time
FROM pg_stat_archiver;
```

制造一个低风险写入/等待 WAL switch 的验证应在维护流程中提前设计。

不要等几小时后才发现：

```text
新 Primary 能写
但 WAL archive credential 不工作
```

---

#### Step 5：让三个 Region 的 App 指向 FRA

更新：

```dotenv
DATABASE_URL=postgresql://app_user:...@10.77.0.11:5432/social?sslmode=disable
```

按 Region 逐个重启：

```bash
cd /opt/socialapp
docker compose up -d
```

先从一个 Region smoke test，再全部恢复。

建议未来把 DB endpoint 抽象为：

```text
db-primary.internal
```

或 local PgBouncer/HAProxy，以降低改 6 个 App 实例的操作量；但在当前只有三台 VPS 时，不必为了这个再引入一个跨洲一致性控制面。

---

#### Step 6：业务 Smoke Test

至少：

```text
login
read own profile
create test post
read same post
delete test post
send test message（若有聊天）
```

从三个 Region 都执行。

检查 `/version` 和 request logs，确认请求并不是只有 FRA Region 才成功。

---

#### Step 7：立刻让“幸存的另一个 Standby”改跟新 Primary

这是很多手工 Failover 文档会漏掉的一步。

如果：

```text
FRA = 新 Primary
SJC = 幸存 Standby
SIN = 已 Fence
```

SJC 原本仍配置：

```text
primary_conninfo → SIN
primary_slot_name → sjc_slot（属于旧 SIN）
```

如果不处理：

```text
SJC 会持续尝试连接已经宕机的旧 Primary
```

系统实际上变成：

```text
FRA = 唯一一份正在前进的数据
```

##### 7.1 在 FRA 创建给 SJC 的 slot

FRA：

```bash
sudo -u postgres psql
```

```sql
SELECT * FROM pg_create_physical_replication_slot('sjc_slot');
```

如果提示同名已存在，先确认该 slot 是否确实用于当前 SJC，不要直接 drop。

##### 7.2 修改 SJC 上游

SJC：

```bash
sudo -u postgres psql <<'SQL'
ALTER SYSTEM SET primary_conninfo =
  'host=10.77.0.11 port=5432 user=replicator application_name=sjc connect_timeout=3';
ALTER SYSTEM SET primary_slot_name = 'sjc_slot';
SQL
```

因为第 10.7 节 `.pgpass` 已提前包含 FRA：

```text
10.77.0.11:5432:replication:replicator:...
```

所以无需把 replication password 写进 `primary_conninfo`。

重启 SJC PostgreSQL：

```bash
sudo systemctl restart postgresql
```

验证 SJC：

```bash
sudo -u postgres psql -Atqc \
  "select pg_is_in_recovery();"

sudo -u postgres psql -Atqc \
  "show primary_conninfo;"

sudo -u postgres psql -Atqc \
  "show primary_slot_name;"
```

预期：

```text
t
host=10.77.0.11 ...
sjc_slot
```

FRA 查看：

```sql
SELECT
    application_name,
    client_addr,
    state,
    sync_state
FROM pg_stat_replication;
```

应该看到 SJC streaming。

---

#### Step 8：把旧 SIN 重建成 FRA 的 Standby

SIN 恢复后：

**禁止直接启动成可写 Primary。**

安全默认方案：

```text
fresh pg_basebackup
```

尤其当数据库不大时，重建往往比事故现场临时调 `pg_rewind` 更稳。

##### 8.1 FRA 创建 SIN slot

FRA：

```sql
SELECT * FROM pg_create_physical_replication_slot('sin_slot');
```

##### 8.2 SIN 停库并重做 base backup

SIN：

```bash
sudo systemctl stop postgresql

hostname
ip addr show wg0
```

确认确实是：

```text
10.77.0.13
```

再执行：

```bash
sudo rm -rf /var/lib/postgresql/18/main/*

sudo -u postgres pg_basebackup \
  -h 10.77.0.11 \
  -U replicator \
  -D /var/lib/postgresql/18/main \
  -R \
  -X stream \
  -S sin_slot \
  --progress

sudo systemctl start postgresql
```

验证：

```bash
sudo -u postgres psql -Atqc \
  "select pg_is_in_recovery();"
```

预期：

```text
t
```

---

#### Step 9：大库时可使用 `pg_rewind`，但必须提前演练

PostgreSQL 的 `pg_rewind` 专门用于 timeline 分叉后把旧 Primary 快速同步到新 Primary。

它通常比完整 base backup 更快，但有明确前提：

- target data directory 必须停止。
- `wal_log_hints=on` 或满足数据校验和前提。
- 所需 WAL 必须可获得。
- 失败后 target data directory 可能需要直接重建。
- rewind 后必须正确设置 Standby recovery，否则旧节点可能再次分叉。

因此本文生产默认：

```text
数据库较小 → fresh pg_basebackup
数据库很大 + 已测试 pg_rewind → 使用 pg_rewind
```

不要第一次使用 `pg_rewind` 就在真实灾难中现学。

---

#### Step 10：恢复双副本状态并收尾

最终应恢复：

```text
FRA = Primary
SJC = Standby
SIN = Standby
```

FRA：

```sql
SELECT
    application_name,
    client_addr,
    state,
    sync_state,
    replay_lsn
FROM pg_stat_replication;
```

再检查：

```sql
SELECT
    slot_name,
    active,
    wal_status,
    safe_wal_size
FROM pg_replication_slots;
```

然后：

- 确认 pgBackRest 新 Primary backup 正常。
- 确认 WAL archive 正常。
- 确认三个 App Region 都使用 FRA。
- 确认 Cloudflare 所有 pool 健康。
- 删除无用旧 slots。
- 写事故时间线。
- 记录实际 RTO/RPO。
- 复盘为什么 Primary 故障以及为什么选择 FRA。

---

#### Step 11：不要立刻 Failback

SIN 原来是 Primary，不代表恢复后必须马上切回去。

更稳妥：

```text
FRA 已稳定成为 Primary
→ 继续运行
→ 在计划维护窗口再决定是否 Switchover 回 SIN
```

频繁故障现场来回切主，会显著增加误操作概率。


### 19.5 Replication Slot 导致磁盘上涨

检查：

```sql
SELECT
    slot_name,
    active,
    wal_status,
    safe_wal_size,
    restart_lsn
FROM pg_replication_slots;
```

如果某 Standby 已永久废弃：

```sql
SELECT pg_drop_replication_slot('old_slot');
```

只能在确认：

```text
该 slot 不再需要
```

后执行。

---

### 19.6 Object Storage 故障

不要让：

```text
上传失败
```

导致：

```text
用户帖子数据库事务也进入不可恢复半状态
```

推荐状态机：

```text
draft
uploading
uploaded
processing
published
failed
```

对象存储异常时：

- 暂停新媒体上传。
- 纯文本帖子仍可服务。
- 已存在媒体优先依赖 CDN/cache。
- 不删除 DB metadata。
- 后台任务指数退避。

---

## 20. 灾难恢复演练

每季度至少完整做一次。

---

### 20.1 Drill A：Kill App

```bash
docker stop socialapp-app_a-1
```

确认：

- Caddy 自动切 App B。
- 无明显 5xx。
- 告警收到。

---

### 20.2 Drill B：Kill Region

临时禁用/关机 FRA。

确认：

- Cloudflare 健康状态变化。
- 欧洲测试请求进入其它 Region。
- App 不依赖 FRA local state。

---

### 20.3 Drill C：WireGuard Partition

阻断 FRA → SIN DB。

确认：

- `/health/edge` 正确失败。
- FRA 被移出。
- 其它 Region 正常。
- 恢复 WG 后自动恢复。

---

### 20.4 Drill D：Primary Failover

在维护窗口模拟 SIN Primary down。

严格执行第 19.4 节。

记录：

```text
Detection time
Decision time
Fence time
Promotion time
App update time
Total RTO
Lost transactions / RPO
```

---

### 20.5 Drill E：PITR

从对象存储恢复到独立临时机。

检查：

```text
能否真的打开数据库
能否登录测试账号
能否找到指定时间前的数据
migration version 是否正确
```

备份成功日志不等于恢复成功。

---

## 21. 安全基线

### 21.1 网络

公网只开放必要服务。

```text
443 → Cloudflare only
22  → admin VPN / allowlisted IP
51820 → WireGuard peers
```

严禁公网：

```text
5432
6379
4222
8222
```

---

### 21.2 Secret

Secret 不：

- 放 Git。
- 放 Docker image。
- 打到日志。
- 放客户端。
- 共享给不需要的人。

第一阶段可以：

```text
root/deploy-owned .env
chmod 600
```

后续升级：

```text
SOPS + age
Vault
Cloud secret manager
```

---

### 21.3 Cloudflare Token

CI Token 采用最小权限：

- 只允许目标 Zone。
- 只允许 Load Balancer / DNS 必需操作。
- 不使用 Global API Key。

---

### 21.4 Password

用户密码：

```text
Argon2id
```

或可靠框架当前推荐算法。

不要自行实现密码 hash。

---

### 21.5 JWT

建议：

```text
access token 短期
refresh token server controlled
```

JWT key：

- 定期 rotate。
- 支持 `kid`。
- 新旧 key 有平滑 overlap。
- 不在 mobile binary 内保存 server private key。

---

### 21.6 Admin

Admin API：

- 强制 MFA。
- 细粒度角色。
- audit log。
- 高风险动作二次确认。
- 与普通用户接口分权限。

---

### 21.7 Dependency / Image

CI：

- dependency audit。
- container image scan。
- 固定 base image / digest。
- 定期安全更新。
- 不让生产机自动拉 `latest`。

OS：

```bash
apt update
apt upgrade
```

应该有明确维护窗口。

---

## 22. 容量规划与性能注意事项

### 22.1 跨洲数据库 RTT 是客观存在的

例如：

```text
FRA App
   ↓
SIN PostgreSQL
```

一次事务如果有：

```text
BEGIN
SELECT
UPDATE
INSERT
COMMIT
```

并且 ORM 每一步都产生网络 round trip，延迟会被成倍放大。

优化原则：

- 合并 SQL。
- 事务内减少 round trip。
- 用 prepared statements / connection pool。
- 避免 N+1 query。
- 不在事务中等待外部 HTTP。
- 把只读可陈旧流量逐渐放本地 replica/cache。

---

### 22.2 不要过早调 PostgreSQL 参数

初期优先：

- 足够 RAM。
- NVMe/SSD。
- 磁盘不超售。
- 正确索引。
- 慢查询监控。
- connection pool。

再根据：

```text
pg_stat_statements
EXPLAIN ANALYZE
IO metrics
```

调优。

---

### 22.3 磁盘保留

Primary 建议长期至少：

```text
20~30% free
```

因为空间还会被：

- WAL。
- replication slots。
- temp query。
- vacuum。
- index build。
- backup staging。
- log。

占用。

---

### 22.4 如果 VPS 很小

例如每台只有：

```text
2 vCPU / 2GB RAM
```

不要同时强行运行：

```text
2 App
PostgreSQL
NATS
Prometheus
Grafana
Loki
Elasticsearch
```

优先：

1. App。
2. PostgreSQL。
3. Caddy。
4. 基础 cache。
5. 外部 monitoring。
6. 对象存储外置。

---

## 23. 数据一致性与多区域业务规则

### 23.1 使用全局唯一 ID

不要依赖：

```text
Region-local auto increment
```

来生成跨区域事件 ID。

可使用：

```text
UUIDv7
ULID
```

如果数据库仍只有一个 Primary，普通 bigint identity 也可以。

但 Event / Client-generated object 推荐全局唯一 ID。

---

### 23.2 幂等 Key

对这些接口加入：

```text
Idempotency-Key
```

例如：

- 发帖。
- 发消息。
- 订单。
- 充值。
- 关键写操作。

网络超时后客户端重试，不应该重复生成业务记录。

---

### 23.3 Outbox Pattern

任何：

```text
DB write + publish event
```

都不要写成：

```text
INSERT DB
commit
publish NATS
```

然后希望永远不失败。

使用：

```text
BEGIN
  INSERT business_data
  INSERT outbox
COMMIT
```

再由 worker 发送。

---

### 23.4 Replica 只允许陈旧读

给代码层明确区分：

```text
db.primary
db.replica
```

而不是让 ORM 自动 random read。

最好 API/Repository 明确标记：

```text
ConsistencyStrong
ConsistencyEventual
```

---

## 24. 上线测试计划

正式开放用户之前，必须验证。

---

### 24.1 网络延迟

从每台服务器：

```bash
ping <other-region-public-ip>
ping 10.77.0.11
ping 10.77.0.12
ping 10.77.0.13
```

记录 baseline。

---

### 24.2 API

从欧洲、北美、亚洲测试源：

```bash
curl -sS -o /dev/null \
  -w '%{time_connect} %{time_starttransfer} %{time_total}\n' \
  https://api.example.com/livez
```

确认 Region：

```text
/version
```

---

### 24.3 Load Test

不要第一次压测就在正式用户高峰。

测试：

- login。
- feed。
- create post。
- comments。
- image presign。
- WebSocket connect。
- chat send。

重点观察：

```text
p50
p95
p99
5xx
DB pool wait
DB CPU
disk IO
WAL rate
```

---

### 24.4 Failover

至少验证：

```text
App A down
App B down
Region down
WireGuard to DB down
Replica down
NATS down
Object storage down
```

系统行为必须符合预期。

---

### 24.5 Security Test

检查：

```text
5432 public blocked
6379 public blocked
4222 public blocked
Origin 443 direct public blocked
SSH password login blocked
Root SSH blocked
.env not readable by other users
No secrets in image
No secrets in logs
```

---

## 25. 正式上线步骤

推荐按以下顺序。

### Phase 1 — Infrastructure

- [ ] 3 台 VPS OS 更新。
- [ ] deploy 用户。
- [ ] SSH key。
- [ ] Firewall。
- [ ] WireGuard mesh。
- [ ] hostname / UTC / NTP。

### Phase 2 — Database

- [ ] 选定 Primary。
- [ ] PostgreSQL 18。
- [ ] app role。
- [ ] migration role。
- [ ] replication role。
- [ ] 两个 standby。
- [ ] replication slot limit。
- [ ] lag monitoring。
- [ ] pgBackRest。
- [ ] WAL archive。
- [ ] 首次 full backup。
- [ ] restore drill。

### Phase 3 — App

- [ ] Build immutable image。
- [ ] Deploy FRA。
- [ ] Deploy SJC。
- [ ] Deploy SIN。
- [ ] `/livez`。
- [ ] `/health/edge`。
- [ ] `/version`。
- [ ] local Caddy balancing。

### Phase 4 — Storage

- [ ] Private bucket。
- [ ] CORS。
- [ ] presigned upload。
- [ ] media CDN。
- [ ] private media auth。
- [ ] upload validation。

### Phase 5 — Cloudflare

- [ ] DNS。
- [ ] Origin CA。
- [ ] Full strict。
- [ ] Pools。
- [ ] Health monitor。
- [ ] HTTPS Monitor `Host: api.example.com` / SNI 已验证。
- [ ] 每个 Geo 的 ordered fallback pools 已显式配置并演练。
- [ ] Geo/Dynamic steering。
- [ ] WAF。
- [ ] Rate limits。
- [ ] Origin firewall Cloudflare-only。

### Phase 6 — Observability

- [ ] external uptime。
- [ ] VPS metrics。
- [ ] app metrics。
- [ ] DB metrics。
- [ ] backup alert。
- [ ] replication lag。
- [ ] central logs。
- [ ] error tracking。

### Phase 7 — Failure Drill

- [ ] Kill App。
- [ ] Kill Region。
- [ ] Break WG。
- [ ] Promote DB replica。
- [ ] pg_rewind/reseed old primary。
- [ ] PITR。

### Phase 8 — Go Live

- [ ] 限量用户。
- [ ] 观察 24h。
- [ ] 扩大流量。
- [ ] 记录 baseline。
- [ ] 固化 Runbook。

---

## 26. 深度 Review：查缺补漏后的修正项

本节是对整套方案从：

```text
可用性
一致性
安全性
可恢复性
可运维性
复杂度
```

六个维度进行 Review 后得到的最终修正。

---

### Review 1：三地 App 有高可用，但数据库仍是核心故障域

问题：

```text
三地 App 全活
≠
数据库全活
```

修正：

- 明确定义 Single-Primary。
- 两地 async replica。
- Continuous WAL archive。
- 人工 promotion Runbook。
- 明确 RPO/RTO。

残余风险：

Primary region 故障时仍需要人工决策。

未来解决：

```text
Primary Region 内额外增加 DB HA nodes
+
Region 间 Async DR
```

---

### Review 2：不能把 Replica 当 Backup

问题：

逻辑删除会同步。

修正：

```text
pgBackRest + Offsite S3 + WAL + PITR + Restore Drill
```

上线门槛：

**没有完成一次真实恢复，不允许把备份项标记为完成。**

---

### Review 3：Replication Slot 可能把磁盘写爆

问题：

远端 Standby 断几天：

```text
slot 保留 WAL
→ pg_wal 增长
→ disk 100%
→ Primary crash
```

修正：

- 设置 `max_slot_wal_keep_size`。
- WAL archive。
- `pg_replication_slots` 监控。
- disk 80/90 告警。
- replica disconnected 告警。

---

### Review 4：Failover 最大风险不是“慢”，而是 Split-Brain

问题：

仅因为 FRA 到 SIN 不通就 promote FRA：

```text
SIN 可能还活着
```

会产生两个 writable primary。

修正：

Failover 强制：

```text
Fence old primary
→ Verify
→ Promote
```

旧 Primary 回归时：

```text
pg_rewind
or
fresh basebackup
```

绝不直接恢复写流量。

---

### Review 5：本地 Replica 不能无脑承担所有 GET

问题：

写后立即读到旧数据。

修正：

默认：

```text
strong read → primary
```

只有明确允许 eventual consistency 的 endpoint 才：

```text
local replica
```

---

### Review 6：健康检查不能只判断“进程活着”

问题：

App 进程活着但：

```text
FRA → SIN DB network broken
```

用户所有 API 都失败。

修正：

Cloudflare `/health/edge` 应包含：

```text
core DB connectivity
```

但不要把所有非核心第三方服务纳入，否则小依赖故障会把整个 Region 摘掉。

---

### Review 7：Docker + UFW 是常见误区

问题：

Docker NAT / published ports 的行为可能与普通 UFW 规则预期不同。

修正：

- Caddy 跑 host。
- App 只 publish 到 `127.0.0.1`。
- Valkey 不 publish host port。
- PostgreSQL host 只 listen WireGuard。
- NATS gateway 只绑定 WireGuard。
- 如果未来让 Docker 直接 publish 公网端口，必须单独审查 Docker firewall / `DOCKER-USER` chain。

---

### Review 8：Cloudflare 不等于 Origin 自动安全

问题：

Origin IP 被历史 DNS、日志、邮件系统等泄露后，攻击者可绕过 Cloudflare。

修正：

```text
Origin 443 = Cloudflare IP allowlist only
+
Origin CA
+
Full strict
```

更高安全等级：

```text
Authenticated Origin Pulls / mTLS
```

---

### Review 9：跨洲 NATS 不应该承担永久聊天记录

问题：

Core NATS / 网关短暂异常不应该导致聊天消息永久丢失。

修正：

```text
PostgreSQL message
+
Transactional Outbox
+
NATS realtime fanout
+
Reconnect DB catch-up
```

---

### Review 10：不要做三节点跨洲持久化 quorum

问题：

三台 VPS 分别位于三洲：

```text
WAN latency
packet loss
maintenance
```

都会进入 quorum 路径。

修正：

当前阶段：

```text
NATS Gateway = realtime federation
DB = durable source of truth
```

未来如果需要 JetStream/Kafka：

```text
同一 Region 内 3 个低 RTT nodes
```

---

### Review 11：发布与 Migration 必须解耦

问题：

6 个实例同时迁移或 destructive DDL 会造成版本不兼容。

修正：

```text
Expand
→ Deploy
→ Backfill
→ Contract
```

Migration job 单实例执行。

---

### Review 12：本地 Cache 不能承担安全关键全局限流

问题：

攻击者可从不同 Region 获得多份 quota。

修正：

安全关键接口：

```text
Cloudflare
+
global durable state
+
local limiter
```

三层组合。

---

### Review 13：监控不能和故障域完全重合

问题：

三台机器都不可达时，内部监控也不可达。

修正：

必须有：

```text
external synthetic monitor
```

推荐独立 SaaS 或第四台小监控节点。

---

### Review 14：跨洲 DB 延迟需要在代码层控制

问题：

一个业务请求内大量 SQL round-trip 会把 WAN RTT 放大。

修正：

- 连接池。
- 减少事务 round trip。
- 消灭 N+1。
- 一次请求尽量少事务。
- 允许陈旧的读逐渐下沉 replica/cache。
- 后期按 Home Region 分片。

---

### Review 15：社交媒体上传必须防止对象存储变成攻击入口

问题：

Presigned URL 不做限制会被滥用。

修正：

- TTL。
- size。
- MIME。
- owner prefix。
- checksum。
- scan。
- decode。
- transcode。
- moderation。
- private ACL。

---

### Review 16：数据驻留/隐私可能改变数据库架构

问题：

如果 EU 用户数据有严格的数据驻留、跨境传输或客户合同要求：

```text
EU App → Singapore Primary
```

可能不仅是性能问题，也是合规设计问题。

修正：

上线前完成：

- 数据分类。
- 数据存储位置清单。
- backup location。
- log location。
- subprocessors。
- retention。
- deletion workflow。

如果要求 EU 数据必须留在 EU，则需要：

```text
regional data ownership / sharding
```

而不是继续沿用一个全球 Primary。

该部分应结合适用法律、合同与专业合规意见。

---


### Review 17：Standby “能复制”不等于“能立刻当 Primary”

问题：

初稿如果只给初始 SIN 配置：

```text
listen_addresses
pg_hba
archive
```

FRA/SJC 虽然可以 replay WAL，但 Promote 后可能不能被 App 或另一个 Standby 连接。

最终修正：

```text
三台都预配置自身 WG listen address
三台都允许 replication peers
三台都允许 App WG 网段
三台都预装 pgBackRest
```

---

### Review 18：UFW 默认 deny 会同时挡住 wg0 内层流量

问题：

只开放公网 51820 只说明 WireGuard tunnel 可以建立，不代表 tunnel 内的 5432 一定能访问。

最终修正：

```bash
ufw allow in on wg0 from 10.77.0.0/24 to any port 5432 proto tcp
```

NATS Gateway 等其它内部服务也按端口精确授权。

---

### Review 19：Failover 后必须立即重挂“另一个幸存 Standby”

问题：

Promote FRA 后，SJC 的 `primary_conninfo` 仍然指向旧 SIN。

如果只改 App：

```text
业务恢复
但数据库冗余已消失
```

最终修正：

```text
新 Primary 创建 SJC slot
→ SJC primary_conninfo 改到 FRA
→ SJC restart
→ 验证 streaming
→ 再重建旧 SIN
```

---

### Review 20：备份必须跟随 Primary 角色，而不是跟随机器名

问题：

只在 SIN 配置 cron：

```text
SIN down
→ FRA promoted
→ 数据库继续写
→ 定时 backup 永久留在 SIN
```

最终修正：

三台都配置：

```text
pgBackRest
archive_command
role-aware backup script
```

脚本只有 `pg_is_in_recovery() = false` 的当前 Primary 才执行。

---

### Review 21：Cloudflare HTTPS Monitor 必须显式 Host/SNI

问题：

Endpoint 是 raw IP 时，默认：

```text
Host: raw-ip
```

可能导致 Caddy virtual host 或 TLS name mismatch。

最终修正：

```text
HTTPS
Host: api.example.com
Path: /health/edge
```

并在真实 LB Preview/Event Log 中验证 monitor。

---

### Review 22：Geo Steering 的跨区 Failover 不能靠想当然

问题：

只为 Europe 指定 FRA，然后希望它自动按“最近”去 SJC/SIN，不够明确。

最终修正：

为每个 geographic group 写完整有序列表：

```text
EU   = FRA → SJC → SIN
NA   = SJC → FRA → SIN
Asia = SIN → FRA → SJC
```

然后真实 disable endpoint 做演练。

---

### Review 23：Docker 发布端口与 UFW 的组合必须单独审计

问题：

Docker 官方明确说明 published port 流量可能绕过 UFW 常规路径。

最终修正：

```text
Caddy 在 host
App 仅 bind 127.0.0.1
Valkey 不 publish
DB 不在 Docker
```

未来需要公网 publish 时使用 Docker-aware firewall 设计并测试。

---

### Review 24：PITR 必须显式目标动作与隔离恢复环境

问题：

PITR 不是“跑一个 restore 命令”就完成。

最终修正：

```text
独立临时环境
--type=time
--target=<UTC>
--target-action=promote
业务一致性检查
记录实际 RTO
```

恢复机不得误连生产业务流量。

---

## 27. 最终上线 Checklist

### Infrastructure

- [ ] 三台 OS 版本统一。
- [ ] UTC 时间正确。
- [ ] SSH root 禁止。
- [ ] Password SSH 禁止。
- [ ] deploy key 已备份。
- [ ] UFW 默认 deny。
- [ ] 5432 不在公网。
- [ ] 6379 不在公网。
- [ ] 4222 不在公网。
- [ ] WireGuard 三地互通。
- [ ] `wg0` 上 5432 防火墙规则已验证。
- [ ] Docker published ports 已确认不会意外暴露公网。

### Application

- [ ] App stateless。
- [ ] 至少 2 local instances 或明确接受单实例短暂中断。
- [ ] `/livez` 正确。
- [ ] `/health/edge` 正确。
- [ ] `/version` 包含 SHA + Region。
- [ ] Client timeout 合理。
- [ ] 重试仅用于幂等操作。
- [ ] Idempotency-Key 已覆盖重要写入。
- [ ] JWT/refresh token 可撤销。

### Database

- [ ] Single Primary 已确认。
- [ ] `pg_hba.conf` 只允许必要网段。
- [ ] app_user 不是 superuser。
- [ ] 两个 Standby 正常。
- [ ] FRA/SJC/SIN 都提前配置自身 WG `listen_addresses`。
- [ ] 三台 `pg_hba.conf` 都支持未来 Promote 后的 App/replication 连接。
- [ ] replication lag monitor。
- [ ] replication slot monitor。
- [ ] `max_slot_wal_keep_size` 已根据磁盘评估。
- [ ] failover Runbook 已演练。
- [ ] old-primary fencing 已理解。
- [ ] `pg_rewind`/reseed 流程已测试。

### Backup

- [ ] S3/offsite repository。
- [ ] WAL archive。
- [ ] Full backup。
- [ ] Diff/Incr schedule。
- [ ] Backup failure alert。
- [ ] 三台均有 role-aware backup 脚本，只有当前 Primary 执行。
- [ ] Failover 后新 Primary 的 WAL archive 已单独验证。
- [ ] PITR restore 成功。
- [ ] Backup encryption key 独立保存。

### Edge

- [ ] Cloudflare proxy enabled。
- [ ] Full (strict)。
- [ ] Origin certificate installed。
- [ ] 3 pools。
- [ ] Health monitor。
- [ ] HTTPS Monitor `Host: api.example.com` / SNI 已验证。
- [ ] 每个 Geo 的 ordered fallback pools 已显式配置并演练。
- [ ] Geo/Dynamic steering。
- [ ] Fallback order。
- [ ] WebSocket enabled。
- [ ] WAF/rate-limit policy。
- [ ] Origin 443 Cloudflare-only。
- [ ] `CF-Connecting-IP` 仅在可信 origin path 中使用。

### Media

- [ ] VPS 不保存永久用户媒体。
- [ ] Object Storage private-by-default。
- [ ] Presigned upload TTL。
- [ ] Upload size limit。
- [ ] MIME validation。
- [ ] Private media authorization。
- [ ] Cleanup orphan objects。
- [ ] Media processing retry。

### CI/CD

- [ ] Build once。
- [ ] Immutable image。
- [ ] Digest recorded。
- [ ] Region-by-region deploy。
- [ ] Drain before deploy。
- [ ] Rollback tested。
- [ ] Migration single-runner。
- [ ] Expand/Contract migration policy。
- [ ] Production approval gate。

### Observability

- [ ] External uptime。
- [ ] 5xx alert。
- [ ] latency alert。
- [ ] disk alert。
- [ ] DB connection alert。
- [ ] replication alert。
- [ ] backup alert。
- [ ] centralized logs。
- [ ] request ID。
- [ ] error tracking。

### Disaster Recovery

- [ ] Kill App drill。
- [ ] Kill Region drill。
- [ ] WG partition drill。
- [ ] DB promotion drill。
- [ ] PITR drill。
- [ ] Runbook 联系人/权限清单。
- [ ] Provider console access 已测试。

---

## 28. 后续扩展路线

不要一开始就建设最终形态。

### Stage 1 — 当前

```text
3 VPS
Cloudflare
Single DB Primary
2 async replicas
Object Storage
Docker Compose
```

这就是本文方案。

### Stage 2 — 用户增长

加入：

- PgBouncer。
- local read replicas。
- external observability。
- queue/outbox worker。
- NATS gateway。
- dedicated DB VPS。
- dedicated worker nodes。

### Stage 3 — 单 Region 多机

例如 Frankfurt：

```text
fra-app-01
fra-app-02
fra-db-01
fra-db-02
fra-db-03
fra-worker-01
```

此时才值得：

```text
Kubernetes per region
```

不是全球一个 cluster。

### Stage 4 — Regional Data Ownership

用户分配：

```text
user.home_region
```

例如：

```text
EU   → FRA
NA   → SJC
APAC → SIN
```

写入本地 DB shard。

跨 Region 采用：

- event replication。
- feed fanout。
- global directory。
- async materialized views。

这是从：

```text
全球单主
```

走向：

```text
真正多区域数据架构
```

的阶段。

不要在没有真实用户量、延迟指标和一致性需求之前提前支付这部分复杂度。

---

## 29. 官方参考资料

以下文档用于核对本文关键实现方式。上线时仍应以厂商最新文档为准。

### Cloudflare

- Load Balancing Geo Steering  
  https://developers.cloudflare.com/load-balancing/understand-basics/traffic-steering/steering-policies/geo-steering/

- Load Balancing Traffic Steering  
  https://developers.cloudflare.com/load-balancing/understand-basics/traffic-steering/

- Dynamic Steering  
  https://developers.cloudflare.com/load-balancing/understand-basics/traffic-steering/steering-policies/dynamic-steering/

- Load Balancing Monitors  
  https://developers.cloudflare.com/load-balancing/monitors/

- WebSockets  
  https://developers.cloudflare.com/network/websockets/

- Cloudflare IP Addresses / Origin Allowlisting  
  https://developers.cloudflare.com/fundamentals/concepts/cloudflare-ip-addresses/

- Protect Your Origin Server  
  https://developers.cloudflare.com/fundamentals/security/protect-your-origin-server/

- Origin CA  
  https://developers.cloudflare.com/ssl/origin-configuration/origin-ca/

- Authenticated Origin Pulls  
  https://developers.cloudflare.com/ssl/origin-configuration/authenticated-origin-pull/explanation/

### PostgreSQL

- PostgreSQL 18 Replication Settings  
  https://www.postgresql.org/docs/18/runtime-config-replication.html

- PostgreSQL Failover  
  https://www.postgresql.org/docs/18/warm-standby-failover.html

- `pg_basebackup`  
  https://www.postgresql.org/docs/18/app-pgbasebackup.html

- `pg_rewind`  
  https://www.postgresql.org/docs/18/app-pgrewind.html

- `pg_replication_slots`  
  https://www.postgresql.org/docs/18/view-pg-replication-slots.html

- PostgreSQL Ubuntu / PGDG packages  
  https://www.postgresql.org/download/linux/ubuntu/

### Backup

- pgBackRest Configuration  
  https://pgbackrest.org/configuration.html

### WireGuard

- WireGuard Quick Start  
  https://www.wireguard.com/quickstart/

### Caddy

- Caddy `reverse_proxy`  
  https://caddyserver.com/docs/caddyfile/directives/reverse_proxy

- Caddy TLS  
  https://caddyserver.com/docs/caddyfile/directives/tls

### NATS

- NATS Documentation  
  https://docs.nats.io/

---


### 深度 Review 额外核对

- PostgreSQL 18 Replication Configuration (`max_slot_wal_keep_size`)  
  https://www.postgresql.org/docs/18/runtime-config-replication.html

- PostgreSQL 18 Failover / STONITH  
  https://www.postgresql.org/docs/18/warm-standby-failover.html

- PostgreSQL `pg_rewind`  
  https://www.postgresql.org/docs/18/app-pgrewind.html

- PostgreSQL Continuous Archiving on Standby  
  https://www.postgresql.org/docs/18/warm-standby.html

- Cloudflare Load Balancing Monitors / Host Header  
  https://developers.cloudflare.com/load-balancing/monitors/

- Cloudflare Manage Monitors  
  https://developers.cloudflare.com/load-balancing/monitors/create-monitor/

- Cloudflare Geo Steering  
  https://developers.cloudflare.com/load-balancing/understand-basics/traffic-steering/steering-policies/geo-steering/

- Cloudflare Adaptive Routing / Failover Across Pools  
  https://developers.cloudflare.com/load-balancing/understand-basics/adaptive-routing/

- Docker Packet Filtering / UFW caveat  
  https://docs.docker.com/engine/network/packet-filtering-firewalls/

- pgBackRest Current User Guide  
  https://pgbackrest.org/user-guide.html

## 结论

对只有 Frankfurt、San Jose、Singapore 三台 VPS 的社交 App，当前最合理的生产落地方案不是追求“所有东西都三地强一致”，而是把系统拆成不同一致性层：

```text
入口
→ 全球多 Region Active-Active

应用
→ Stateless

缓存
→ Region Local / Disposable

实时通知
→ Eventual / Recoverable

数据库写入
→ Single Primary / Strong

数据库容灾
→ Async Replica + WAL + PITR

媒体
→ Object Storage + CDN
```

第一阶段最重要的不是“架构看起来像大厂”，而是：

```text
知道数据在哪里
知道请求怎么走
知道服务器挂了怎么办
知道数据库挂了怎么办
知道备份能不能恢复
知道发布失败怎么回滚
```

当以上每项都已经被真实演练过，你才真正拥有了一套可以正式上线的生产系统。
