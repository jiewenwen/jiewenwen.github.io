---
layout: post
title: "编写高质量的 AGENTS.md：给 AI 编码代理建立长期有效的工程规则"
date: 2026-07-26 00:00:00 +0800
categories: [AI]
tags: [AGENTS.md, AI编程, Codex, Claude Code, Cursor, 工程规范, 最佳实践, AI Agent]
---

随着 Codex、Claude Code、Cursor Agent 等 AI 编码代理逐渐参与真实软件工程，团队会遇到一个非常现实的问题：**如何让 AI 不仅“写出能运行的代码”，还能够长期遵守项目的架构约束、工程规范和质量标准？**仅靠每次任务中临时编写 Prompt，通常无法解决这个问题。

任务 Prompt 更适合描述“这一次要做什么”，而项目还需要一份长期生效的规则，用于回答以下问题：

- 修改代码前需要检查什么？
- 哪些架构边界不能突破？
- 是否允许顺手重构？
- 什么情况下需要增加测试？
- 什么才算真正完成任务？
- AI 可以自主决定到什么程度？
- 哪些行为是明确禁止的？

`AGENTS.md` 正是用来解决这些问题的。

它可以被看作一份面向 AI 编码代理的工程协作协议。

本文将讨论 `AGENTS.md` 应该包含哪些基本原则、哪些内容不适合放进去，以及如何构建一份稳定、清晰、可执行的项目级规则。

**实践提示**：`AGENTS.md` 一般放在仓库根目录，并提交到版本控制中。不同 AI 编码工具对文件名的约定略有差异——Codex 使用 `AGENTS.md`，Claude Code 使用 `CLAUDE.md`，Cursor 使用 `.cursorrules`，GitHub Copilot 使用 `.github/copilot-instructions.md`。如果团队同时使用多个工具，可以在 `AGENTS.md` 中维护主规则，再通过符号链接或简短引用文件指向它，避免规则分散和不一致。

------

## 一、AGENTS.md 的定位

`AGENTS.md` 不应该是一份普通的项目说明文档，也不应该是一条无限增长的超级 Prompt。

它最合适的定位是：**定义 AI 编码代理在当前代码仓库中必须长期遵守的工作方式、工程边界和完成标准。**

可以将项目中的指令分为三个层次。

### 第一层：长期工程原则

这类规则适合写入 `AGENTS.md`：

- 修改前必须理解调用链。
- 以实际代码为主要事实来源。
- 优先修复根因。
- 不得削弱安全检查。
- 不得通过删除测试让代码通过。
- 不得擅自改变业务语义。
- 完成任务必须提供验证证据。

这些原则通常不会因为某一个功能开发任务结束而失效。

### 第二层：项目具体约束

这类内容也适合写入 `AGENTS.md`：

- 项目采用什么架构。
- 各目录/模块分别承担什么职责。
- 模块 A 是否允许直接访问模块 B 的数据存储。
- 组件是否允许绕过中间层直接拼接外部地址。
- 数据持久化相关的文件存放在哪里。
- 测试命令、构建命令和检查命令是什么。
- 哪些模块属于安全关键模块。

这些规则与当前仓库长期绑定。

### 第三层：单次任务要求

这类内容不适合长期放在 `AGENTS.md`：

- 本次实现某个具体功能。
- 修复某个 UI 组件的布局问题。
- 将某个模块的响应时间降低到指定范围。
- 本次只输出 Review 报告，不修改代码。
- 本轮优先处理某几个文件。

这些要求应该放在本次任务 Prompt、Issue、任务清单或执行计划中。

一个简单的判断标准是：**如果这条规则在三个月后执行另一个任务时仍然适用，它大概率适合写进 `AGENTS.md`。**

------

## 二、以实际实现作为事实来源

AI 编码代理很容易过度相信文档、注释、文件名和任务描述。

但真实项目中，文档经常落后于实现，注释也可能没有及时更新。

因此，`AGENTS.md` 中首先应该明确：

- 当前可执行代码是主要事实来源。
- 数据持久化相关文件（如数据库迁移脚本）是对应数据结构的事实来源。
- 接口/API 定义和实际实现是接口行为的事实来源。
- 测试只能证明其覆盖范围内的行为，不能代替完整实现分析。
- 文档与代码冲突时，必须进一步检查完整调用链。

推荐写法：

