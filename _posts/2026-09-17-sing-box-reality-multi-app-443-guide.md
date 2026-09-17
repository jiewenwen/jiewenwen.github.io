---
layout: post
title: "单公网 IP 下 TCP 443 端口复用方案"
date: 2026-09-17 00:00:00 +0800
categories: [Development]
tags: [sing-box, vless, reality, HAProxy, Nginx, SNI, TCP 443, 端口复用, Let's Encrypt, vps]
---

只有一台 VPS、一个公网 IP，却要同时运行 sing-box 的 `VLESS + Vision + Reality` 和多个 Web/API App，又不想让任何客户端改端口——这是一道很典型的“单端口复用”题。本文给出一套按顺序可直接落地的方案：**Reality 客户端继续连接公网 `IP:443`，每个 App 继续使用标准 `https://域名`，而公网 TCP 443 只由 HAProxy 独占，再按外层可见的 SNI 分流给 Nginx 或 sing-box。**

**适用系统**：Debian / Ubuntu 系 Linux VPS（systemd）
**核心组件**：HAProxy（L4/SNI 分流）+ Nginx（Web TLS 终止与多 App 反代）+ sing-box（VLESS + Vision + Reality）
**前提假设**：已有一套工作正常的 Reality 服务端与客户端（例如《{% post_url 2025-11-29-sing-box-vless-reality %}》中的配置）；本文只改造入口，不修改 UUID、密钥等任何客户端参数。

---

## 第一部分：结论与关键原理

### 1. 先读结论

本文解决下面这个典型问题：

- 只有一台 VPS、一个公网 IPv4（可选再有 IPv6）；
- sing-box 已经使用 `VLESS + xtls-rprx-vision + Reality`；
- Reality 客户端当前直接连接 `服务器公网 IP:443`；
- 同一台 VPS 还要部署多个 Web/API App；
- 每个 App 都希望用户继续通过标准 HTTPS 访问，例如：

```text
https://api.example.com
https://admin.example.com
https://files.example.com
```

最终采用：

```text
                         TCP 443
Internet ────────────────────────────────────┐
                                             ▼
                                          HAProxy
                                   （只看 ClientHello/SNI）
                                             │
                       ┌─────────────────────┴─────────────────────┐
                       │                                           │
             已知 App SNI                                  其他/未知/无 SNI
                       │                                           │
                       ▼                                           ▼
             127.0.0.1:8443                              127.0.0.1:24443
                    Nginx                                      sing-box
                       │                                           │
          ┌────────────┼────────────┐                   VLESS + Vision + Reality
          ▼            ▼            ▼
       App 1         App 2         App 3
       :8001         :8002         :8003
```

客户端侧不需要把 Reality 改成 `24443`：

```text
Reality 客户端：公网IP:443
App 1：        https://api.example.com
App 2：        https://admin.example.com
App 3：        https://files.example.com
```

`24443`、`8443`、`8001` 等全部只是服务器内部端口。

---

### 2. 关键原理

#### 2.1 为什么多个 App 可以共用 443

Web 服务通常不是每个 App 都直接监听公网 443，而是：

```text
公网 :443
   ↓
反向代理 / TLS 入口
   ├─ api.example.com   → App 1 :8001
   ├─ admin.example.com → App 2 :8002
   └─ files.example.com → App 3 :8003
```

公网只有一个 443，但 Nginx 可以根据 HTTPS SNI / HTTP Host 为多个域名提供服务。

#### 2.2 Reality 为什么使问题复杂

原来：

```text
公网 TCP :443 → sing-box Reality
```

如果 Nginx 也尝试：

```text
公网 TCP :443 → Nginx
```

同一个 `IP + TCP + 443` 会发生端口冲突。

所以必须让一个入口进程独占公网 TCP 443，再把不同连接分流。

#### 2.3 HAProxy 在这里不终止 TLS

HAProxy 工作在 `mode tcp`：

- 不持有 App 证书；
- 不解密 Web TLS；
- 不终止 Reality TLS；
- 只暂存最开始的 TLS ClientHello；
- 读取可见的 SNI；
- 判断连接送 Nginx 还是 sing-box；
- 然后原始 TCP/TLS 数据继续透传。

因此 App 的 TLS 仍由 Nginx 完成，Reality 的 TLS/Reality 握手仍由 sing-box 完成。

---

### 3. 方案成立的必要条件

#### 3.1 App 必须用域名访问

本方案用 SNI 判断 Web 流量。

正常：

```text
https://api.example.com
```

不建议期待下面这种方式进入 Web：

```text
https://203.0.113.10
```

客户端直接用 IP 做 HTTPS 时可能没有可用于匹配 App 的域名 SNI，因此会命中 HAProxy 的默认 Reality 后端。

**结论：App/API 应使用域名，不要把公网 IP 当成正式 HTTPS API 地址。**

#### 3.2 Reality 的可见 SNI 不能与 App SNI 相同

例如 App 域名：

```text
api.example.com
admin.example.com
files.example.com
```

那么 Reality 客户端用于 TLS 的 `server_name` 不要是这些域名。

正常示例：

```text
Reality server       = 203.0.113.10
Reality server_port  = 443
Reality server_name  = www.some-other-domain.example
```

如果 Reality 客户端 `server_name` 是 IP，通常不会发送该 IP 作为 SNI；无 SNI 会命中默认 Reality 后端，也可以工作。

如果出现：

```text
Reality server_name = api.example.com
```

同时 `api.example.com` 被配置成 Web SNI，则该 Reality 连接会被 HAProxy 送给 Nginx，导致 Reality 失败。

#### 3.3 App ClientHello 的 SNI 必须在外层可见

方案依赖 HAProxy `req.ssl_sni` 对第一个 TLS ClientHello 做内容检查。

以下场景要特别注意：

- ECH（Encrypted Client Hello）；
- 某些非常规 ClientHello 构造/分片方式；
- 客户端完全不发送 SNI。

对普通 HTTPS 客户端，一般没有问题。

**如果你的 App 域名启用了 ECH，本方案不应依赖普通 `req.ssl_sni` 做可靠域名路由。** HAProxy 看到的可能只是 Outer ClientHello 的外层名称。

#### 3.4 本文默认 HAProxy 与 Nginx 安装在宿主机

主路径假设：

```text
HAProxy → 宿主机 systemd 服务
Nginx   → 宿主机 systemd 服务
sing-box→ 宿主机 systemd 服务（Docker 变体后文另讲）
App     → 可宿主机，也可 Docker
```

如果 HAProxy/Nginx/sing-box 全部运行在 Docker/Kubernetes，网络命名空间与回环地址关系不同，不要原样照抄 `127.0.0.1`。

#### 3.5 Reality 握手目标不要无意回指本机

本文假设 Reality 当前的 `handshake.server` 已经正常工作。

上线前建议确认：

```bash
getent ahosts YOUR_REALITY_HANDSHAKE_SERVER
```

如果 `handshake.server` 恰好解析到本 VPS 的公网 IP，可能形成复杂回流路径。除非这是你有意设计且已验证的方案，否则建议先搞清楚现有 Reality 握手路径再切换。

---

### 4. 一个必须接受的副作用：sing-box 看不到原始 TCP 源 IP

这是本架构中最容易被忽略的一点。

HAProxy 是普通 TCP 代理时：

```text
真实客户端 198.51.100.20
       ↓
HAProxy
       ↓ 新建后端 TCP 连接
127.0.0.1:24443
       ↓
sing-box
```

因此 sing-box 通常会把 TCP 对端看成：

```text
127.0.0.1
```

而不是：

```text
198.51.100.20
```

#### 4.1 对 Reality 功能本身是否有影响

通常 **VLESS/Reality 身份认证本身不依赖真实 TCP 源 IP**，所以正常连接一般不受影响。

#### 4.2 哪些功能会受影响

如果你的 sing-box 配置依赖：

```text
source_ip_cidr
source_ip_is_private
source_port
按客户端源 IP 做日志统计/封禁/限流
```

这些行为需要重新评估。

#### 4.3 为什么不直接给 Reality 加 PROXY Protocol

HAProxy 可以给支持 PROXY Protocol 的后端添加头部，但这会在原协议字节前插入额外数据。

本文不能假设你的 VLESS + Reality inbound 可以直接接受这种额外前缀，所以 Reality 后端配置：

```haproxy
server singbox_local 127.0.0.1:24443
```

**不要加：**

```text
send-proxy
send-proxy-v2
```

如果你强依赖 Reality 的真实客户端源 IP，需要另行设计透明代理/TPROXY、独立公网 IP 或其他保源方案；本文不提供未经你环境验证的透明代理命令。

#### 4.4 Web App 的真实 IP 可以保留

Web 分支使用：

```text
HAProxy --PROXY v2--> Nginx
```

Nginx 再恢复 `$remote_addr`，因此 App 可以得到真实客户端 IP。

---

### 5. 示例参数

下面所有值都要替换。

| 用途 | 示例值 |
|---|---|
| VPS 公网 IPv4 | `203.0.113.10` |
| VPS 公网 IPv6（可选） | `2001:db8::10` |
| App 1 域名 | `api.example.com` |
| App 2 域名 | `admin.example.com` |
| App 3 域名 | `files.example.com` |
| sing-box Reality 内部监听 | `127.0.0.1:24443` |
| Nginx HTTPS 内部监听 | `127.0.0.1:8443` |
| App 1 | `127.0.0.1:8001` |
| App 2 | `127.0.0.1:8002` |
| App 3 | `127.0.0.1:8003` |

`203.0.113.0/24`、`2001:db8::/32` 和 `example.com` 是文档示例，不要照抄到生产环境。

---

## 第二部分：部署前体检

### 6. 确认当前 sing-box 实际配置文件

不要默认一定是 `/etc/sing-box/config.json`。

先看 systemd：

```bash
sudo systemctl cat sing-box
```

关注 `ExecStart=`。

再看状态：

```bash
sudo systemctl status sing-box --no-pager
```

如果配置文件确实是：

```text
/etc/sing-box/config.json
```

后文命令可以直接使用；否则全部替换成真实路径。

---

### 7. 记录当前监听状态

```bash
sudo ss -lntp
sudo ss -lnup
```

重点看：

```bash
sudo ss -lntp | grep -E ':(80|443|8443|24443|8001|8002|8003)\b' || true
sudo ss -lnup | grep ':443\b' || true
```

切换前通常会看到：

```text
TCP *:443 → sing-box
```

#### 7.1 TCP 443 与 UDP 443 是不同端口空间

本文共享的是：

```text
TCP 443
```

所以：

```text
TCP 443 → HAProxy
UDP 443 → 理论上仍可由其他程序监听
```

数字相同但协议不同不会产生同一个 socket 冲突。

不过本文基础方案**不启用 HTTP/3/QUIC**，因此不需要开放 UDP 443。

---

### 8. 备份

```bash
sudo mkdir -p /root/reality-multiapp-backup

sudo cp -a /etc/sing-box \
  /root/reality-multiapp-backup/sing-box-$(date +%F-%H%M%S)

if [ -d /etc/nginx ]; then
  sudo cp -a /etc/nginx \
    /root/reality-multiapp-backup/nginx-$(date +%F-%H%M%S)
fi

if [ -f /etc/haproxy/haproxy.cfg ]; then
  sudo cp -a /etc/haproxy/haproxy.cfg \
    /root/reality-multiapp-backup/haproxy-$(date +%F-%H%M%S).cfg
fi
```

也建议保存当前 sing-box 版本：

```bash
sing-box version
```

---

### 9. DNS 体检：A、AAAA、CAA

安装 `dig`（如果没有）：

```bash
sudo apt update
sudo apt install -y dnsutils
```

检查：

```bash
dig +short A api.example.com
dig +short AAAA api.example.com
dig +short CAA api.example.com


dig +short A admin.example.com
dig +short AAAA admin.example.com


dig +short A files.example.com
dig +short AAAA files.example.com
```

#### 9.1 IPv4

每个 App 至少应有：

```text
api.example.com    A    203.0.113.10
admin.example.com  A    203.0.113.10
files.example.com  A    203.0.113.10
```

#### 9.2 如果不打算支持 IPv6

**不要保留错误或历史遗留的 AAAA。**

如果域名存在：

```text
AAAA → 某个旧 IPv6 / 其他服务器
```

要么修正，要么删除。

#### 9.3 如果要支持 IPv6

必须同时满足：

```text
AAAA → 本 VPS 正确公网 IPv6
VPS 确实有该 IPv6
80/tcp IPv6 可达
443/tcp IPv6 可达
HAProxy 接受 IPv6 443
```

检查本机：

```bash
ip -6 addr show scope global
ip -6 route
```

#### 9.4 为什么 AAAA 对证书很重要

Let’s Encrypt 在域名同时有 `A` 与 `AAAA` 时，会优先尝试 IPv6 做验证。

所以即使：

```text
IPv4 网站完全正常
```

只要 AAAA 错误，也可能导致 HTTP-01 签发失败。

#### 9.5 CAA

如果 DNS 配置了 CAA，确认没有禁止你使用的 CA。

如果 Certbot 返回 CAA 相关错误，先处理 DNS，不要反复重试申请证书。

---

### 10. 部署阶段建议暂时不要套 CDN/反向代理

本文主流程假设：

```text
App 域名 DNS → 直接指向该 VPS
```

如果使用 Cloudflare 等 CDN，部署阶段建议先使用 DNS-only/直连方式完成验证。

原因：

- 排查 SNI 更直接；
- 排查证书更直接；
- HAProxy 看到的源 IP、Nginx 的真实 IP 链会因为 CDN 改变；
- CDN/ECH/Origin 访问控制属于额外一层设计。

部署稳定后再决定是否加 CDN。

---

### 11. 检查 Reality SNI 与 App 域名冲突

列出 App：

```text
api.example.com
admin.example.com
files.example.com
```

再查看 Reality 客户端配置中的：

```text
server
server_port
server_name
```

必须保证：

```text
Reality 可见 server_name ∉ App 域名集合
```

如果 `server_name` 是 IP 或客户端明确禁用了 SNI，则会走 HAProxy 默认 Reality 后端。

---

### 12. 检查 App 自身监听方式

推荐：

```text
App 1 → 127.0.0.1:8001
App 2 → 127.0.0.1:8002
App 3 → 127.0.0.1:8003
```

检查：

```bash
sudo ss -lntp | grep -E ':(8001|8002|8003)\b' || true
```

不推荐直接：

```text
0.0.0.0:8001
0.0.0.0:8002
0.0.0.0:8003
```

否则可能绕过 Nginx 和 HTTPS 直接访问 App。

---

### 13. 防火墙规划

公网通常只需要：

```text
SSH 端口/tcp
80/tcp
443/tcp
```

**注意：如果 SSH 不是 22，请使用你自己的 SSH 端口。**

例如 UFW 已经启用且 SSH 是 22：

```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw status verbose
```

不要对公网放行：

```text
24443/tcp
8443/tcp
8001/tcp
8002/tcp
8003/tcp
```

基础方案也不需要：

```text
443/udp
```

云厂商 Security Group / Firewall 也要同步检查。

**不要在未确认 SSH 放行前贸然执行 `ufw enable`。**

---

## 第三部分：入口组件与 Web 证书

### 14. 安装 HAProxy、Nginx、Certbot 与测试工具

Debian/Ubuntu：

```bash
sudo apt update
sudo apt install -y \
  haproxy \
  nginx \
  certbot \
  curl \
  openssl \
  dnsutils
```

检查：

```bash
haproxy -v
nginx -v
certbot --version
openssl version
```

检查 Nginx realip 模块：

```bash
nginx -V 2>&1 | grep -- --with-http_realip_module
```

本文 Nginx Web 分支需要 realip 模块恢复 HAProxy 传来的源 IP。

> 说明：本文使用发行版 Certbot 包完成 `webroot` 和自动续期。不要同时混用多个不同来源的 Certbot 安装，避免 timer/配置混乱。

---

### 15. 配置 Nginx TCP 80

这一阶段 Reality 继续占用 TCP 443，不动。

创建 ACME Webroot：

```bash
sudo mkdir -p /var/www/acme/.well-known/acme-challenge
sudo chmod 755 /var/www/acme
sudo chmod 755 /var/www/acme/.well-known
sudo chmod 755 /var/www/acme/.well-known/acme-challenge
```

删除发行版默认站点软链接：

```bash
sudo rm -f /etc/nginx/sites-enabled/default
```

创建：

```bash
sudo nano /etc/nginx/sites-available/apps-http
```

#### 15.1 IPv4-only 版本

```nginx
server {
    listen 80 default_server;
    server_name _;
    return 444;
}

server {
    listen 80;

    server_name
        api.example.com
        admin.example.com
        files.example.com;

    location ^~ /.well-known/acme-challenge/ {
        root /var/www/acme;
        default_type text/plain;
        try_files $uri =404;
    }

    # 443 切换完成前，先不要跳转 HTTPS。
    location / {
        return 404;
    }
}
```

#### 15.2 如果域名发布了 AAAA

在两个 server 中分别加上：

```nginx
listen [::]:80 default_server;
```

和：

```nginx
listen [::]:80;
```

最终要确保 IPv6 的 80 端口真的能从公网访问。

#### 15.3 启用

```bash
sudo ln -sfn \
  /etc/nginx/sites-available/apps-http \
  /etc/nginx/sites-enabled/apps-http

sudo nginx -t
sudo systemctl enable nginx
sudo systemctl restart nginx
```

确认：

```bash
sudo ss -lntp | grep ':80\b'
```

---

### 16. 在申请证书前手工测试 ACME Webroot

```bash
echo 'acme-ok' | \
  sudo tee /var/www/acme/.well-known/acme-challenge/test.txt
```

从**服务器外部**测试 IPv4：

```bash
curl -4 http://api.example.com/.well-known/acme-challenge/test.txt
curl -4 http://admin.example.com/.well-known/acme-challenge/test.txt
curl -4 http://files.example.com/.well-known/acme-challenge/test.txt
```

应返回：

```text
acme-ok
```

如果有 AAAA，还要从具备 IPv6 的外部网络测试：

```bash
curl -6 http://api.example.com/.well-known/acme-challenge/test.txt
```

如果 `curl -4` 成功但 `curl -6` 失败，而 DNS 存在 AAAA：

**先修 IPv6/AAAA，不要继续 Certbot。**

---

### 17. 申请证书

建议每个独立 App 域名单独一个 lineage，新增/删除最清晰。

把邮箱替换成你的地址。

#### App 1

```bash
sudo certbot certonly --webroot \
  -w /var/www/acme \
  --cert-name api.example.com \
  --email YOUR_EMAIL@example.com \
  --agree-tos \
  --non-interactive \
  -d api.example.com
```

#### App 2

```bash
sudo certbot certonly --webroot \
  -w /var/www/acme \
  --cert-name admin.example.com \
  --email YOUR_EMAIL@example.com \
  --agree-tos \
  --non-interactive \
  -d admin.example.com
```

#### App 3

```bash
sudo certbot certonly --webroot \
  -w /var/www/acme \
  --cert-name files.example.com \
  --email YOUR_EMAIL@example.com \
  --agree-tos \
  --non-interactive \
  -d files.example.com
```

查看：

```bash
sudo certbot certificates
```

不要只凭猜测写证书路径。

正常首次 lineage 一般是：

```text
/etc/letsencrypt/live/api.example.com/fullchain.pem
/etc/letsencrypt/live/api.example.com/privkey.pem
```

如果你以前申请过同名证书，实际目录可能不同；以：

```bash
sudo certbot certificates
```

输出为准。

#### 17.1 通配符证书

`HTTP-01` 不能申请 `*.example.com` 通配符证书。

如果你需要 wildcard，需要使用 DNS-01，属于另一套 ACME 流程。

---

## 第四部分：Nginx 内部 HTTPS 与 App 反代

### 18. 增加一个“失败关闭”的默认 8443 TLS 虚拟主机

创建：

```bash
sudo nano /etc/nginx/sites-available/00-reject-8443
```

内容：

```nginx
server {
    listen 127.0.0.1:8443 ssl proxy_protocol default_server;
    server_name _;

    # 8443 只允许本机 HAProxy 连接。
    set_real_ip_from 127.0.0.1;
    real_ip_header proxy_protocol;

    # 未匹配到明确 App server_name 时直接拒绝 TLS 握手，
    # 不返回其他 App 的默认/错误证书。
    ssl_reject_handshake on;
}
```

启用：

```bash
sudo ln -sfn \
  /etc/nginx/sites-available/00-reject-8443 \
  /etc/nginx/sites-enabled/00-reject-8443
```

`ssl_reject_handshake` 需要 Nginx 1.19.4+。当前受支持 Debian/Ubuntu 通常满足；最终以 `nginx -t` 为准。

---

### 19. App 1：api.example.com

```bash
sudo nano /etc/nginx/sites-available/api.example.com
```

```nginx
server {
    listen 127.0.0.1:8443 ssl proxy_protocol;
    server_name api.example.com;

    ssl_certificate     /etc/letsencrypt/live/api.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.example.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;

    # 只信任从本机 HAProxy 来的 PROXY Protocol。
    set_real_ip_from 127.0.0.1;
    real_ip_header proxy_protocol;

    access_log /var/log/nginx/api.example.com.access.log;
    error_log  /var/log/nginx/api.example.com.error.log;

    location / {
        proxy_pass http://127.0.0.1:8001;
        proxy_http_version 1.1;

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;

        # 故意覆盖客户端自己伪造的 X-Forwarded-For；
        # HAProxy 已经通过 PROXY v2 提供真实 TCP 源地址。
        proxy_set_header X-Forwarded-For   $remote_addr;

        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-Host  $host;
        proxy_set_header X-Forwarded-Port  443;

        # 普通 HTTP 后端不需要 Upgrade 时可清空 Connection。
        proxy_set_header Connection "";

        proxy_connect_timeout 10s;
        proxy_read_timeout 60s;
        proxy_send_timeout 60s;
    }
}
```

启用：

```bash
sudo ln -sfn \
  /etc/nginx/sites-available/api.example.com \
  /etc/nginx/sites-enabled/api.example.com
```

---

### 20. App 2：admin.example.com

```bash
sudo nano /etc/nginx/sites-available/admin.example.com
```

```nginx
server {
    listen 127.0.0.1:8443 ssl proxy_protocol;
    server_name admin.example.com;

    ssl_certificate     /etc/letsencrypt/live/admin.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/admin.example.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;

    set_real_ip_from 127.0.0.1;
    real_ip_header proxy_protocol;

    access_log /var/log/nginx/admin.example.com.access.log;
    error_log  /var/log/nginx/admin.example.com.error.log;

    location / {
        proxy_pass http://127.0.0.1:8002;
        proxy_http_version 1.1;

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $remote_addr;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-Host  $host;
        proxy_set_header X-Forwarded-Port  443;
        proxy_set_header Connection        "";

        proxy_connect_timeout 10s;
        proxy_read_timeout 60s;
        proxy_send_timeout 60s;
    }
}
```

启用：

```bash
sudo ln -sfn \
  /etc/nginx/sites-available/admin.example.com \
  /etc/nginx/sites-enabled/admin.example.com
```

---

### 21. App 3：files.example.com

```bash
sudo nano /etc/nginx/sites-available/files.example.com
```

```nginx
server {
    listen 127.0.0.1:8443 ssl proxy_protocol;
    server_name files.example.com;

    ssl_certificate     /etc/letsencrypt/live/files.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/files.example.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;

    set_real_ip_from 127.0.0.1;
    real_ip_header proxy_protocol;

    access_log /var/log/nginx/files.example.com.access.log;
    error_log  /var/log/nginx/files.example.com.error.log;

    # 按你的业务调整。
    client_max_body_size 100m;

    location / {
        proxy_pass http://127.0.0.1:8003;
        proxy_http_version 1.1;

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $remote_addr;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-Host  $host;
        proxy_set_header X-Forwarded-Port  443;
        proxy_set_header Connection        "";

        proxy_connect_timeout 10s;
        proxy_read_timeout 60s;
        proxy_send_timeout 60s;
    }
}
```

启用：

```bash
sudo ln -sfn \
  /etc/nginx/sites-available/files.example.com \
  /etc/nginx/sites-enabled/files.example.com
```

---

### 22. 检查 Nginx

```bash
sudo nginx -t
sudo systemctl reload nginx
sudo ss -lntp | grep ':8443\b'
```

应看到：

```text
127.0.0.1:8443 → nginx
```

#### 22.1 为什么直接 curl 8443 会失败

下面这种普通请求：

```bash
curl -k https://127.0.0.1:8443/
```

预期可能失败，并出现类似：

```text
broken header while reading PROXY protocol
```

这是正常的。

因为 `8443` 要求连接起始处先收到 HAProxy 的 PROXY Protocol v2 头，然后才是 TLS ClientHello。

---

## 第五部分：HAProxy SNI 分流

### 23. 创建 App 域名白名单

```bash
sudo mkdir -p /etc/haproxy
sudo nano /etc/haproxy/app-domains.lst
```

内容：

```text
api.example.com
admin.example.com
files.example.com
```

#### 23.1 规则原则

这个文件只放真正的 Web/App HTTPS 域名。

**不要放：**

```text
Reality 的伪装 server_name
Reality handshake.server
无关域名
```

#### 23.2 不要把 `*.example.com` 当通配符直接写进列表

本文使用的是精确 SNI 列表匹配。

不要以为：

```text
*.example.com
```

会自动代表所有子域名。

生产上显式列出实际 App hostname 最安全，也最不容易误把 Reality SNI 路由到 Web。

---

### 24. 配置 HAProxy

备份：

```bash
sudo cp -a /etc/haproxy/haproxy.cfg \
  /etc/haproxy/haproxy.cfg.bak-$(date +%F-%H%M%S)
```

编辑：

```bash
sudo nano /etc/haproxy/haproxy.cfg
```

#### 24.1 IPv4-only 推荐配置

```haproxy
global
    log /dev/log local0
    log /dev/log local1 notice
    user haproxy
    group haproxy
    daemon

defaults
    log global
    mode tcp
    option tcplog
    option dontlognull

    timeout connect 5s

    # Reality 可能存在长连接；这里选择较宽松的 TCP 空闲超时。
    # 如果你确定业务不需要，可按实际场景缩短。
    timeout client 24h
    timeout server 24h

frontend public_tls_443
    bind 0.0.0.0:443
    mode tcp

    # 等待完整 TLS ClientHello；最多等待 5 秒。
    tcp-request inspect-delay 5s
    tcp-request content accept if { req.ssl_hello_type 1 }

    # 只有明确列出的 App SNI 才进入 Web。
    acl is_web_app req.ssl_sni -i -f /etc/haproxy/app-domains.lst
    use_backend nginx_https if is_web_app

    # 未知 SNI、无 SNI、非 TLS/无法识别的流量默认交给 Reality。
    default_backend singbox_reality

backend nginx_https
    mode tcp

    # Web 分支使用 PROXY v2 保留客户端 IP。
    server nginx_local 127.0.0.1:8443 send-proxy-v2

backend singbox_reality
    mode tcp

    # Reality 分支不要添加 PROXY Protocol。
    server singbox_local 127.0.0.1:24443
```

#### 24.2 如果域名有 AAAA：使用 IPv4 + IPv6 双栈监听

把：

```haproxy
bind 0.0.0.0:443
```

替换为：

```haproxy
bind [::]:443 v4v6
```

HAProxy 官方文档说明该形式可以在支持的平台上让一个 listener 接受 IPv4 与 IPv6。

不要同时盲目添加多个会互相覆盖的 dual-stack bind；以：

```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
```

和最终 `ss` 输出为准。

#### 24.3 检查 HAProxy

```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
```

必须成功。

此时**不要启动/重启 HAProxy**，因为 sing-box 仍然占用公网 TCP 443。

---

### 25. 关于 HAProxy `inspect-delay`

```haproxy
tcp-request inspect-delay 5s
tcp-request content accept if { req.ssl_hello_type 1 }
```

意义：

- TCP 建立后暂时缓存初始数据；
- 一旦完整 TLS ClientHello 可解析就立即继续；
- 并不是每个正常 TLS 连接固定多等 5 秒；
- `5s` 是最大等待窗口。

对于只连接端口却迟迟不发数据的扫描连接，这个窗口会占用一定资源。

如果服务器高并发且受扫描严重，可以在充分测试后缩短，但不要为了几毫秒优化导致慢网络 ClientHello 尚未完整就误走默认 Reality。

---

## 第六部分：迁移 sing-box 并正式切换 TCP 443

### 26. 修改 Reality inbound

原配置可能类似：

```json
{
  "type": "vless",
  "tag": "vless-reality-in",
  "listen": "::",
  "listen_port": 443,
  "users": [
    {
      "uuid": "你的UUID",
      "flow": "xtls-rprx-vision"
    }
  ],
  "tls": {
    "enabled": true,
    "reality": {
      "enabled": true,
      "handshake": {
        "server": "你的Reality握手目标",
        "server_port": 443
      },
      "private_key": "你的私钥",
      "short_id": [
        "你的short-id"
      ]
    }
  }
}
```

只把监听修改为：

```json
"listen": "127.0.0.1",
"listen_port": 24443
```

即：

```json
{
  "type": "vless",
  "tag": "vless-reality-in",
  "listen": "127.0.0.1",
  "listen_port": 24443,
  "users": [
    {
      "uuid": "你的UUID",
      "flow": "xtls-rprx-vision"
    }
  ],
  "tls": {
    "enabled": true,
    "reality": {
      "enabled": true,
      "handshake": {
        "server": "你的Reality握手目标",
        "server_port": 443
      },
      "private_key": "你的私钥",
      "short_id": [
        "你的short-id"
      ]
    }
  }
}
```

原则上不要改：

```text
UUID
flow
private_key
short_id
handshake.server
handshake.server_port
客户端 public_key
客户端 short_id
客户端 server_name
```

本文只是把服务端监听从公网 443 移到本机 24443。

---

### 27. 检查 sing-box 配置

如果配置文件是 `/etc/sing-box/config.json`：

```bash
sudo sing-box check -c /etc/sing-box/config.json
```

如果不是，替换为真实路径。

**不要跳过这一步。**

---

### 28. 切换前最后检查

三个配置都必须先通过：

```bash
sudo nginx -t
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo sing-box check -c /etc/sing-box/config.json
```

并确认 App 本身可访问：

```bash
curl -v http://127.0.0.1:8001/
curl -v http://127.0.0.1:8002/
curl -v http://127.0.0.1:8003/
```

应用本身如果有 `/health`，优先测健康检查 endpoint。

---

### 29. 重启 sing-box，使公网 443 空出来

```bash
sudo systemctl restart sing-box
sudo systemctl status sing-box --no-pager
```

检查：

```bash
sudo ss -lntp | grep ':24443\b'
```

应看到：

```text
127.0.0.1:24443 → sing-box
```

再检查 TCP 443：

```bash
sudo ss -lntp | grep ':443\b' || true
```

此时应暂时没有 TCP 443 listener。

如果仍然是 sing-box，说明：

- 修改的不是 systemd 实际读取的配置；或
- 有第二个 sing-box 进程/容器；或
- 还有其他 inbound 占 443。

不要继续启动 HAProxy，先排查。

---

### 30. 启动 HAProxy

```bash
sudo systemctl enable haproxy
sudo systemctl restart haproxy
sudo systemctl status haproxy --no-pager
```

确认：

```bash
sudo ss -lntp | grep ':443\b'
```

IPv4-only 常见输出：

```text
0.0.0.0:443 → haproxy
```

双栈配置常见输出可能是：

```text
[::]:443 → haproxy
```

只要实际同时能接预期 IPv4/IPv6 即可。

---

### 31. 确保全部服务开机自启

```bash
sudo systemctl enable nginx
sudo systemctl enable haproxy
sudo systemctl enable sing-box
```

查看：

```bash
systemctl is-enabled nginx haproxy sing-box
```

---

### 32. 使用 308，而不是默认写 301

编辑：

```bash
sudo nano /etc/nginx/sites-available/apps-http
```

把：

```nginx
location / {
    return 404;
}
```

改为：

```nginx
location / {
    return 308 https://$host$request_uri;
}
```

保留 ACME location 在前：

```nginx
location ^~ /.well-known/acme-challenge/ {
    root /var/www/acme;
    default_type text/plain;
    try_files $uri =404;
}
```

完整逻辑：

```text
/.well-known/acme-challenge/* → 不重定向，直接在 80 提供文件
其他 HTTP 请求              → 308 到 HTTPS
```

为什么用 308：

- `GET` 正常；
- `POST` 保持 `POST`；
- `PUT` 保持 `PUT`；
- 请求体保持不变；
- 更适合 API。

应用：

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## 第七部分：完整验证

### 33. 检查所有监听端口

```bash
sudo ss -lntp | grep -E ':(80|443|8443|24443|8001|8002|8003)\b' || true
sudo ss -lnup | grep ':443\b' || true
```

IPv4-only 的目标状态类似：

```text
0.0.0.0:80       → nginx
0.0.0.0:443      → haproxy
127.0.0.1:8443   → nginx
127.0.0.1:24443  → sing-box
127.0.0.1:8001   → App 1
127.0.0.1:8002   → App 2
127.0.0.1:8003   → App 3
```

如果 App 是 Docker，端口进程显示可能是 Docker 代理/容器相关进程。

---

### 34. 本机绕过 DNS 测 App SNI 分流

即使公网 DNS 还在传播，也可以本机走 HAProxy：

```bash
curl -vk \
  --resolve api.example.com:443:127.0.0.1 \
  https://api.example.com/
```

同理：

```bash
curl -vk \
  --resolve admin.example.com:443:127.0.0.1 \
  https://admin.example.com/

curl -vk \
  --resolve files.example.com:443:127.0.0.1 \
  https://files.example.com/
```

如果证书是正式有效证书，可以去掉 `-k`：

```bash
curl -v \
  --resolve api.example.com:443:127.0.0.1 \
  https://api.example.com/
```

---

### 35. 从外部机器绕过 DNS 测公网路径

```bash
curl -v \
  --resolve api.example.com:443:203.0.113.10 \
  https://api.example.com/
```

然后测实际 DNS：

```bash
curl -v https://api.example.com/
```

---

### 36. OpenSSL 检查证书/SNI

```bash
openssl s_client \
  -connect 203.0.113.10:443 \
  -servername api.example.com \
  </dev/null
```

应该看到 `api.example.com` 对应的 Web 证书链。

也可：

```bash
openssl s_client \
  -connect 127.0.0.1:443 \
  -servername api.example.com \
  -verify_hostname api.example.com \
  </dev/null
```

---

### 37. 测试 HTTP → HTTPS 是否保留 API 方法

可以先检查响应码：

```bash
curl -I http://api.example.com/
```

预期：

```text
HTTP/... 308 Permanent Redirect
Location: https://api.example.com/...
```

如果你的 API 有无副作用测试 endpoint，可以再验证 POST 重定向行为；不要对生产写接口随便发送测试数据。

---

### 38. 检查 Web 真实客户端 IP

从外部访问：

```bash
curl https://api.example.com/
```

服务器：

```bash
sudo tail -n 20 /var/log/nginx/api.example.com.access.log
```

Nginx `$remote_addr` 应是外部客户端公网 IP，而不是：

```text
127.0.0.1
```

如果 App 有打印请求 header，也应收到：

```text
X-Real-IP: <真实客户端IP>
X-Forwarded-For: <真实客户端IP>
X-Forwarded-Proto: https
X-Forwarded-Port: 443
```

---

### 39. 测 Reality

客户端保持原来：

```text
server       = 原公网 IP
server_port  = 443
uuid         = 不变
flow         = xtls-rprx-vision
public_key   = 不变
short_id     = 不变
server_name  = 原值
```

**不要把客户端端口改成 24443。**

连接后检查：

```bash
sudo journalctl -u sing-box -n 100 --no-pager
```

以及实际访问外网验证 Reality 流量。

#### 39.1 注意 sing-box 日志中的 peer/source IP

经过 HAProxy 后，如果 sing-box 日志记录 TCP 来源，通常会看到 HAProxy/回环地址。

这是架构预期，不代表 Reality 连接异常。

---

### 40. IPv6 完整验证（仅发布 AAAA 时）

从外部 IPv6 网络：

```bash
curl -6 -v https://api.example.com/
```

DNS：

```bash
dig +short AAAA api.example.com
```

服务器监听：

```bash
sudo ss -lntp | grep ':443\b'
```

如果发布了 AAAA，就不要只测 `curl -4`。

---

## 第八部分：证书自动续期

### 41. 创建 Nginx reload deploy hook

```bash
sudo mkdir -p /etc/letsencrypt/renewal-hooks/deploy
```

```bash
sudo tee /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh >/dev/null <<'SCRIPT'
#!/bin/sh
set -eu
nginx -t
systemctl reload nginx
SCRIPT
```

```bash
sudo chmod 755 \
  /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
```

HAProxy 不终止 Web TLS，也不加载这些 App 证书，所以 App 证书更新后通常只需 reload Nginx。

---

### 42. 检查 Certbot timer

```bash
systemctl list-timers --all | grep -i certbot || true
systemctl status certbot.timer --no-pager || true
```

不同发行版/安装来源可能用 timer 或 cron。

---

### 43. 测试续期

基本测试：

```bash
sudo certbot renew --dry-run
```

如果当前 Certbot 支持，可同时测试 deploy hook：

```bash
sudo certbot renew --dry-run --run-deploy-hooks
```

如果你的发行版 Certbot 不认识 `--run-deploy-hooks`，执行基本 dry-run 后，再手工验证：

```bash
sudo /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
```

---

## 第九部分：新增 App、协议兼容、Docker 与 CDN

### 44. 新增 new.example.com → 127.0.0.1:8004

必须按顺序操作，**最后一步才加入 HAProxy SNI 列表**。

#### 44.1 DNS

```text
new.example.com A 203.0.113.10
```

如果支持 IPv6：

```text
new.example.com AAAA 你的正确公网IPv6
```

#### 44.2 确认 App

```bash
curl -v http://127.0.0.1:8004/
```

#### 44.3 把 new.example.com 加入 Nginx 80 的 server_name

编辑：

```bash
sudo nano /etc/nginx/sites-available/apps-http
```

加入：

```text
new.example.com
```

然后：

```bash
sudo nginx -t
sudo systemctl reload nginx
```

#### 44.4 ACME 测试

```bash
curl http://new.example.com/.well-known/acme-challenge/test.txt
```

#### 44.5 申请证书

```bash
sudo certbot certonly --webroot \
  -w /var/www/acme \
  --cert-name new.example.com \
  --email YOUR_EMAIL@example.com \
  --agree-tos \
  --non-interactive \
  -d new.example.com
```

#### 44.6 创建 Nginx HTTPS 站点

```nginx
server {
    listen 127.0.0.1:8443 ssl proxy_protocol;
    server_name new.example.com;

    ssl_certificate     /etc/letsencrypt/live/new.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/new.example.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;

    set_real_ip_from 127.0.0.1;
    real_ip_header proxy_protocol;

    location / {
        proxy_pass http://127.0.0.1:8004;
        proxy_http_version 1.1;

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $remote_addr;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-Host  $host;
        proxy_set_header X-Forwarded-Port  443;
        proxy_set_header Connection        "";
    }
}
```

检查：

```bash
sudo nginx -t
sudo systemctl reload nginx
```

#### 44.7 最后才加入 HAProxy

```bash
echo 'new.example.com' | \
  sudo tee -a /etc/haproxy/app-domains.lst
```

检查是否重复：

```bash
sort /etc/haproxy/app-domains.lst | uniq -d
```

检查配置：

```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
```

reload：

```bash
sudo systemctl reload haproxy
```

#### 44.8 验证

```bash
curl -v https://new.example.com/
```

#### 为什么 HAProxy 一定最后改

如果你先把域名加入 HAProxy，而 Nginx 站点/证书还没准备好，该域名流量已经会被送给 Nginx。

本文虽然加了 `ssl_reject_handshake` 默认站点以失败关闭，但正确运维顺序仍应是：

```text
DNS
→ App
→ 80/ACME
→ 证书
→ Nginx 8443
→ nginx -t + reload
→ HAProxy 域名列表
→ haproxy -c + reload
```

---

### 45. WebSocket

如果某个 App 使用 WebSocket，不要继续使用普通站点中的：

```nginx
proxy_set_header Connection "";
```

WebSocket location 应显式传 Upgrade。

推荐在 `/etc/nginx/conf.d/websocket-map.conf`：

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}
```

然后 WebSocket location：

```nginx
location /ws/ {
    proxy_pass http://127.0.0.1:8001;
    proxy_http_version 1.1;

    proxy_set_header Host              $host;
    proxy_set_header X-Real-IP         $remote_addr;
    proxy_set_header X-Forwarded-For   $remote_addr;
    proxy_set_header X-Forwarded-Proto https;

    proxy_set_header Upgrade           $http_upgrade;
    proxy_set_header Connection        $connection_upgrade;

    proxy_read_timeout 1h;
}
```

检查：

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

### 46. SSE / 长轮询 / 流式响应

SSE 可能需要：

```nginx
proxy_buffering off;
proxy_read_timeout 1h;
```

具体以 App 框架为准。

否则 Nginx 默认缓冲/超时可能表现为：

- 消息迟迟不推送；
- 60 秒左右被断开；
- 流式响应被缓存。

---

### 47. gRPC

gRPC 不应简单照抄普通：

```nginx
proxy_pass http://127.0.0.1:xxxx;
```

通常需要 Nginx 的 `grpc_pass`，以及正确的 HTTP/2 配置。

HAProxy 这一层是 TLS passthrough，因此不会破坏 ALPN；真正是否支持 HTTP/2/gRPC 取决于 Nginx TLS listener 与后端配置。

如果你的 App 是 gRPC，请为该 App 单独写 Nginx gRPC server/location。

---

### 48. HTTP/2

基础配置即使只协商 HTTP/1.1 也能正常提供 HTTPS API。

如果希望启用 HTTP/2，要根据你的 Nginx 版本使用对应语法。

不同 Nginx 版本存在：

```text
listen ... http2
```

与较新：

```text
http2 on;
```

的差异。

不要为了开启 HTTP/2 直接复制不兼容当前 Nginx 版本的语法；先：

```bash
nginx -v
nginx -t
```

HAProxy `mode tcp` 不终止 TLS，所以客户端的 ALPN 会原样到达 Nginx。

---

### 49. HTTP/3 / QUIC / UDP 443

HTTP/3 使用 QUIC，也就是 UDP。

本文基础架构：

```text
TCP 443 → HAProxy
```

并没有处理：

```text
UDP 443
```

所以基础方案请不要主动向客户端广告 h3/QUIC。

技术上：

```text
TCP 443 和 UDP 443 可以由不同程序同时监听
```

但如果真的要给 Nginx 开公网 UDP 443/HTTP3，需要额外设计：

- UDP 防火墙；
- Nginx QUIC 支持；
- 多域名证书；
- HTTP/3 Alt-Svc；
- IPv6；
- 绕过 HAProxy 后的访问控制/真实 IP。

本文不把它混入基础部署。

---

### 50. App 在 Docker 中

推荐只发布到宿主机回环：

```yaml
services:
  api:
    image: your-api-image
    ports:
      - "127.0.0.1:8001:8080"

  admin:
    image: your-admin-image
    ports:
      - "127.0.0.1:8002:8080"

  files:
    image: your-files-image
    ports:
      - "127.0.0.1:8003:8080"
