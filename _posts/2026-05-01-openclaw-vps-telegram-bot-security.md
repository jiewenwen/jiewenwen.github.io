---
layout: post
title: "OpenClaw 安装、配置与 Telegram Bot 接入教程：从 VPS 部署到安全加固"
date: 2026-05-01 13:20:00 +0800
categories: [AI]
tags: [OpenClaw, Telegram Bot, VPS, Gateway, Security]
---

这篇文章记录一次在 Debian/Ubuntu VPS 上部署 OpenClaw，并通过 Telegram Bot 接入对话的完整流程。目标不是一次性打开所有功能，而是先跑通一条稳定、可维护、相对安全的主线：

```text
VPS 安装 OpenClaw
→ 启动本地 Gateway
→ 创建或复用 Telegram Bot
→ 配置 Bot Token
→ 完成 DM pairing
→ 改成 allowlist
→ 做基础安全加固
```

OpenClaw 的功能很多，第一次部署时最容易被各种 provider、skills、hooks、gateway 选项分散注意力。建议先把 Gateway 和 Telegram 私聊打通，再逐步添加模型、技能和自动化能力。

---

## 一、OpenClaw 是什么？

OpenClaw 是一个可以运行在个人设备或服务器上的 AI Assistant / Gateway。它不是大模型本身，而是一个本地优先的 AI 助手框架：你可以把它连接到 OpenAI、DeepSeek、Anthropic、Google、Ollama 等模型服务，再通过 Telegram、WhatsApp、Slack、Discord 等聊天渠道和它交互。

可以简单理解为：

```text
Telegram 私聊 Bot
        ↓
OpenClaw Gateway
        ↓
LLM Provider，例如 DeepSeek / OpenAI / Ollama
```

部署完成后，你就可以像和一个 Telegram 联系人聊天一样使用自己的 AI 助手。

---

## 二、部署环境

本文以 Debian/Ubuntu VPS 为例。

示例环境：

```text
系统：Debian / Ubuntu
用户：root 或具备 sudo 权限的普通用户
Gateway 默认端口：18789
Telegram 接入方式：BotFather 创建 Bot + Bot Token
```

OpenClaw 常见安装方式有两种：

1. 使用官方安装脚本。
2. 先安装 Node.js，再用 npm 全局安装。

第一次部署建议使用官方安装脚本，少踩环境依赖的坑。更习惯手动控制环境的人，可以选择 npm 方式。

---

## 三、安装前准备

先登录 VPS：

```bash
ssh root@你的服务器IP
```

更新系统并安装基础工具：

```bash
sudo apt update
sudo apt install -y curl git ca-certificates build-essential nano
```

如果使用普通用户，确保该用户具备 sudo 权限。

---

## 四、安装 OpenClaw

### 方式一：官方安装脚本

执行：

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

安装完成后，如果没有自动进入配置流程，可以手动运行：

```bash
openclaw onboard --install-daemon
```

`--install-daemon` 会走托管 Gateway 安装路径，把 OpenClaw Gateway 安装为后台服务。这样即使退出 SSH，Gateway 也能继续运行。

### 方式二：npm 安装

如果希望手动安装 Node.js，可以用 NodeSource 安装较新的 Node.js：

```bash
curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -
sudo apt install -y nodejs
```

检查版本：

```bash
node -v
npm -v
```

然后安装 OpenClaw：

```bash
npm install -g openclaw@latest
openclaw onboard --install-daemon
```

---

## 五、Onboarding 怎么选？

执行 `openclaw onboard --install-daemon` 后，会进入交互式配置流程，通常会涉及：

- 模型 Provider
- Search Provider
- Skills
- Hooks
- Gateway
- Control UI
- Agent TUI
- Telegram 或其他聊天渠道

第一次部署不建议一次性开启所有功能。推荐顺序是：

1. 先配置一个可用的模型 Provider。
2. Search Provider 可以先跳过。
3. Skills 可以保持默认，缺依赖项不用强行补齐。
4. Hooks 先选择 `Skip for now`。
5. Gateway 安装为 service。
6. 先确认 Gateway 正常运行。
7. 最后再接入 Telegram。

