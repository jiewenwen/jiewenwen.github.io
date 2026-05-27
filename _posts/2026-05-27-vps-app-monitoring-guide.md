---
layout: post
title: "小白也能看懂：VPS 上部署 App 后端后，如何监控服务器和应用异常"
date: 2026-05-27 12:00:00 +0800
categories: [Development]
tags: [vps, Docker, Prometheus, Grafana, Uptime Kuma, Sentry, Node Exporter, 后端监控]
---

如果你在 VPS 上部署了自己的 App 后端，例如 Go、Node.js、Java、Python 后端，那么上线之后最重要的问题不是“服务能不能跑起来”，而是：

> 服务挂了，我能不能第一时间知道？  
> 用户访问慢了，我能不能定位原因？  
> VPS CPU、内存、磁盘异常，我能不能提前发现？  
> 后端出现 500、panic、数据库连接失败，我能不能查到日志？

很多小白一开始部署后端时，只会用：

```bash
docker ps
docker logs
~~~

或者偶尔 SSH 上去看一下机器状态。但这远远不够。

本文会从零开始，讲一套适合个人开发者、小团队、早期 App 项目的监控方案。

目标是：**不追求复杂，但要实用、安全、可告警、可定位问题。**

------

## 一、我们到底要监控什么？

VPS 上部署 App 后端之后，至少要监控四类东西：

| 类型           | 监控内容                             |
| -------------- | ------------------------------------ |
| 可用性监控     | 网站、API、健康检查接口是否能访问    |
| VPS 性能监控   | CPU、内存、磁盘、网络、系统负载      |
| App 后端监控   | 请求量、接口耗时、错误率、数据库状态 |
| 日志和异常监控 | 500 错误、panic、慢请求、异常堆栈    |

简单来说：

```text
Uptime Kuma：看服务还活着没
Prometheus：收集监控数据
Node Exporter：收集 VPS 系统数据
Grafana：把数据画成图表
Sentry：收集后端异常和错误堆栈
Loki：后期用于集中收集日志
```

对于小白来说，不建议一上来就搭很复杂的监控平台。可以分阶段来。

------

## 二、推荐的小白监控方案

我推荐的组合是：

```text
第一阶段：Uptime Kuma + Prometheus + Node Exporter + Grafana
第二阶段：后端接入 /metrics 和 Sentry
第三阶段：Loki + Alloy 做日志集中化
```

其中第一阶段最重要，建议所有 VPS 后端项目都部署。

### 第一阶段必须部署

| 工具          | 作用                                  |
| ------------- | ------------------------------------- |
| Uptime Kuma   | 监控网站/API 是否能访问，异常后发通知 |
| Node Exporter | 采集 VPS 的 CPU、内存、磁盘、网络数据 |
| Prometheus    | 存储监控数据                          |
| Grafana       | 展示监控图表                          |

### 第二阶段建议部署

| 工具           | 作用                          |
| -------------- | ----------------------------- |
| Sentry         | 监控后端异常、panic、错误堆栈 |
| App `/metrics` | 暴露后端性能指标给 Prometheus |

### 第三阶段后期部署

| 工具  | 作用                                  |
| ----- | ------------------------------------- |
| Loki  | 存储和查询日志                        |
| Alloy | 采集系统日志、Nginx 日志、Docker 日志 |

------

## 三、整体架构图

可以简单理解为：

```text
用户
 ↓
Nginx / Caddy
 ↓
App 后端
 ↓
数据库 / Redis
```

监控系统则是：

```text
Uptime Kuma  --->  检查 https://api.example.com/health 是否正常

Node Exporter ---> 采集 VPS CPU / 内存 / 磁盘 / 网络
       ↓
Prometheus -----> 存储监控数据
       ↓
Grafana --------> 展示图表和告警

App 后端 -------> 暴露 /metrics
       ↓
Prometheus -----> 采集 App 性能指标

App 后端 -------> Sentry
                 上报 panic / exception / 慢请求
