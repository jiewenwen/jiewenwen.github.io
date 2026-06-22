---
title: 使用 sing-box 部署 VLESS + Vision + Reality 节点（服务端 + 多客户端配置）
date: 2025-11-29 12:00:00 +0800
categories: [Network]
tags: [sing-box, vless, reality, vps, proxy, Mihomo, Clash]
---


## **1. 环境要求**

建议：

- **系统**：Debian 12 / Ubuntu 22.04（其他版本也可）
- **架构**：x86_64 / ARM 均支持
- **端口**：开放 443/TCP
- **客户端**：iOS Shadowrocket / Android sing-box / Windows & macOS Mihomo 或 sing-box

无需：

- 域名
- 证书
- 面板（如 s-ui）

Reality 最大优势就是模拟真实网站 TLS 握手，几乎无法被区分。服务端部署完成后，第 7 章会逐一介绍各平台客户端的具体配置。


## **2. 安装 sing-box**

更新系统后执行：

```bash
apt update && apt upgrade -y
bash <(curl -fsSL https://sing-box.app/install.sh)
```

查看版本：

```bash
sing-box version
```

配置文件路径：

```bash
/etc/sing-box/config.json
```

## **3. 生成 Reality 配置所需参数**

Reality 需要：

- UUID（每个用户唯一）
- PrivateKey / PublicKey（Reality 密钥对）
- Short ID（4~8 字节 hex）
- 伪装网站（无需属于你）

### 3.1 生成 UUID

最稳方式：

```bash
sing-box generate uuid
```

或系统自身：

```bash
cat /proc/sys/kernel/random/uuid
```

### 3.2 生成 Reality 密钥对

```bash
sing-box generate reality-keypair
```

输出示例：

```
PrivateKey: y9X2EXAMPLE...
PublicKey: 90dfEXAMPLE...
```

⚠️ PrivateKey = 服务端用，
⚠️ PublicKey = 客户端用

### 3.3 生成 Short ID

```bash
sing-box generate rand --hex 8
```

示例输出：

```
ab12cd34
```

### 3.4 选择伪装网站（Reality 必须）

推荐：

- `www.apple.com`
- `www.microsoft.com`
- `www.cloudflare.com`

不需要 DNS，不需要证书。本教程使用：

```
www.microsoft.com
```

## **4. 编写 sing-box 配置文件（完整可用）**

编辑：

```bash
nano /etc/sing-box/config.json
```

粘贴以下内容：

```json
{
  "log": {
    "level": "info",
    "timestamp": true
  },
  "inbounds": [
    {
      "type": "vless",
      "tag": "vless-in",
      "listen": "::",
      "listen_port": 443,
      "users": [
        {
          "uuid": "REPLACE_UUID",
          "flow": "xtls-rprx-vision"
        }
      ],
      "tls": {
        "enabled": true,
        "server_name": "www.microsoft.com",
        "reality": {
          "enabled": true,
          "handshake": {
            "server": "www.microsoft.com",
            "server_port": 443
          },
          "private_key": "REPLACE_PRIVATE_KEY",
          "short_id": [
            "REPLACE_SHORT_ID"
          ]
        }
      }
    }
  ],
  "outbounds": [
    {
      "type": "direct",
      "tag": "direct"
    }
  ]
}
```

需要替换：

| 字段                  | 示例                                   |
| --------------------- | -------------------------------------- |
| `REPLACE_UUID`        | `5ba3bdcd-1ab1-4f1c-948a-5dfd5c823bd3` |
| `REPLACE_PRIVATE_KEY` | `y9X2EXAMPLE...`                       |
| `REPLACE_SHORT_ID`    | `ab12cd34`                             |

保存：`Ctrl + O`， 
退出：`Ctrl + X`

## **5. 检查与启动服务**

### 5.1 检查 JSON 格式（重要）

```bash
sing-box check -c /etc/sing-box/config.json
```

### 5.2 启动与开机自启

```bash
systemctl enable sing-box
systemctl start sing-box
```

### 5.3 查看状态

