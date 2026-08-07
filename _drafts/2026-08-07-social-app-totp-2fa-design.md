---
layout: post
title: "社交 App TOTP 双因素认证（2FA）技术设计：从原理到上线"
date: 2026-08-07 00:00:00 +0800
categories: [Security]
tags: [TOTP, 2FA, MFA, 双因素认证, RFC 6238, 安全设计, Authenticator, 认证]
---

> **文档类型**：Technical Design / Security Design
> **适用系统**：前后端分离社交 App（Mobile + Web）
> **认证方式**：Password + TOTP Authenticator
> **兼容客户端**：Google Authenticator、Microsoft Authenticator、1Password、Authy、Bitwarden 及其他兼容 RFC 6238 的 Authenticator
> **状态**：设计方案（可用于开发落地）
> **优先级**：High

## 摘要

当前社交 App 仅靠「账号 + 密码」登录，密码一旦泄露、被撞库或在第三方网站复用，账号就面临被接管的风险。本文是一份面向前后端分离架构社交 App 的 **TOTP（Time-based One-Time Password，基于时间的一次性密码）双因素认证（2FA）技术设计文档**，覆盖从背景、原理、架构、开启与登录全流程，到数据库、加密、限流、审计、测试与上线灰度的完整设计，可作为研发团队直接落地的参考。

**适合读者**：

- 后端 / 架构师：重点看流程编排、数据库设计、加密与并发控制；
- 前端 / 客户端开发：重点看二维码、Manual Key、验证码输入与 UX；
- 安全工程师：重点看防重放、防暴力、防降级、日志脱敏与账号恢复。

**核心设计决策（TL;DR）**：

- 采用 RFC 6238 标准 TOTP，不绑定任何 Authenticator 厂商，也不需要集成厂商 SDK；
- 参数统一为 SHA1 / 6 位 / 30 秒 / 160-bit 随机 Secret / Base32 编码 / ±1 验证窗口；
- Secret 必须**加密存储**（AES-256-GCM + KMS 信封加密），不能做不可逆哈希；
- 登录改为两阶段：密码通过 → 下发 MFA Challenge → TOTP 验证通过后才签发正式 Token；
- 完整覆盖防重放（`lastAcceptedStep` + 数据库原子更新）、三层限流、Recovery Code 与账号恢复流程；
- 末尾附测试方案、上线检查清单与灰度策略，保证方案可落地、可验收。

------

## 一、背景与目标

### 1.1 背景

当前社交 App 已经具备基础账号认证能力，用户通过「账号 + 密码」登录系统。随着用户规模增长和账号价值的提升（例如创作者、企业号、管理员账号），仅靠密码认证面临以下风险：

- **密码泄露**：数据库被拖库、社工或钓鱼导致密码外泄；
- **撞库攻击**：攻击者利用其他网站泄露的账号密码批量尝试本系统登录；
- **弱密码攻击**：用户使用简单密码，可被暴力枚举或字典攻击；
- **第三方网站密码复用**：用户在多站共用同一密码，一处泄露、处处遭殃；
- **账号被盗 / 高价值账号被接管**：密码一旦失守，账号即被完全控制；
- **管理员账号被攻击**：后台权限账号若被攻破，影响面是系统级的。

为降低上述风险，系统计划增加基于 TOTP 的双因素认证能力。用户开启 2FA 后，登录流程由：

```text
账号 + 密码
```

变为：

```text
账号 + 密码
      ↓
Authenticator 6 位动态验证码
      ↓
完成登录
```

其中 Authenticator 可以是 Google Authenticator、Microsoft Authenticator、1Password、Authy、Bitwarden，以及其他任何兼容 TOTP / RFC 6238 的应用。

> **为什么选择 TOTP 而不是其他 2FA 方式？**
> - 标准成熟、免费、生态兼容性最好，所有主流 Authenticator 开箱即用；
> - 验证码在客户端本地生成，服务器无需发送短信 / 邮件，成本低且不易被运营商拦截；
> - SMS / Email OTP 都存在被拦截、被钓鱼的风险，只适合作为辅助手段；
> - Push、WebAuthn 需要厂商或平台生态支持，可作为后续演进方向（见「路线图」）。
>
> 本系统**不绑定**某一家 Authenticator，也不需要集成 Google Authenticator SDK 或 Microsoft Authenticator SDK——只需要按 RFC 6238 实现或调用成熟 TOTP 库即可。

### 1.2 设计目标

本项目需要实现以下能力：

1. 用户可以**主动开启** TOTP 2FA；
2. 后端生成唯一、高熵的 TOTP Secret；
3. 前端展示**二维码**和**手动输入密钥（Manual Key）**；
4. Authenticator 扫描二维码后生成动态验证码；
5. 用户输入验证码**确认绑定**后，Factor 才正式生效；
6. 登录时密码验证成功后，要求输入 TOTP；
7. **TOTP 验证成功后才签发完整登录 Token**（Access + Refresh）；
8. 支持 Recovery Code（恢复码），用于丢失 Authenticator 时登录；
9. 支持关闭 2FA（需重新验证身份）；
10. 支持更换 Authenticator（安全迁移，不锁死账号）；
11. 支持手机丢失后的账号恢复流程（含冷静期）；
12. 支持验证码错误次数限制（防暴力破解）；
13. 防止同一个 TOTP 被重复使用（防重放）；
14. TOTP Secret 加密保存；
15. 所有敏感操作均记录安全审计日志；
16. 对用户发送 2FA 开启、关闭、恢复等安全通知。

### 1.3 非目标

第一阶段**不包含**：

- 自研 Authenticator App（无必要，标准 TOTP 已兼容）；
- SMS OTP 作为主 2FA（可拦截、成本高）；
- Email OTP 作为主 2FA（可拦截，安全性弱）；
- Push Authentication（依赖厂商推送链路，第一阶段不引入）；
- Hardware Security Key（硬件成本与支持成本高）；
- Passkey / WebAuthn（依赖平台生态，作为第三阶段演进）。

后续可在密码之上叠加多种因素：

```text
Password
   +
 ┌───────────────┐
 │ TOTP          │
 │ Passkey       │
 │ Security Key  │
 └───────────────┘
```

对于管理员、高价值创作者、企业账号等高风险账户，未来**建议优先支持 Passkey / WebAuthn**，因为 TOTP 无法抵御实时钓鱼攻击（攻击者可在钓鱼页面同时骗取密码和验证码）。TOTP 仍是当前性价比最高、覆盖面最广的 2FA 方案。

------

## 二、TOTP 原理与参数

### 2.1 TOTP 基本原理

本方案采用 **RFC 6238** 标准 TOTP。Authenticator 与服务器共享同一个 Secret，各自根据当前时间独立计算出相同的 6 位验证码，**双方不需要任何网络通信**：

```text
            Secret
             │
       ┌─────┴─────┐
       │           │
       ▼           ▼
Authenticator    Server
       │           │
Current Time    Current Time
       │           │
       └─────┬─────┘
             │
         TOTP 算法
             │
             ▼
          123456
```

只要满足以下三个条件，双方就能独立算出相同验证码：

```text
Secret 相同
+
时间基本同步
+
算法参数相同
```

TOTP 的数学本质是 HOTP（RFC 4226）：以 `counter = floor(unixTime / period)` 作为计数，用 `HMAC-SHA1(Secret, counter)` 计算结果并进行动态截断（Dynamic Truncation），最终得到 6 位数字。因为计数器随时间推进，验证码每 30 秒变化一次，旧验证码自然失效。

### 2.2 推荐 TOTP 参数

全系统统一使用以下参数，避免多套参数互相不兼容：

```yaml
type: TOTP
algorithm: SHA1
digits: 6
period: 30
secretLength: 20 bytes
secretEncoding: Base32
validationWindow: [-1, 0, +1]
```

| 参数 | 值 | 说明 |
|---|---:|---|
| Algorithm | SHA1 | Authenticator 兼容性最好，所有主流 App 均支持 |
| Digits | 6 | 标准 6 位动态验证码 |
| Period | 30 秒 | 每 30 秒更新一次 |
| Secret | 160 bit（20 字节） | 密码学随机数，熵足够且为 Authenticator 常见标准 |
| Encoding | Base32 | RFC 4648 Base32，Authenticator 标准格式 |
| Window | ±1 | 容忍客户端少量时间误差（前 1 个窗口 + 当前 + 后 1 个窗口） |

> **为什么验证窗口是 ±1？**
> 用户手机时钟可能偏差几秒到几十秒。允许「前 1 个窗口 + 当前窗口」即可覆盖绝大多数情况（用户通常不会故意把时钟调慢很多），同时把「未来窗口」限制在 +1，避免验证码被提前使用。**不要轻易扩大到 ±3**：窗口越大，同一时刻"有效"的验证码越多，留给攻击者尝试的空间越大。如果需要更大容错，更稳妥的做法是允许「过去 1~2 个窗口、未来 0~1 个窗口」。

### 2.3 常见误区：SHA1 ≠ 弱哈希

这里使用的 HMAC-SHA1 是 TOTP 算法的一部分，**并不是用 SHA1 来存储密码**。TOTP 中的 SHA1 只参与一次性验证码的计算，不承担长期机密存储的职责；且由于 Secret 是 160-bit 高熵随机值，HMAC-SHA1 在实际攻击面中是安全的（这也是 RFC 6238 的默认配置）。

而**密码本身的存储必须使用慢哈希**，例如：

```text
Argon2id

或

bcrypt / scrypt
```