### Skills 提示不是报错

你可能会看到类似：

```text
Skills status
Eligible: 8
Missing requirements: 48
Unsupported on this OS: 7
Blocked by allowlist: 0
```

这不是错误，只是说明部分技能缺少依赖或 API Key。第一次部署只要能完成基础对话即可，不需要把所有 skills 都装上。

### Hooks 先跳过

如果看到：

```text
Enable hooks?
- Skip for now
- boot-md
- bootstrap-extra-files
- command-logger
- session-memory
```

建议第一次选择：

```text
Skip for now
```

Hooks 是自动化扩展机制，例如启动时加载额外上下文、记录命令、保存 session memory 等。对于首次接入 Telegram，它不是必要项。

---

## 六、确认 Gateway 正常运行

Onboarding 完成后，先检查 Gateway。

```bash
openclaw gateway status
```

正常情况下会看到类似：

```text
Runtime: running
Connectivity probe: ok
Listening: 127.0.0.1:18789
Dashboard: http://127.0.0.1:18789/
```

再检查端口：

```bash
ss -lntp | grep 18789
```

理想输出类似：

```text
LISTEN 0 511 127.0.0.1:18789 0.0.0.0:* users:(("node",pid=5352,fd=28))
```

也可以用 curl 测试 Dashboard：

```bash
curl -i http://127.0.0.1:18789/
```

如果返回 `HTTP/1.1 200 OK`，说明 Web UI 已经可以访问。

---

## 七、常见 Gateway 问题排查

### 1. `connect ECONNREFUSED 127.0.0.1:18789`

如果看到：

```text
Health check failed: connect ECONNREFUSED 127.0.0.1:18789
Gateway: not detected
```

说明 Gateway 没有正常监听 18789。可以执行：

```bash
openclaw doctor --fix
openclaw gateway install --force
openclaw gateway restart
openclaw gateway status
```

再检查端口：

```bash
ss -lntp | grep 18789
```

### 2. `Port 18789 is already in use`

如果看到：

```text
Port 18789 is already in use
Gateway already running locally
```

这不一定是错误。通常说明 Gateway 进程已经在运行，只是健康检查可能短暂超时。

先等几秒再检查：

```bash
sleep 5
openclaw gateway status
```

如果仍然异常，再重启：

```bash
openclaw gateway restart
sleep 5
openclaw gateway status
```

不要重复手动启动多个 Gateway 进程，否则更容易端口冲突。

### 3. 查看日志

如果 Gateway 起不来，可以看 systemd 用户服务日志：

```bash
journalctl --user -u openclaw-gateway -n 100 --no-pager
```

也可以看 OpenClaw 日志：

```bash
openclaw logs --follow
```

或按实际日期查看临时日志：

```bash
tail -f /tmp/openclaw/openclaw-日期.log
```

---

## 八、创建或找回 Telegram Bot

OpenClaw 接入 Telegram 需要一个 Telegram Bot Token。

打开 Telegram，搜索官方 BotFather：

```text
@BotFather
```

### 创建新 Bot

发送：

```text
/newbot
```

按提示输入：

```text
Bot 显示名称：YOUR Agent
Bot username：your_agent_bot
```

注意 username 必须以 `bot` 结尾。

创建完成后，BotFather 会返回一个 token，格式类似：