```md
## Source of Truth

- Treat executable code, data definitions, interface specifications,
  configuration, and tests as the primary sources of truth.
- Do not infer behavior from filenames, comments, or documentation
  without verifying the implementation.
- When documentation conflicts with code, inspect the complete runtime
  path before deciding which behavior is intended.
```

这里需要注意，“以代码为准”并不意味着代码永远正确。

更准确的含义是：**在分析系统当前到底做了什么时，以实际运行路径为准；在判断系统应该做什么时，还需要结合业务要求和架构设计。**

------

## 三、先调查，再修改

AI 编码代理常见的问题之一，是看到一个局部问题后立刻开始改代码。

例如，一个模块返回了意外结果，代理可能直接修改该模块的实现，却没有检查：

- 是否还有其他调用方依赖当前行为；
- 是否存在共享的底层逻辑；
- 是否涉及跨模块的状态一致性；
- 是否有其他组件依赖当前的返回值格式；
- 是否存在相关的集成测试或端到端测试。

因此，应该在 `AGENTS.md` 中要求代理在修改前完成基本调查。

```md
## Understand Before Changing

- Inspect relevant entry points, callers, dependencies, data flows,
  and tests before modifying code.
- Search all usages before changing a shared type, public interface,
  schema, protocol, or business rule.
- Do not replace an existing design until its purpose and constraints
  are understood.
```

这条规则能够显著减少局部修改造成的连锁回归。

对于以下改动，更应该强制要求搜索全部引用：

- 公共类型/接口；
- 数据结构/持久化字段；
- 错误码/状态码；
- 权限规则；
- 认证/授权结构；
- 模块间通信协议/事件；
- 全局配置项。

------

## 四、优先解决根因，而不是叠加补丁

AI 很擅长快速让错误消失，但“错误消失”不等于“问题解决”。

例如：

- 认证机制有问题，却在某个入口额外判断一次状态；
- 数据同步设计错误，却增加定时刷新；
- 状态模型不完整，却增加更多布尔字段；
- 数据竞争未解决，却增加重试；
- 生命周期管理错误，却在多个地方手动清理资源。

这些做法短期内可能通过测试，但会继续积累架构债。

因此，应明确要求：

```md
## Solve Root Causes

- Prefer fixing the underlying cause over adding guards, exceptions,
  duplicated logic, retries, compatibility paths, or special cases.
- Do not hide invalid states or suppress failures merely to make
  tests or builds pass.
- When the root cause cannot be resolved safely within scope,
  document the limitation explicitly.
```

判断一个方案是不是根因修复，可以提出三个问题：

1. 同类问题是否还可能在其他入口出现？
2. 当前修改是在统一约束，还是增加特殊分支？
3. 删除这段补丁后，底层模型是否仍然正确？

真正的根因修复通常会让系统规则更统一，而不是让判断条件越来越多。

------

## 五、保持最小必要改动，但不要机械追求最小 Diff

“最小改动”是软件工程中的重要原则，但它经常被错误理解。

合理的最小改动是：**只修改完成目标所必需的范围，不引入无关变化。**

不合理的最小改动是：即使底层设计已经明显错误，也只增加一行条件判断，避免改动更多文件。

因此，`AGENTS.md` 中应同时强调“范围聚焦”和“结构正确”。

```md
## Keep Changes Focused

- Make the smallest coherent change that fully solves the requested problem.
- Do not refactor, rename, reformat, reorganize, or upgrade unrelated code.
- A focused change must still be structurally correct; do not preserve
  a defective design solely to minimize the diff.
```

这里的关键词是 `smallest coherent change`，即“最小的完整改动”。

一个改动可能涉及多个文件，但只要这些文件共同构成完整解决方案，它仍然属于最小必要改动。

相反，只改一个文件，却留下不一致的调用路径，并不是真正的小改动。

------

## 六、默认保持业务语义不变

在重构、清理和性能优化任务中，AI 很容易把“看起来可以简化”的逻辑删除。

但有些代码看似冗余，实际上承担着：

- 权限约束；
- 防重放；
- 状态校验；
- 兼容处理；
- 并发保护；
- 业务兜底；
- 数据完整性约束。

因此，除非任务明确要求改变行为，否则应该默认保持：

- 业务规则；
- 权限语义；
- 接口语义；
- 错误语义；
- 数据持久化语义；
- 用户可感知行为。

