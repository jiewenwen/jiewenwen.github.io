---
title: 从零配置 sing-box 部署 VLESS + grpc + Reality 节点
date: 2026-01-09 23:00:00 +0800
categories: [Network]
tags: [sing-box, vless, reality, vps, proxy]
---


## **1. 环境要求**

建议：

- **系统**：Debian 12 / Ubuntu 22.04（其他版本也可）
- **架构**：x86_64 / ARM 均支持
- **端口**：开放 8443/TCP
- **客户端**：iOS **Shadowrocket**
- **域名**：本配置需要一个域名，自行购买。本文以**vps.example.com**示例

无需：

- 证书
- 面板（如 s-ui）

Reality 最大优势就是模拟真实网站 TLS 握手，几乎无法被区分。


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
      "tag": "vless-grpc-reality",
      "listen": "::",
      "listen_port": 8443,
      "users": [
        {
          "uuid": "REPLACE_UUID",
          "flow": ""
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
      "transport": { 
        "type": "grpc", 
        "service_name": "grpc" 
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
ufw allow 8443/tcp
ufw reload
```

若 VPS 有安全组（阿里、AWS、Oracle 等）：

👉 控制台开放 **TCP 8443** 即可。

## **7. Shadowrocket 客户端配置**

**首先配置域名指向IP**。本文使用的是cloudflare购买的域名，比如 example.com。在cloudflare中添加该域名的dns记录。A/AAAA 记录指向 VPS，必须为灰云（DNS Only），不能使用橙云（proxy）。
- 示例：

| 类型 | 名称 | 值     | 代理状态  |
| ---- | ---- | ------ | --------- |
| A    | **vps**    | 157.17.17.43 | DNS only |

然后打开 Shadowrocket → 右上角 + → **Type: VLESS**

填写以下内容：

| 项目                  | 值                                          |
| --------------------- | ------------------------------------------- |
| **Address**           | vps.example.com                             |
| **Port**              | 8443                                         |
| **UUID**              | 上面生成的 UUID                             |
| **transport**         | grpc                                        |
| **Host**              | vps.example.com                            |
| **Service Name**      | grpc                         |
| **TLS**               | 开启                                        |
| **SNI**               | www.microsoft.com                         |
| **Public Key**        | Reality 的 PublicKey                        |
| **Short ID**          | 你的 Short ID                               |
| **Fingerprint**       | `chrome` 或默认（Shadowrocket 默认为 Safari） |

保存后连接 → Safari 访问：

```
https://ipinfo.io
```

若显示 VPS IP = 成功

## **8. 常见错误与排查**

### 8.1 TLS handshake error

原因：

- PublicKey 填错
- SNI 错误
- Short ID 不一致
- 私钥与公钥不匹配

### 8.2 Connected 但无法上网

原因：

- 8443 未开放
- JSON 文件有语法错误
- IPv6 环境但 Shadowrocket 没开 IPv6
- 错误的伪装域名