```

------

## 四、准备工作

本文默认你的 VPS 是 Ubuntu 或 Debian。

你需要准备：

```text
一台 VPS
一个域名
Docker 和 Docker Compose
Nginx 或 Caddy
一个已经部署好的 App 后端
```

示例域名：

```text
api.example.com      后端 API
status.example.com   Uptime Kuma
grafana.example.com  Grafana
```

你可以根据自己的域名替换。

------

## 五、第一步：后端必须提供健康检查接口

在搭监控之前，后端最好先提供一个健康检查接口。

例如：

```http
GET /health
```

正常时返回：

```json
{
  "status": "ok",
  "version": "1.0.0",
  "database": "ok",
  "redis": "ok"
}
```

异常时返回：

```json
{
  "status": "error",
  "database": "failed"
}
```

建议 HTTP 状态码这样设计：

| 状态       | HTTP Code |
| ---------- | --------- |
| 后端正常   | 200       |
| 数据库异常 | 500       |
| Redis 异常 | 500       |
| 服务异常   | 500       |

为什么需要 `/health`？

因为它可以让监控系统判断：

```text
后端进程是否活着
数据库是否正常
Redis 是否正常
API 是否能正常响应
```

如果你只是监控网站首页，有时并不能发现真正的后端问题。

------

## 六、安装 Docker 和 Docker Compose

如果 VPS 还没有安装 Docker，可以执行：

```bash
curl -fsSL https://get.docker.com | sh
```

把当前用户加入 docker 组：

```bash
sudo usermod -aG docker $USER
```

然后重新登录 SSH。

检查 Docker 是否安装成功：

```bash
docker version
docker compose version
```

如果能看到版本号，说明 Docker 已经安装成功。

------

## 七、部署 Uptime Kuma

Uptime Kuma 用来监控：

```text
API 是否能访问
网站是否能打开
HTTPS 证书是否快过期
VPS 端口是否开放
服务异常时发送通知
```

### 1. 创建目录

```bash
sudo mkdir -p /opt/uptime-kuma
cd /opt/uptime-kuma
```

### 2. 创建 docker-compose.yml

```bash
sudo nano docker-compose.yml
```

写入：

```yaml
services:
  uptime-kuma:
    image: louislam/uptime-kuma:2
    container_name: uptime-kuma
    restart: unless-stopped
    volumes:
      - ./data:/app/data
    ports:
      - "127.0.0.1:3001:3001"