两者是两件完全不同的事，不要混淆，更不要因为"TOTP 用了 SHA1"就放松对密码哈希的要求。

------

## 三、整体架构

### 3.1 系统架构总览

```text
┌──────────────────────────────────────────────┐
│          Mobile App / Web Frontend           │
│    （二维码 / Manual Key / OTP / 恢复码）      │
└──────────────────────┬───────────────────────┘
                       │ HTTPS
                       ▼
┌──────────────────────────────────────────────┐
│                API Gateway                   │
└──────────────────────┬───────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────┐
│                Auth Service                  │
│                                              │
│   Password Auth / MFA Enrollment / TOTP      │
│   Verification / MFA Challenge / Recovery    │
│   Rate Limiting / Security Audit             │
└───────┬──────────────┬──────────────┬────────┘
        │              │              │
        ▼              ▼              ▼
   (User DB)        (Redis)      (KMS / Key Vault)
        │
        ├── Users
        ├── MFA Factor
        ├── Recovery Codes
        └── Security Audit
```

核心职责划分：

```text
Frontend
    ↓
API
    ↓
Auth Service
    ├── Password Authentication
    ├── MFA Enrollment
    ├── TOTP Verification
    ├── MFA Challenge
    ├── Recovery Codes
    ├── Rate Limiting
    └── Security Audit
            ↓
      Database / Redis / KMS
```

### 3.2 后端技术组件

后端需要的基础组件：

```text
TOTP Library
+
Cryptographic Random Generator
+
AES-GCM / KMS
+
Database
+
Redis
+
现有认证框架
```

不同语言都可以使用成熟、经过审计的 TOTP 库，**不要在业务代码里自己实现 TOTP 算法**：

| 技术栈 | 推荐组件 |
|---|---|
| Node.js | `otplib`，二维码可用 `qrcode`，随机数用 Node `crypto`，缓存用 Redis |
| Java / Spring Boot | `java-totp` 或 `java-otp`，二维码可用 ZXing，缓存用 Redis，加密用 KMS SDK |
| Python | `pyotp`，二维码可用 `qrcode`，加密用 `cryptography` / KMS SDK |
| Go | `github.com/pquerna/otp/totp`，随机数用 `crypto/rand`，加密用 AES-GCM / KMS |
| .NET | `Otp.NET`，二维码可用 QRCoder，密钥保护用 ASP.NET Data Protection / KMS |

> 具体选哪个库不影响整体系统设计——只要它实现 RFC 6238 且支持我们约定的参数（SHA1 / 6 位 / 30 秒 / Base32）即可。选择标准：维护活跃、API 简单、允许我们传入自己的随机数源与时钟，方便测试。

### 3.3 前端需要的能力

前端**不负责验证 TOTP**（验证永远在服务端进行），只负责展示与交互：

```text
展示二维码
+
显示 Manual Secret
+
接收 6 位验证码
+
调用后端 API
+
展示 2FA 状态
+
管理 Recovery Code
```

**二维码由谁生成？** 两种方式都可以：

1. **前端生成**：后端返回 `otpauth://...` URI，前端本地渲染二维码。移动端方案：
   - Flutter：`qr_flutter`
   - React Native：`react-native-qrcode-svg`
   - Web：`qrcode` / `qrcode.react` 等
2. **后端生成 PNG**：后端直接返回二维码图片：

```text
Backend
    ↓
QR PNG
    ↓
Frontend Image
```

这种情况下移动端甚至不需要二维码库。推荐优先用**方式 1**（前端渲染），因为后端不需要引入图片生成依赖，且二维码内容只在客户端内存中存在，更容易控制缓存与泄露面。

------
## 四、开启 2FA 完整流程

### 4.1 用户状态模型

用户的 TOTP Factor 建议设置三种状态，而不是简单的"开启 / 未开启"：

```text
PENDING
ACTIVE
REVOKED
```

状态机：

```text
[初始] ── 生成 Secret 并保存 ──> PENDING
PENDING ── 首次验证码验证成功 ──> ACTIVE
PENDING ── 超时 / 用户取消 ──> REVOKED
ACTIVE  ── 关闭 / 换绑 / 管理员重置 ──> REVOKED
```

> **为什么不能"生成 Secret 后立即 ACTIVE"？**
> 因为用户完全有可能：没有扫描二维码、Authenticator 配置失败、页面被关闭、网络中断。如果 Secret 一生成就算"已开启"，用户会在不知情的情况下被锁定，或者留下一个"名义上开启、实际从未绑定"的假 2FA。
>
> 正确流程必须是：

```text
生成 Secret
↓
PENDING
↓
用户输入 Authenticator Code
↓
后端验证成功
↓
ACTIVE
```

即：**必须用用户手机上的 Authenticator 实际产出的验证码来证明"绑定成功"**，这一步也是防呆、防误开启的关键。

### 4.2 开启前置身份验证

用户入口：

```text
Settings
→ Security
→ Two-Factor Authentication
→ Enable
```

开启 2FA 属于**敏感操作**，不能仅凭已有的 Access Token 完成——因为 Access Token 可能来自被偷的会话，攻击者会"帮用户"开启 2FA 从而锁死用户。因此建议要求用户**重新验证密码**：

```text
Enter your password
```

如果未来已经存在多个 MFA Factor，则升级为：

```text
Password
+
Existing MFA
```

（多因素场景下，修改安全设置至少需要验证"密码 + 现有任一 MFA"。）

### 4.3 创建 TOTP Enrollment

API：

```http
POST /api/v1/security/mfa/totp/enroll
Authorization: Bearer <accessToken>
Content-Type: application/json
```

Request：

```json
{
  "password": "********"
}
```

后端执行步骤：

```text
1. 检查 Access Token（是否有效、是否过期）
2. 获取 userId
3. 验证用户密码（防 CSRF / 防会话劫持场景下的误操作）
4. 检查是否已有 ACTIVE TOTP（已有则返回 MFA_ALREADY_ENABLED）
5. 删除过期的 PENDING Enrollment（避免堆积）
6. 生成随机 Secret（CSPRNG，20 字节）
7. Secret Base32 Encode
8. Secret 加密（AES-GCM / KMS，见「数据与存储设计」）
9. 保存 PENDING Factor
10. 创建 otpauth URI
11. 返回 Enrollment 信息（otpauth URI + Manual Key + 有效期）
```

> **补充建议**：对该接口本身也要做限流（例如 `mfa:enroll:user:{userId}`，见「Redis 设计」），防止攻击者反复生成大量 PENDING 记录或骚扰用户。

### 4.4 Secret 生成

Secret 是 TOTP 安全性的根基，**必须使用密码学安全随机数**：

```text
20 bytes = 160 bit
```

伪代码：

```pseudo
secretBytes = secureRandom(20)

secretBase32 = base32Encode(secretBytes)
```

**禁止**使用以下弱随机来源：

```pseudo
secret = random()            // 非密码学随机
secret = UUID only           // UUID 有结构，熵不足
secret = currentTimestamp    // 可预测
secret = MD5(userId)         // 与用户数据相关，可被推断
```

必须使用平台提供的 CSPRNG：

```text
SecureRandom      // Java
crypto/rand       // Go
crypto.randomBytes // Node.js
secrets           // Python
RandomNumberGenerator.GetBytes // .NET
```

> 20 字节（160 bit）的随机熵足以抵御暴力枚举。若未来要升级到更高级的 Authenticator 协议，可考虑 32 字节，但当前 20 字节完全满足 RFC 6238 且与所有主流 Authenticator 兼容。

### 4.5 otpauth URI

标准格式：

```text
otpauth://totp/{issuer}:{account}
?secret={secret}
&issuer={issuer}
&algorithm=SHA1
&digits=6
&period=30
```

示例：

```text
otpauth://totp/MySocialApp:john@example.com?secret=JBSWY3DPEHPK3PXPJBSWY3DPEHPK3PXP&issuer=MySocialApp&algorithm=SHA1&digits=6&period=30
```

要点：

- `issuer` 与 `account` 必须做 **URL Encode**（百分号编码），否则含特殊字符的账号会导致 URI 解析错误；
- 推荐 `issuer = App 品牌名`，`account = 用户 Email 或稳定用户名`，例如 `MySocialApp:john@example.com`；
- **不要**在 URI 中暴露数据库 User ID、手机号、内部 UUID 等，除非业务确有需要；
- 部分 Authenticator 会校验 `issuer` 参数与标签前缀是否一致（防止 URI 混淆攻击），所以两端要保持一致。

### 4.6 Enrollment API Response

示例响应：

```json
{
  "success": true,
  "data": {
    "enrollmentId": "mfa_enroll_018fa57d",
    "otpauthUri": "otpauth://totp/...",
    "manualKey": "ABCD EFGH IJKL MNOP QRST UVWX YZAB CDEF",
    "issuer": "MySocialApp",
    "accountName": "john@example.com",
    "expiresIn": 600
  }
}
```

其中 `manualKey` 是 20 字节 Secret 的 Base32 表示（160 bit = 32 个 Base32 字符），按 4 位一组展示便于手动输入。

- `enrollmentId` 建议有效期 **10 分钟**（过期后需重新发起 enroll）；
- 二维码 URI 本质上包含**永久有效的 Secret**，属于高度敏感数据；
- HTTP Response 必须携带：

```http
Cache-Control: no-store
Pragma: no-cache
```

防止代理、浏览器缓存敏感数据。

### 4.7 前端 Enrollment 页面

