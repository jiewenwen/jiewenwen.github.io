---
title: VLESS + Vision + REALITY 单节点在 Clash/Mihomo 与 sing-box 上的配置实践
date: 2026-06-04 12:00:00 +0800
categories: [Network]
tags: [sing-box, vless, reality, vps, proxy, Mihomo, Clash]
---

最近我在服务器上部署了基于 **VLESS + Vision + REALITY** 的代理节点。相比传统 TLS 节点，REALITY 不依赖自有域名证书，也不要求服务端暴露一个完整网站，更适合个人单节点使用。服务器配置可参考文章：[使用 sing-box 部署 VLESS + Vision + Reality 节点](https://blog.flynoa.cc/posts/sing-box-vless-reality/)。

这篇文章只记录客户端侧配置，主要覆盖两类场景：

- **Clash/Mihomo**：适合 Clash Verge Rev、Mihomo Party、Clash Meta for Android、NekoBox 等使用 Mihomo/Clash Meta 内核的客户端。
- **sing-box**：适合 Windows/macOS/Linux 上本地端口代理，也适合 Android 上的 sing-box for Android TUN/VPN 模式。

需要先说明：传统老版 Clash 不支持 VLESS + Vision + REALITY。这里说的 Clash 配置，实际指 **Mihomo / Clash Meta 内核**。

## **1. 节点参数**

一个 VLESS + Vision + REALITY 节点通常可以写成类似下面的 URL：

```text
vless://UUID@服务器地址:端口?encryption=none&flow=xtls-rprx-vision&security=reality&sni=www.bing.com&fp=chrome&pbk=REALITY公钥&sid=short_id&type=tcp#节点名
```

常用参数对应关系如下：

| URL 参数 | 含义 |
| --- | --- |
| `UUID` | VLESS 用户 ID |
| `服务器地址` | VPS 的 IP 或域名 |
| `端口` | 服务端监听端口，常见为 `443` |
| `flow` | Vision 流控，通常为 `xtls-rprx-vision` |
| `sni` | REALITY 使用的 SNI，需要与服务端 `serverNames` 对应 |
| `fp` | 客户端 TLS 指纹，常用 `chrome` |
| `pbk` | REALITY 公钥，客户端填写 |
| `sid` | REALITY short ID，需要与服务端 `shortIds` 对应 |
| `type` | 传输类型，本文使用 `tcp` |

客户端只需要填写 **REALITY 公钥**。服务端配置里的 `privateKey` 是私钥，不要填到客户端。

## **2. Clash/Mihomo 单节点配置**

下面是一份最小可用的 Mihomo 配置。把其中的占位值替换成自己的节点参数即可。

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
    servername: 你的REALITY伪装域名
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

需要重点替换这些字段：

| Mihomo 字段 | 填写内容 |
| --- | --- |
| `server` | VPS IP 或域名 |
| `port` | 服务端监听端口 |
| `uuid` | VLESS 用户 ID |
| `servername` | 服务端 REALITY `serverNames` 中的域名 |
| `reality-opts.public-key` | REALITY 公钥 |
| `reality-opts.short-id` | 服务端 `shortIds` 中的一个值 |
| `client-fingerprint` | 与 URL 中的 `fp` 对应，常用 `chrome` |
| `skip-cert-verify` | REALITY 场景下常见为 `true` |

如果服务端配置是：

```json
"serverNames": [
  "www.bing.com"
]
```

那么 Mihomo 中应写：

```yaml
servername: www.bing.com
```

`default-nameserver` 是引导 DNS，用来解析 `nameserver` 和 `proxy-server-nameserver` 里的 DNS 服务域名，例如 `cloudflare-dns.com` 和 `dns.google`。它必须写成 IP，只有在启动 DNS 查询链路时使用，不是普通网站域名的主要解析策略。

这里容易混淆的是 `network: tcp`、`udp: true` 和 `packet-encoding: xudp`：

- `network: tcp` 表示 VLESS 到服务端的传输层是 TCP，也就是 URL 中的 `type=tcp`。
- `udp: true` 表示这个代理节点允许客户端应用产生的 UDP 流量，例如 QUIC、HTTP/3、游戏或部分 DNS 流量。
- `packet-encoding: xudp` 表示这些 UDP 包会用 XUDP 方式封装，再通过 VLESS 连接转发；它不是把 VLESS 传输层改成 UDP。

如果只需要 TCP 代理，可以把 `udp` 改成 `false`，并删除 `packet-encoding: xudp`。如果需要 UDP 转发，保留这两项，并确认服务端兼容 XUDP。

`skip-cert-verify: true` 是 REALITY 节点在 Mihomo 中的常见写法，安全性主要依赖 `public-key`、`short-id` 与 REALITY 握手校验。普通 TLS 节点不要直接照抄这个值。

## **3. sing-box 本地端口模式**

下面配置适合 Windows/macOS/Linux 上的本地代理端口模式。它在 `127.0.0.1:7890` 监听 HTTP/SOCKS 混合端口，浏览器或系统代理指向这个端口即可。

这份示例按 sing-box 1.12+ 的配置风格编写：DNS 服务器使用新格式，国内规则使用远程 `rule_set`，不再直接写已废弃的 `geosite`/`geoip` 字段。

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
        "server_name": "你的REALITY伪装域名",
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

这份配置的逻辑是：

- 本机开放 `127.0.0.1:7890`，同时支持 HTTP 和 SOCKS 代理。
- 国内域名、国内 IP、私有地址走 `direct`。
- 其他流量默认走 `proxy`，也就是 VLESS + Vision + REALITY 节点。
- DNS 查询由 sing-box 接管，并统一使用 Cloudflare / Google DNS。
- 远程规则集会被缓存到 `cache_file`，避免每次启动都重新下载。
- `packet_encoding: "xudp"` 是 UDP 包编码，不改变 VLESS 的 TCP 传输层；不需要 UDP 转发时可以删除这一项。

如果你的 `server` 填的是域名，`domain_resolver: "cloudflare"` 会让 sing-box 使用 Cloudflare DNS 解析节点域名。更稳妥的做法是直接填写 VPS IP，避免节点域名解析异常导致无法启动代理链路。

## **4. Android sing-box TUN 模式**

Android 上更推荐 sing-box for Android 的 TUN/VPN 模式，而不是只开本地代理端口。TUN 模式可以接管大部分 App 流量，不需要每个 App 单独设置代理。

下面配置与上一节使用同一个 VLESS 出站，只把入站从 `mixed` 改成 `tun`。

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
        "server_name": "你的REALITY伪装域名",
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

Android 图形客户端会通过 VpnService 管理 TUN 接口，`interface_name` 一般不需要手动写。分应用代理或绕过也更建议在 sing-box for Android 的界面里设置，而不是直接手写 `include_package` 或 `exclude_package`。

## **5. 参数对应关系**

为了避免填错，可以按下面的关系检查：

| VLESS URL 参数 | Mihomo 字段 | sing-box 字段 |
| --- | --- | --- |
| `UUID` | `uuid` | `uuid` |
| `服务器地址` | `server` | `server` |
| `端口` | `port` | `server_port` |
| `flow=xtls-rprx-vision` | `flow` | `flow` |
| `sni=xxx` | `servername` | `tls.server_name` |
| `fp=chrome` | `client-fingerprint` | `tls.utls.fingerprint` |
| `pbk=xxx` | `reality-opts.public-key` | `tls.reality.public_key` |
| `sid=xxx` | `reality-opts.short-id` | `tls.reality.short_id` |
| `type=tcp` | `network: tcp` | 不写 `transport`，默认就是 TCP |
| UDP 转发 | `udp` + `packet-encoding` | `packet_encoding` |

最容易填错的是 `servername`、`public_key`、`short_id` 和 `flow`。其中 `public_key` 必须是 REALITY 公钥，不是服务端私钥。

注意：sing-box 的 `network` 字段表示这个出站允许处理 `tcp`、`udp` 哪类业务流量，不等同于 VLESS URL 里的 `type=tcp`。如果想保留 UDP 转发能力，通常不要把 sing-box 的 `network` 固定为 `tcp`。

## **6. 检查与排错**

### 6.1 检查配置格式

sing-box 配置保存后，建议先执行：

```bash
sing-box check -c config.json
```

如果是 Linux 服务器或桌面系统，也可以用格式化命令统一排版：

```bash
sing-box format -w -c config.json
```

Mihomo 图形客户端通常会在导入配置时提示 YAML 错误；如果客户端支持内核参数，也可以用 Mihomo 的配置检查功能先测试。

### 6.2 连不上时优先检查

1. 客户端内核是否为 Mihomo/Clash Meta，而不是传统 Clash。
2. `uuid` 是否与服务端一致。
3. `servername` 是否在服务端 `serverNames` 列表里。
4. `public_key` 是否来自 REALITY 公钥，而不是 `privateKey`。
5. `short_id` 是否在服务端 `shortIds` 列表里。
6. `flow` 是否为 `xtls-rprx-vision`。
7. 服务端端口是否放行，例如安全组和防火墙都开放了 `443/tcp`。
8. sing-box 远程规则集是否能下载；首次启动无法访问 GitHub 时，可以先临时删除 `rule_set` 规则，确认节点本身可用后再恢复。

### 6.3 short ID 可以为空吗？

REALITY 服务端的 `shortIds` 可以配置为空字符串，但不建议这样做。有些客户端对空 short ID 的处理不一致。更稳妥的做法是设置一个短 ID，例如：

```json
"shortIds": [
  "abcd1234"
]
```

客户端对应填写：

```yaml
short-id: abcd1234
```

sing-box 中则是：

```json
"short_id": "abcd1234"
```

### 6.4 Android 应该用 NekoBox 还是 sing-box？

如果只是导入一个节点快速使用，NekoBox 更省事。

如果想统一管理 DNS、路由、TUN 和分应用规则，sing-box for Android 更合适。

### 6.5 Windows 应该用 Mihomo 还是 sing-box？

如果想要图形界面、规则管理和订阅管理，优先选择 Clash Verge Rev 或 Mihomo Party。

如果想要跨平台配置一致、日志明确、规则更可控，可以直接使用 sing-box。

## **7. 参考资料**

- [Mihomo VLESS 配置](https://wiki.metacubex.one/config/proxies/vless/)
- [Mihomo DNS 配置](https://wiki.metacubex.one/config/dns/)
- [sing-box VLESS 出站](https://sing-box.sagernet.org/configuration/outbound/vless/)
- [sing-box TLS/REALITY 字段](https://sing-box.sagernet.org/configuration/shared/tls/)
- [sing-box TUN 入站](https://sing-box.sagernet.org/configuration/inbound/tun/)
- [sing-box 路由与规则集](https://sing-box.sagernet.org/configuration/route/)

## **8. 总结**

VLESS + Vision + REALITY 的客户端配置并不复杂，关键是保证服务端和客户端参数一一对应：

1. `uuid` 必须一致。
2. `server` 和 `port` 必须正确。
3. `flow` 使用 `xtls-rprx-vision`。
4. `servername` 匹配服务端 `serverNames`。
5. 客户端填写 REALITY 公钥。
6. `short_id` 匹配服务端 `shortIds`。
7. Clash 客户端必须使用 Mihomo/Clash Meta 内核。
8. sing-box 新版本应使用 `rule_set`，不要继续依赖旧的 `geosite`/`geoip` 直写规则。
