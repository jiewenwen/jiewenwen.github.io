---
layout: post
title: "用 Codex 构建可验证的多 Agent 编码工作流：A 实施、B 审查、C 验证"
date: 2026-08-19 00:00:00 +0800
categories: [AI]
tags: [Codex, AI编程, Multi-Agent, Code Review, 自动化验证, 工程规范, 最佳实践, AI Agent]
---

当一个 App 已经完成需求梳理、架构整改和任务拆分之后，真正困难的往往不再是"应该怎么改"，而是：

> **如何把几十甚至上百个整改任务稳定、可控、可验证地落地到代码里。**

如果使用 VS Code 中的 Codex 进行开发，一个很自然的想法是：

- 开一个窗口 A，让 Codex 负责实施；
- 开一个窗口 B，让另一个 Codex 对 A 的实施进行独立审查；
- 再增加一个 C，用测试、构建、静态检查等确定性工具进行客观验证。

最终形成：

```text
A = Implementer
B = Reviewer
C = Verifier
```

也就是：

> **A + B + C**

这比简单的"双 Agent 左右互搏"更加可靠。

因为真正的软件工程质量，不应该建立在"两个 AI 都觉得没问题"之上，而应该建立在：

```text
Implementation
+
Independent Review
+
Deterministic Verification
```

之上。

**适用场景**：本文面向代码整改、技术债治理、安全修复、系统重构和大型 Refactor 等项目。当方案设计已经完成、任务清单已经确定、主要矛盾是"如何稳定落地"时，这套工作流价值最大。

**不太适合的场景**：需求尚不明确的原型探索、一次性小改动、临时调试。这些场景引入三个角色只会增加不必要的开销——具体边界见文末「附录 C」。

**核心结论（TL;DR）**：

- 单 Agent 会上下文漂移、自我审查存在天然偏差、测试通过不代表需求完整；
- 双 Agent（A + B）本质都是语言模型，可能出现"一致但错误"的相关性判断；
- 引入 C（确定性验证）并贯穿整个实施闭环，才能让"可验证"落到实处；
- 执行顺序推荐 **A → C → B**，而不是 A → B → C；
- 一 Task 一 Commit，B 只 Review 稳定 Commit，且 B 默认不修改代码；
- 最终判定：**DONE = B PASS + C PASS**，而不是"A 说完成了"。

**全文结构**：

1. 单 Agent 与双 Agent 的不足（一 ~ 二）
2. A + B + C 模型与角色隔离（三 ~ 五）
3. A：Implementer 的实施纪律（六 ~ 八）
4. B：Independent Reviewer 的审查纪律（九 ~ 十二）
5. Git 作为 Agent 之间的同步协议（十三 ~ 十五）
6. C：Verifier 的确定性验证（十六 ~ 十九）
7. 执行顺序、状态机与记录协议（二十 ~ 二十四）
8. 落地方式：VS Code 布局与 Git Worktree（二十五 ~ 二十九）
9. 本质与最终原则（三十 ~ 三十二）
10. 附录：Prompt 示例、落地检查清单与适用边界

---

## 一、为什么单 Agent 不够

假设我们已经有一份完整的整改计划：

```text
FE-01 修复登录状态管理
FE-02 重构 API Error Handler
FE-03 统一前端请求重试策略

BE-01 重构 Auth Middleware
BE-02 增加 Refresh Token Rotation
BE-03 修复并发 Refresh Race Condition
BE-04 完成数据库 Migration

SEC-01 修复敏感信息泄漏
SEC-02 增加 Resource Ownership 校验

TEST-01 增加认证 Integration Tests
TEST-02 增加关键流程 E2E Tests
```

最简单的方式当然是：

```text
把完整整改方案交给 Codex
        ↓
让它一路实施到底
```

但随着任务数量增加，这种模式很容易出现几个问题。

### 1. 上下文漂移

一开始 Codex 可能严格按照任务实施。

做到后面，它可能开始：

```text
顺手重构
顺手优化
顺手改 API
顺手抽象
顺手调整数据结构
```

最终得到的代码也许"更优雅"，但已经偏离原始整改要求。

