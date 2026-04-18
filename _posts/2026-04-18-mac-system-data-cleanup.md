---
layout: post
title: "Mac 存储空间清理教程：找出 System Data 过大的真正原因"
date: 2026-04-18 00:40:00 +0800
categories: [macOS]
tags: [Mac, 存储清理, System Data, Xcode, Docker]
---

很多人清理 Mac 存储空间时，第一眼看到的都是这几个问题：

- **System Data 特别大**
- 系统设置里的分类看不懂
- 明明没装多少东西，磁盘却快满了

如果这台 Mac 既是日常办公机，又装了 **Xcode、iOS Simulator、Docker**，空间异常膨胀的原因往往不是系统本身，而是开发环境留下的可再生数据。

这篇文章提供一套可复用、相对稳妥的排查与清理流程。先查清，再删除。所有 `rm -rf` 命令执行前，都建议二次确认路径。

---

## 一、先建立正确认知

在 macOS 里，**System Data 并不等于系统文件本体**。

它经常还会包含这些内容：

- Xcode 测试数据
- Simulator 数据
- 编译缓存
- Docker 镜像与构建缓存
- App 容器数据
- 系统缓存和日志

所以，看到 **System Data 很大**，不要先怀疑系统坏了，先做排查。

---

## 二、先排除常见“大户”

### 1. 查看本地 Time Machine 快照

```bash
tmutil listlocalsnapshots /
```

如果输出了很多快照，说明本地快照占了一部分空间。
如果只有类似下面的提示而没有快照列表，这一项通常可以排除：

```bash
Snapshots for disk /:
```

### 2. 查看 iPhone/iPad 本地备份

```bash
du -sh ~/Library/Application\ Support/MobileSync/Backup
```

如果提示：

```bash
No such file or directory
```

说明没有本地 iPhone/iPad 备份目录，这一项也可以排除。

---

## 三、直接用 `du` 找真实占用源

系统设置的分类不一定精确，最直接的方法是统计 `~/Library` 下各目录的实际大小。

先看重点目录：

```bash
du -sh ~/Library/Caches
du -sh ~/Library/Application\ Support
du -sh ~/Library/Developer
du -sh ~/Library/Containers
du -sh ~/Library/Group\ Containers
```

再看总体排名：

```bash
du -sh ~/Library/* 2>/dev/null | sort -hr | head -20
```

核心思路只有一句话：不看“系统怎么分类”，只看“谁实际占得最多”。

---

## 四、如果 `~/Library/Developer` 很大，优先排查 Xcode 残留

如果 `~/Library/Developer` 达到几十 GB 甚至上百 GB，大概率是 Xcode 或测试环境数据。

继续执行：

```bash
du -sh ~/Library/Developer/* 2>/dev/null | sort -hr
du -sh ~/Library/Developer/Xcode/* 2>/dev/null | sort -hr
```

重点关注：

- `~/Library/Developer/XCTestDevices`
- `~/Library/Developer/CoreSimulator`
- `~/Library/Developer/Xcode/DerivedData`

它们通常对应：

### 1. `XCTestDevices`

Xcode 测试运行相关数据。这个目录异常变大时，常见原因是测试设备残留、测试缓存累积，或异常测试后遗留的大量派生数据。

### 2. `CoreSimulator`

iOS 模拟器数据，包括模拟器设备、设备内 App 数据、模拟器系统环境。

### 3. `DerivedData`

Xcode 编译缓存目录。通常可以直接删除，是最常见且相对安全的清理项之一。

---

## 五、如何相对安全地清理 Xcode/Simulator 占用

开始前请先退出：

- Xcode
- Simulator
- Instruments
- 正在跑测试的终端窗口

### 1. 清理 `DerivedData`

这是通常最安全的一项：

```bash
rm -rf ~/Library/Developer/Xcode/DerivedData
```

删除后不影响源码，只会让下次编译变慢一些。

### 2. 清理不可用模拟器

```bash
xcrun simctl delete unavailable
```

这条命令会删除已无效、不可用的模拟器数据，通常较安全。

### 3. 彻底清理模拟器设备数据

如果 `CoreSimulator` 仍然很大：

```bash
rm -rf ~/Library/Developer/CoreSimulator/Devices/*
```

这会清空模拟器设备及其 App 数据，影响包括：

- 模拟器回到较干净状态
- 模拟器中的登录状态、缓存、测试数据会丢失
- 不影响项目源码

### 4. 清理 `XCTestDevices`

如果这一项特别大（比如几十 GB、上百 GB），优先处理。

先看具体占用：

```bash
du -sh ~/Library/Developer/XCTestDevices/* 2>/dev/null | sort -hr | head -30
```

更稳妥的方式是先备份再观察：

```bash
mv ~/Library/Developer/XCTestDevices ~/Library/Developer/XCTestDevices.backup
mkdir -p ~/Library/Developer/XCTestDevices
```

确认 Xcode 正常后，再删除备份：

```bash
rm -rf ~/Library/Developer/XCTestDevices.backup
```

如果你已确认只是残留缓存，也可以直接清空：

```bash
rm -rf ~/Library/Developer/XCTestDevices/*
```

---

## 六、如果 `~/Library/Containers` 很大，重点看 Docker 和聊天软件

另一个常见大户是：