```md
## Preserve Intended Behavior

- Unless explicitly requested, preserve existing business rules,
  security semantics, public interface behavior, data persistence behavior,
  and user-visible behavior.
- Do not remove logic merely because it appears unused until all runtime
  and indirect usages have been investigated.
- Refactoring must preserve externally observable behavior unless the task
  explicitly defines a behavior change.
```

尤其是在涉及多个模块或服务的项目中，“行为不变”不能只看 UI 是否相同，还要检查：

- 输入/输出字段是否变化；
- 返回状态/错误码是否变化；
- 错误信息是否变化；
- 数据排序是否变化；
- 分页行为是否变化；
- 重试语义是否变化；
- 缓存失效时机是否变化；
- 模块间通信的顺序和格式是否变化。

------

## 七、明确架构边界

如果 `AGENTS.md` 只能保留一类项目特有规则，应该优先保留架构边界。

因为 AI 编码代理通常会选择最直接的实现路径。

例如：

- 为了让某个模块快速获取数据，让它直接访问另一个模块的内部存储；
- 为了少写一层抽象，让入口层直接实现核心业务逻辑；
- 为了方便复用，让领域逻辑依赖具体的外部框架或通信协议；
- 为了快速修复，让多个模块共同修改同一份全局状态。

这些做法短期内很快，长期却会使系统耦合迅速扩大。

可以在 `AGENTS.md` 中明确规定：

```md
## Respect Architectural Boundaries

- Keep presentation, business logic, data access, and infrastructure
  responsibilities separated according to the project's architecture.
- Do not place business rules in UI components, network handlers,
  data adapters, or serialization code.
- Do not bypass established abstractions merely because a direct
  dependency is easier.
- New dependencies must follow the direction defined by the architecture.
```

对于具体项目，还应该进一步写清楚各层职责。

例如（以分层架构为例）：

```md
## Layer Boundaries

- Entry points (UI, CLI, API handlers) may accept input, invoke
  application services, and format output.
- Entry points must not contain business rules or data access logic.
- Application services coordinate use cases and transaction boundaries.
- Data access objects are responsible only for persistence behavior.
- Domain logic must not depend on specific frameworks, databases,
  messaging systems, or communication protocols.
```

无论项目采用分层架构、六边形架构、Clean Architecture 还是其他模式，原则是一样的：

```md
## Module Boundaries

- Each module owns its data and exposes well-defined interfaces.
- Modules must not reach into each other's internal storage directly.
- Shared state must go through designated coordination points.
- Utility code must not silently grow business logic.
```

规则越具体，AI 越不容易为了方便突破边界。

------

## 八、禁止制造新的技术债

除了告诉 AI 应该做什么，还要明确告诉它哪些行为不能接受。

常见的新技术债包括：

- 复制粘贴业务逻辑；
- 无解释的硬编码；
- 永久存在的临时兼容分支；
- 空实现；
- 只返回固定值的伪实现；
- 为通过编译而增加的占位代码；
- 忽略错误；
- 大范围使用宽泛类型；
- 不安全的强制类型转换；
- 吞掉异常后继续运行；
- 在生产代码中保留调试输出。

可以写成：

```md
## Avoid New Technical Debt

- Do not introduce duplicated business logic, unexplained hard-coded
  values, placeholder implementations, silent fallbacks, or permanent
  temporary workarounds.
- Do not use broad types, unchecked casts, ignored errors, or disabled
  validation to conceal design problems.
- Do not add TODO or FIXME comments unless the unresolved work is
  unavoidable and clearly explained.
```

对 `TODO` 和 `FIXME` 应该特别谨慎。

很多代理会在没有真正完成任务时写一句：

```text
TODO: implement this properly later
```

然后将任务标记为完成。

更合理的原则是：**如果当前任务要求完成该功能，就不能用 TODO 代替实现。**

只有确实超出当前范围、且不影响当前正确性的事项，才适合记录为后续工作。

------

## 九、防止过度设计

与“写得太随意”相反，AI 编码代理还有另一个常见倾向：过度抽象。

例如，一个简单功能可能被扩展为：

- 新增通用接口；
- 新增抽象工厂；
- 新增事件总线；
- 新增插件系统；
- 新增三层适配器；
- 新增通用 Repository；
- 新增大量只有一个实现的 Protocol。

这些抽象看起来很“专业”，但可能没有解决任何真实问题。

因此，`AGENTS.md` 应该同时限制过度设计：