### 2. 自我审查存在天然偏差

如果 Codex 刚刚写完某段代码，再问：

```text
请检查一下刚才的实现有没有问题。
```

它很容易沿着原来的思路继续证明自己的方案。

也就是说，当：

```text
Implementer
=
Reviewer
```

时，缺少真正意义上的独立判断。

### 3. 测试通过不代表任务完成

例如整改要求是：

```text
所有资源读取接口必须校验 Resource Ownership。
```

Codex 修改了：

```text
GET /api/orders/:id
```

但可能漏掉：

```text
GET /api/orders/:id/items
GET /api/invoices/:id
GET /api/files/:id
```

此时：

```text
lint PASS
typecheck PASS
test PASS
build PASS
```

依然不能说明整改任务完整完成。

因为：

```text
代码正确
≠
需求完整
```

---

## 二、为什么只用 A + B 仍然不够

于是我们自然会想到：

```text
A = Developer
B = Reviewer
```

A 负责写代码，B 负责独立 Review。

这已经比单 Agent 强很多。

但仍然存在一个根本问题：

> **A 和 B 都是语言模型。**

它们可能出现相关性错误（correlated errors）。

例如：

```text
A 错误理解某个业务规则
        ↓
实现了一套逻辑
        ↓
B 阅读代码
        ↓
觉得"逻辑合理"
        ↓
PASS
```

问题在于：

```text
两个 AI 一致
```

并不意味着：

```text
代码一定正确
```

因此需要第三个角色——一个不依赖模型主观判断的确定性角色。

---

## 三、A + B + C 模型

最终架构：

```text
A = Implementer

B = Reviewer

C = Verifier
```

其中：

### A：负责实施

职责：

```text
理解 Task
修改代码
补充测试
执行本地检查
形成 Git Checkpoint
```

### B：负责独立审查

职责：

```text
对照 Requirement
检查 Git Diff
检查遗漏
检查边界条件
检查回归风险
检查安全问题
检查测试有效性
给出 PASS / FAIL
```

### C：负责确定性验证

C 不是第三个 Codex。

C 是：

```text
lint
typecheck
unit test
integration test
E2E test
build
static analysis
security scan
API contract test
migration test
```

C 的特点是：

> **不参与讨论，只给结果。**

例如：

```text
TypeScript Compile: PASS
Unit Test: PASS
Integration Test: FAIL
E2E: PASS
Build: PASS
```

它不会因为代码"看起来合理"而放过问题。

---

## 四、完整架构

整个流程可以表示为：

```text
                  MASTER REMEDIATION PLAN
                           │
                           ▼
                       TASK QUEUE
                           │
                           ▼
                    Select One Task
                           │
                           ▼
                 ┌─────────────────┐
                 │        A        │
                 │   IMPLEMENTER   │
                 └────────┬────────┘
                          │
                    Implement Code
                          │
                          ▼
                 ┌─────────────────┐
                 │        C        │
                 │    VERIFIER     │
                 └────────┬────────┘
                          │
                lint / test / build
                          │
                     PASS?
                    ┌─────┴─────┐
                    │           │
                   NO          YES
                    │           │
                    ▼           ▼
                  Fix A      Git Commit
                                │
                                ▼
                      ┌─────────────────┐
                      │        B        │
                      │    REVIEWER     │
                      └────────┬────────┘
                               │
                         PASS / FAIL
                         ┌─────┴─────┐
                         │           │
                        FAIL        PASS
                         │           │
                         ▼           ▼
                         A          DONE
                         │
                      Fix Code
                         │
                         ▼
                         C
                         │
                         ▼
                         B
```

这里非常重要的一点是：

> **C 不应该只在最后出现。**

而应该贯穿整个实施闭环。

---

## 五、三个角色必须严格隔离

A + B + C 能否真正发挥作用，关键取决于：

> **Role Separation（角色隔离）。**

三个角色不能互相越界。

推荐定义：

```text
A：
有代码修改权

B：
有审查权
有否决权
默认没有代码修改权

C：
有验证权
没有设计权
没有解释权
```

---

## 六、A：Implementer

A 的目标不是"让代码变得更漂亮"。