推荐 UI 布局：

```text
Enable Authenticator

1. Open your Authenticator app
2. Scan this QR code

┌─────────────────┐
│                 │
│     QR CODE     │
│                 │
└─────────────────┘

Can't scan?

Setup key:
ABCD EFGH IJKL MNOP QRST UVWX YZAB CDEF

[Copy]

Enter the 6-digit code:

[ _ _ _ _ _ _ ]

[Verify & Enable]
```

交互建议：

- 二维码下方始终显示 **Manual Key + Copy 按钮**；
- 验证码输入框支持粘贴、自动去掉空格、数字键盘；
- 展示"验证码每 30 秒刷新"的倒计时提示，避免用户用过期验证码反复失败；
- 二维码页面建议**禁止截图**（Android 可用 `FLAG_SECURE`），详见「截图保护」。

### 4.8 为什么必须提供 Manual Key

用户很可能在**同一台手机**上同时运行社交 App 和 Authenticator。这种情况下，用户无法让摄像头去扫描自己当前屏幕上的二维码。

因此必须同时提供：

```text
Manual Setup Key
```

并支持：

```text
Copy
```

**不建议鼓励用户截图保存二维码**，因为二维码内容就是永久 Secret。截图可能被：

- Google Photos / iCloud Photos
- 第三方相册
- 各类云备份服务

自动上传到云端，等于把 Secret 泄露出去。正确姿势是：能扫码就扫码，不能扫码就手动输入 Manual Key，并在输入时提醒用户"此密钥仅显示一次"。

### 4.9 首次验证码验证

接口：

```http
POST /api/v1/security/mfa/totp/enroll/confirm
Authorization: Bearer <accessToken>
```

Request：

```json
{
  "enrollmentId": "mfa_enroll_018fa57d",
  "code": "123456"
}
```

后端流程：

```text
查询 Enrollment
↓
检查 userId（必须是本人）
↓
检查状态 = PENDING
↓
检查是否过期
↓
解密 Secret
↓
验证 TOTP
↓
成功
↓
Factor → ACTIVE
↓
生成 Recovery Codes
```

### 4.10 TOTP 验证算法

设当前 Unix 时间戳为 `T`，则时间窗口：

```text
timeStep = floor(unixTime / 30)
```

为容忍轻微时间误差，验证以下三个窗口：

```text
timeStep - 1
timeStep
timeStep + 1
```

即窗口 `[-1, 0, +1]`。若匹配到任一窗口，则把 `matchedStep` 记为已验证窗口，用于后续防重放（见「防止 TOTP 重复使用」）。

> **两个实现细节**：
> 1. **常量时间比较**：验证码比对应使用常量时间比较（constant-time compare），避免通过响应时间推断出部分正确信息（理论上 6 位码空间小，时序攻击影响有限，但养成好习惯成本很低）；
> 2. **不要轻易扩大到 ±3**：窗口越大，同一时刻"有效"的验证码越多，攻击者的尝试空间越大。

### 4.11 首次启用事务

启用 2FA 涉及多张表的写入（Factor、Recovery Codes、Audit），**必须使用数据库事务**保证原子性：

```pseudo
BEGIN TRANSACTION

factor = findPendingFactor(
    enrollmentId,
    userId
)

if factor == null:
    return MFA_ENROLLMENT_NOT_FOUND

if factor.expiresAt < now:
    return MFA_ENROLLMENT_EXPIRED

secret = decrypt(factor.secret)

matchedStep = verifyTotp(
    secret,
    code,
    window = [-1, 0, +1]
)

if matchedStep == null:
    registerFailure()
    return INVALID_TOTP

activate(factor)

factor.lastAcceptedStep = matchedStep

revokeOtherPendingFactors(userId)

recoveryCodes = generateRecoveryCodes(userId)

writeSecurityAudit(
    userId,
    MFA_TOTP_ENABLED
)

COMMIT
```

> 注意顺序：先验证、后激活；`lastAcceptedStep` 在激活的同时写入，避免并发重复确认；其他 PENDING Factor 一并作废，保证"一个用户最多一个待确认 Factor"。

------

## 五、Recovery Code（恢复码）

### 5.1 Recovery Code 生成与展示

用户成功开启 TOTP 后，**立即**生成恢复码并展示（只展示一次）：

- 数量：**10 个**
- 每个：**12～16 位高熵随机字符**（建议去掉易混淆字符，如 `0/O`、`1/I`）

示例：

```text
W73Q-MG2K-NP8X
D92M-KH7R-WX4Q
K61Y-PL9D-FA7T
...
```

前端展示：

```text
Recovery Codes

These codes can be used if you lose access
to your Authenticator.

Each code can only be used once.

[Download]
[Copy]
```

交互要点：

- 页面必须提醒"**仅显示一次**，请立即保存"；
- 提供 Download / Copy，方便用户离线保存；
- 离开页面后，前端不得再展示明文恢复码；
- 若用户**重新生成**恢复码，旧的恢复码必须全部作废（见「Recovery Code 必须单次使用」与「重新生成恢复码」）。

### 5.2 Recovery Code 数据库存储

数据库**不能保存明文**恢复码——数据库一旦泄露，恢复码就等于密码。

正确做法：

```text
Recovery Code
      ↓
Secure Hash
      ↓
Database
```

推荐两种哈希方案：

```text
Argon2id
```

或

```text
HMAC-SHA256（使用服务器端保密 Key）
```

- Argon2id：对每个恢复码做慢哈希，安全但入库 / 校验开销略高（10 个恢复码 × 每次登录校验，可接受）；
- HMAC-SHA256 + 服务端 Key：速度快、实现简单，前提是 Key 必须单独保管（如 KMS / 环境变量 + 密钥管理）。

两者都满足"恢复码**只显示一次**，后端无法恢复明文"的要求——这意味着恢复码一旦展示给用户，后台和客服都无法再查询到明文，这是设计预期，不是缺陷。

------
## 六、登录流程

### 6.1 正常登录流程总览

原流程：

```text
Password
   ↓
Access Token
```

修改为（两阶段）：

```text
App                     Auth Service              Database
 │── username + password ──>│                        │
 │                          │── verify user/password─>│
 │                          │<── result ─────────────│
 │                          │                        │
 │  （MFA 未开启）           │── 直接签发 Token ──────>│
 │<── Access + Refresh ─────│                        │
 │                          │                        │
 │  （MFA 已开启）           │── 下发 MFA Challenge ──>│
 │<── MFA_REQUIRED ─────────│                        │
 │── challenge + code ──────>│                        │
 │                          │── verify TOTP factor ──>│
 │                          │<── result ─────────────│
 │<── Access + Refresh ──────│                        │
```

> 关键原则：**密码验证通过 ≠ 登录成功**。必须等 MFA 也验证通过，才视为完整登录。

### 6.2 登录第一阶段

API：

```http
POST /api/v1/auth/login
```

Request：

```json
{
  "username": "john@example.com",
  "password": "********"
}
```

**未开启 MFA**：

```json
{
  "success": true,
  "data": {
    "accessToken": "...",
    "refreshToken": "..."
  }
}
```

**已开启 MFA**：

```json
{
  "success": true,
  "data": {
    "status": "MFA_REQUIRED",
    "mfaChallenge": "mfac_018fa72d",
    "methods": [
      "TOTP",
      "RECOVERY_CODE"
    ],
    "expiresIn": 300
  }
}
```

> **非常重要**：密码验证通过但 MFA 尚未完成时，**不能签发任何正常 Access Token / Refresh Token**，只能签发一个"仅能用于完成 MFA 挑战"的凭据。否则攻击者拿到密码后即可绕过 MFA 直接登录。

### 6.3 MFA Challenge

MFA Challenge 推荐存在 **Redis**，便于设置 TTL、并发访问与跨节点共享：

```text
Key:

mfa:challenge:{challengeId}
```

Value：

```json
{
  "userId": "user_xxx",
  "purpose": "LOGIN",
  "passwordVerified": true,
  "createdAt": 1723000000,
  "expiresAt": 1723000300,
  "attempts": 0,
  "ip": "203.0.113.1",
  "deviceId": "device_xxx"
}
```

TTL：**5 分钟**（与 `expiresIn: 300` 一致）。

### 6.4 Challenge 安全要求

Challenge 必须：

- 使用密码学随机 ID（不可猜测，如 `mfac_` + 随机 128-bit）；
- 短时间有效（TTL 5 分钟）；
- **单次使用**（验证成功后立即删除）；
- 与 `userId` 绑定；
- 与认证用途绑定（`purpose`，登录 / 敏感操作等场景分开）；
- 有最大失败次数（默认 5 次）；
- 成功后立即删除；
- **不能**直接访问普通业务 API。

禁止设计类似这样的接口：

```text
/password-success-token
```

即"密码验证通过后返回一个与正式 Access Token 同等权限的令牌"。这等于把 MFA 变成摆设。

### 6.5 TOTP 登录验证接口

```http
POST /api/v1/auth/mfa/totp/verify
```

Request：

```json
{
  "mfaChallenge": "mfac_018fa72d",
  "code": "123456"
}
```

验证链路：

```text
Challenge
↓
User
↓
TOTP Factor
↓
Secret Decrypt
↓
TOTP Verify
↓
Replay Protection
↓
Challenge Consume
↓
Issue Tokens
```

成功：

```json
{
  "success": true,
  "data": {
    "accessToken": "...",
    "refreshToken": "...",
    "expiresIn": 3600
  }
}
```