```bash
systemctl status sing-box
```

若显示 `active (running)` 即成功。

查看日志：

```bash
journalctl -u sing-box -n 100
```

## **6. 防火墙与安全组配置**

若启用 UFW：

```bash
ufw allow 443/tcp
ufw reload
```

若 VPS 有安全组（阿里、AWS、Oracle 等）：

👉 控制台开放 **TCP 443** 即可。

## **7. 客户端配置**

服务端部署完成后，下面分别介绍各平台客户端的配置方式。

一个 VLESS + Vision + REALITY 节点通常可写成如下 URL，支持该格式的客户端可直接导入：

```text
vless://UUID@你的服务器IP:443?encryption=none&flow=xtls-rprx-vision&security=reality&sni=www.microsoft.com&fp=chrome&pbk=REALITY公钥&sid=short_id&type=tcp#节点名
```

各参数含义见第 8 节的对照表。

### 7.1 Shadowrocket（iOS）

打开 Shadowrocket → 右上角 + → **Type: VLESS**

填写以下内容：

| 项目                  | 值                                          |
| --------------------- | ------------------------------------------- |
| **Address**           | VPS IP（支持 IPv4 或 IPv6）                 |
| **Port**              | 443                                         |
| **UUID**              | 上面生成的 UUID                             |
| **TLS**               | 开启                                        |
| **Flow**              | `xtls-rprx-vision`                          |
| **Server Name / SNI** | `www.microsoft.com`                         |
| **Public Key**        | Reality 的 PublicKey                        |
| **Short ID**          | 你的 Short ID                               |
| **Fingerprint**       | `chrome` 或默认（Shadowrocket 默认为 Safari） |

保存后连接 → Safari 访问：

```
https://ipinfo.io
```

若显示 VPS IP = 成功。

### 7.2 Clash/Mihomo

**注意**：传统老版 Clash 不支持 VLESS + Vision + REALITY。以下配置适用于 **Mihomo / Clash Meta 内核**，可在 Clash Verge Rev、Mihomo Party、Clash Meta for Android、NekoBox 等客户端中使用。

下面是一份最小可用的 Mihomo 配置，替换占位值即可使用：

```yaml
mixed-port: 7890
allow-lan: false
mode: rule
log-level: info
ipv6: true

dns:
  enable: true
  ipv6: true
  default-nameserver:
    - 1.1.1.1
    - 8.8.8.8
  enhanced-mode: fake-ip
  nameserver:
    - https://cloudflare-dns.com/dns-query
    - https://dns.google/dns-query
  proxy-server-nameserver:
    - https://cloudflare-dns.com/dns-query
    - https://dns.google/dns-query
  fake-ip-filter:
    - "*.lan"
    - "*.local"

proxies:
  - name: "VLESS-Reality"
    type: vless
    server: 你的服务器IP或域名
    port: 443
    uuid: 你的UUID
    network: tcp
    udp: true
    tls: true
    flow: xtls-rprx-vision
    packet-encoding: xudp
    servername: www.microsoft.com
    client-fingerprint: chrome
    skip-cert-verify: true
    reality-opts:
      public-key: 你的REALITY公钥
      short-id: 你的short_id

proxy-groups:
  - name: PROXY
    type: select
    proxies:
      - "VLESS-Reality"
      - DIRECT

rules:
  - DOMAIN-SUFFIX,local,DIRECT
  - IP-CIDR,127.0.0.0/8,DIRECT,no-resolve
  - IP-CIDR,10.0.0.0/8,DIRECT,no-resolve
  - IP-CIDR,172.16.0.0/12,DIRECT,no-resolve
  - IP-CIDR,192.168.0.0/16,DIRECT,no-resolve
  - GEOIP,CN,DIRECT
  - MATCH,PROXY
```

需要替换的字段：