而是：

> **准确完成当前 Task。**

例如：

```text
TASK: AUTH-03

实现 Refresh Token Rotation。
```

要求：

```text
1. 每次使用 refresh token 后生成新的 refresh token；

2. 旧 token 立即失效；

3. 支持 token reuse detection；

4. 不修改 access token API；

5. 增加 integration tests。
```

A 的工作流程应该固定为：

```text
读取 Requirement
      ↓
阅读相关代码
      ↓
理解影响范围
      ↓
实施修改
      ↓
补充测试
      ↓
调用 C
      ↓
修复失败项
      ↓
检查 Git Diff
      ↓
Commit
```

---

## 七、A 不应该拥有需求解释自由

整改阶段和探索开发阶段不同。

如果技术方案已经确定，那么 A 的权限应该受到限制。

例如不要告诉 A：

```text
请优化整个认证系统。
```

而应该：

```text
只实施 AUTH-03。

不要改变任务范围。

不要主动修改现有 API。

如果发现与本任务无直接关系的问题，
记录为 Finding，
不要顺手修复。
```

这样可以有效防止：

```text
Scope Creep（范围蔓延）
```

---

## 八、一次只处理一个 Task

Multi-Agent Workflow 最常见的问题，不是模型能力不足，而是：

> **Task 太大。**

不推荐：

```text
TASK-01

重构整个认证系统。
```

推荐拆成：

```text
AUTH-01
统一 Access Token Validation

AUTH-02
重构 Refresh Token Storage

AUTH-03
实现 Refresh Token Rotation

AUTH-04
增加 Token Reuse Detection

AUTH-05
处理 Concurrent Refresh

AUTH-06
补充 Integration Tests
```

任务粒度越小：

```text
Diff 越小

Review 越准确

Regression 越容易发现

失败越容易回滚

上下文越稳定
```

---

## 九、B：Independent Reviewer

B 的身份不是：

```text
第二个 Developer
```

而应该是：

```text
Independent Reviewer
```

它不负责回答：

```text
"如果是我，我会怎么实现"
```

而负责回答：

```text
"当前实现是否真正满足原始要求"
```

这是两个完全不同的问题。

---

## 十、B 应该基于 Requirement，而不是 A 的解释

例如 A 完成后说：

```text
我已经解决了 Refresh Token Race Condition，
同时完善了并发保护和异常处理。
```

不要把这段话作为 B 的主要输入。

否则容易产生：

```text
Anchoring Bias（锚定偏差）
```

更好的 B 输入应该只有：

```text
Original Requirement
+
Current Code
+
Git Diff
+
Tests
```

即：

```text
Requirement
    +
Implementation
    ↓
Reviewer
    ↓
Independent Conclusion
```

而不是：

```text
Developer Conclusion
    ↓
Reviewer Verification
```

---

## 十一、B 应该检查什么

B 的 Review 至少应该覆盖以下几个维度。

### 1. Requirement Coverage

逐条检查：

```text
Requirement 1 → DONE / NOT DONE

Requirement 2 → DONE / NOT DONE

Requirement 3 → DONE / NOT DONE
```

不是笼统地说：

```text
整体看起来完成了。
```

### 2. Scope

检查：

```text
有没有修改与 Task 无关的模块？

有没有扩大整改范围？

有没有改变原有 API Contract？

有没有修改不应该修改的数据结构？
```

### 3. Regression

例如修改：

```text
Auth Middleware
```

需要考虑：

```text
login
logout
refresh
session restore
API auth
WebSocket auth
background jobs
mobile client
```

是否受到影响。

### 4. Error Handling

检查：

```text
invalid input
null
empty value
timeout
network failure
database failure
retry
partial failure
```

### 5. Concurrency

重点检查：

```text
race condition
duplicate request
retry
idempotency
transaction
lock
parallel execution
```

### 6. Security

涉及用户和权限时，检查：

```text
Authentication
Authorization
Resource Ownership
Privilege Escalation
Replay Attack
Token Leakage
Sensitive Logging
Input Validation
```

### 7. Test Quality

不是只检查：

```text
有没有 Test
```

而是检查：