### 6.6 防止 TOTP 重复使用

这是**极易被忽视**的实现细节。假设验证码 `123456` 在 `10:10:00 ～ 10:10:29` 有效，攻击者截获后可能立即重复提交。因此需要为每个 Factor 保存：

```text
lastAcceptedStep
```

例如：

```text
57291821
```

验证规则：当某个时间窗口已经被成功使用过：

```text
currentStep <= lastAcceptedStep
```

则直接拒绝，返回：

```text
CODE_ALREADY_USED
```

这样同一窗口的验证码即使被截获，也只能用一次。

### 6.7 防止并发 Replay

不能只做简单的读-改-写：

```pseudo
if step > lastStep:
    updateLastStep()
```

因为两个并发请求可能同时通过检查。必须使用**数据库原子更新**：

```sql
UPDATE mfa_totp_factor
SET last_accepted_step = :step
WHERE id = :factor_id
AND (
    last_accepted_step IS NULL
    OR last_accepted_step < :step
);
```

然后检查**受影响行数**：

- `affected rows = 1`：本次验证码第一次被接受，继续后续流程；
- `affected rows = 0`：验证码已经被（并发地）使用过，返回 `CODE_ALREADY_USED`。

> 补充：也可以再用 Redis 做一层**短时防重放**（例如 `SET mfa:used:step:{factorId}:{step} NX EX 60`），作为数据库层的加速与兜底，但**数据库原子更新是必须的主防线**，不能省略。

### 6.8 登录验证码限流

6 位验证码理论空间只有：

```text
000000 ～ 999999
```

约 **100 万**种组合。如果完全不限流，攻击者每秒尝试几十次，几小时内就能把某个账号的当前窗口枚举完。因此**必须防暴力破解**，建议至少三层限流：

```text
Account（账号级）
+
Challenge（挑战级）
+
IP（IP 级）
```

三层限流都要实现，缺一不可：Challenge 层挡住"单次挑战内狂试"，Account 层挡住"换挑战继续试"，IP 层挡住"批量扫号"。

### 6.9 Challenge 限制

建议：

```text
一个 Challenge 最多 5 次验证码尝试
```

例如：

```text
attempts >= 5
```

则：

```text
Challenge Expired
```

用户需要重新输入密码，发起新的登录（即重新走一遍密码 → MFA）。

### 6.10 Account 级限流

Redis Key：

```text
mfa:failure:user:{userId}
```

示例策略：

```text
5 min / 10 attempts
```

连续失败后增加等待时间（渐进式退避）：

```text
1～3 次：正常
4 次：等待 10 秒
5 次：等待 30 秒
6 次：等待 60 秒
...
```

**关键**：不要因为用户生成了一个新的 Challenge，就清空 Account Failure Counter。否则攻击者可以：

```text
创建 Challenge
↓
尝试 5 次
↓
重新创建 Challenge
↓
再尝试 5 次
```

无限循环，直接绕过限流。Account 级计数器必须以 `userId` 为维度、以较长时间（如 15 分钟～1 小时）为窗口独立统计。

### 6.11 IP 级限流

同时限制：

```text
mfa:failure:ip:{ip}
```

防止单个 IP 批量攻击多个账号：

```text
User A
User B
User C
...
```

IP 限流阈值可以比账号级宽松一些（例如 15 分钟 / 50 次失败），主要用来阻断"分布式批量扫号"，而不是惩罚正常用户。

### 6.12 登录页面 UX

密码验证成功后进入：

```text
Two-Factor Authentication

Enter the 6-digit code from your
Authenticator app.

[ _ _ _ _ _ _ ]

[Verify]

Can't access your Authenticator?

Use a recovery code
```

验证码输入框建议：

- 数字键盘（Numeric Keyboard）；
- 固定 6 位；
- 支持粘贴；
- 自动去掉空格 / 分隔符；
- 不自动持久化输入内容；
- 不写入 Crash Log；
- 不写入 Analytics；
- 不写入 Debug Log。

是否自动提交（输入满 6 位自动 Verify）可以根据团队 UX 习惯决定，但必须同时提供显式的 [Verify] 按钮，并处理好"Challenge 已过期"的情况（用户输入过程中挑战过期时，给出友好提示并引导重新登录）。

### 6.13 Recovery Code 登录

接口：

```http
POST /api/v1/auth/mfa/recovery/verify
```

Request：

```json
{
  "mfaChallenge": "mfac_xxx",
  "recoveryCode": "W73Q-MG2K-NP8X"
}
```

后端流程：

```text
1. Validate Challenge
2. 查询未使用 Recovery Codes
3. Verify Hash
4. Atomic Mark Used
5. Consume Challenge
6. Issue Login Tokens
7. Security Notification
```

> Recovery Code 登录同样要走 MFA Challenge（即必须先输对密码），不能"直接用恢复码登录"绕过密码；并且恢复码登录是**高危信号**，应当触发安全通知与风控关注（见「安全通知」）。

### 6.14 Recovery Code 必须单次使用

数据库字段：

```text
usedAt
```

成功使用后：

```text
NULL
↓
2026-08-07T11:00:00Z
```

再次提交同一个恢复码：

```text
拒绝
```

并记录安全事件：

```text
RECOVERY_CODE_REPLAY_ATTEMPT
```

> 防并发：与 TOTP 防重放同理，标记已使用要使用**原子更新**（`UPDATE ... SET used_at = now() WHERE code_hash = :hash AND used_at IS NULL`，检查受影响行数），避免并发提交同一个恢复码都成功。

------

## 七、关闭、更换与恢复

### 7.1 用户关闭 2FA

入口：

```text
Settings
→ Security
→ Two-Factor Authentication
→ Disable
```

关闭 2FA 是**降低安全等级**的操作，绝不能仅凭一个 `POST /disable + Bearer Token` 就执行——否则攻击者一旦拿到会话，就能一键关掉 MFA 再为所欲为。推荐重新验证：

```text
Password
+
Current TOTP
```

API：

```http
POST /api/v1/security/mfa/totp/disable
```

Request：

```json
{
  "password": "********",
  "code": "123456"
}
```

成功后：

```text
TOTP Factor → REVOKED
Recovery Codes → REVOKED
所有未完成 MFA Challenge → 删除
Security Audit → 记录（MFA_TOTP_DISABLED）
Security Notification → 发送（MFA Disabled）
```

> 补充建议：关闭 2FA 属于高风险动作，可以再叠加"冷静期"或"通知 + 确认弹窗"。如果产品允许，关闭后建议旋转 Refresh Token、并可选择撤销其他会话（见「关闭 MFA 后的 Session 策略」）。

### 7.2 更换 Authenticator

**不要**设计成：

```text
删除旧 Authenticator
↓
配置新 Authenticator
```

因为用户在第二步失败后会陷入"旧设备已删、新设备未配"的两难，彻底锁死账号。

正确流程（安全迁移）：

```text
验证密码 ──> 验证旧 TOTP ──> 生成新 Secret ──> 新 Factor = PENDING
   ──> 用户扫描新二维码 ──> 验证新 TOTP ──> 新 Factor = ACTIVE
   ──> 旧 Factor = REVOKED
```

即：**新 Factor 验证成功之前，旧 Factor 始终保持 ACTIVE**，这样任何一步失败，用户都还能用旧 Authenticator 登录。整个替换建议使用**安全事务**控制（生成新 PENDING → 确认新 ACTIVE → 撤销旧 REVOKED 在同一事务内完成）。

### 7.3 用户丢失 Authenticator

恢复优先级：

```text
Recovery Code
↓
已有可信设备
↓
严格账号恢复流程
```

**不要**设计成"点 Forgot 2FA → 发一封 Email → 直接关闭 2FA"，否则攻击者只要攻破邮箱就能绕过 TOTP，2FA 形同虚设。

社交 App 应根据**账号价值**定义不同恢复级别，例如：

- **普通用户**：

```text
Email ownership
+
Password
+
Device history / risk check
+
Cooldown
```

- **高风险用户**（创作者、管理员、企业号）：

```text
Email
+
Identity verification（实名 / 视频 / 证件核验）
+
Manual review
+
Security cooldown
```

### 7.4 账号恢复冷静期

对于直接重置 MFA 的高风险恢复，可以考虑设置：

```text
12～24 小时 Security Cooldown
```

冷静期内：

- 给旧设备发送 Push；
- 给邮箱发送通知；
- 提供"不是本人操作"的申诉 / 中止入口；
- 限制修改密码；
- 限制绑定支付方式；
- 限制导出敏感数据。

是否实施以及时长取决于 App 风险等级——冷静期越长越安全，但对真实用户越不友好，需要产品权衡。

------

## 八、数据与存储设计

### 8.1 数据库设计原则

推荐使用**独立的 MFA Factor 表**，不要简单地把 `totp_secret` 塞进 `users` 表：

- 职责单一，便于扩展；
- 未来增加 `Passkey / Security Key / Push MFA` 时无需改 `users` 表结构；
- Secret 的加密字段、生命周期（PENDING / ACTIVE / REVOKED）与用户主数据解耦，便于做数据生命周期管理（如账号注销时清理）。

### 8.2 MFA Factor Table

PostgreSQL 示例：