| Mihomo 字段 | 填写内容 |
| --- | --- |
| `server` | VPS IP 或域名 |
| `port` | 服务端监听端口 |
| `uuid` | VLESS 用户 ID |
| `servername` | 服务端配置的 `server_name`，本文为 `www.microsoft.com` |
| `reality-opts.public-key` | REALITY 公钥 |
| `reality-opts.short-id` | 服务端 `short_id` 的值 |
| `client-fingerprint` | 常用 `chrome` |
| `skip-cert-verify` | REALITY 场景下设为 `true` |

关于 `network`、`udp` 和 `packet-encoding` 的说明：

- `network: tcp`：VLESS 到服务端的传输层为 TCP。
- `udp: true`：允许代理节点转发 UDP 流量（QUIC、HTTP/3、游戏等）。
- `packet-encoding: xudp`：UDP 包以 XUDP 方式封装后经 VLESS 转发，不改变传输层。

如果只需 TCP 代理，可将 `udp` 改为 `false` 并删除 `packet-encoding: xudp`。

`skip-cert-verify: true` 是 REALITY 节点在 Mihomo 中的常见写法，安全性依赖 `public-key`、`short-id` 与 REALITY 握手校验。普通 TLS 节点不要照抄此值。

### 7.3 sing-box 本地端口模式（Windows/macOS/Linux）

以下配置在 `127.0.0.1:7890` 监听 HTTP/SOCKS 混合端口，浏览器或系统代理指向该端口即可。配置按 sing-box 1.12+ 风格编写：DNS 使用新格式，国内规则使用远程 `rule_set`。

```json
{
  "log": {
    "level": "info",
    "timestamp": true
  },
  "dns": {
    "servers": [
      {
        "tag": "cloudflare",
        "type": "tls",
        "server": "1.1.1.1"
      },
      {
        "tag": "google",
        "type": "tls",
        "server": "8.8.8.8"
      }
    ],
    "final": "cloudflare"
  },
  "inbounds": [
    {
      "type": "mixed",
      "tag": "mixed-in",
      "listen": "127.0.0.1",
      "listen_port": 7890
    }
  ],
  "outbounds": [
    {
      "type": "vless",
      "tag": "proxy",
      "server": "你的服务器IP或域名",
      "server_port": 443,
      "uuid": "你的UUID",
      "flow": "xtls-rprx-vision",
      "packet_encoding": "xudp",
      "domain_resolver": "cloudflare",
      "tls": {
        "enabled": true,
        "server_name": "www.microsoft.com",
        "utls": {
          "enabled": true,
          "fingerprint": "chrome"
        },
        "reality": {
          "enabled": true,
          "public_key": "你的REALITY公钥",
          "short_id": "你的short_id"
        }
      }
    },
    {
      "type": "direct",
      "tag": "direct"
    },
    {
      "type": "block",
      "tag": "block"
    }
  ],
  "route": {
    "rules": [
      {
        "action": "sniff"
      },
      {
        "protocol": "dns",
        "action": "hijack-dns"
      },
      {
        "ip_is_private": true,
        "action": "route",
        "outbound": "direct"
      },
      {
        "rule_set": "geosite-cn",
        "action": "route",
        "outbound": "direct"
      },
      {
        "rule_set": "geoip-cn",
        "action": "route",
        "outbound": "direct"
      }
    ],
    "rule_set": [
      {
        "type": "remote",
        "tag": "geosite-cn",
        "format": "binary",
        "url": "https://raw.githubusercontent.com/SagerNet/sing-geosite/rule-set/geosite-cn.srs"
      },
      {
        "type": "remote",
        "tag": "geoip-cn",
        "format": "binary",
        "url": "https://raw.githubusercontent.com/SagerNet/sing-geoip/rule-set/geoip-cn.srs"
      }
    ],
    "final": "proxy",
    "auto_detect_interface": true,
    "default_domain_resolver": "cloudflare"
  },
  "experimental": {
    "cache_file": {
      "enabled": true
    }
  }
}
```

配置逻辑：

- 本机开放 `127.0.0.1:7890`，同时支持 HTTP 和 SOCKS 代理。
- 国内域名、国内 IP、私有地址走 `direct`。
- 其他流量默认走 VLESS + Vision + REALITY 节点。
- DNS 查询由 sing-box 接管，统一使用 Cloudflare / Google DNS。
- 远程规则集缓存到 `cache_file`，避免每次启动重新下载。
- `packet_encoding: "xudp"` 是 UDP 包编码，不需要 UDP 转发可删除。