```text
Test 有没有真正覆盖 Requirement
```

例如：

```text
Refresh Token Rotation
```

至少应考虑：

```text
正常刷新

旧 Token 失效

非法 Token

过期 Token

Token Reuse

Concurrent Refresh
```

---

## 十二、B 默认不要修改代码

这是整个模型最重要的纪律之一。

正确权限模型：

```text
A = Write

B = Review

C = Verify
```

而不是：

```text
A = Write

B = Review + Write
```

否则很快会变成：

```text
A 修改 auth.ts
        ↓
B Review auth.ts
        ↓
B 顺手修复
        ↓
A 同时继续修改
        ↓
代码状态漂移
```

最终很难确认：

```text
谁改了什么

Review 的是什么版本

失败应该归因到哪个改动
```

---

## 十三、不要实时互搏，要回合制

两个 Agent 同时操作同一个项目，看起来效率很高：

```text
A Coding

B Reviewing
```

但实际上：

```text
B 看到的文件状态
```

可能几分钟后就被 A 改掉。

于是：

```text
Review Input
≠
Final Implementation
```

更好的方式：

```text
A 完成稳定状态
      ↓
C 验证
      ↓
Git Commit
      ↓
B Review
```

也就是说：

> **Review Stable Checkpoint，而不是 Review Moving Target。**

---

## 十四、Git Commit 是 Agent 之间的同步协议

在这个工作流中，Git 不只是版本管理工具。

它还是：

> **Agent Communication Protocol（Agent 之间的通信协议）。**

例如：

```text
AUTH-03
   ↓
A Implement
   ↓
C Verify
   ↓
commit abc123
   ↓
B Review abc123
```

这样 B Review 的对象永远明确：

```text
Commit abc123
```

而不是：

```text
A 当前工作目录里的某个瞬时状态
```

---

## 十五、一 Task 一 Commit

建议：

> **One Task, One Commit。**

例如：

```bash
fix(AUTH-03): implement refresh token rotation
```

下一项：

```bash
fix(AUTH-04): detect refresh token reuse
```

这样 Reviewer 可以直接：

```bash
git show <commit>
```

而不需要面对：

```text
4000 行 Diff
37 个文件
9 个 Task 混在一起
```

---

## 十六、C：Verifier

A + B + C 和普通双 Agent 工作流最大的区别，就是 C。

C 的目标是：

> **把能机械判断的问题全部交给机器。**

避免 B 把时间浪费在：

```text
代码是否能编译

类型有没有报错

单测有没有挂

Lint 有没有失败
```

这些问题应该由 C 判断。

B 应该把注意力放在：

```text
需求完整性

逻辑漏洞

遗漏

风险

安全

边界条件
```

---

## 十七、C 应该包含哪些检查

根据项目类型不同，C 可以包括：

```text
lint
format check
typecheck
unit test
integration test
E2E
build
migration test
API contract test
static analysis
dependency scan
security scan
```

例如一个 Web App：

```bash
npm run lint
npm run typecheck
npm run test
npm run test:integration
npm run build
```

如果是 monorepo，可以进一步：

```text
frontend checks
backend checks
shared package checks
database migration checks
```

---

## 十八、C 的价值在于确定性

A 和 B 都可能说：

```text
我认为这个实现没有问题。
```

C 不会。

C 只会给出：

```text
PASS
```

或者：

```text
FAIL
```

例如：

```text
Lint: PASS

TypeCheck: PASS

Unit: PASS

Integration: FAIL

Build: PASS
```

那么当前 Task 就不能进入 Review PASS。

---

## 十九、C 不能替代 B

反过来也要注意：

```text
All Tests PASS
```

不能代替 Code Review。

因为自动化验证只能测试：

```text
已经被写成断言的东西
```

它不知道：

```text
有没有漏接口

Requirement 是否理解错了

有没有不必要的架构修改

有没有未来维护风险

有没有安全设计缺陷
```

所以：

```text
C ≠ B
```

同理：

```text
B ≠ C
```

两者是互补关系，缺一不可。

---

## 二十、推荐的执行顺序

一个 Task 最稳的生命周期是：