```md
## Avoid Overengineering

- Do not design for speculative future requirements.
- Prefer existing project patterns and the simplest design that
  satisfies current requirements.
- Add an abstraction only when it establishes a meaningful boundary,
  removes proven duplication, isolates expected change,
  or improves testability.
- Do not introduce frameworks, event buses, plugin systems, or additional
  architectural layers without demonstrated need.
```

一个抽象是否合理，可以通过以下问题判断：

1. 它是否隔离了真实存在的变化？
2. 它是否消除了已经出现的重复？
3. 它是否建立了明确的架构边界？
4. 它是否显著提高了测试能力？
5. 它是否只有一个实现，而且看不到第二个实现的现实需求？

如果前四个问题都是否定的，而第五个问题是肯定的，这个抽象很可能没有必要。

------

## 十、安全和数据完整性必须高于实现便利

对于涉及用户身份、权限控制、敏感数据或需要保证数据一致性的项目，安全规则必须直接写进 `AGENTS.md`。

不能假设 AI 会自然遵守所有安全边界。

建议至少包括以下原则：

```md
## Security and Data Integrity

- Treat all externally-provided identity, authorization, ownership,
  time, state, and resource references as untrusted.
- Enforce authorization and business invariants on trusted boundaries,
  not at the input layer.
- Never expose or log passwords, tokens, verification codes,
  private keys, secrets, or sensitive personal data.
- Consider transactions, concurrency, idempotency, retries,
  partial failure, and rollback for multi-step state changes.
- Never weaken security controls to simplify implementation
  or make tests pass.
```

特别需要强调的是：

### 外部输入默认不可信

以下信息不能只依赖调用方或客户端提供：

- 用户/主体身份标识；
- 角色和权限声明；
- 资源所有权归属；
- 操作时间戳；
- 金额、数量等业务数值；
- 操作是否被允许的标志位。

调用方可以提交请求意图，但可信边界内必须重新验证权限和业务约束。

### 多步骤状态变更必须考虑失败

例如，一个操作需要：

1. 写入持久化存储；
2. 更新缓存或索引；
3. 发送通知或事件；
4. 写入审计记录。

任何一步都可能失败。

代理必须考虑：

- 哪些操作需要原子性（事务）；
- 哪些操作可以安全重试；
- 重试是否会重复执行；
- 是否需要幂等键；
- 部分成功后如何恢复或补偿；
- 缓存与持久化存储不一致时以谁为准。

------

## 十一、严格控制依赖变更

AI 代理非常容易为一个小问题引入新依赖。

例如，为了处理一个简单字符串操作，引入大型工具库；为了一个小状态机，引入完整框架。

因此应规定：

```md
## Dependency Discipline

- Prefer the standard library and existing project dependencies.
- Add a production dependency only when its value clearly exceeds
  its maintenance and security cost.
- Do not upgrade unrelated dependencies or modify lockfiles
  without necessity.
- Explain every new production dependency in the final report.
```

新依赖不只是多一个包，它还意味着：

- 新的供应链风险；
- 新的许可证要求；
- 新的升级成本；
- 新的兼容性问题；
- 新的构建体积；
- 新的维护责任。

所以依赖变更应该被视为架构决策，而不是普通代码修改。

------

## 十二、测试是实现的一部分

有些 AI 代理会把测试视为任务结束后的附加工作。

更合理的工程原则是：**对行为的修改和对行为的验证，属于同一个实现过程。**

建议写入：

```md
## Testing and Verification

- Add or update tests whenever behavior changes.
- Bug fixes should include a regression test when reasonably possible.
- Run the narrowest relevant checks first, then broader checks
  appropriate to the affected scope.
- Never delete, skip, weaken, or rewrite a valid test solely
  to make the implementation pass.
- Review the final diff for regressions, unintended changes,
  debug code, and temporary files.
```

### 为什么先运行最相关测试

如果每改一行代码就运行整个项目测试，反馈会很慢。

更合理的顺序是：

1. 当前函数或模块的单元测试；
2. 当前功能的集成测试；
3. 受影响模块的完整测试；
4. 项目级构建和全量测试。

这种方式既提高效率，也能够更快定位问题。

### Bug 修复应该增加回归测试

回归测试的核心价值不是提高覆盖率数字，而是记录：**这个问题曾经发生过，并且以后不能再次发生。**

一个高质量 Bug 修复通常应包括：

