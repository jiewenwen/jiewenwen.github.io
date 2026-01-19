---
title: sing-box + VLESS gRPC Reality + WARP 完整教程
date: 2026-01-19 11:00:00 +0800
categories: [Network]
tags: [sing-box, vless, reality, vps, proxy]
---

## **前言**

考虑到平时用机房IP使用AI工具如Chatgpt、Claude等，容易降智。解决方案最好的无疑是买一个住宅IP。但是考虑到价格相对昂贵，基于能省则省原则，本文考虑到第二种折中方案，即vps节点出口套用Warp。本文解决的一个经典场景即是，vps开了双栈，ipv4走warp出口，ipv6走直连。

目标效果：

| 入口      | 出口               |
| --------- | ------------------ |
| IPv4 节点 | 走 Cloudflare WARP |
| IPv6 节点 | 直连 VPS           |

适用环境：

- VPS：DMIT / 其他海外 VPS
- 系统：Debian / Ubuntu
- sing-box：1.12.17
- 客户端：Shadowrocket / Clash / sing-box
- 协议：VLESS + gRPC + Reality

------

## **一、准备工作**

你需要：

1. 一台海外 VPS（支持 IPv4 + IPv6）
2. 一个域名
3. 两个子域名：
   - `v4.example.com` → 只绑定 **A 记录**
   - `v6.example.com` → 只绑定 **AAAA 记录**

DNS 示例：

| 域名           | 记录 | 指向     |
| -------------- | ---- | -------- |
| v4.example.com | A    | VPS IPv4 |
| v6.example.com | AAAA | VPS IPv6 |

------

## **二、安装 sing-box**

### 2.1 下载

```bash
apt update && apt upgrade -y
bash <(curl -fsSL https://sing-box.app/install.sh)
```

### 2.2 查看版本

```bash
sing-box version
```

------

## **三、安装 Cloudflare WARP（wgcf）**

WARP 本质是一个 WireGuard 隧道，我们用 `wgcf` 生成配置。

### 3.1 下载 wgcf

```bash
wget https://github.com/ViRb3/wgcf/releases/download/wgcf_2.2.30_linux_amd64
chmod +x wgcf_2.2.30_linux_amd64
mv wgcf_2.2.30_linux_amd64 /usr/local/bin/wgcf
```

### 3.2 注册并生成配置

```bash
wgcf register
wgcf generate
```

生成文件：

```
wgcf-profile.conf
```

------

## **四、读取 WARP 参数**

```bash
cat wgcf-profile.conf
```

你会看到类似：

```ini
[Interface]
PrivateKey = AAAABBBBCCCC
Address = 172.16.0.2/32, 2606:4700:110::/128

[Peer]
PublicKey = DDDDEEEEFFFF
Endpoint = 162.159.192.1:2408
AllowedIPs = 0.0.0.0/0, ::/0
PersistentKeepalive = 25
```

记住这些：

| 项目       | 用途            |
| ---------- | --------------- |
| PrivateKey | WARP 私钥       |
| Address    | `本机隧道` IP   |
| PublicKey  | Cloudflare 公钥 |
| Endpoint   | WARP 服务器     |
| AllowedIPs | 全流量          |
| Keepalive  | 保活            |

------

## **五、生成 Reality 密钥、UUID、ShortID**

生成 Reality 密钥

```bash
sing-box generate reality-keypair
```

得到：

```
PrivateKey: XXXXX
PublicKey: YYYYY
```

生成UUID

```bash
sing-box generate uuid
```

生成ShortID

```bash
sing-box generate rand --hex 8
```

------

## **六、完整 sing-box 配置**

编辑配置文件：

```bash
nano /etc/sing-box/config.json
```

粘贴以下内容：