```

这里要注意：

```yaml
127.0.0.1:3001:3001
```

这表示 Uptime Kuma 只监听本机，不直接暴露到公网。

这是一个很重要的安全习惯。

### 3. 启动 Uptime Kuma

```bash
docker compose up -d
```

查看容器：

```bash
docker ps
```

查看日志：

```bash
docker logs -f uptime-kuma
```

------

## 八、用 Nginx 反代 Uptime Kuma

假设你准备用：

```text
https://status.example.com
```

访问 Uptime Kuma。

### 1. 安装 Nginx

```bash
sudo apt update
sudo apt install -y nginx
```

### 2. 创建 Nginx 配置

```bash
sudo nano /etc/nginx/sites-available/uptime-kuma
```

写入：

```nginx
server {
    listen 80;
    server_name status.example.com;

    location / {
        proxy_pass http://127.0.0.1:3001;
        proxy_http_version 1.1;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### 3. 启用配置

```bash
sudo ln -s /etc/nginx/sites-available/uptime-kuma /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

### 4. 申请 HTTPS 证书

安装 Certbot：

```bash
sudo apt install -y certbot python3-certbot-nginx
```

申请证书：

```bash
sudo certbot --nginx -d status.example.com
```

完成后，打开：

```text
https://status.example.com
```

第一次进入会让你创建管理员账号。

------

## 九、Uptime Kuma 应该监控哪些东西？

进入 Uptime Kuma 后，可以添加下面几个监控项。

### 1. 监控后端健康接口

类型选择：

```text
HTTP(s)
```

URL 填：

```text
https://api.example.com/health
```

建议设置：

```text
检查间隔：60 秒
失败重试：3 次
恢复通知：开启
```

这样可以避免偶发网络波动导致误报。

------

### 2. 监控网站首页

如果你还有官网，可以监控：

```text
https://example.com
```

------

### 3. 监控 SSH 端口

类型选择：

```text
TCP Port
```

Host：

```text
你的 VPS IP
```

Port：

```text
22
```

这样可以知道 VPS 的 SSH 端口是否还活着。

------

### 4. 监控 HTTPS 证书

Uptime Kuma 的 HTTP(s) 监控可以检查证书过期。

建议开启证书过期提醒，避免证书突然过期导致 API 无法访问。

------

## 十、部署 Prometheus + Node Exporter + Grafana

这一套用于监控 VPS 性能。

它可以让你看到：

```text
CPU 使用率
内存使用率
磁盘空间
网络流量
系统负载
磁盘 IO
服务器运行时间
```

### 1. 创建目录

```bash
sudo mkdir -p /opt/monitoring
cd /opt/monitoring
```

### 2. 创建 Prometheus 配置

```bash
sudo nano prometheus.yml
```

写入：

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: "vps"
    static_configs:
      - targets: ["node-exporter:9100"]

  - job_name: "prometheus"
    static_configs:
      - targets: ["prometheus:9090"]
```

含义：

| 配置               | 含义                     |
| ------------------ | ------------------------ |
| scrape_interval    | 每隔多久采集一次数据     |
| node-exporter:9100 | 采集 VPS 系统指标        |
| prometheus:9090    | 采集 Prometheus 自身指标 |

------

### 3. 创建 docker-compose.yml

```bash
sudo nano docker-compose.yml
```

写入：

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus-data:/prometheus
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.retention.time=30d"
    ports:
      - "127.0.0.1:9090:9090"
    networks:
      - monitoring

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    pid: host
    volumes:
      - /:/host:ro,rslave
    command:
      - "--path.rootfs=/host"
    expose:
      - "9100"
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    volumes:
      - ./grafana-data:/var/lib/grafana
    ports:
      - "127.0.0.1:3000:3000"
    networks:
      - monitoring

networks:
  monitoring:
    driver: bridge
```

### 4. 启动服务

```bash
docker compose up -d
```

查看容器：

```bash
docker ps
```

应该能看到：

```text
prometheus
node-exporter
grafana
```

------

## 十一、用 Nginx 反代 Grafana

假设你准备使用：

```text
https://grafana.example.com
```

### 1. 创建 Nginx 配置

```bash
sudo nano /etc/nginx/sites-available/grafana
```

写入：

```nginx
server {
    listen 80;
    server_name grafana.example.com;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### 2. 启用配置

```bash
sudo ln -s /etc/nginx/sites-available/grafana /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

### 3. 申请 HTTPS 证书

```bash
sudo certbot --nginx -d grafana.example.com
```

然后打开：

```text
https://grafana.example.com
```

Grafana 默认账号密码通常是：

```text
admin / admin
```

第一次登录后，系统会要求你修改密码。

请务必使用强密码。

------

## 十二、Grafana 添加 Prometheus 数据源

进入 Grafana 后，按下面步骤操作：

```text
Connections
→ Data sources
→ Add data source
→ Prometheus
```

URL 填：

```text
http://prometheus:9090
```

然后点击：

```text
Save & test
```

如果显示成功，说明 Grafana 已经可以读取 Prometheus 数据。

------

## 十三、导入 VPS 监控面板

Grafana 可以导入现成的 Node Exporter 面板。

常用 Dashboard ID：

```text
1860
```

操作步骤：

```text
Dashboards
→ New
→ Import
→ 输入 1860
→ 选择 Prometheus 数据源
→ Import
```

导入后，你就能看到 VPS 的各种指标。

例如：

```text
CPU
内存
磁盘
网络
系统负载
磁盘 IO
系统运行时间
```

------

## 十四、小白应该重点看哪些指标？

### 1. CPU

重点看：

```text
CPU 使用率
Load Average
```

简单判断：

| 情况                 | 说明           |
| -------------------- | -------------- |
| CPU 长期低于 50%     | 正常           |
| CPU 长期高于 70%     | 需要关注       |
| CPU 长期高于 90%     | 可能性能不足   |
| Load 高于 CPU 核心数 | 服务器压力偏大 |

例如你是 2 核 VPS，如果：

```text
Load Average 长期大于 2
```

说明服务器可能已经比较忙。

------

### 2. 内存

重点看：

```text
Available Memory
Swap Used
```

判断标准：

| 情况             | 说明               |
| ---------------- | ------------------ |
| 可用内存大于 30% | 正常               |
| 可用内存小于 15% | 需要关注           |
| Swap 持续增长    | 内存可能不够       |
| 出现 OOM Kill    | 后端可能被系统杀死 |

检查是否出现 OOM：

```bash
dmesg -T | grep -i "killed process"
```

或者：

```bash
journalctl -k | grep -i oom
```

------

### 3. 磁盘

重点看：

```text
Disk Usage
Disk IO
Inode Usage
```

判断标准：

| 情况               | 建议                 |
| ------------------ | -------------------- |
| 磁盘使用率 > 80%   | 开始清理             |
| 磁盘使用率 > 90%   | 立即处理             |
| inode 使用率 > 80% | 检查是否有大量小文件 |
| IO wait 很高       | 磁盘可能成为瓶颈     |

查看磁盘空间：

```bash
df -h
```

查看 inode：

```bash
df -i
```

查看大目录：

```bash
sudo du -h --max-depth=1 / | sort -h
```

------

### 4. 网络

重点看：

```text
上传流量
下载流量
连接数
是否有异常流量
```

如果网络有问题，常见表现是：

```text
接口变慢
连接超时
Nginx 出现 502
Nginx 出现 504
用户反馈打不开
```

------

## 十五、后端也需要自己的监控指标

只监控 VPS 是不够的。

因为 VPS 正常，不代表 App 正常。

例如：

```text
CPU 很低，但数据库连接失败
内存正常，但登录接口全部 500
磁盘正常，但 Redis 挂了
网络正常，但某个 API 慢到 5 秒
```

所以后端应该额外暴露 `/metrics` 接口。

Prometheus 可以定期抓取这个接口。

建议后端监控这些指标：

| 指标                  | 作用             |
| --------------------- | ---------------- |
| HTTP 请求总数         | 看流量变化       |
| HTTP 请求耗时         | 看接口是否变慢   |
| 5xx 错误率            | 看服务是否异常   |
| 数据库查询耗时        | 定位慢查询       |
| Redis 状态            | 判断缓存是否正常 |
| 登录成功/失败数量     | 监控关键业务     |
| 注册成功/失败数量     | 监控关键业务     |
| 消息发送成功/失败数量 | 监控核心功能     |

------

## 十六、Go 后端接入 Prometheus 示例

如果你的后端是 Go，可以使用 Prometheus client。

安装：

```bash
go get github.com/prometheus/client_golang/prometheus
go get github.com/prometheus/client_golang/prometheus/promhttp
```

最简单示例：

```go
package main

import (
    "net/http"

    "github.com/prometheus/client_golang/prometheus/promhttp"
)

func main() {
    http.Handle("/metrics", promhttp.Handler())
    http.ListenAndServe(":2112", nil)
}
```

然后在 Prometheus 配置中添加：

```yaml
  - job_name: "app-backend"
    static_configs:
      - targets: ["127.0.0.1:2112"]
```

如果你的后端在 Docker 网络里，也可以写容器名：

```yaml
  - job_name: "app-backend"
    static_configs:
      - targets: ["backend:2112"]
```

注意：要确保 Prometheus 和后端容器在同一个 Docker 网络里。

------

## 十七、后端日志应该怎么写？

日志不要随便打印。

推荐使用结构化日志。

一条正常请求日志可以这样：

```text
time=2026-05-27T10:00:00Z level=info request_id=abc123 method=POST path=/api/login status=200 latency=35ms user_id=1001 ip=1.2.3.4
```

一条错误日志可以这样：

```text
time=2026-05-27T10:01:00Z level=error request_id=def456 method=POST path=/api/login status=500 latency=120ms error="database connection failed"
```

关键字段包括：

| 字段       | 作用                |
| ---------- | ------------------- |
| time       | 发生时间            |
| level      | info / warn / error |
| request_id | 一次请求的唯一 ID   |
| method     | GET / POST          |
| path       | 请求路径            |
| status     | HTTP 状态码         |
| latency    | 请求耗时            |
| user_id    | 用户 ID             |
| ip         | 客户端 IP           |
| error      | 错误原因            |

其中最重要的是：

```text
request_id
```

因为用户反馈问题时，如果能拿到 request_id，你就可以快速找到对应日志。

------

## 十八、常用排查命令

### 1. 查看 Docker 容器

```bash
docker ps
```

查看所有容器，包括已经停止的：

```bash
docker ps -a
```

------

### 2. 查看后端日志

```bash
docker logs -f your-backend
```

查看最近 200 行：

```bash
docker logs --tail=200 your-backend
```

查看最近 1 小时：

```bash
docker logs --since=1h your-backend
```

------

### 3. 查看 Docker Compose 日志

进入你的 App 目录：

```bash
cd /opt/app
```

查看所有服务日志：

```bash
docker compose logs -f
```

只看后端：

```bash
docker compose logs -f backend
```

------

### 4. 查看 systemd 服务日志

如果你的后端不是 Docker，而是 systemd 服务：

```bash
journalctl -u your-app -f
```

最近 200 行：

```bash
journalctl -u your-app -n 200
```

最近 1 小时：

```bash
journalctl -u your-app --since "1 hour ago"
```

------

### 5. 查看 Nginx 日志

错误日志：

```bash
sudo tail -f /var/log/nginx/error.log
```

访问日志：

```bash
sudo tail -f /var/log/nginx/access.log
```

查看最近的 500 错误：

```bash
sudo grep " 500 " /var/log/nginx/access.log | tail -50
```

查看最近的 502 错误：

```bash
sudo grep " 502 " /var/log/nginx/access.log | tail -50
```

查看最近的 504 错误：

```bash
sudo grep " 504 " /var/log/nginx/access.log | tail -50
```

------

## 十九、常见故障排查

### 故障一：Uptime Kuma 显示 API 挂了

先在自己电脑上测试：

```bash
curl -i https://api.example.com/health
```

然后到 VPS 上测试：

```bash
curl -i http://127.0.0.1:你的后端端口/health
```

如果 VPS 本机访问也失败，说明后端本身有问题。

继续检查：

```bash
docker ps
docker logs --tail=200 your-backend
```

如果后端容器没运行，可以尝试启动：

```bash
docker start your-backend
```

如果是 Docker Compose：

```bash
cd /opt/app
docker compose up -d
```

------

### 故障二：网站显示 502 Bad Gateway

502 通常表示：

```text
Nginx 正常，但后端挂了，或者 Nginx 连不上后端。
```

检查 Nginx：

```bash
sudo nginx -t
sudo systemctl status nginx
```

检查后端：

```bash
docker ps
docker logs --tail=200 your-backend
```

检查端口：

```bash
ss -tulpn | grep 你的后端端口
```

本机测试后端：

```bash
curl -i http://127.0.0.1:你的后端端口/health
```

------

### 故障三：网站显示 504 Gateway Timeout

504 通常表示：

```text
后端响应太慢。
```

可能原因：

```text
数据库查询太慢
后端死锁
外部 API 超时
CPU 打满
内存不足
网络异常
```

检查 CPU：

```bash
top
```

检查内存：

```bash
free -h
```

查看后端日志：

```bash
docker logs --since=10m your-backend
```

查看 Nginx 错误日志：

```bash
sudo tail -100 /var/log/nginx/error.log
```

------

### 故障四：后端容器经常自动重启

查看容器状态：

```bash
docker ps
```

如果看到状态里经常出现：

```text
Restarting
```

说明容器在反复崩溃。

查看日志：

```bash
docker logs --tail=300 your-backend
```

检查是否被系统 OOM Kill：

```bash
dmesg -T | grep -i "killed process"
```

如果是 OOM，说明内存不足。

可以临时增加 swap：

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

永久启用 swap：

```bash
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

------

## 二十、基础告警建议

### 1. Uptime Kuma 告警

建议监控：

```text
https://api.example.com/health
https://example.com
VPS SSH 端口
HTTPS 证书过期时间
```

建议配置：

| 项目         | 建议  |
| ------------ | ----- |
| 检查间隔     | 60 秒 |
| 失败重试     | 3 次  |
| 恢复通知     | 开启  |
| 证书过期提醒 | 开启  |

------

### 2. Grafana 告警

后期可以添加：

| 指标           | 告警阈值               |
| -------------- | ---------------------- |
| CPU 使用率     | 5 分钟内持续 > 85%     |
| 内存可用率     | < 10%                  |
| 磁盘使用率     | > 80% 警告，> 90% 严重 |
| Load Average   | 高于 CPU 核心数 1.5 倍 |
| API 5xx 错误率 | > 1%                   |
| P95 延迟       | > 500ms 或 1s          |
| 后端进程重启   | 10 分钟内多次重启      |

------

## 二十一、接入 Sentry 监控后端异常

Sentry 适合监控：

```text
panic
exception
错误堆栈
慢请求
线上版本新增错误
```

如果你的后端是 Go，可以安装：

```bash
go get github.com/getsentry/sentry-go
```

基础示例：

```go
package main

import (
    "log"
    "time"

    "github.com/getsentry/sentry-go"
)

func main() {
    err := sentry.Init(sentry.ClientOptions{
        Dsn: "你的 Sentry DSN",
        TracesSampleRate: 0.2,
        Environment: "production",
        Release: "app-backend@1.0.0",
    })
    if err != nil {
        log.Fatalf("sentry.Init: %s", err)
    }

    defer sentry.Flush(2 * time.Second)

    sentry.CaptureMessage("backend started")
}
```

建议把 Sentry DSN 放在环境变量中，不要写死在代码里：

```env
SENTRY_DSN=你的 DSN
APP_ENV=production
APP_VERSION=1.0.0
```

------

## 二十二、后期再部署 Loki 做日志集中化

前期可以先用：

```bash
docker logs
journalctl
tail -f /var/log/nginx/error.log
```

等服务复杂后，再部署：

```text
Loki + Grafana Alloy + Grafana
```

Loki 的作用是集中存储和查询日志。

适合这些场景：

```text
多个后端服务
多个 VPS
日志很多
需要按 request_id 搜索
需要排查某个用户请求链路
```

对于小白或早期项目，Loki 不是第一优先级。

可以等你的 App 稳定上线后再加。

------

## 二十三、安全注意事项

这几个端口不要直接暴露公网：

```text
3000  Grafana
3001  Uptime Kuma
9090  Prometheus
9100  Node Exporter
3100  Loki
```

推荐做法：

```text
只绑定 127.0.0.1
通过 Nginx / Caddy 反代访问
Grafana 使用强密码
最好只允许 Tailscale / VPN / 固定 IP 访问
```

查看当前监听端口：

```bash
ss -tulpn
```

如果你使用 UFW 防火墙：

```bash
sudo ufw status
```

建议只开放：

```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

不要开放：

```bash
sudo ufw allow 9090
sudo ufw allow 9100
sudo ufw allow 3000
```

------

## 二十四、推荐目录结构

建议把 App 和监控分开放：

```text
/opt/app/
  docker-compose.yml
  .env
  logs/

/opt/uptime-kuma/
  docker-compose.yml
  data/

/opt/monitoring/
  docker-compose.yml
  prometheus.yml
  prometheus-data/
  grafana-data/
```

这样以后维护比较清楚：

```text
/opt/app            放你的后端服务
/opt/uptime-kuma    放可用性监控
/opt/monitoring     放 Prometheus / Grafana / Node Exporter
```

------

## 二十五、日常检查清单

### 每天看一次

```text
Uptime Kuma 有没有红色告警
Grafana CPU 是否异常
Grafana 内存是否异常
Grafana 磁盘是否快满
后端有没有大量 500
```

### 每周看一次

```text
磁盘空间
Docker 日志大小
Nginx 错误日志
后端错误日志
系统更新
证书有效期
```

查看 Docker 占用：

```bash
docker system df
```

清理无用镜像：

```bash
docker image prune
```

谨慎使用：

```bash
docker system prune -a
```

因为它可能删除你后面还需要的镜像。

------

## 二十六、最小可用版本总结

如果你不想一次性部署太多东西，至少先做到下面这些：

```text
1. 后端提供 /health
2. Uptime Kuma 监控 /health
3. Prometheus + Node Exporter 监控 VPS
4. Grafana 展示 CPU / 内存 / 磁盘 / 网络
5. 后端日志包含 request_id
6. 关键错误接入 Sentry
7. 监控后台不要直接暴露公网
```

最终推荐组合：

```text
必须：
Uptime Kuma
Prometheus
Node Exporter
Grafana

强烈推荐：
Sentry

后期再加：
Loki
Grafana Alloy
Alertmanager
```

------

## 结语

对于 VPS 上部署 App 后端的个人开发者来说，监控系统不需要一开始就特别复杂。

真正重要的是：

```text
服务挂了，你能知道；
服务器资源异常，你能知道；
后端报错，你能查到；
用户反馈问题，你能定位；
磁盘快满、证书快过期，你能提前处理。
```

所以，最推荐的起步方案就是：

```text
Uptime Kuma + Prometheus + Node Exporter + Grafana
```

然后随着项目发展，再逐步接入：

```text
Sentry
Loki
Grafana Alloy
Alertmanager
```

监控不是为了看起来专业，而是为了在服务出问题时，能够尽快发现、尽快定位、尽快恢复。

```
---

这篇文章已经是标准 Jekyll Markdown 格式，包含 Front Matter，可以直接放到 `_posts` 目录发布。标题、分类和 tags 也已经配置好了。
```