```sql
CREATE TABLE mfa_totp_factor (
    id UUID PRIMARY KEY,

    user_id UUID NOT NULL,

    status VARCHAR(20) NOT NULL,

    secret_ciphertext BYTEA NOT NULL,

    secret_nonce BYTEA,

    encryption_key_id VARCHAR(128),

    algorithm VARCHAR(16)
        NOT NULL DEFAULT 'SHA1',

    digits SMALLINT
        NOT NULL DEFAULT 6,

    period_seconds SMALLINT
        NOT NULL DEFAULT 30,

    last_accepted_step BIGINT,

    pending_expires_at TIMESTAMPTZ,

    confirmed_at TIMESTAMPTZ,

    revoked_at TIMESTAMPTZ,

    created_at TIMESTAMPTZ
        NOT NULL DEFAULT NOW(),

    updated_at TIMESTAMPTZ
        NOT NULL DEFAULT NOW()
);
```

索引：

```sql
CREATE INDEX idx_mfa_totp_user
ON mfa_totp_factor(user_id);
```

保证"一个用户最多一个 ACTIVE TOTP"（PostgreSQL 部分唯一索引）：

```sql
CREATE UNIQUE INDEX uk_user_active_totp
ON mfa_totp_factor(user_id)
WHERE status = 'ACTIVE';
```

> 字段说明：`secret_nonce` 为 AES-GCM 的随机 Nonce；`encryption_key_id` 记录用哪把 KMS Key 加密，便于密钥轮换时定位需要重加密的数据；`last_accepted_step` 是防重放的核心字段（见「防止 TOTP 重复使用」）。

### 8.3 Recovery Code Table

```sql
CREATE TABLE mfa_recovery_code (
    id UUID PRIMARY KEY,

    user_id UUID NOT NULL,

    factor_id UUID NOT NULL,

    code_hash VARCHAR(255) NOT NULL,

    used_at TIMESTAMPTZ,

    revoked_at TIMESTAMPTZ,

    created_at TIMESTAMPTZ
        NOT NULL DEFAULT NOW()
);
```

索引：

```sql
CREATE INDEX idx_recovery_user
ON mfa_recovery_code(user_id);
```

> 补充：`code_hash` 建议采用 Argon2id 或 HMAC-SHA256（服务端 Key）计算，绝不明文存储；`factor_id` 关联到 Factor，方便换绑 / 关闭时统一作废。

### 8.4 Security Audit Table

建议复用已有的统一审计日志体系；若从零建表，示例：

```sql
CREATE TABLE security_audit_log (
    id UUID PRIMARY KEY,

    user_id UUID,

    event_type VARCHAR(64) NOT NULL,

    ip_address VARCHAR(64),

    device_id VARCHAR(128),

    user_agent TEXT,

    metadata JSONB,

    created_at TIMESTAMPTZ
        NOT NULL DEFAULT NOW()
);
```

> 审计日志默认**只追加、不修改、不删除**，并设置合理的保留周期；`metadata` 只放事件上下文（设备、来源、是否成功），严禁放入敏感数据。

### 8.5 Security Events

至少记录以下安全事件：

```text
MFA_ENROLL_STARTED
MFA_TOTP_ENABLED
MFA_TOTP_FAILED
MFA_TOTP_DISABLED
MFA_TOTP_REPLACED

MFA_LOGIN_SUCCESS
MFA_LOGIN_FAILED

MFA_RECOVERY_CODE_USED
MFA_RECOVERY_CODE_FAILED

MFA_RECOVERY_CODES_REGENERATED

MFA_RESET_REQUESTED
MFA_RESET_COMPLETED
```

**铁律**：Audit Log 绝不能记录：

```text
TOTP Secret
OTP Code
Recovery Code
otpauth URI
QR Code
Password
```

审计是给安全分析看的，不是给攻击者留的"资料库"。

### 8.6 Secret 为什么不能 Hash

密码可以哈希存储，是因为服务器只需要：

```text
Password
↓
Hash
↓
DB
```

验证时只需 `Verify(password)`，不需要知道原密码。

但 TOTP 完全不同，服务器每次验证 OTP 都必须用**原始 Secret** 重新计算期望值：

```text
Secret + Current Time
↓
Generate Expected OTP
```

所以：

```text
TOTP Secret
```

**不能做不可逆哈希**，必须：

```text
Encrypted Secret
```

即：可解密、可重算，但存储时不可读。

### 8.7 Secret 存储方案

推荐架构：

```text
TOTP Secret ──AES-256-GCM──> Ciphertext ──> Database
     │
     └── 密钥由 KMS / HSM 提供
```

推荐：

```text
AES-256-GCM
+
AWS KMS

或

Google Cloud KMS

或

Azure Key Vault

或

企业 HSM
```

> 选择标准：加密密钥（KEK）**绝对不能**与密文存在同一个数据库 / 同一台机器上，否则"拖库即解密"。KMS / HSM 的价值就是让密钥与数据分离、密钥不可导出、可审计、可轮换。

### 8.8 Envelope Encryption（信封加密）

更完整的生产方案：

```text
TOTP Secret
     ↓
Data Encryption Key（DEK，每用户 / 每 Factor 一把）
     ↓
AES-GCM Encrypt
     ↓
Encrypted Secret
```

DEK 再由：

```text
KMS Master Key（KEK）
```

保护（KMS 加密 DEK，产生 wrapped DEK）。数据库保存：

```text
secretCiphertext
encryptedDataKey
nonce
keyId
```

验证验证码时：

```text
DB
↓
Encrypted Secret + Encrypted DEK
↓
KMS Decrypt（unwrap DEK）
↓
Decrypt Secret
↓
Memory
↓
Verify TOTP
↓
Immediately discard（用后即焚）
```

> 信封加密的好处：KMS 调用次数少（DEK 可缓存于内存短暂复用）、单把密钥泄露影响面可控、未来可以做 KMS 主密钥轮换（仅需用新 KEK 重新包裹 DEK，无需重加密所有 Secret）。前提是 DEK 缓存生命周期要短，且进程内存中的明文 Secret 用后立即清零。

### 8.9 Secret 禁止出现的位置

TOTP Secret / otpauth URI **禁止**进入以下任何位置：

```text
Application Log
Nginx Access Log Query Params
APM
Sentry Breadcrumbs
Crashlytics
Analytics
Firebase Analytics
Datadog Tags
Mixpanel
Amplitude
Debug Console
Database Audit Dump
Support Ticket
```

因此：**不要通过 URL Query Parameter 发送 Secret**。

错误：

```http
GET /mfa/setup?secret=ABCDEF
```

正确：

```http
POST /mfa/setup
```

并在 **Response Body** 返回（配合 `Cache-Control: no-store`）。

> 原因：URL 会被记入 Nginx / 网关 / CDN 访问日志、浏览器历史、APM 的 URL 采集，是最高频的泄露通道之一。

### 8.10 二维码安全

二维码内容就是 TOTP Secret，因此**二维码不是普通 UI 资源**，需要特殊对待：

```http
Cache-Control: no-store
```

并避免：

- CDN 缓存；
- 图片缓存服务；
- 图片 URL 永久保存；
- 对象存储公开 URL；
- Analytics 自动截图；
- Server Log 记录。

最安全的做法：

```text
Backend
↓
otpauth URI
↓
Frontend Memory
↓
Render QR
```

页面退出后销毁内存中的 URI / 二维码，不落盘、不进缓存。

### 8.11 Redis 设计

推荐使用 Redis 存储以下状态：

```text
MFA Challenge
Rate Limit Counter
Pending Login State
Temporary Reauthentication
```

Key 约定示例：

```text
mfa:challenge:{challengeId}      TTL: 300 sec
mfa:fail:user:{userId}           TTL: 可更长（如 15min～1h）
mfa:fail:ip:{ip}                 TTL: 可更长
mfa:enroll:user:{userId}         TTL: 600 sec（enroll 限流）
```

- `mfa:challenge:*`：挑战数据 + TTL，天然支持过期清理；
- `mfa:fail:user:*`：账号级失败计数（**不随新 Challenge 重置**，见「Account 级限流」）；
- `mfa:fail:ip:*`：IP 级失败计数；
- `mfa:enroll:user:*`：防止反复生成大量 Secret / PENDING 记录。

> Redis 中的 Key 命名建议带版本前缀（如 `v1:`），方便后续迁移；所有敏感值在 Redis 中也不应以明文形式出现（Challenge 本身不含 Secret，但尽量保持最小化）。

------
## 九、API 设计

### 9.1 API 汇总

| 功能 | 接口 | 说明 |
|---|---|---|
| 开启 Enrollment | `POST /api/v1/security/mfa/totp/enroll` | 生成 Secret 与 Enrollment（需密码） |
| 确认 Enrollment | `POST /api/v1/security/mfa/totp/enroll/confirm` | 验证首个 TOTP，正式开启 |
| MFA 状态 | `GET /api/v1/security/mfa` | 查询 2FA 状态与剩余恢复码 |
| 登录 | `POST /api/v1/auth/login` | 密码验证，返回 Token 或 MFA Challenge |
| 登录验证 TOTP | `POST /api/v1/auth/mfa/totp/verify` | 用 TOTP 完成挑战 |
| 登录使用恢复码 | `POST /api/v1/auth/mfa/recovery/verify` | 用恢复码完成挑战 |
| 关闭 TOTP | `POST /api/v1/security/mfa/totp/disable` | 需密码 + 当前 TOTP |
| 更换 TOTP | `POST /api/v1/security/mfa/totp/replace/start`<br>`POST /api/v1/security/mfa/totp/replace/confirm` | 两段式安全迁移 |
| 重新生成恢复码 | `POST /api/v1/security/mfa/recovery/regenerate` | 需密码 + TOTP，旧码全部作废 |