```

不推荐：

```yaml
ports:
  - "8001:8080"
```

因为通常会绑定宿主机所有地址。

部署后一定实际检查：

```bash
sudo ss -lntp | grep ':8001\b'
```

---

### 51. sing-box 在 Docker 中

如果 HAProxy 在宿主机，而 sing-box 在 Docker，不能把容器自己的 `127.0.0.1` 当宿主机回环。

一种简单方式：

```yaml
services:
  sing-box:
    image: your-sing-box-image
    ports:
      - "127.0.0.1:24443:24443"
```

容器内 sing-box：

```json
"listen": "0.0.0.0",
"listen_port": 24443
```

宿主机 HAProxy 仍然：

```haproxy
server singbox_local 127.0.0.1:24443
```

这样 24443 只发布在宿主机回环，不直接暴露公网。

如果你采用 Docker 私有 bridge 网络让 HAProxy 也进容器，则应改用容器服务名/IP，而不是本文宿主机 `127.0.0.1` 模式。

---

### 52. 如果 App 以后接 Cloudflare/CDN

此时访问链可能变成：

```text
真实用户
  ↓
CDN
  ↓
HAProxy
  ↓
Nginx
  ↓
App
```

那么 HAProxy PROXY v2 能保存的“真实 TCP 源 IP”通常是：

```text
CDN Edge IP
```

而不是最终用户 IP。

最终用户 IP 可能在 CDN 专用 HTTP Header 中。

此时不能简单相信来自任意公网客户端的：

```text
CF-Connecting-IP
X-Forwarded-For
```

否则可伪造。

正确 CDN 方案通常还需要：

- 源站只允许 CDN IP 段；
- 明确信任 CDN 的真实 IP header；
- 定期更新 CDN 网段；
- 考虑 ECH 与 SNI 分流关系；
- 重新设计 Nginx real-ip 链。

**本文主配置针对客户端直连 VPS 的 App HTTPS。**

---

## 第十部分：安全加固

### 53. 最小暴露原则

最终应是：

```text
公网：
SSH/tcp
80/tcp
443/tcp