- 能复现原问题的测试；
- 修复实现；
- 测试由失败变为通过；
- 相关范围内的其他测试仍然通过。

------

## 十三、明确“完成”的定义

很多 AI 任务停在“代码已经写出来”这一步。

但真实工程中的完成至少还包括：

- 能否构建；
- 测试是否通过；
- 类型检查是否通过；
- 格式是否正确；
- 是否留下调试代码；
- 是否误改无关文件；
- 是否更新必要文档；
- 是否真实执行过验证命令。

因此，`AGENTS.md` 中应该明确 Definition of Done。

```md
## Definition of Done

A task is complete only when:

- The requested behavior is fully implemented.
- Relevant builds, tests, lint checks, formatting checks,
  and type checks pass.
- Architecture and security constraints remain satisfied.
- No placeholder, duplicate, dead, debug, or temporary
  implementation remains.
- Documentation is updated when public behavior, architecture,
  configuration, or operational procedures change.
- Any check that could not be run is explicitly reported.
- No result, command execution, test outcome, or implementation
  status is claimed without evidence.
```

最后一条尤其重要：**不得声称执行了实际上没有执行的测试或命令。**

AI 的输出可能非常确信，但确信不能替代证据。

如果环境中无法运行某项测试，应该明确写出：

- 哪个测试没有执行；
- 为什么不能执行；
- 已经执行了哪些替代检查；
- 还存在哪些风险。

------

## 十四、复杂任务应维护执行计划

简单修改不需要复杂规划，但跨模块功能、数据库迁移、系统重构和安全加固通常无法靠一次性 Prompt 稳定完成。

因此，可以规定复杂任务使用单独的 `PLANS.md` 或执行计划文件。

```md
## Planning Complex Work

- For complex features, cross-cutting changes, migrations,
  or significant refactors, create and maintain an execution plan.
- Keep the plan current as discoveries, decisions, progress,
  and verification results change.
- Continue through the plan autonomously unless blocked by a genuinely
  ambiguous business decision, an irreversible action, missing credentials,
  or a safety concern.
```

好的执行计划不是静态任务列表，而应该持续记录：

- 当前目标；
- 已完成事项；
- 新发现的问题；
- 已做出的架构决策；
- 剩余风险；
- 验证结果；
- 下一步工作。

这样即使任务跨越多个会话，代理也能够恢复上下文，而不是每次重新理解整个项目。

------

## 十五、自主推进与谨慎决策之间的边界

如果规则过于严格，AI 会频繁停止并询问用户。

如果规则过于宽松，AI 又可能擅自做出高风险决策。

因此，应该明确哪些事情可以自主处理，哪些事情必须谨慎。

```md
## Working Style

- Investigate independently before asking questions.
- Use reasonable defaults for minor implementation details
  that do not alter business semantics.
- Do not make irreversible, destructive, security-sensitive,
  or product-defining decisions without explicit authorization.
- State uncertainty and limitations honestly.
- Never fabricate implementation, testing, review,
  or command-execution results.
```

通常可以自主决定的事项包括：

- 局部变量命名；
- 符合现有风格的文件组织；
- 明确的空值处理；
- 必要的测试补充；
- 无业务影响的小型内部实现细节。

不应该擅自决定的事项包括：

- 删除用户数据；
- 修改权限模型；
- 降低加密或认证强度；
- 改变公开接口；
- 修改核心业务规则；
- 执行不可逆迁移；
- 删除大范围代码；
- 替换核心基础设施；
- 引入新的商业或产品语义。

------

## 十六、哪些内容不应该写入 AGENTS.md

为了避免文件持续膨胀，还需要明确哪些内容应该放到其他位置。

### 不要放入完整架构设计

`AGENTS.md` 可以说明架构边界，但不应该包含几十页系统设计。

更合适的方式是：

```md
- Follow the architecture defined in `docs/architecture.md`.
- For major structural changes, update the architecture document first.
```

### 不要放入一次性任务清单

某次修复、某个版本目标、某轮 Review 范围，都应该放进任务 Prompt 或计划文件。

### 不要放入容易变化的数据

例如：

- 当前剩余多少个功能模块；
- 当前有多少失败测试；
- 某个临时分支名称；
- 某次发布的日期；
- 某个开发阶段的进度。

这些内容很快会过期。

### 不要重复语言本身的基础规范

例如：

- Swift 变量使用驼峰命名；
- Go 代码应该运行 `gofmt`；
- TypeScript 应避免明显类型错误。