### 9.2 推荐 Error Code

统一业务错误码：

```text
MFA_REQUIRED

MFA_CHALLENGE_INVALID
MFA_CHALLENGE_EXPIRED
MFA_CHALLENGE_ATTEMPTS_EXCEEDED

TOTP_INVALID
TOTP_ALREADY_USED

MFA_ENROLLMENT_INVALID
MFA_ENROLLMENT_EXPIRED
MFA_ENROLLMENT_ALREADY_IN_PROGRESS

MFA_ALREADY_ENABLED
MFA_NOT_ENABLED

RECOVERY_CODE_INVALID
RECOVERY_CODE_ALREADY_USED

RATE_LIMITED

REAUTHENTICATION_REQUIRED
```

> 不要向攻击者泄露过多内部状态（例如"你的验证码差 1 位就对"这类信息）。错误码要面向客户端做 UX 分支，但细节一律模糊化。

### 9.3 Error Response

统一响应格式：

```json
{
  "success": false,
  "error": {
    "code": "TOTP_INVALID",
    "message": "Invalid verification code."
  }
}
```

**禁止**返回类似：

```text
Secret incorrect
Matched previous timestep
Expected code = 123456
```

这类信息等于直接帮助攻击者。HTTP 状态码建议按语义区分：`400`（参数错误）、`401`（认证失败 / TOTP 错误）、`403`（权限不足 / 需重新认证）、`409`（冲突，如已开启 2FA）、`429`（限流）。

------

## 十、Token 与敏感操作

### 10.1 Token 设计（amr 声明）

已有 JWT 架构的情况下，建议在 Token 中增加 **`amr`（Authentication Methods References）** 声明，记录本次认证使用的方法：

普通密码登录：

```json
{
  "sub": "user_123",
  "amr": [
    "pwd"
  ]
}
```

完成 TOTP：

```json
{
  "sub": "user_123",
  "amr": [
    "pwd",
    "otp"
  ]
}
```

> 有了 `amr`，后续敏感操作可以精确判断"本次会话是否真正完成了 MFA"，而不是靠猜测或额外的状态查询。

### 10.2 敏感操作 Step-Up Authentication

即使用户已登录，以下操作也建议**要求重新验证 MFA**（Step-Up / 会话内二次认证）：

```text
修改密码

修改邮箱

修改手机号

关闭 2FA

重新生成 Recovery Codes

查看或导出隐私数据

删除账号

修改支付信息

修改创作者收款信息

管理员权限操作
```

实现方式：

```text
Access Token
+
Recent MFA Verification
```

要求用户**最近 5～15 分钟**内完成过 MFA 验证（例如在 Redis 中记录 `authn:recent:{userId}` 时间戳），否则弹出二次验证。

### 10.3 社交 App 特别需要保护的操作

社交产品常见的高价值行为：

```text
Change Password
Change Email
Change Phone

Disable MFA

Delete Account

Export Personal Data

Change Username

Transfer / Claim High-value Handle

Creator Monetization

Payment Account

Advertising Account

Developer / API Credentials

Moderator / Admin Actions
```

这些操作建议统一支持：

```text
Step-Up MFA
```

避免"登录时做了 MFA，但改绑邮箱这类关键操作却不验证"的安全漏洞。

------

## 十一、设备与会话

### 11.1 Trusted Device（可选）

后续可以考虑增加"信任此设备 30 天"，但**第一版可以先不做**（降低复杂度与风险面）。

如果要做，**不能**简单保存 `mfaPassed=true` 这种布尔值，而应签发**服务器控制的 Trusted Device Token**，并绑定：

```text
userId
deviceId
tokenId
createdAt
expiresAt
revokedAt
```

同时允许用户在：

```text
Settings
→ Security
→ Trusted Devices
```

查看和撤销。

> 重要：即使设备被信任，**高风险操作仍不能跳过 MFA**（见「敏感操作 Step-Up Authentication」）。

### 11.2 设备管理

建议社交 App 同步提供设备管理入口：

```text
Security
├── Two-Factor Authentication
├── Recovery Codes
├── Logged-in Devices
└── Security Activity
```

当用户怀疑账号被盗时，可以一键：

```text
Log out all other devices
```

设备管理 + MFA 配合使用，能显著提升账号恢复与风控体验。

### 11.3 Refresh Token 与 MFA

**必须完成 MFA 之后才能签发正式的 Refresh Token**。

错误：

```text
Password Success
↓
Refresh Token
↓
MFA
```

因为攻击者拿到 Refresh Token 后可以直接绕过 MFA 长期续期。

正确：

```text
Password Success
↓
MFA Challenge
↓
MFA Success
↓
Access Token + Refresh Token
```

### 11.4 关闭 MFA 后的 Session 策略

建议关闭 MFA 后：

```text
Rotate Refresh Token
```

高安全场景可以进一步：

```text
Revoke Other Sessions
```

并要求其他设备重新登录。最低限度也必须：

```text
Security Notification
```

通知用户"2FA 已被关闭"，防止攻击者悄悄降级安全设置。

------
## 十二、安全通知与管理后台

### 12.1 安全通知

以下事件建议通过 Email / Push 通知用户：

| 事件 | 文案示例 |
|---|---|
| MFA 已开启 | `Two-factor authentication was enabled on your account.` |
| MFA 已关闭 | `Two-factor authentication was disabled.` |
| Authenticator 已更换 | `Your Authenticator app was changed.` |
| 恢复码被使用 | `A recovery code was used to sign into your account.` |
| MFA 被重置 | `Two-factor authentication was reset.` |

通知中可以包含：

```text
Time
Device
Approximate Location
IP information
```

帮助用户判断"是不是我本人操作"，但**绝对不要**包含任何敏感 Secret。

### 12.2 管理后台

客服或管理员**绝对不应该**能够：

```text
查看用户 TOTP Secret
```

后台只能看到（只读）：

```text
MFA Enabled: Yes

Type: Authenticator

Enabled At: ...

Recovery Codes Remaining: ...

Last MFA Success: ...
```

> 如果后台能看到 Secret，等于把 KMS 加密的防线打穿了一半——任何后台泄露 / 内部人员泄露都可能造成批量 Secret 泄露。务必从权限模型上直接禁止。

### 12.3 管理员 Reset MFA

如果业务允许客服重置 MFA，**必须严格权限控制**，推荐带审批流：

```text
Support Agent
↓
Request MFA Reset
↓
Risk Check
↓
Privileged Approval
↓
MFA Reset
↓
Audit Log
↓
User Notification
```

**不要**设计一个普通客服按钮"Disable 2FA"点击立即生效——这是账号接管攻击的高发点。每次 Reset 都必须：记录审计、通知用户、在用户侧展示"MFA 已被重置"的安全提醒。

### 12.4 防止 MFA Downgrade

确保攻击者无法通过以下任一入口绕过 MFA：

```text
旧 API
Mobile Legacy API
Web Login API
OAuth Login
Third-party Login
Password Reset
Device Migration
```

典型漏洞示例：

```text
/api/v1/login    → 要求 MFA，没问题
/api/v0/login    → 仍直接返回 Token，严重漏洞！
```

上线前必须**逐一审查所有登录入口**，统一强制 MFA 策略，不能只改新版接口而漏掉旧版。

### 12.5 OAuth / Social Login

社交 App 很可能支持：

```text
Google Login
Apple Login
Facebook Login
```

如果用户已经在平台内开启了 TOTP，那么：

```text
OAuth Identity Success
↓
仍然要求 TOTP
```

除非产品**明确**把 OAuth Provider 的高可信认证作为替代 Factor（需要显式声明并评估风险）。

第一版建议简单统一，避免不同登录方式行为不一致：

```text
Password Login
Google Login
Apple Login
↓
如果 account.mfaEnabled
↓
MFA Challenge
```

### 12.6 Password Reset 与 MFA

**密码重置不能自动关闭 2FA。**

正确：

```text
Password Reset
↓
Password Changed
↓
MFA Still Enabled
```

错误做法是：

```text
攻破邮箱
↓
Reset Password
↓
顺便绕过 TOTP
```

这会严重降低 MFA 的价值。密码重置后，MFA 应保持开启；如果用户同时丢失了邮箱与 Authenticator，应走「用户丢失 Authenticator」的严格恢复流程，而不是悄悄关掉 2FA。

------

## 十三、基础设施与安全防护

### 13.1 时钟同步

TOTP 依赖时间，服务器必须保持时间准确：

```text
NTP
```

确保所有 Auth Server 时间一致；多节点部署时：

```text
Auth Node A
Auth Node B
Auth Node C
```

都必须同步。时间偏差可能导致：

```text
Authenticator 生成正确验证码
↓
服务器一直认为错误
```

→ 用户会误以为功能坏了，实际上是时钟漂移。建议在监控中加入"服务器时钟偏移"指标。

### 13.2 多机房部署

如果应用跨 Region（例如 US / EU / Asia），需要确保：

1. 时间同步（NTP 各机房独立但都对准 UTC）；
2. `lastAcceptedStep` **全局一致**（防重放数据不能因机房不同而"各算各的"）；
3. MFA Challenge 能跨节点读取（Redis 用 Centralized 或 Regional + 一致性策略）；
4. Rate Limit 状态一致（账号级计数全局生效）。

> 特别注意：Replay Protection 数据**必须避免**因最终一致性导致验证码在多个机房被重复接受。要么使用同一 Redis 集群，要么对 `lastAcceptedStep` 采用强一致存储；若采用"就近读取 + 异步复制"，必须在复制延迟窗口内继续允许重放风险，或直接禁止。