回环：
127.0.0.1:24443 sing-box
127.0.0.1:8443  Nginx HTTPS
127.0.0.1:800x  Apps
```

---

### 54. 不要泄露 Reality 私密参数

不要公开：

```text
private_key
UUID
short_id
其他认证凭据
```

`public_key` 本身是给客户端使用的公钥，但完整节点配置仍不建议公开。

---

### 55. App 域名列表采用白名单

HAProxy 配置核心安全/稳定原则：

```text
明确 App SNI → Web
其他全部     → Reality
```

不要反过来写成“大量猜测 Reality 域名、其余都 Web”，否则扫描/未知 SNI 更容易误进 Web 默认站点。

---

### 56. Nginx 8443 必须只监听回环

正确：

```nginx
listen 127.0.0.1:8443 ssl proxy_protocol;
```

不要：

```nginx
listen 0.0.0.0:8443 ssl proxy_protocol;
```

否则外部用户可能直接访问一个必须携带 PROXY Protocol 的端口，并扩大攻击面。

---

### 57. `set_real_ip_from` 不要信任全网

本文：

```nginx
set_real_ip_from 127.0.0.1;
real_ip_header proxy_protocol;
```

这是因为只有本机 HAProxy 应该能连接 8443。

不要为了“省事”写：

```nginx
set_real_ip_from 0.0.0.0/0;
```

否则如果 8443 被意外暴露，攻击者可以伪造 PROXY 源地址。

---

### 58. HSTS 要最后开

如果你之后要加：

```text
Strict-Transport-Security
```

先确保：

- 所有子域名 HTTPS 都长期稳定；
- 没有仍需要 HTTP 的子域名；
- 证书自动续期已经验证。

不要在刚切换时就盲目加 `includeSubDomains; preload`。

---

## 第十一部分：故障排查与运行维护

### 59. HAProxy：`Address already in use`

```bash
sudo ss -lntp | grep ':443\b'
```

如果仍看到：

```text
sing-box → *:443
```

说明 sing-box 没有真正切到 24443。

继续查：

```bash
sudo systemctl cat sing-box
sudo systemctl status sing-box --no-pager
ps aux | grep '[s]ing-box'
```

如果 sing-box 是 Docker：

```bash
docker ps --format 'table {{.Names}}\t{{.Ports}}'
```

---

### 60. App 域名被送到 Reality

检查：

```bash
cat /etc/haproxy/app-domains.lst
```

必须精确包含：

```text
api.example.com
```

检查：

```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl reload haproxy
```

再：

```bash
curl -vk \
  --resolve api.example.com:443:127.0.0.1 \
  https://api.example.com/