除非项目有特殊要求，否则优先通过格式化工具、Lint 配置和编译器进行约束，而不是全部重复写进 `AGENTS.md`。

### 不要写无法验证的空泛要求

例如：

```md
- Write high-quality code.
- Follow best practices.
- Make the architecture elegant.
- Ensure excellent performance.
```

这些规则没有明确的执行标准。

更好的写法是将其转化为具体行为：

```md
- Do not duplicate business logic.
- Run the relevant performance benchmark when changing a hot path.
- Preserve module dependency direction.
- Add regression tests for bug fixes.
```

------

## 十七、推荐的 AGENTS.md 文件结构

一份实用的 `AGENTS.md` 可以采用以下结构：

```md
# AGENTS.md

## Repository Overview

## Source of Truth

## Repository Map

## Build and Test Commands

## Understand Before Changing

## Architecture Boundaries

## Preserve Intended Behavior

## Security and Data Integrity

## Dependency Discipline

## Testing and Verification

## Project-Specific Prohibitions

## Definition of Done

## Planning Complex Work

## Final Report Requirements
```

其中，真正与项目有关的部分主要是：

- Repository Overview；
- Repository Map；
- Build and Test Commands；
- Architecture Boundaries；
- Project-Specific Prohibitions。

其余部分可以在多个项目之间复用。

关于 `Final Report Requirements`：这是指任务完成后 AI 代理应当提交的最终报告内容，例如变更摘要、新增依赖说明、未执行检查及原因、已知风险等。目的是确保每次任务结束时，人类开发者不需要重新阅读全部 Diff 就能理解发生了什么。

### 多工具兼容建议

如果团队同时使用多种 AI 编码工具，需要注意不同工具的配置文件名不同：

| 工具 | 配置文件 |
|------|----------|
| Codex (OpenAI) | `AGENTS.md` |
| Claude Code (Anthropic) | `CLAUDE.md` |
| Cursor | `.cursorrules` |
| GitHub Copilot | `.github/copilot-instructions.md` |
| Windsurf | `.windsurfrules` |

建议以 `AGENTS.md` 作为主规则文件，其他配置文件通过简短引用指向它（例如 `CLAUDE.md` 中写 `Follow the rules defined in AGENTS.md`），或使用符号链接让多个文件名指向同一份内容，避免规则版本漂移。

------

## 十八、一个可直接使用的核心模板

下面是一份适合作为起点的精简模板。