```bash
~/Library/Containers
```

可用下面命令快速查看：

```bash
du -sh ~/Library/Containers/* 2>/dev/null | sort -hr | head -30
```

常见目录包括：

- `com.docker.docker`
- `com.tencent.xinWeChat`
- Telegram、邮件、网盘等应用容器

---

## 七、Docker 占用很高时，怎么清理

如果 `~/Library/Containers/com.docker.docker` 很大（比如十几 GB 到几十 GB），先看结构：

```bash
docker system df
```

重点看：

- `Images`
- `Build Cache`

如果大部分是 `RECLAIMABLE`，说明这些空间基本可回收。

### 1. 先清理 Build Cache

```bash
docker builder prune -a
```

### 2. 再清理未使用镜像

```bash
docker image prune -a
```

这会删除未被当前容器使用的镜像。

### 3. 或者一步清理大部分可回收内容

```bash
docker system prune -a
```

通常会清理：

- 未使用镜像
- 构建缓存
- 已停止容器
- 未使用网络

### 4. 不建议一开始就加 `--volumes`

```bash
docker system prune -a --volumes
```

这条命令更激进，会连 volume 一起清理。volume 里可能有：

- 数据库数据
- 项目持久化数据
- 容器业务文件

除非你非常确定 volume 没有重要数据，否则先不要使用。

---

## 八、如何确认空间是否真的释放

清理后不要只看系统设置的颜色条，最好在终端再确认一次。

### 1. 查看根卷空间

```bash
df -h /
```


重点看：`Size`、`Used`、`Avail`、`Capacity`。

### 2. 再看 Developer 是否明显下降

```bash
du -sh ~/Library/Developer
du -sh ~/Library/Developer/* 2>/dev/null | sort -hr
```

### 3. 必要时重启一次 Mac

macOS 的存储分类页面不是总会即时刷新。有时需要等待几分钟，或重启后再看：

**系统设置 -> 通用 -> 储存**

---

## 九、一套推荐的排查顺序（可直接照做）

### 第一步：排除常见来源

```bash
tmutil listlocalsnapshots /
du -sh ~/Library/Application\ Support/MobileSync/Backup
```

### 第二步：定位 `~/Library` 下的大目录

```bash
du -sh ~/Library/* 2>/dev/null | sort -hr | head -20
```

### 第三步：如果 `Developer` 很大，继续细查

```bash
du -sh ~/Library/Developer/* 2>/dev/null | sort -hr
du -sh ~/Library/Developer/Xcode/* 2>/dev/null | sort -hr
```

### 第四步：清理 Xcode/Simulator 残留

```bash
rm -rf ~/Library/Developer/Xcode/DerivedData
xcrun simctl delete unavailable
rm -rf ~/Library/Developer/CoreSimulator/Devices/*
```

若 `XCTestDevices` 特别大，再处理：

```bash
rm -rf ~/Library/Developer/XCTestDevices/*
```

### 第五步：检查 Docker

```bash
du -sh ~/Library/Containers/* 2>/dev/null | sort -hr | head -30
docker system df
```

### 第六步：清理 Docker 缓存与镜像

```bash
docker builder prune -a
docker image prune -a
docker system prune -a
```

---

## 十、哪些可以删，哪些要谨慎

### 一般可以删

- `~/Library/Developer/Xcode/DerivedData`
- 不再需要的 Simulator 数据
- `XCTestDevices` 残留
- Docker Build Cache
- 未使用的 Docker 镜像

### 需要谨慎处理

- Docker volumes
- `~/Library/Application Support` 里的应用数据
- 聊天软件本地数据
- 你不确定用途的数据库文件

### 不建议乱删

- `/System`
- `/Library` 下不认识的系统文件
- 不明来源的配置和驱动文件

---

## 十一、结论

如果你的 Mac 是开发机，遇到 **System Data 过大** 时，优先排查的通常不是系统本身，而是：

- `~/Library/Developer/XCTestDevices`
- `~/Library/Developer/CoreSimulator`
- `~/Library/Developer/Xcode/DerivedData`
- `~/Library/Containers/com.docker.docker`

真正有效的清理方法，不是盯着系统设置里的颜色条，而是：

> 用 `du` 找出最大目录，再有针对性地清理可再生数据。

这样更安全，也更高效。

---

## 附：本文命令汇总

```bash
tmutil listlocalsnapshots /
du -sh ~/Library/Application\ Support/MobileSync/Backup

du -sh ~/Library/Caches
du -sh ~/Library/Application\ Support
du -sh ~/Library/Developer
du -sh ~/Library/Containers
du -sh ~/Library/Group\ Containers
du -sh ~/Library/* 2>/dev/null | sort -hr | head -20

du -sh ~/Library/Developer/* 2>/dev/null | sort -hr
du -sh ~/Library/Developer/Xcode/* 2>/dev/null | sort -hr
du -sh ~/Library/Containers/* 2>/dev/null | sort -hr | head -30

rm -rf ~/Library/Developer/Xcode/DerivedData
xcrun simctl delete unavailable
rm -rf ~/Library/Developer/CoreSimulator/Devices/*
rm -rf ~/Library/Developer/XCTestDevices/*

docker system df
docker builder prune -a
docker image prune -a
docker system prune -a

df -h /
```