```

---

### 61. Reality 失败但 App 正常

第一优先检查：

```text
Reality client server_name
```

是否等于 App 域名。

例如：

```text
Reality server_name = api.example.com
```

且：

```text
api.example.com ∈ app-domains.lst
```

那么 HAProxy 会把 Reality 送到 Nginx。

再检查：

```bash
sudo ss -lntp | grep ':24443\b'
sudo journalctl -u sing-box -n 200 --no-pager
```

---

### 62. Nginx：`broken header while reading PROXY protocol`

意味着 Nginx 8443 收到的连接开头不是合法 PROXY Protocol。

正确组合：

HAProxy：

```haproxy
server nginx_local 127.0.0.1:8443 send-proxy-v2
```

Nginx：

```nginx
listen 127.0.0.1:8443 ssl proxy_protocol;
```

如果有人直接：

```bash
curl https://127.0.0.1:8443
```

出现该错误属于预期。

---

### 63. Nginx 502 Bad Gateway

先绕过 Nginx 访问 App：

```bash
curl -v http://127.0.0.1:8001/
```

检查：

```bash
sudo tail -n 100 \
  /var/log/nginx/api.example.com.error.log
```

常见：

- App 未启动；
- App 监听了另一个地址/端口；
- Docker 端口未发布；
- App 实际只支持 HTTPS，而 Nginx 用了 `http://`；
- Unix socket 权限错误。