```text
1234567890:AAxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

这个 token 非常重要，不要公开，不要提交到 GitHub。

### 找回已有 Bot

如果以前创建过 Bot，但忘了 token，可以在 BotFather 里发送：

```text
/mybots
```

选择对应 Bot 后，可以查看或重置 API Token。

如果不确定这个旧 Bot 以前做什么用，建议检查：

- Bot username
- Description
- About
- Commands
- 是否在某个群或频道里
- 旧服务器或项目里的 `.env`、`docker-compose.yml`、`openclaw.json`

如果怀疑 token 泄露过，建议在 BotFather 里 revoke current token，重新生成一个新 token。

---

## 九、配置 Telegram 接入 OpenClaw

Telegram 不需要执行 `openclaw channels login telegram`，把 Bot Token 写入配置或环境变量后重启 Gateway 即可。

OpenClaw 主配置文件一般在：

```bash
~/.openclaw/openclaw.json
```

编辑：

```bash
nano ~/.openclaw/openclaw.json
```

加入 Telegram 配置：

```json
{
  "channels": {
    "telegram": {
      "enabled": true,
      "botToken": "这里换成你的 Telegram Bot Token",
      "dmPolicy": "pairing",
      "groups": {
        "*": {
          "requireMention": true
        }
      }
    }
  }
}
```

如果 `openclaw.json` 已经有其他内容，不要直接全部覆盖。只需要把下面这段合并到原有的 `channels` 对象里：

```json
"telegram": {
  "enabled": true,
  "botToken": "你的 Telegram Bot Token",
  "dmPolicy": "pairing",
  "groups": {
    "*": {
      "requireMention": true
    }
  }
}
```

保存 nano：

```text
Ctrl + O
Enter
Ctrl + X
```

然后重启 Gateway：

```bash
openclaw gateway restart
sleep 5
openclaw gateway status
```

确认 Gateway 仍然正常：

```text
Runtime: running
Connectivity probe: ok
Listening: 127.0.0.1:18789
```

---

## 十、Telegram 首次配对

打开你的 Telegram Bot，发送：

```text
/start
```

当 `dmPolicy` 设置为 `pairing` 时，未知发送者会收到一个短配对码。服务器上执行：

```bash
openclaw pairing list telegram
```

如果出现 pairing code，例如：

```text
ABCD1234
```

批准它：

```bash
openclaw pairing approve telegram ABCD1234
```

然后回到 Telegram，发送：

```text
你好，你现在能工作吗？
```

如果 Bot 能回复，说明 Telegram 私聊接入成功。

需要注意：DM pairing 只授权私聊访问，不等于授权这个用户在群里控制 Bot。群聊还需要单独配置 group allowlist、`groupPolicy` 或 `groups` 等规则。

---

## 十一、改成 allowlist

第一次接入时用 `pairing` 比较方便，但长期运行建议改成 `allowlist`。

区别是：

```text
pairing：陌生用户可以发起配对流程，等待你批准
allowlist：只允许配置里的 Telegram 数字用户 ID 对话
open：允许所有人私聊，必须显式配置 allowFrom: ["*"]
disabled：忽略所有私聊
```

个人 Bot 建议使用：

```json
"dmPolicy": "allowlist"
```

并明确写入自己的 Telegram 数字 ID。

### 获取自己的 Telegram 数字 ID

可以先看日志：

```bash
openclaw logs --follow
```

然后给 Bot 发一条消息，在日志里找类似：

```text
from.id: 5272077915
```

这个数字就是 Telegram 用户 ID。

也可以调用 Telegram API：

```bash
curl "https://api.telegram.org/bot你的BotToken/getUpdates"
```

在输出里找：

```json
"from": {
  "id": 5272077915
}
```

### 配置 allowlist

编辑配置：

```bash
nano ~/.openclaw/openclaw.json
```

把 Telegram 配置改成：

```json
{
  "channels": {
    "telegram": {
      "enabled": true,
      "botToken": "你的 Telegram Bot Token",
      "dmPolicy": "allowlist",
      "allowFrom": ["5272077915"],
      "groups": {
        "*": {
          "requireMention": true
        }
      }
    }
  }
}
```

`allowFrom` 使用数字 Telegram 用户 ID。`@username` 不适合作为长期配置，升级后也可能需要 `openclaw doctor --fix` 才能解析。

保存后重启：

```bash
openclaw gateway restart
```

再测试 Telegram 私聊。

---

## 十二、ownerAllowFrom 与命令权限

如果你希望通过 Telegram 执行 owner-only 命令、审批高风险操作或使用更强的控制能力，仅仅 `allowFrom` 可能不够。官方文档强调：DM 访问权限和命令 owner 权限是两层概念。

对于单用户 Bot，建议确认配置里有明确 owner：

```json
{
  "commands": {
    "ownerAllowFrom": ["telegram:5272077915"]
  }
}
```

如果你不确定当前配置状态，可以先运行：

```bash
openclaw doctor --fix
openclaw gateway restart
```

再检查 Telegram 是否能正常对话和执行你需要的命令。

---

## 十三、群聊配置建议

如果只是个人使用，建议只使用私聊 Bot，不要一开始就加入群聊或频道。

如果确实要加入群聊，保留：

```json
"groups": {
  "*": {
    "requireMention": true
  }
}
```

这样只有在群里提及 Bot 时才会触发，例如：

```text
@your_agent_bot 帮我总结一下上面的内容
```

不建议让 Bot 自动读取所有群消息。

在 BotFather 里，建议保持 Privacy Mode 开启：

```text
/setprivacy
→ 选择你的 bot
→ Enable
```

还要记住：群组访问和私聊访问是分开的。即使某个用户已经完成 DM pairing，也不代表他自动获得群聊触发权限。

---

## 十四、提高响应速度

如果接入后能对话但反应很慢，通常不是 Telegram 慢，而是模型慢或网络链路慢。

例如使用推理模型并开启较高 reasoning：

```text
deepseek-v4-pro
think medium
```

这类模型通常会先思考再回复，延迟明显高于普通 chat 模型。

可以尝试：

```text
把 reasoning / think 从 medium 改成 low
或者换成更快的 chat / flash 模型
```

测试时在 Telegram 里发：

```text
只回复 OK
```

如果短消息也很慢，说明模型 API 或 VPS 到模型 API 的网络延迟较高。

可以同时查看日志：

```bash
openclaw logs --follow
```

并测试网络：

```bash
curl -I https://api.telegram.org
curl -I https://api.deepseek.com
```

---

## 十五、安全加固建议

OpenClaw 可以连接模型、文件、工具和聊天渠道，安全配置非常重要。

### 1. 不要把 18789 暴露到公网

确认监听地址是：

```text
127.0.0.1:18789
```

执行：

```bash
ss -lntp | grep 18789
```

理想结果是监听本地回环地址，而不是：

```text
0.0.0.0:18789
```

如果需要远程打开 Dashboard，用 SSH 隧道：

```bash
ssh -L 18789:127.0.0.1:18789 root@你的服务器IP
```

然后本地浏览器打开：

```text
http://127.0.0.1:18789
```

### 2. 防火墙只开放必要端口

至少确认 SSH 可访问：

```bash
sudo ufw allow 22/tcp
sudo ufw enable
sudo ufw status
```

如果误开放了 18789：

```bash
sudo ufw delete allow 18789/tcp
```

### 3. 长期使用 allowlist

建议长期使用：

```json
"dmPolicy": "allowlist",
"allowFrom": ["你的 Telegram 数字 ID"]
```

不要长期使用：

```json
"dmPolicy": "open"
```

如果确实要 `open`，OpenClaw 要求显式配置：

```json
"allowFrom": ["*"]
```

这类配置只适合你完全理解风险的场景。

### 4. 保护配置文件权限

OpenClaw 配置文件里可能有 Telegram Bot Token 和模型 API Key，应限制权限：

```bash
chmod 700 ~/.openclaw
chmod 600 ~/.openclaw/openclaw.json
```

检查：

```bash
ls -la ~/.openclaw/openclaw.json
```

理想权限类似：

```text
-rw------- 1 root root ... openclaw.json
```

### 5. 不要把 Token 上传 GitHub

检查本机是否有明文 token：

```bash
grep -R "TELEGRAM\|BOT_TOKEN\|OPENAI\|DEEPSEEK\|API_KEY\|botToken" ~ -n 2>/dev/null
```

如果项目目录会上传 GitHub，确保 `.env`、`openclaw.json`、`docker-compose.yml` 不包含敏感信息，或已加入 `.gitignore`。

### 6. 尽量不要长期用 root 跑

本文为了方便演示使用 root，但长期运行更建议新建普通用户：

```bash
adduser openclaw
usermod -aG sudo openclaw
su - openclaw
```

然后在普通用户下重新安装和配置 OpenClaw。

原因是 AI agent 未来可能调用工具、读取文件、执行命令。用 root 权限运行，风险会明显放大。

### 7. 运行安全审计

```bash
openclaw security audit
openclaw security audit --deep
```

如果有 warning，再逐项处理。

---

## 十六、USER.md 和 IDENTITY.md 要不要更新？

Telegram 接入成功后，OpenClaw 可能会询问是否更新：

```text
USER.md
IDENTITY.md
```

可以简单理解为：

```text
USER.md：关于用户是谁、偏好、常用场景
IDENTITY.md：关于这个 agent 自己是谁、如何回复、要遵守什么原则
```

建议只写基础信息，不要写太多敏感内容。

示例：

```text
名字：Bob
称呼：Bob
时区：Asia/Shanghai
用途：个人技术助手，用于 App 开发、AI 研发、服务器运维、Telegram 草稿整理和 AI 新闻筛选。
安全偏好：不要自动执行高风险操作；涉及删除文件、修改系统服务、公开发布、发送消息、暴露 token/API key 的操作，必须先征求确认。
回复语言：默认使用中文。
```

---

## 十七、常用命令汇总

查看版本：

```bash
openclaw --version
```

运行检查：

```bash
openclaw doctor
openclaw doctor --fix
```

查看 Gateway 状态：

```bash
openclaw gateway status
```

重启 Gateway：

```bash
openclaw gateway restart
```

强制重装 Gateway service：

```bash
openclaw gateway install --force
```

查看端口：

```bash
ss -lntp | grep 18789
```

测试 Dashboard：

```bash
curl -i http://127.0.0.1:18789/
```

查看日志：

```bash
openclaw logs --follow
```

或：

```bash
journalctl --user -u openclaw-gateway -n 100 --no-pager
```

查看 Telegram pairing：

```bash
openclaw pairing list telegram
```

批准 Telegram pairing：

```bash
openclaw pairing approve telegram 你的CODE
```

安全审计：

```bash
openclaw security audit --deep
```

---

## 十八、完整最小流程

如果只想快速部署，可以按下面最小流程执行：

```bash
sudo apt update
sudo apt install -y curl git ca-certificates build-essential nano

