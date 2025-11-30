---
title: 使用自域名 + CA 证书部署 Hysteria2 节点
date: 2025-11-29 12:00:00 +0800
categories: [Tutorial, Network]
tags: [Hysteria2, HY2, Cloudflare, Shadowrocket, sing-box, VPS, TLS, ACME]
---

本教程使用 Cloudflare 域名（例如 example.com）与 ACME 证书在 VPS 上部署 Hysteria2（HY2）节点，并提供 Shadowrocket 客户端配置示例。适用于 Debian / Ubuntu / CentOS 等常见发行版。

## **Hysteria2 是什么**
- 基于 QUIC，抗丢包、低延迟，更适合跨境网络
- 可通过 HTTPS/HTTP3 伪装，支持自动 ACME 证书获取
- 在移动宽带（移动/联通/电信）环境表现优秀

## **1. 准备工作**
### 1.1 一台 VPS
支持 IPv4 或 IPv6（双栈更佳）。

### 1.2 Cloudflare 域名
- 在cloudflare中添加域名的dns记录。A/AAAA 记录指向 VPS（可使用 hy 等子域名），必须为灰云（DNS Only），不能使用橙云
- 示例：

| 类型 | 名称 | 值     | 代理状态  |
| ---- | ---- | ------ | --------- |
| A    | @    | 157.17.17.43 | DNS only |

## 2. 安装 Hysteria2（官方脚本）
运行官方一键脚本：

```bash
bash <(curl -fsSL https://get.hy2.sh/)
```

脚本会安装最新版 hysteria2、创建 systemd 服务，并生成 `/etc/hysteria/config.yaml`。

## **3. 修改 hy2 配置文件**
编辑服务器 Hysteria2 配置

```bash
sudo nano /etc/hysteria/config.yaml
```

示例配置（替换为自己的域名、邮箱和 Token）：

```yaml
listen: :443

acme:
  domains:
    - "example.com"
  email: you@example.com

auth:
  type: password
  password: "YourStrongPassword"

masquerade:
  type: proxy
  proxy:
    url: https://bing.com/
    rewriteHost: true
```

保存后重启并查看日志：

```bash
sudo systemctl restart hysteria-server.service
```

设置开机启动：
```bash
sudo systemctl enable hysteria-server.service
```

检查启动状态：
```bash
sudo systemctl status hysteria-server.service
```

如果提示active(running)表示启动成功，并且证书已申请成功。如果提示failed失败，可能是防火墙未放行443/udp端口。

## **4. Shadowrocket 客户端连接（iOS）**
### 4.1 基础配置

| 字段   | 填写内容           |
| ------ | ------------------ |
| 类型   | Hysteria2          |
| 服务器 | example.com        |
| 端口   | 443                |
| 密码   | YourStrongPassword |

### 4.2 TLS 设置（必须开启）

| 字段           | 填写内容    |
| -------------- | ----------- |
| TLS            | 开启        |
| SNI            | example.com |
| Allow Insecure | 关闭        |

> 必须填写 SNI，以匹配服务器证书域名，否则无法握手。

### 4.3 Bandwidth（可选）
- 不填：使用 BBR，更稳定（推荐）
- 填写：使用 Brutal，更快但可能不稳

### 4.4 其他设置
- Obfs：关闭
- QUIC 版本：默认
- 心跳：默认

## **5. 节点测试**
Shadowrocket 中：
1) 点击刚添加的 HY2 节点
2) 启用代理
3) 打开浏览器访问任意网站

可以正常使用即部署成功。

## **6. 常见问题**
**Q1：Cloudflare 可以开橙云吗？**

不可以。HY2 基于 QUIC，HTTP CDN 无法反代此类流量，必须使用灰云（DNS only）。

**Q2：HY2 可以走 IPv6 吗？**

可以，客户端会优先使用 IPv6。

**Q3：必须使用 ACME 证书吗？**

不强制，但 ACME（Let’s Encrypt）更安全，Shadowrocket 兼容性也更好，无需开启 Allow Insecure，强烈推荐。