---

### 64. Certbot HTTP-01 失败

检查：

```bash
sudo ss -lntp | grep ':80\b'
sudo nginx -t
sudo ufw status verbose

dig +short A api.example.com
dig +short AAAA api.example.com
dig +short CAA api.example.com
```

外部测试：

```bash
curl -4 http://api.example.com/.well-known/acme-challenge/test.txt
curl -6 http://api.example.com/.well-known/acme-challenge/test.txt
```

如果有 AAAA，尤其关注 IPv6。

HTTP-01 本身使用 TCP 80。

---

### 65. App HTTPS IPv4 正常、部分用户失败

很常见原因：DNS 有 AAAA，但 443 只监听 IPv4。

检查：

```bash
dig +short AAAA api.example.com
sudo ss -lntp | grep ':443\b'
```

如果有 AAAA，HAProxy 应使用双栈 listener，例如：

```haproxy
bind [::]:443 v4v6
```

并确保云防火墙允许 IPv6 TCP 443。

---

### 66. 访问 `https://公网IP` 却不是 App

这是预期。

没有已知 App SNI 时：

```text
HAProxy → default_backend singbox_reality
```

正式 Web 应使用：

```text
https://api.example.com
```

而不是公网 IP。

---

### 67. ECH 开启后 App 突然被送 Reality