### 13.3 日志脱敏

建议添加**统一日志过滤器**，对以下字段一律脱敏为：

```text
[REDACTED]
```

```text
password
code
otp
totp
secret
manualKey
otpauthUri
recoveryCode
```

示例：

错误（泄露验证码）：

```text
verify TOTP code=123456
```

正确（只记上下文，不记敏感值）：

```text
TOTP verification failed
userId=user_xxx
challengeId=xxx
```

> 日志过滤器要加在**日志框架层面**（全局 filter），而不是在每个 Controller 里手动处理，否则一定会漏。

### 13.4 APM / Sentry

特别检查以下工具的采集配置：

```text
Sentry
Datadog
New Relic
Crashlytics
Firebase
ELK
CloudWatch
```

因为这些工具经常自动记录：

```text
Request Body
Response Body
URL
Breadcrumb
```

必须把 MFA 相关 API 的 Payload 从采集范围中**排除**（例如配置 `ignore` 规则、在 SDK 层脱敏）。

### 13.5 API 防缓存

以下敏感接口的响应必须禁止缓存：

```text
/mfa/totp/enroll
/mfa/recovery/regenerate
```

```http
Cache-Control: no-store
Pragma: no-cache
```

并且**不要经过 CDN 缓存**（敏感接口应直接从源站返回）。

### 13.6 CSRF

如果 Web 版本使用 Cookie Session，则敏感接口必须有：

```text
CSRF Protection
```

（如 SameSite Cookie + CSRF Token 双重防护）。如果移动 App 使用：

```text
Authorization: Bearer
```

则重点保护 Token 的存储与传递（HTTPS、不进 URL、防日志泄露）。

### 13.7 App 本地存储

客户端**不要**永久保存：

```text
TOTP Secret
Recovery Codes
用户输入 OTP
```

Access Token / Refresh Token 的存储：

- **iOS**：`Keychain`
- **Android**：`Keystore` / Encrypted SharedPreferences（或更安全的 EncryptedStorage）
- **Flutter / React Native**：使用对应的安全存储插件（如 `flutter_secure_storage` / `react-native-keychain`）

> 即使用户短暂停留的页面数据（如验证码输入）也不建议写入明文 SharedPreferences / UserDefaults。

### 13.8 截图保护

对于 Recovery Codes 页面，可以考虑 Android 使用：

```text
FLAG_SECURE
```

禁止截图。但**是否启用要权衡用户体验**——用户需要把恢复码保存到别处，完全禁止截图可能反而诱导用户拍照外传。

更现实的设计：

```text
允许 Copy / Download
+
明确安全提示
```

TOTP 二维码页面则可以更严格地考虑禁止截图（因为二维码本身就是 Secret）。

------
## 十四、上线与灰度

### 14.1 数据库 Migration

上线流程建议分步进行，降低一次性变更的风险：

```text
Step 1
创建 MFA 相关表（mfa_totp_factor、mfa_recovery_code、security_audit_log）

Step 2
部署后端，但默认 feature flag off（接口存在但不开放）

Step 3
部署前端 MFA UI（在 flag 关闭时隐藏入口）

Step 4
内部账号测试（QA / 内部员工真实走一遍 开启 → 登录 → 恢复 流程）

Step 5
灰度开放（按百分比 / 按用户群）

Step 6
正式开放
```

> 数据库迁移建议使用带版本的 migration 工具（如 Flyway / Liquibase / Alembic），并保证向前兼容：新增表存在时，旧版本代码仍能正常运行。

### 14.2 Feature Flag

建议使用配置开关：

```text
feature.totpMfa.enabled
```

甚至可以支持按用户比例灰度：

```text
feature.totpMfa.userPercent
```

例如：

```text
1%
5%
20%
100%
```

> 灰度节奏：内部账号 → 1% 真实用户 → 观察 MFA 指标与客服工单 → 逐步提升 → 全量。Feature Flag 的评估要放在请求链路最前面，做到"开关即生效、无需发版"。

### 14.3 Metrics

建议监控以下技术指标：

```text
MFA enrollment started
MFA enrollment completed
MFA enrollment failure
MFA login success
MFA login failure
MFA recovery used
MFA disabled
MFA reset
MFA challenge expired
Rate limit triggered
```

**Metrics 绝对不能携带**：

```text
OTP
Secret
Recovery Code
```

（指标只统计计数与分位数，不记录敏感值。）

### 14.4 Product Metrics

产品侧可以关注：

```text
2FA Adoption Rate         2FA 采用率
Enrollment Completion Rate 开启完成率
TOTP Failure Rate          TOTP 失败率
Recovery Code Usage Rate   恢复码使用率
MFA Reset Rate             MFA 重置率
Login Drop-off after MFA   登录在 MFA 步骤后的流失
```

如果 `Enrollment Started` 很高但 `Enrollment Completed` 很低，说明扫码或 UI 流程可能存在问题（例如二维码不清晰、Manual Key 不明显、验证码输入体验差），需要排查引导流程。

### 14.5 Security Metrics

安全侧重点监控：

```text
TOTP failures / minute
Unique users attacked
Unique IPs
High failure IPs
Recovery attempts
MFA reset attempts
MFA disabled after suspicious login
```

可以对异常行为触发风控（例如短时间内大量失败、恢复码高频使用、MFA 被重置后立即发生敏感操作等）。

------

## 十五、测试方案

### 15.1 Unit Test

**Secret**

```text
Secret 长度正确（20 字节）
Secret 每次生成不同
Secret Base32 编码正确
```

**TOTP**

```text
正确验证码通过
错误验证码失败
上一窗口允许
当前窗口允许
下一窗口允许
超出窗口失败
```

**Replay**

```text
验证码首次成功
同一验证码第二次失败（CODE_ALREADY_USED）
```

**Enrollment**

```text
Pending 正常
Expired Pending 失败
Confirmed 后状态为 ACTIVE
重复 Confirm 失败
```

### 15.2 Integration Test

覆盖完整链路：

```text
Password → MFA Challenge
Challenge → TOTP → Token
Challenge Expired
Wrong TOTP
Rate Limit
Recovery Code Login
Disable MFA
Replace MFA
Password Reset + MFA
OAuth Login + MFA
```

### 15.3 并发测试

**必须**测试：

```text
同一个 TOTP
同时发送 10 个请求
```

预期结果：

```text
只有一个成功
其余全部 CODE_ALREADY_USED
```

这是验证防重放（原子更新）最重要的安全测试，漏掉它等于没做防并发验证。

### 15.4 时间边界测试

例如验证码切换点：

```text
12:00:29
12:00:30
```

需要确保 `[-1, 0, +1]` 窗口行为符合预期：窗口切换瞬间、前后窗口都正确接受 / 拒绝。

### 15.5 Rate Limit 测试

模拟：

```text
1 个账号 → 1000 次验证码尝试
```

确保不会一直允许（Account 级限流生效）。

同时模拟：

```text
1 个 IP → 1000 个账号
```

确保存在 IP 级防护。

### 15.6 Recovery Code 测试

测试：

```text
有效 Recovery Code
错误 Recovery Code
已经使用的 Recovery Code
并发重复使用（原子标记）
撤销后的 Recovery Code
重新生成后旧 Recovery Code 失效
```

### 15.7 Security Test

上线前建议进行渗透测试，重点检查：

```text
MFA Bypass
Challenge ID Guessing
Challenge Replay
TOTP Replay
Rate Limit Bypass
Legacy API Bypass
OAuth Bypass
Password Reset Bypass
Race Condition
Secret Leakage
Log Leakage
Recovery Flow
```

------
## 十六、工程实现

### 16.1 前端开发任务拆分

**Security Settings 页面**

```text
2FA Status
Enable Button
Disable Button
Replace Authenticator
Recovery Codes
```

**Enrollment 页面**

```text
QR Code
Manual Key
Copy
6-digit Input
Confirm
Error Handling
```

**登录 MFA 页面**

```text
6-digit Input
Verify
Recovery Code Entry
Challenge Expiration
Retry
```

**Recovery Codes 页面**

```text
List
Copy
Download
Confirmation（确认已保存）
```

### 16.2 后端开发任务拆分

按领域拆分服务，避免所有逻辑堆在一个 Controller：

```text
TotpService
MfaEnrollmentService
MfaChallengeService
RecoveryCodeService
MfaAuditService
MfaRateLimitService
```

### 16.3 TotpService

职责：

```text
generateSecret()
generateOtpAuthUri()
verify()
calculateTimeStep()
```

TotpService **不负责**：

```text
Database
Login Token
HTTP
```

保持纯算法组件独立，方便单元测试与复用。

### 16.4 MfaEnrollmentService

职责：

```text
startEnrollment()
confirmEnrollment()
replaceFactor()
disableFactor()
regenerateRecoveryCodes()
```

### 16.5 MfaChallengeService

职责：

```text
createLoginChallenge()
getChallenge()
consumeChallenge()
failChallenge()
expireChallenge()
```

### 16.6 RecoveryCodeService

职责：

```text
generateCodes()
hashCode()
verifyCode()
consumeCode()
revokeCodes()
```

### 16.7 RateLimitService

建议统一封装：

```text
checkUserLimit()
checkIpLimit()
registerFailure()
registerSuccess()
```

不要把 Redis 限流逻辑散落在 Controller 中——否则每个接口的限流行为会不一致。