如果 `server` 填的是域名，`domain_resolver: "cloudflare"` 会让 sing-box 使用 Cloudflare DNS 解析节点域名。更稳妥的做法是直接填写 VPS IP。

### 7.4 Android sing-box TUN 模式

Android 上推荐使用 sing-box for Android 的 TUN/VPN 模式，可接管大部分 App 流量，无需每个 App 单独设置代理。

以下配置与上一节共用同一个 VLESS 出站，入站从 `mixed` 改为 `tun`：

```json
{
  "log": {
    "level": "info",
    "timestamp": true
  },
  "dns": {
    "servers": [
      {
        "tag": "cloudflare",
        "type": "tls",
        "server": "1.1.1.1"
      },
      {
        "tag": "google",
        "type": "tls",
        "server": "8.8.8.8"
      }
    ],
    "final": "cloudflare"
  },
  "inbounds": [
    {
      "type": "tun",
      "tag": "tun-in",
      "address": [
        "172.19.0.1/30",
        "fdfe:dcba:9876::1/126"
      ],
      "auto_route": true,
      "stack": "system"
    }
  ],
  "outbounds": [
    {
      "type": "vless",
      "tag": "proxy",
      "server": "你的服务器IP或域名",
      "server_port": 443,
      "uuid": "你的UUID",
      "flow": "xtls-rprx-vision",
      "packet_encoding": "xudp",
      "domain_resolver": "cloudflare",
      "tls": {
        "enabled": true,
        "server_name": "www.microsoft.com",
        "utls": {
          "enabled": true,
          "fingerprint": "chrome"
        },
        "reality": {
          "enabled": true,
          "public_key": "你的REALITY公钥",
          "short_id": "你的short_id"
        }
      }
    },
    {
      "type": "direct",
      "tag": "direct"
    },
    {
      "type": "block",
      "tag": "block"
    }
  ],
  "route": {
    "rules": [
      {
        "action": "sniff"
      },
      {
        "protocol": "dns",
        "action": "hijack-dns"
      },
      {
        "ip_is_private": true,
        "action": "route",
        "outbound": "direct"
      },
      {
        "rule_set": "geosite-cn",
        "action": "route",
        "outbound": "direct"
      },
      {
        "rule_set": "geoip-cn",
        "action": "route",
        "outbound": "direct"
      }
    ],
    "rule_set": [
      {
        "type": "remote",
        "tag": "geosite-cn",
        "format": "binary",
        "url": "https://raw.githubusercontent.com/SagerNet/sing-geosite/rule-set/geosite-cn.srs"
      },
      {
        "type": "remote",
        "tag": "geoip-cn",
        "format": "binary",
        "url": "https://raw.githubusercontent.com/SagerNet/sing-geoip/rule-set/geoip-cn.srs"
      }
    ],
    "final": "proxy",
    "default_domain_resolver": "cloudflare"
  },
  "experimental": {
    "cache_file": {
      "enabled": true
    }
  }
}
```

Android 图形客户端会通过 VpnService 管理 TUN 接口，`interface_name` 一般不需要手动填写。分应用代理或绕过建议在 sing-box for Android 界面中设置。

如果只是导入一个节点快速使用，NekoBox 更省事；如果想统一管理 DNS、路由、TUN 和分应用规则，sing-box for Android 更合适。

## **8. 参数对应关系**

为避免填错，以下列出 VLESS URL 参数在各客户端配置中的对应关系：