本架构根据外层 ClientHello SNI 分流。

ECH 下 HAProxy 可能只能看到 Outer ClientHello 的外层/诱饵名称。

如果你控制的 App 域名开始发布 ECH 配置：

- 停止依赖普通 SNI 作为唯一 L4 路由依据；或
- 关闭该 App 的 ECH；或
- 使用独立公网 IP / 独立入口；或
- 重新设计边界代理。

不要把 ECH 路由异常当作 Nginx 证书问题排查半天。

---

### 68. WebSocket 约 60 秒断开

检查：

```nginx
proxy_read_timeout
```

以及 WebSocket ping。

例如：

```nginx
proxy_read_timeout 1h;
```

同时确认 Upgrade/Connection headers 已配置。

---

### 69. HAProxy 日志在哪里

进程启动错误通常：

```bash
sudo journalctl -u haproxy -n 200 --no-pager
```

实时：

```bash
sudo journalctl -u haproxy -f
```

但具体 `tcplog` 是否进入 journal、`/var/log/haproxy.log` 或 syslog 其他文件，取决于发行版 rsyslog/journald 配置。

可查：

```bash
sudo grep -R "haproxy" /etc/rsyslog.conf /etc/rsyslog.d 2>/dev/null || true
```

---

### 70. 常用日志