curl -fsSL https://openclaw.ai/install.sh | bash

openclaw onboard --install-daemon

openclaw gateway status
ss -lntp | grep 18789
curl -i http://127.0.0.1:18789/

nano ~/.openclaw/openclaw.json

openclaw gateway restart
openclaw gateway status
```

Telegram 侧：

```text
BotFather → /newbot 或 /mybots
复制 Bot Token
写入 ~/.openclaw/openclaw.json
Telegram 私聊 Bot → /start
服务器执行 openclaw pairing list telegram
服务器执行 openclaw pairing approve telegram CODE
```

跑通后改成：

```json
"dmPolicy": "allowlist",
"allowFrom": ["你的 Telegram 数字 ID"]
```

如需 owner 命令权限，再确认：

```json
"commands": {
  "ownerAllowFrom": ["telegram:你的 Telegram 数字 ID"]
}
```

---

## 十九、结语

这次部署的核心经验是：

```text
先让 Gateway 正常运行
再接 Telegram
先用 pairing 跑通
再改 allowlist 加固
不要暴露 18789
不要长期用 root 跑高权限 agent
```

OpenClaw 的安装并不复杂，但它涉及 Gateway、模型 Provider、Telegram Bot、systemd 用户服务、配置文件和安全策略。第一次部署时不要追求一步到位，先抓住这几条主线：

```text
Gateway 正常监听 127.0.0.1:18789
Telegram Bot Token 配好
DM pairing / allowlist 正确
ownerAllowFrom 按需配置
模型 Provider 可用
```

这样基本就可以稳定使用。

---

## 参考链接

- [OpenClaw 官方文档](https://docs.openclaw.ai/)
- [OpenClaw Onboard CLI](https://docs.openclaw.ai/cli/onboard)
- [OpenClaw Telegram Channel](https://docs.openclaw.ai/channels/telegram)
- [OpenClaw Pairing](https://docs.openclaw.ai/channels/pairing)
- [OpenClaw Gateway Configuration](https://docs.openclaw.ai/gateway/configuration)
- [OpenClaw Security](https://docs.openclaw.ai/gateway/security)
- [OpenClaw GitHub](https://github.com/openclaw/openclaw)