```text
Requirement
    ↓
A Implement
    ↓
C Verify
    ↓
A Fix
    ↓
C PASS
    ↓
Commit
    ↓
B Review
    ↓
FAIL?
    ↓
A Fix
    ↓
C Verify
    ↓
Commit
    ↓
B Re-review
    ↓
PASS
```

因此：

```text
A → C → B
```

其实比：

```text
A → B → C
```

更加合理。

因为 B 不应该浪费精力去 Review 一份：

```text
连 build 都过不了
```

的代码。

---

## 二十一、推荐状态机

如果整改任务很多，可以给每个 Task 一个明确状态：

```text
TODO
 ↓
IMPLEMENTING
 ↓
VERIFYING
 ↓
IMPLEMENTED
 ↓
REVIEWING
 ↓
┌─────────────┐
│             │
▼             ▼
REJECTED     PASSED
│             │
▼             ▼
FIXING        DONE
│
└────→ VERIFYING
```

---

## 二十二、每个 Task 最好拥有自己的记录

例如：

```yaml
id: AUTH-03

title: Refresh Token Rotation

status: REVIEWING

requirements:
  - rotate refresh token on successful refresh
  - invalidate previous token
  - preserve current access-token API

implementation_commit:
  - abc123

verification:
  lint: PASS
  typecheck: PASS
  unit_test: PASS
  integration_test: PASS
  build: PASS

review:
  status: FAIL

findings:
  - severity: P1
    description: concurrent refresh scenario is not handled

  - severity: P2
    description: expired refresh token path is not covered
```

这样整个整改过程不再是：

```text
一堆 AI 对话记录
```

而变成：

```text
一套有状态的软件工程流程
```

---

## 二十三、统一 PASS / FAIL 协议

B 的输出不要是：

```text
整体写得不错，
还有一些地方可以完善。
```

因为这种结论无法驱动工作流。

建议固定输出：

```text
STATUS: PASS
```

或者：

```text
STATUS: FAIL
```

如果失败：

```text
STATUS: FAIL

P0:
- None

P1:
- Concurrent refresh may issue two valid refresh tokens.

P2:
- Integration test does not cover expired refresh tokens.

P3:
- Error naming is inconsistent.
```

---

## 二十四、严重等级

推荐定义：

```text
P0 = Critical

必须立即修复。
存在严重安全、数据一致性或系统不可用风险。
```

```text
P1 = Blocking

当前 Task 不允许 PASS。
```

```text
P2 = Should Fix

原则上应在当前 Task 中处理。
```

```text
P3 = Optional

代码质量或维护性建议。
```

PASS 条件例如：

```text
P0 = 0

P1 = 0
```

对于高风险整改可以要求：

```text
P0 = 0
P1 = 0
P2 = 0
```

---

## 二十五、推荐 VS Code 布局

最简单的方式：

```text
VS Code Window A
/project

Codex Role:
IMPLEMENTER
```

以及：

```text
VS Code Window B
/project

Codex Role:
REVIEWER
```

但要严格要求：

```text
B 不修改文件
```

这样成本最低。

---

## 二十六、更稳定的方式：Git Worktree

如果项目较大，推荐进一步隔离工作环境：

```text
/project
```

用于 A。

```text
/project-review
```

用于 B。

例如：

```text
workspace/

├── project/
│   └── Codex A
│
└── project-review/
    └── Codex B
```

A 负责：

```text
implementation branch
```

B 负责：

```text
stable review workspace
```

这样可以避免：

```text
两个 Codex 同时访问不断变化的 Working Tree
```

---

## 二十七、不要让 B 提前看到 A 的思考过程

为了保持独立性，不建议把 A 的大量解释交给 B。

B 最好只看到：

```text
Task Requirement

Git Diff

Relevant Code

Test Results
```

而不是：

```text
A 为什么选择这个架构

A 为什么认为这是正确的

A 认为哪些风险已经解决
```

因为这些信息会影响 Reviewer 的独立判断。

---

## 二十八、把 AI 的主观判断与机器的客观结果分离

一个完整 Task 最终应该形成两类证据。

### 主观证据（来自 B）