HAProxy：

```bash
sudo journalctl -u haproxy -f
```

Nginx：

```bash
sudo journalctl -u nginx -f
sudo tail -f /var/log/nginx/api.example.com.access.log
sudo tail -f /var/log/nginx/api.example.com.error.log
```

sing-box：

```bash
sudo journalctl -u sing-box --output cat -f
```

---

### 71. 一键拓扑检查

```bash
echo '=== TCP LISTEN ==='
sudo ss -lntp | grep -E ':(80|443|8443|24443|8001|8002|8003)\b' || true

echo
echo '=== UDP 443 ==='
sudo ss -lnup | grep ':443\b' || true

echo
echo '=== HAProxy config ==='
sudo haproxy -c -f /etc/haproxy/haproxy.cfg

echo
echo '=== Nginx config ==='
sudo nginx -t

echo
echo '=== sing-box config ==='
sudo sing-box check -c /etc/sing-box/config.json

echo
echo '=== Enabled ==='
systemctl is-enabled haproxy nginx sing-box || true

echo
echo '=== Active ==='
systemctl is-active haproxy nginx sing-box || true
```

如果你的 sing-box 配置文件不是 `/etc/sing-box/config.json`，记得改最后的命令。

---

### 72. 重启 VPS 后必须再验证一次

完成部署后，维护窗口允许时建议执行一次真实 reboot 验证开机顺序：

```bash
sudo reboot
```

重新登录后：

```bash
systemctl is-active haproxy nginx sing-box
sudo ss -lntp | grep -E ':(80|443|8443|24443)\b'
```

然后实际测试：

```bash
curl https://api.example.com/
```

并测试 Reality 客户端。

如果当前 VPS 不能接受重启风险，可以先不 reboot，但上线验收里应保留“重启后恢复”测试项。

---

## 第十二部分：回滚、验收与速查

### 73. 快速恢复 sing-box 独占 TCP 443

如果切换后 Reality 有严重问题：

#### 73.1 停 HAProxy

```bash
sudo systemctl stop haproxy
```

#### 73.2 把 sing-box 改回公网 443

例如：

```json
"listen": "::",
"listen_port": 443
```

或直接恢复备份配置。

如果你以前是：

```json
"listen": "0.0.0.0"
```