### 16.8 Controller 层

Controller 只负责：

```text
Request
↓
Validation
↓
Service
↓
Response
```

不要把以下逻辑直接写在 Controller：

```text
TOTP Algorithm
Crypto
Database Transaction
```

### 16.9 推荐后端模块结构

```text
src/

auth/
    login/
    token/
    password/

security/
    mfa/
        controller/
        service/
        repository/
        model/

        totp/
            TotpService

        challenge/
            MfaChallengeService

        recovery/
            RecoveryCodeService

        audit/
            MfaAuditService
```

------

## 十七、API Contract 示例

### 17.1 GET MFA Status

```http
GET /api/v1/security/mfa
Authorization: Bearer xxx
```

Response：

```json
{
  "success": true,
  "data": {
    "enabled": true,
    "factors": [
      {
        "type": "TOTP",
        "status": "ACTIVE",
        "confirmedAt": "2026-08-07T01:00:00Z"
      }
    ],
    "recoveryCodesRemaining": 7
  }
}
```

### 17.2 Enrollment Response

```json
{
  "success": true,
  "data": {
    "enrollmentId": "enr_7e19...",
    "otpauthUri": "otpauth://...",
    "manualKey": "ABCD EFGH IJKL MNOP QRST UVWX YZAB CDEF",
    "expiresAt": "2026-08-07T02:20:00Z"
  }
}
```

### 17.3 MFA Login Challenge

```json
{
  "success": true,
  "data": {
    "authenticationStatus": "MFA_REQUIRED",
    "mfaChallenge": "mfac_f33...",
    "methods": [
      "TOTP",
      "RECOVERY_CODE"
    ],
    "expiresAt": "2026-08-07T02:10:00Z"
  }
}
```

------

## 十八、推荐安全默认值

```yaml
mfa:
  totp:
    algorithm: SHA1
    digits: 6
    periodSeconds: 30
    secretBytes: 20
    allowedPastWindows: 1
    allowedFutureWindows: 1

  enrollment:
    ttlSeconds: 600

  loginChallenge:
    ttlSeconds: 300
    maxAttempts: 5

  recoveryCodes:
    count: 10

  rateLimit:
    enabled: true
```

> 具体次数应结合现有风控体系调整；所有默认值建议集中在一个配置文件中，便于统一修改与灰度。

------
## 十九、路线图

### 19.1 第一阶段 MVP

如果需要快速上线，第一阶段至少实现：

```text
✓ TOTP Secret
✓ QR / Manual Key
✓ Confirm Enrollment
✓ Login Challenge
✓ TOTP Login
✓ Secret Encryption
✓ Basic Rate Limit
✓ Recovery Codes
✓ Disable MFA
✓ Security Notification
✓ Audit Log
```

**不要**为了赶进度删掉：

```text
Secret Encryption
Rate Limit
Recovery Flow
Challenge
```

这些不是"增强功能"，而是完整 MFA 系统不可分割的一部分。

### 19.2 第二阶段

可以增加：

```text
Trusted Device
Device Management
Advanced Risk Detection
MFA Step-Up
Admin MFA Enforcement
Creator MFA Enforcement
MFA Adoption Campaign
Suspicious Login Detection
```

### 19.3 第三阶段

增加：

```text
Passkey / WebAuthn
```

最终多因素架构：

```text
Account
   │
   ├── Password
   │
   ├── TOTP
   │
   ├── Recovery Codes
   │
   └── Passkey
```

对管理员可以要求：

```text
Passkey / Hardware Key
```

获得更强的抗钓鱼能力。

------

## 二十、上线检查清单

### 20.1 Backend

- [ ] 使用成熟 TOTP Library
- [ ] Secret 使用 CSPRNG 生成
- [ ] Secret >= 160 bit
- [ ] Secret Base32
- [ ] TOTP SHA1 / 6 digits / 30 sec
- [ ] 验证 Window = ±1
- [ ] Secret 加密存储
- [ ] KMS Key 与 DB 分离
- [ ] Enrollment 有 TTL
- [ ] Enrollment Confirm 后才 ACTIVE
- [ ] 登录第一阶段不签发正式 Token
- [ ] Challenge 单次使用
- [ ] Challenge 有 TTL
- [ ] Challenge 有次数限制
- [ ] User Rate Limit
- [ ] IP Rate Limit
- [ ] TOTP Replay Protection
- [ ] Replay DB Update 原子化
- [ ] Recovery Code 哈希保存
- [ ] Recovery Code 单次使用
- [ ] Disable MFA 需要重新认证
- [ ] Replace MFA 使用安全迁移流程
- [ ] Password Reset 不自动关闭 MFA
- [ ] OAuth Login 不能绕过 MFA
- [ ] Legacy API 不能绕过 MFA
- [ ] 所有服务器时间同步
- [ ] Audit Log
- [ ] Security Notifications

### 20.2 Frontend

- [ ] Security Settings
- [ ] 2FA Enabled / Disabled 状态
- [ ] QR 页面
- [ ] Manual Key
- [ ] Copy Manual Key
- [ ] 6 位验证码输入
- [ ] MFA Login 页面
- [ ] Recovery Code Login
- [ ] Recovery Codes 展示
- [ ] Replace Authenticator
- [ ] Disable MFA
- [ ] Challenge Expired UX
- [ ] Rate Limit UX
- [ ] 敏感字段不写 Analytics
- [ ] 敏感字段不写 Crash Log
- [ ] Sensitive Screen Review

### 20.3 Infrastructure

- [ ] HTTPS Only
- [ ] Redis
- [ ] KMS / Key Vault
- [ ] NTP
- [ ] Log Redaction
- [ ] APM Sensitive Data Filter
- [ ] CDN No Cache
- [ ] Security Monitoring
- [ ] Alerts

------
## 二十一、推荐最终架构

本社交 App 第一阶段推荐：

```text
┌───────────────────────────────────────────┐
│                Mobile App                 │
│                                           │
│ QR / Manual Key / OTP / Recovery Code     │
└───────────────────┬───────────────────────┘
                    │ HTTPS
                    ▼
┌───────────────────────────────────────────┐
│               Auth Service                │
│                                           │
│ Password                                  │
│ TOTP                                      │
│ MFA Challenge                             │
│ Recovery                                  │
│ Rate Limiting                             │
│ Audit                                     │
└───────┬──────────────┬──────────────┬─────┘
        │              │              │
        ▼              ▼              ▼

   PostgreSQL        Redis          KMS
        │
        ├── Users
        ├── MFA Factor
        ├── Recovery Codes
        └── Security Audit
```

推荐认证链路：

```text
账号 / OAuth
      ↓
Primary Authentication
      ↓
MFA Enabled ?
   │          │
   No        Yes
   │          │
   │          ▼
   │     MFA Challenge
   │          ↓
   │        TOTP
   │          ↓
   │     Replay Check
   │          ↓
   └──────────┤
              ↓
      Access Token
      Refresh Token
```

------

## 二十二、关键安全原则总结

整个项目最重要的不是"生成一个 6 位数字"，而是围绕它的完整安全设计：

```text
1. Secret 安全存储（加密 + KMS）

2. 登录 Challenge（两阶段登录）

3. 验证失败限流（防暴力）

4. TOTP Replay Protection（防重放）

5. Recovery Code（丢失恢复）

6. 关闭 / 更换 MFA 的身份验证

7. Password Reset 不绕过 MFA

8. OAuth / Legacy API 不绕过 MFA

9. 敏感信息不进入日志

10. 完整账号恢复流程
```

TOTP 算法本身只占整个实现的一小部分。一个生产可用的 MFA 系统，本质上是：

```text
Authentication State Machine
+
Secret Management
+
Risk Control
+
Account Recovery
+
Security Audit
```

而不是单纯：

```text
Authenticator
+
6-digit Code
```

------

## 二十三、推荐实施顺序

建议开发团队按以下顺序推进，每阶段可独立验收：

```text
Phase 1   数据库 + Secret Encryption
Phase 2   TOTP Service
Phase 3   Enrollment API
Phase 4   Enrollment Frontend
Phase 5   Login MFA Challenge
Phase 6   Login MFA UI
Phase 7   Rate Limit + Replay Protection
Phase 8   Recovery Codes
Phase 9   Disable / Replace
Phase 10  Security Notification + Audit
Phase 11  Integration / Security Test
Phase 12  Feature Flag 灰度上线
```

------

## 二十四、技术决策结论

本项目采用：

```text
Standard TOTP / RFC 6238
```

而不是集成某一个 Authenticator 厂商。Authenticator 只是 Secret 的客户端持有者和验证码生成器。

系统边界：

```text
Authenticator
    │
    │ 不访问我们的 API
    │
    ▼
用户读取验证码
    │
    ▼
Frontend
    │
    │ HTTPS
    ▼
Backend
    │
    ▼
TOTP Verification
```

最终推荐参数：

```text
TOTP
HMAC-SHA1
6 digits
30 seconds
160-bit random Secret
Base32
±1 verification window
```

后端：

```text
TOTP Library
+
Database
+
Redis
+
KMS
+
Existing Authentication Framework
```

前端：

```text
QR Renderer
+
6-digit Input
+
Security Settings UI
```

**不需要**：

```text
Google Authenticator SDK
Microsoft Authenticator SDK
自建验证码服务
Authenticator 与服务器实时通信
```

该架构标准、成熟、兼容性高，并且后续可以平滑升级到 Passkey / WebAuthn。