```text
Requirement covered

No obvious regression

Security model acceptable

No missing call sites
```

### 客观证据（来自 C）

```text
Lint PASS

TypeCheck PASS

Tests PASS

Build PASS
```

最终：

```text
DONE
=
B PASS
+
C PASS
```

而不是：

```text
DONE
=
A 说完成了
```

---

## 二十九、最终推荐架构

完整模型：

```text
                      REMEDIATION PLAN
                             │
                             ▼
                         TASK QUEUE
                             │
                             ▼
                       ONE TASK ONLY
                             │
                             ▼
                   ┌─────────────────┐
                   │        A        │
                   │   IMPLEMENTER   │
                   └────────┬────────┘
                            │
                        Code Change
                            │
                            ▼
                   ┌─────────────────┐
                   │        C        │
                   │    VERIFIER     │
                   └────────┬────────┘
                            │
           lint / typecheck / tests / build
                            │
                         PASS?
                    ┌───────┴───────┐
                    │               │
                   NO              YES
                    │               │
                    ▼               ▼
                 A FIX          GIT COMMIT
                                    │
                                    ▼
                           ┌─────────────────┐
                           │        B        │
                           │    REVIEWER     │
                           └────────┬────────┘
                                    │
                              PASS / FAIL
                           ┌────────┴────────┐
                           │                 │
                          FAIL              PASS
                           │                 │
                           ▼                 ▼
                         A FIX              DONE
                           │
                           ▼
                           C
                           │
                           ▼
                           B
```

---

## 三十、这实际上是一个 Mini Engineering Team

到了这里，Codex 已经不只是：

```text
AI Coding Assistant
```

而是在一个软件工程系统中承担角色。

整个结构类似一个小型研发团队：

```text
Product / Architect
        │
        ▼
Requirement
        │
        ▼
Developer
        │
        ▼
CI / QA
        │
        ▼
Code Reviewer
        │
        ▼
Merge
```

映射到 Codex：

```text
Remediation Plan
        │
        ▼
Codex A
Implementer
        │
        ▼
C
Automated Verification
        │
        ▼
Codex B
Independent Reviewer
        │
        ▼
PASS
```

这已经非常接近一个小型研发团队的协作方式。

---

## 三十一、A + B + C 真正解决的不是"写代码速度"

它真正解决的是：

```text
错误如何被发现

错误如何被阻断

任务如何被追踪

代码如何被验证

改动如何被回滚

需求如何避免遗漏
```

也就是说：

> **A + B + C 的目标不是让 Codex 更聪明，而是让整个系统对 Codex 的错误更加不敏感。**

这是一个非常重要的区别。

---

## 三十二、最终原则

如果把整个方法压缩成几条规则，可以总结为：

```text
1. 一次只做一个 Task。

2. A 只负责实施。

3. C 负责所有可以机械验证的问题。

4. B 负责独立判断需求是否真正完成。

5. B 默认不修改代码。

6. Review 稳定 Commit，不 Review 动态工作目录。

7. 一 Task 一 Commit。

8. B 不依赖 A 的自我评价。

9. C PASS 不等于 B PASS。

10. DONE = B PASS + C PASS。
```

最终闭环：

```text
Implement
    ↓
Verify
    ↓
Commit
    ↓
Review
    ↓
Fix
    ↓
Verify Again
    ↓
Re-review
    ↓
PASS
```

---

## 附录 A：A / B 的 Starter Prompt 示例

原则讲清楚之后，附上可以直接使用的启动 Prompt，方便落地。

### A：Implementer 的 Prompt

```text
你是 Implementer。

当前任务：
<粘贴 Task 描述>

要求：
1. 只实施这个 Task，不要扩大范围；
2. 不修改与任务无关的模块，不改变现有 API Contract；
3. 修改代码前先阅读相关调用链，理解影响范围；
4. 为本次改动补充必要测试；
5. 完成后运行指定的验证命令（lint / typecheck / test / build）；
6. 提交时使用约定式提交，例如 fix(AUTH-03): ...；
7. 如果发现与任务无关的问题，记录为 Finding，不要顺手修复；
8. 最后汇报：改了哪些文件、验证结果、Commit Hash、Findings。
```