就恢复你**原来的值**，不要机械照抄 `::`。

#### 73.3 检查并重启

```bash
sudo sing-box check -c /etc/sing-box/config.json
sudo systemctl restart sing-box
```

确认：

```bash
sudo ss -lntp | grep ':443\b'
```

应重新看到 sing-box 占 TCP 443。

#### 73.4 是否需要停 Nginx

不需要为了 Reality 回滚而删除：

```text
127.0.0.1:8443
127.0.0.1:800x
```

它们不会与公网 TCP 443 冲突。

但此时 App HTTPS 会无法从公网 443 访问，这是回滚到“Reality 优先”的预期结果。

#### 73.5 避免 reboot 后 HAProxy 又抢回 443

如果决定长期回滚：

```bash
sudo systemctl disable haproxy
```

否则下次重启可能再次与 sing-box 竞争 443。

---

### 74. 上线验收清单（必须全部打勾）

#### DNS

- [ ] `A` 指向正确 VPS IPv4。
- [ ] 没有错误 AAAA。
- [ ] 如果有 AAAA，VPS/80/443/HAProxy IPv6 都已验证。
- [ ] CAA 不阻止证书 CA。
- [ ] App 域名没有无意经过未知 CDN/代理。

#### Reality

- [ ] 客户端仍然连接 `公网IP:443`。
- [ ] 客户端 Reality `server_name` 不等于 App 域名。
- [ ] sing-box 服务端只监听 `127.0.0.1:24443`。
- [ ] Reality 真实连接测试成功。
- [ ] 已接受/处理 sing-box 看不到原始客户端 TCP 源 IP 的影响。
- [ ] 配置中若有 `source_ip_cidr` 等源 IP 规则，已重新评估。

#### HAProxy

- [ ] 公网 TCP 443 只由 HAProxy 监听。
- [ ] `haproxy -c` 通过。
- [ ] App hostname 精确存在于 `app-domains.lst`。
- [ ] Reality 的 SNI/伪装域名不在 `app-domains.lst`。
- [ ] Web backend 有 `send-proxy-v2`。
- [ ] Reality backend 没有 `send-proxy*`。
- [ ] 如果有 AAAA，HAProxy 使用 IPv4+IPv6 listener。

#### Nginx

- [ ] 80/tcp 对 ACME 可达。
- [ ] `/.well-known/acme-challenge/` 不被 HTTPS 重定向覆盖。
- [ ] 其他 HTTP 使用 308 跳 HTTPS。
- [ ] 8443 只监听 `127.0.0.1`。
- [ ] 8443 使用 `proxy_protocol`。
- [ ] `set_real_ip_from` 只信任 127.0.0.1。
- [ ] 默认 8443 TLS 虚拟主机使用 `ssl_reject_handshake`。
- [ ] `nginx -t` 通过。

#### Certificates

- [ ] `certbot certificates` 正常。
- [ ] 实际证书路径与 Nginx 一致。
- [ ] `certbot renew --dry-run` 成功。
- [ ] deploy hook 能 reload Nginx。

#### Apps

- [ ] App 不直接对公网暴露 800x。
- [ ] 每个 App 本地 `curl 127.0.0.1:800x` 正常。
- [ ] 每个 App 公网 HTTPS 正常。
- [ ] App 看到正确客户端 IP。
- [ ] WebSocket/SSE/gRPC 如有使用已单独验证。

#### Firewall

- [ ] SSH 端口允许。
- [ ] TCP 80 允许。
- [ ] TCP 443 允许。
- [ ] 24443/8443/800x 不对公网开放。
- [ ] 没有无意开放 UDP 443。

---

### 75. 最终拓扑

```text
公网
│
├── TCP 80
│      ↓
│    Nginx
│      ├── /.well-known/acme-challenge/* → ACME webroot
│      └── 其他 → 308 HTTPS
│
└── TCP 443
       ↓
     HAProxy
       │
       ├── SNI 精确命中 /etc/haproxy/app-domains.lst
       │       ↓
       │     PROXY Protocol v2
       │       ↓
       │     127.0.0.1:8443
       │       ↓
       │     Nginx TLS
       │       ├── api.example.com   → 127.0.0.1:8001
       │       ├── admin.example.com → 127.0.0.1:8002
       │       └── files.example.com → 127.0.0.1:8003
       │
       └── 未知 SNI / 无 SNI / 默认流量
               ↓
             普通 TCP 透传，不加 PROXY 头
               ↓
             127.0.0.1:24443
               ↓
             sing-box
             VLESS + Vision + Reality
```

#### 三句话记住

```text
1. Reality 客户端继续连接：公网 IP:443。
2. 所有 App 用户继续访问：https://各自域名（默认 TCP 443）。
3. 公网 TCP 443 只由 HAProxy 监听；HAProxy 再按外层可见 SNI 分给 Nginx 或 sing-box。
```

再加一句重要副作用：

```text
4. Web 可通过 PROXY v2 恢复真实客户端 IP；Reality 经 HAProxy 普通 TCP 代理后，sing-box 默认看不到原始 TCP 源 IP。
```

---

### 76. 官方参考

以下文档是本方案关键能力的依据，建议上线前按你实际安装版本再次核对。

#### sing-box

- TLS / Reality / `server_name` / SNI 行为：  
  https://sing-box.sagernet.org/configuration/shared/tls/
- VLESS inbound / `xtls-rprx-vision`：  
  https://sing-box.sagernet.org/configuration/inbound/vless/
- Route Rule / `source_ip_cidr` 等：  
  https://sing-box.sagernet.org/configuration/route/rule/

#### HAProxy

- Configuration Manual，`req.ssl_sni`、`tcp-request inspect-delay`：  
  https://www.haproxy.com/documentation/haproxy-configuration-manual/latest/
- PROXY Protocol / `send-proxy-v2`：  
  https://www.haproxy.com/documentation/haproxy-configuration-tutorials/proxying-essentials/client-ip-preservation/enable-proxy-protocol/
- IPv4/IPv6 bind：  
  https://www.haproxy.com/documentation/haproxy-configuration-tutorials/proxying-essentials/configuration-basics/frontends/

#### Nginx

- HTTP Real IP module：  
  https://nginx.org/en/docs/http/ngx_http_realip_module.html
- SSL module / `ssl_reject_handshake`：  
  https://nginx.org/en/docs/http/ngx_http_ssl_module.html
- Reverse proxy：  
  https://nginx.org/en/docs/http/ngx_http_proxy_module.html
- WebSocket proxying：  
  https://nginx.org/en/docs/http/websocket.html

#### Certbot / Let’s Encrypt

- Certbot Webroot / renewal hooks：  
  https://eff-certbot.readthedocs.io/en/stable/using.html
- Let’s Encrypt Challenge Types：  
  https://letsencrypt.org/docs/challenge-types/
- Let’s Encrypt IPv6 validation：  
  https://letsencrypt.org/docs/ipv6-support/

---

### 77. 最终建议

对“**一个公网 IP + Reality IP:443 直连 + 多个 App HTTPS**”这个需求，这套方案的优点是：

- Reality 客户端地址/端口不改；
- App 仍全部使用标准 HTTPS 443；
- App 数量可以继续扩展；
- TLS 边界清晰；
- HAProxy 不需要持有 Web 私钥；
- Web 真实客户端 IP 可通过 PROXY v2 保留；
- 未知/无 SNI 默认送 Reality，符合你的 Reality IP 直连场景；
- 回滚路径简单。

它最重要的两个边界也必须明确：

1. **依赖外层 ClientHello SNI，所以 ECH 等隐藏实际 SNI 的机制会破坏这种 L4 域名分流。**
2. **Reality 经普通 HAProxy TCP 代理后，sing-box 默认失去真实客户端 TCP 源 IP。**

如果这两个边界对你的业务都可以接受，该方案适合作为一台 VPS 上 Reality 与多个 HTTPS App 共存的长期结构。