```md
# AGENTS.md

## Source of Truth

- Treat executable code, data definitions, API specifications,
  configuration, and tests as the primary sources of truth.
- Verify actual implementation before relying on comments or documentation.
- When documentation conflicts with code, inspect the full runtime path.

## Understand Before Changing

- Inspect relevant entry points, callers, dependencies, data flows,
  and tests before modifying code.
- Search all usages before changing shared types, public interfaces,
  schemas, protocols, or business rules.
- Do not replace a design until its purpose and constraints are understood.

## Solve Root Causes

- Prefer fixing the underlying cause over adding special cases,
  duplicated logic, retries, or compatibility paths.
- Do not suppress failures merely to make builds or tests pass.
- Document unresolved limitations explicitly.

## Keep Changes Focused

- Make the smallest coherent change that fully solves the task.
- Do not refactor, rename, reformat, reorganize, or upgrade unrelated code.
- Do not preserve a defective design solely to minimize the diff.

## Preserve Intended Behavior

- Unless explicitly requested, preserve business rules, security semantics,
  public interface behavior, data persistence behavior,
  and user-visible behavior.
- Investigate all direct and indirect usages before removing code.
- Refactoring must preserve externally observable behavior.

## Respect Architectural Boundaries

- Keep presentation, business logic, data access, and infrastructure
  responsibilities separated according to the project's architecture.
- Do not place business rules in UI components, network handlers,
  data adapters, or serialization code.
- Do not bypass established abstractions for convenience.
- Preserve the dependency direction defined by the architecture.

## Avoid New Technical Debt

- Do not introduce duplicated business logic, unexplained hard-coded
  values, placeholder implementations, silent fallbacks,
  or permanent temporary workarounds.
- Do not ignore errors or disable validation to conceal problems.
- Do not use TODO or FIXME as a substitute for required implementation.

## Avoid Overengineering

- Do not design for speculative future requirements.
- Prefer existing project patterns and the simplest correct design.
- Add abstractions only when they establish a meaningful boundary,
  remove proven duplication, isolate expected change,
  or improve testability.
- Do not introduce new frameworks or architectural layers
  without demonstrated need.

## Security and Data Integrity

- Treat externally-provided identity, authorization, ownership,
  time, and state as untrusted.
- Enforce authorization and business invariants on trusted boundaries.
- Never expose or log passwords, tokens, verification codes,
  private keys, or secrets.
- Consider transactions, concurrency, idempotency, retries,
  partial failure, and rollback.
- Never weaken security controls to simplify implementation.

## Dependency Discipline

- Prefer the standard library and existing dependencies.
- Add a production dependency only when clearly necessary.
- Do not upgrade unrelated dependencies or modify lockfiles unnecessarily.
- Explain every new production dependency in the final report.

## Testing and Verification

- Add or update tests whenever behavior changes.
- Bug fixes should include regression tests when reasonably possible.
- Run focused tests first, followed by broader relevant checks.
- Never delete, skip, or weaken a valid test solely to make code pass.
- Review the final diff for unintended changes, debug code,
  temporary files, and sensitive information.

## Definition of Done

A task is complete only when:

- The requested behavior is fully implemented.
- Relevant builds, tests, lint checks, formatting checks,
  and type checks pass.
- Architecture and security constraints remain satisfied.
- No placeholder, duplicate, debug, or temporary implementation remains.
- Required documentation is updated.
- Checks that could not be run are explicitly reported.
- All claimed results are supported by actual evidence.

## Complex Work

- Maintain an execution plan for cross-cutting features,
  migrations, and significant refactors.
- Keep progress, discoveries, decisions, risks,
  and verification results current.
- Continue autonomously unless blocked by an ambiguous business decision,
  irreversible action, missing credential, or safety concern.

## Working Style

- Investigate independently before asking questions.
- Use reasonable defaults for minor details that do not alter semantics.
- Do not make destructive, irreversible, security-sensitive,
  or product-defining decisions without authorization.
- State uncertainty and limitations honestly.
- Never fabricate execution or verification results.
```

------

## 十九、如何让规则真正生效

写完 `AGENTS.md` 并不意味着规则一定有效。

还需要定期检查以下问题。

### 规则是否具体

“保持代码质量”很难执行。

“不得复制业务逻辑，Bug 修复必须尽可能增加回归测试”更容易执行。

### 规则是否存在冲突

例如同时写：

- 始终保持最小改动；
- 发现架构问题必须彻底重构。

这两条规则可能产生冲突。

更合理的表达是：**修改范围应保持聚焦，但不得为了减少 Diff 而保留明显错误的设计。**

### 规则是否能够被验证

好的规则通常能够通过以下方式验证：

- 搜索；
- 编译；
- 测试；
- Lint；
- Diff Review；
- 依赖检查；
- 架构检查。

### 文件是否过长

如果 `AGENTS.md` 已经增长到很难完整阅读，应该拆分：

- `AGENTS.md`：核心规则和入口；
- `PLANS.md`：复杂任务执行计划规范；
- `docs/architecture.md`：完整架构；
- `docs/testing.md`：测试策略；
- `docs/security.md`：安全要求；
- 子目录 `AGENTS.md`：局部模块规则。

### 是否包含已经过期的内容

每次较大架构调整后，都应该检查：

- 目录结构是否变化；
- 命令是否仍然可用；
- 架构边界是否变化；
- 禁止事项是否仍然合理；
- 文档引用是否仍然存在。

------

## 二十、结语

`AGENTS.md` 的价值，不在于让 AI 写出更多代码，而在于限制它以错误方式完成任务。

一份高质量的 `AGENTS.md` 应该帮助 AI 编码代理做到：

- 先理解，再修改；
- 以实际实现为依据；
- 优先解决根因；
- 保持修改范围聚焦；
- 尊重现有架构边界；
- 默认保持业务语义稳定；
- 不制造新的技术债；
- 不为未来假设过度设计；
- 不牺牲安全和数据完整性；
- 使用测试和构建结果证明任务完成；
- 对无法验证的部分保持诚实。

最终，`AGENTS.md` 不应该成为一份冗长的口号集合。

它应该是一份简洁、稳定、明确、可验证的工程契约。

任务 Prompt 决定 AI 这一次要做什么，而 `AGENTS.md` 决定它在整个项目中应该以什么方式工作。