### B：Independent Reviewer 的 Prompt

```text
你是 Independent Reviewer。

不要修改任何代码，只做 Review。

输入：
- Original Requirement（原始需求）
- Current Code（当前代码）
- Git Diff（本次提交的 Diff）
- Test Results（测试结果）

审查维度：
1. Requirement Coverage：逐条核对需求是否真正完成；
2. Scope：是否越界修改了无关模块；
3. Regression：是否影响既有功能；
4. Error Handling / Concurrency / Security；
5. Test Quality：测试是否真正覆盖需求。

输出固定为：

STATUS: PASS 或 FAIL

并给出 P0 / P1 / P2 / P3 分级问题清单。
```

**注意**：不要把 A 的解释、A 的自我评价、A 的架构说明粘贴给 B——那是 B 产生 Anchoring Bias 的主要来源。

---

## 附录 B：落地检查清单

把整套方法压缩成一份可勾选的检查清单，实施前对照一遍：

- [ ] 任务清单是否已拆分到足够小（一个 Task 一个提交）？
- [ ] 每个 Task 是否有明确的 Requirement，而不是一句"优化一下"？
- [ ] A 是否只实施当前 Task，并把无关问题记录为 Finding？
- [ ] C 是否贯穿每个 Task（lint / typecheck / test / build），而不是只在最后跑一次？
- [ ] 是否做到一 Task 一 Commit，提交信息带 Task 编号？
- [ ] B 是否只 Review 稳定 Commit，而不是动态工作目录？
- [ ] B 是否拿到了 Original Requirement，而不是 A 的自我评价？
- [ ] B 是否输出了结构化 PASS / FAIL 与 P0 ~ P3 问题清单？
- [ ] B 是否全程没有修改代码？
- [ ] FAIL 后是否走"A 修复 → C 验证 → 新 Commit → B 重新 Review"闭环？
- [ ] DONE 是否同时满足 B PASS 和 C PASS？
- [ ] 每个 Task 是否有状态记录（TODO → IMPLEMENTING → ... → DONE）？

---

## 附录 C：什么时候不需要 A + B + C

这套工作流有明确的开销：三个角色的上下文、多轮验证与 Review 循环，都会消耗时间和 Token。以下场景引入它会得不偿失：

- **原型探索 / 需求还不明确**：还在"试方案"阶段，先让单个 Agent 快速验证可行性，而不是过早套流程；
- **一次性小改动**：改一个配置、修一行 bug，A + C 两步就够，B 的独立审查收益有限；
- **纯文档 / 纯配置变更**：没有可验证的代码行为，C 的确定性验证无从谈起；
- **时间极紧的一次性任务**：如果改动影响面小、可快速回滚，简化为 A + C 即可。

判断标准很简单：

> **只有当"错误发现得越晚代价越高、且任务规模足够大"时，A + B + C 的收益才大于开销。**

技术债治理、安全修复、系统重构正是典型的"错误代价高、规模大"场景，因此最值得使用这套工作流。

---

## 结语

当一个项目已经拥有完整的整改方案和任务清单以后，真正需要优化的就不再只是 Prompt。

更值得设计的是：

> **整个 AI Coding Workflow。**

单 Agent 模式关注的是：

```text
怎么让 AI 一次写对？
```

A + B + C 模式关注的是：

```text
即使 AI 没有一次写对，
系统能不能发现？

发现之后能不能阻止？

阻止之后能不能修复？

修复之后能不能重新验证？
```

这才是真正适合大型整改项目的思路。

最终我们得到的不是一个"更会写代码的 Codex"。

而是一套：

```text
可实施
可验证
可审查
可追踪
可回滚
可持续推进
```

的 AI 软件工程系统。

对于已经完成方案设计、当前主要任务是全面实施落地的项目来说，**A + B + C** 往往比单纯追求更长的 Prompt、更大的 Context 或更多并行 Agent 更有价值。

因为高质量的软件工程，从来不应该依赖：

> "希望这次 AI 没犯错。"

而应该依赖：

> **即使它犯错，系统也能把错误挡在最终代码之外。**