```jsonc
{
  "log": {
    "level": "info",
    "timestamp": true
  },

  "endpoints": [
    {
      "type": "wireguard",
      "tag": "warp",

      // 建议 system=false：让 sing-box 自己托管，不依赖 wg-quick
      "system": false,

      // wgcf 生成的 Address（可以同时填 v4/v6）
      "address": [
        "172.16.0.2/32",
        "2606:4700:110:xxxx:xxxx:xxxx:xxxx:xxxx/128"
      ],

      // wgcf 生成的 PrivateKey
      "private_key": "YOUR_WARP_PRIVATE_KEY_BASE64",

      // 可选：WARP 常见 MTU 1280
      "mtu": 1280,

      // Peer（Cloudflare）
      "peers": [
        {
          // Endpoint：wgcf-profile.conf 里的 Endpoint
          // 可以写域名
          "address": "162.159.192.1",
          "port": 2408,

          // wgcf-profile.conf 里的 PublicKey
          "public_key": "YOUR_WARP_PEER_PUBLIC_KEY_BASE64",

          // 通常 AllowedIPs 全局路由（让走 WARP 的流量全进隧道）
          "allowed_ips": [
            "0.0.0.0/0",
            "::/0"
          ],

          // 如果你拿到了 reserved（WARP 常用）
          "reserved": [0, 0, 0],

          "persistent_keepalive_interval": 25
        }
      ]
    }
  ],

  "inbounds": [
    {
      "type": "vless",
      "tag": "vless-in-4",
      "listen": "0.0.0.0",
      "listen_port": 443,
      "users": [
        {
          "uuid": "YOUR_UUID",
          "flow": ""
        }
      ],
      "transport": {
        "type": "grpc",
        "service_name": "grpc"
      },
      "tls": {
        "enabled": true,
        "server_name": "www.microsoft.com",
        "reality": {
          "enabled": true,
          "handshake": {
            "server": "www.microsoft.com",
            "server_port": 443
          },
          "private_key": "YOUR_REALITY_PRIVATE_KEY",
          "short_id": ["YOUR_SHORT_ID"]
        }
      }
    },

    {
      "type": "vless",
      "tag": "vless-in-6",
      "listen": "::",
      "listen_port": 8443,
      "users": [
        {
          "uuid": "YOUR_UUID",
          "flow": ""
        }
      ],
      "transport": {
        "type": "grpc",
        "service_name": "grpc"
      },
      "tls": {
        "enabled": true,
        "server_name": "www.microsoft.com",
        "reality": {
          "enabled": true,
          "handshake": {
            "server": "www.microsoft.com",
            "server_port": 443
          },
          "private_key": "YOUR_REALITY_PRIVATE_KEY",
          "short_id": ["YOUR_SHORT_ID"]
        }
      }
    }
  ],

  "outbounds": [
    {
      "type": "direct",
      "tag": "direct"
    }
  ],

  "route": {
    // 默认不写也行；建议显式写 direct，避免误会
    "final": "direct",

    "rules": [
      {
        "inbound": ["vless-in-4"],
        "action": "route",
        "outbound": "warp"
      },
      {
        "inbound": ["vless-in-6"],
        "action": "route",
        "outbound": "direct"
      }
    ]
  }
}

```

替换：

- `YOUR_WARP_PRIVATE_KEY`
- `YOUR_WARP_PEER_PUBLIC_KEY`
- `YOUR_REALITY_PRIVATE_KEY`
- `YOUR_UUID`

------

## **七、启动 sing-box**

```bash
systemctl enable sing-box
systemctl restart sing-box
journalctl -u sing-box -f
```

看到：

```
wireguard endpoint warp connected
```

说明成功。

------

## **八、客户端配置**

打开 Shadowrocket → 右上角 + → **Type: VLESS**

填写以下内容：

| 项目                  | 值                                          |
| --------------------- | ------------------------------------------- |
| **Address**           | v4.example.com   （IPv6节点填写 v6.example.com ）                          |
| **Port**              | 443     （IPv6节点填写 8443）                                    |
| **UUID**              | 上面生成的 UUID                             |
| **transport**         | grpc                                        |
| **Host**              | v4.example.com     （IPv6节点填写 v6.example.com ）                       |
| **Service Name**      | grpc                         |
| **TLS**               | 开启                                        |
| **SNI**               | www.microsoft.com                         |
| **Public Key**        | Reality 的 PublicKey                        |
| **Short ID**          | 你的 Short ID                               |
| **Fingerprint**       | `chrome` 或默认（Shadowrocket 默认为 Safari） |

------

## **九、验证是否走 WARP**

IPv4 连接后，访问：

```bash
https://www.cloudflare.com/cdn-cgi/trace
```

应显示 Cloudflare IP。而IPv6 连接后，应显示 DMIT 原生 IP。

------

## **十、最终结构图**

```
IPv4 → VLESS → WARP → Cloudflare → Internet
IPv6 → VLESS → Direct → Internet
```

------

## **十一、总结**

- sing-box 原生支持 WARP
- IPv4 + WARP 更稳定
- IPv6 直连更快
- 双入口分流是最优解
- 适合 ChatGPT / Google / Apple / OpenAI API