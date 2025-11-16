---
title: 手动部署 VLESS + Reality + uTLS 完整教程
date: 2025-11-17 00:12:00 +0800
categories: [VPN]
tags: [VPN, VPS, VLESS, Reality, uTLS, Sing-box]
---



适合情况：

- **客户端：Shadowrocket（iOS）**
- **服务器：Linux（Debian/Ubuntu/CentOS 都可）**
- **基于Sing-box**
- **协议：VLESS + Reality + uTLS（Chrome 指纹）**
- **不依赖 S-UI，X-UI等面板**
- **不暴露面板端口**
- **最高安全性 + 最低指纹特征**

------



## **第一步：安装 Sing-box（官方安装脚本）**



适用于 Debian / Ubuntu：

```bash
bash <(curl -fsSL https://sing-box.app/install.sh)
```

安装完成后，配置文件路径在：

```
/etc/sing-box/config.json
```

可执行文件路径在：

```
/usr/local/bin/sing-box
```

systemd 服务：

```bash
systemctl restart sing-box
systemctl status sing-box
```

------



## **第二步：生成所需密钥和 UUID**



Sing-box 自带工具，我们来生成 Reality 密钥对、Short ID 和 UUID。



### 1. 生成 Reality 密钥对：

```bash
sing-box generate reality-key
```

输出示例（请复制保存）：

```bash
Private key: XXXXXXXXXXXXXXXXXXXX
Public key:  YYYYYYYYYYYYYYYYYYYY
```



### 2. 生成 short_id：

```bash
sing-box generate reality-short-id
```

输出示例（请复制保存）：

```bash
abcd1234
```

> **ShortID 最好 4–8 字节，不要太短，不要太长。**



### 3. 生成 UUID：

```bash
sing-box generate uuid
```

输出示例（请复制保存）：

```bash
zzzzzzzz-zzzz-zzzz-zzzz-zzzzzzzzzzzz
```

------



## **第三步：编写 Sing-box 服务端配置**



路径在：

```bash
nano /etc/sing-box/config.json
```

JSON内容（请使用 **第二步** 生成的值替换“替换为...”部分）：

```json
{
  "log": {
    "level": "warn"
  },
  "inbounds": [
    {
      "type": "vless",
      "listen": "::",
      "listen_port": 443,
      "users": [
        {
          "uuid": "替换为你的UUID"
        }
      ],
      "tls": {
        "enabled": true,
        "server_name": "www.cloudflare.com",
        "reality": {
          "enabled": true,
          "private_key": "替换为你的PrivateKey",
          "short_id": "替换为ShortID"
        },
        "utls": {
          "enabled": true,
          "fingerprint": "chrome"
        }
      }
    }
  ],
  "outbounds": [
    {
      "type": "direct"
    }
  ]
}
```

------



### **关键说明（非常重要）**



### 1. `server_name` 必须与伪装域名一致

推荐伪装域名（客户端和服务器必须一致）：

```
www.cloudflare.com
www.google.com
www.apple.com
```



### 2. Reality 不能与 allowInsecure 混用

Reality 不需要证书，不需要域名解析，不需要 `allowInsecure`。



### 3. uTLS 必须开启且 fingerprint 用 chrome

这是目前最安全、最真实的 TLS 指纹。

------



## **第四步：启动 Sing-box 服务**



检查语法：

```bash
sing-box check -c /etc/sing-box/config.json
```

如果输出：

```bash
Configuration OK
```

启动并设置开机自启：

```bash
systemctl restart sing-box
systemctl enable sing-box
systemctl status sing-box
```

------



## **第五步：服务器防火墙放行 443**



必要（以 UFW 为例）：

```bash
ufw allow 443/tcp
```

Reality 必须使用 TCP（真实流量就是 HTTPS）。

------



## **第六步：Shadowrocket 客户端配置**



打开 Shadowrocket → 添加节点 → 类型选择 **VLESS**

填写（请注意 `SpiderX` 字段的修正）：

| **项目**       | **值**                               |
| -------------- | ------------------------------------ |
| 地址           | 你的服务器IP                         |
| 端口           | 443                                  |
| UUID           | 服务端 UUID（第二步生成）            |
| 加密           | none                                 |
| 流控           | 留空                                 |
| 传输协议       | tcp                                  |
| TLS            | 开启                                 |
| 伪装域名 (SNI) | `www.cloudflare.com`（与服务端一致） |
| Reality        | 开启                                 |
| PublicKey      | 服务端公钥（第二步生成）             |
| ShortID        | 服务端 ShortID（第二步生成）         |
| **SpiderX**    | **留空**                             |
| uTLS           | 开启                                 |
| 指纹           | chrome                               |

------



## **第七步：测试是否正常运行**



### 1. 连接 Shadowrocket

查看是否显示：

```
Connected
```



### 2. 测试 IP

访问 `https://chat.openai.com` 或 `https://www.youtube.com`。



### 3. 测试现实功能

在 Shadowrocket 查看 Log：

如果 Reality 成功，你会看到：

```
Reality handshake OK
```