| VLESS URL 参数 | Shadowrocket | Mihomo 字段 | sing-box 字段 |
| --- | --- | --- | --- |
| `UUID` | UUID | `uuid` | `uuid` |
| 服务器地址 | Address | `server` | `server` |
| 端口 | Port | `port` | `server_port` |
| `flow=xtls-rprx-vision` | Flow | `flow` | `flow` |
| `sni=xxx` | Server Name / SNI | `servername` | `tls.server_name` |
| `fp=chrome` | Fingerprint | `client-fingerprint` | `tls.utls.fingerprint` |
| `pbk=xxx` | Public Key | `reality-opts.public-key` | `tls.reality.public_key` |
| `sid=xxx` | Short ID | `reality-opts.short-id` | `tls.reality.short_id` |
| `type=tcp` | — | `network: tcp` | 默认 TCP，无需额外配置 |
| UDP 转发 | — | `udp` + `packet-encoding` | `packet_encoding` |

最容易填错的是 `servername`、`public_key`、`short_id` 和 `flow`。特别注意 `public_key` 必须是 REALITY 公钥，不是服务端私钥。

## **9. 常见错误与排查**

### 9.1 配置格式检查

sing-box 配置保存后建议先执行：

```bash
sing-box check -c config.json
```

也可用格式化命令统一排版：

```bash
sing-box format -w -c config.json
```

Mihomo 图形客户端通常会在导入配置时提示 YAML 错误。

### 9.2 TLS handshake error

原因：

- Public Key 填错（客户端应填公钥，不是服务端私钥）
- SNI 与服务端 `server_name` 不一致
- Short ID 不一致
- 私钥与公钥不匹配

### 9.3 Connected 但无法上网

原因：

- 443 端口未开放（检查防火墙和安全组）
- JSON/YAML 配置文件有语法错误
- IPv6 环境但客户端未开启 IPv6
- 伪装域名配置错误

### 9.4 连不上时优先检查清单

1. 客户端内核是否为 Mihomo/Clash Meta（传统 Clash 不支持）。
2. `uuid` 是否与服务端一致。
3. `servername` 是否匹配服务端 `server_name`。
4. `public_key` 是否来自 REALITY 公钥，而不是 `privateKey`。
5. `short_id` 是否匹配服务端 `short_id`。
6. `flow` 是否为 `xtls-rprx-vision`。
7. 服务端端口是否放行（安全组 + 防火墙均需开放 443/tcp）。
8. sing-box 远程规则集是否能下载；首次启动无法访问 GitHub 时，可先临时删除 `rule_set` 规则，确认节点可用后再恢复。

### 9.5 short ID 可以为空吗？

服务端 sing-box 配置中 `short_id` 是数组（如 `["abcd1234"]`），客户端配置中则是字符串（如 `"abcd1234"`）。服务端 `short_id` 可以配置为空数组 `[""]`，但不建议，有些客户端对空 short ID 处理不一致。更稳妥的做法是设置一个具体值，客户端对应填写即可。

## **10. 参考资料**

- [Mihomo VLESS 配置](https://wiki.metacubex.one/config/proxies/vless/)
- [Mihomo DNS 配置](https://wiki.metacubex.one/config/dns/)
- [sing-box VLESS 出站](https://sing-box.sagernet.org/configuration/outbound/vless/)
- [sing-box TLS/REALITY 字段](https://sing-box.sagernet.org/configuration/shared/tls/)
- [sing-box TUN 入站](https://sing-box.sagernet.org/configuration/inbound/tun/)
- [sing-box 路由与规则集](https://sing-box.sagernet.org/configuration/route/)

## **11. 总结**

VLESS + Vision + REALITY 的部署分为服务端和客户端两部分：

**服务端**：
1. 安装 sing-box，生成 UUID、Reality 密钥对和 Short ID。
2. 编写配置文件，`server_name` 指向伪装网站，`private_key` 填写服务端私钥。
3. 检查配置格式后启动服务，开放 443 端口。

**客户端**：
1. `uuid` 必须与服务端一致。
2. `server` 和 `port` 必须正确。
3. `flow` 使用 `xtls-rprx-vision`。
4. SNI / `servername` 匹配服务端 `server_name`。
5. 填写 REALITY 公钥（不是私钥）。
6. `short_id` 匹配服务端配置。
7. Clash 客户端必须使用 Mihomo/Clash Meta 内核。
