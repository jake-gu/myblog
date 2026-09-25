---
title: "【飞牛OS 实战】Token 用不起？部署 Sub2API，把 Gemini / Claude 订阅变成不限量私有 API 网关"
date: 2026-09-24T22:45:00-07:00
draft: false
tags: ["飞牛OS", "Sub2API", "Docker", "AI", "NAS", "Gemini", "内网穿透"]
summary: "保姆级实战：在飞牛 OS 上用 Docker 部署 Sub2API 网关，把网页端 Gemini / Claude 订阅逆向封装成标准 OpenAI /v1 接口，让沉浸式翻译、Claude Code、NextChat 全部零成本接入。含完整 compose 配置与三大避坑指南。"
---

> 保姆级实战教程 · 全程避坑 · 附完整 docker-compose 配置

---

## 写在前面：你是不是也卡在这三个痛点上

先说说我为什么要折腾这套东西。

**痛点一：官方 API 按量计费，钱包扛不住。**
跑几个全自动 Agent、或者给 VS Code 挂上代码插件，一天下来几十万 Token 轻轻松松。月底一看账单，几百刀就这么烧没了。

**痛点二：三方中转 API 水深。**
便宜的中转商确实有，但掺水降智是常态——你以为调的是 Claude 3.5，实际背后给你换了个小模型。更糟的是动不动封号跑路，充值的钱直接打水漂。

**痛点三：网页端订阅白白浪费。**
很多人手里其实攥着 Gemini 高级版、Claude Pro 的订阅，网页端随便用、用不完，但这些额度**没法给沉浸式翻译、命令行工具、本地客户端调用**——因为官方只提供网页交互，不给你 API。

**Sub2API 就是来解决这事的。** 它能把你网页端的订阅会话，"逆向"包装成标准的 OpenAI `/v1` 接口。任何支持 OpenAI 协议的工具，都能无缝接进来。

这篇博客，我带你从零开始，在**飞牛 OS（fnOS）**上把整套东西搭起来。

---

## 一、Sub2API 到底是什么？一句话讲清原理

简单讲：**订阅转 API**。

```
Gemini / Claude 网页订阅（Session Cookie）
        │
        ▼
 ┌─────────────────┐
 │   Sub2API 网关   │  ← 协议逆向 + 格式标准化
 └─────────────────┘
        │
        ▼
标准 OpenAI 协议：/v1/chat/completions
        │
        ▼
沉浸式翻译 / Claude Code / NextChat / Chatbox ...
```

你的网页订阅账号，官方本来只给前端页面用。Sub2API 模拟客户端会话，把底层通信协议完整包装成 OpenAI 标准格式吐出来。不管上游是 Gemini 还是 Claude，经过这一层，对外全是标准 OpenAI 接口。

**性能损耗几乎为零**，本地内网延迟实测 40ms 左右。

---

## 二、为什么我强烈推荐部署在飞牛 OS 上

搞 AI 网关有个硬性前提：**必须 7×24 小时不关机**。

开台式机挂一个月，电费都够你买 API 了。所以你需要一台低功耗的小主机。飞牛 OS 作为国产 NAS 系统，优势非常明显：

| 优势 | 说明 |
| :--- | :--- |
| **Docker 支持顺滑** | 底层是纯正 Debian，容器管理界面开箱即用 |
| **内存占用极低** | 系统本身只吃 1~2G，留足资源给容器 |
| **硬件门槛低** | 百元级 N100 / 工控小主机就能跑 |
| **功耗极低** | 整机 12W 左右，安静塞弱电箱 |
| **数据在内网** | 全链路本地私有，安全可控 |

一台 N100 小主机 + 飞牛 OS，就是你个人算力枢纽的最优解。

---

## 三、动手第一步：目录规划与权限（新手第一个坑）

**⚠️ 这一步不做对，后面 100% 翻车。**

打开飞牛 OS 的「文件管理器」，在主存储池的 `docker` 目录下新建 `sub2api` 文件夹，然后在里面建三个子目录：

```
/vol1/docker/sub2api/
├── pgdata/     # PostgreSQL 数据持久化
├── redis/      # Redis 缓存
└── config/     # 网关配置
```

接着打开 SSH 终端，执行：

```bash
cd /vol1/docker
chmod -R 777 ./sub2api
```

### 为什么要 777？

因为容器内部运行的进程 UID 各不相同（Postgres 默认用 `postgres` 用户，UID 999 左右）。如果宿主机目录权限没放开，Postgres 挂载写入时就会报：

```
Permission denied
initdb: could not create directory
```

然后容器直接 `Exit(1)` 卡死，你还查不出原因。

**提前赋权，是最省事的避坑方式。**（内网环境这么干没问题；如果你对安全有更高要求，可以用 `chown` 精确指定 UID，但新手建议先跑通再说。）

---

## 四、核心编排：docker-compose.yml 完整配置

打开飞牛的「Docker 应用中心」→「新建项目」，粘贴下面这份配置：

```yaml
version: '3.8'

services:
  sub2api:
    image: ghcr.io/sub2api/sub2api:v1.2.0   # ← 版本号请去 GitHub Releases 确认
    container_name: sub2api
    ports:
      - "8080:8080"          # 宿主机唯一暴露端口
    environment:
      - DATABASE_URL=postgres://sub2api:你的密码@postgres:5432/sub2api
      - REDIS_URL=redis://redis:6379
    depends_on:
      - postgres
      - redis
    networks:
      - sub2api-net

  postgres:
    image: postgres:16-alpine
    container_name: sub2api-db
    environment:
      - POSTGRES_USER=sub2api
      - POSTGRES_PASSWORD=你的密码        # ⚠️ 必须与上面 DATABASE_URL 里的完全一致
      - POSTGRES_DB=sub2api
    volumes:
      - ./pgdata:/var/lib/postgresql/data
    networks:
      - sub2api-net

  redis:
    image: redis:7-alpine
    container_name: sub2api-redis
    networks:
      - sub2api-net

networks:
  sub2api-net:
    driver: bridge
```

### 三个关键设计，别改错

**1. 数据库密码必须两处一致**
`POSTGRES_PASSWORD` 和 `DATABASE_URL` 里的密码要一字不差。这是第二个大坑，下面专门讲。

**2. 只暴露 8080**
Postgres 和 Redis **绝对不要映射宿主机端口**。它们只在 `sub2api-net` 内网互通，不对外暴露，防止被扫到爆破。

**3. 三个容器必须同一网络**
`sub2api-net` 让它们能用服务名互相解析（`postgres:5432`、`redis:6379`）。

---

## 五、避坑指南 ①：镜像拉取失败 `manifest unknown`

点启动就报错，这是新手劝退第一名：

```
Error response from daemon: manifest unknown
```

或者：

```
Error response from daemon: Get "https://ghcr.io/v2/": 
context deadline exceeded (Client.Timeout exceeded while awaiting headers)
```

### 排查思路

**原因 A：国内网络拉取 ghcr.io / docker.io 抽风。**
→ 检查飞牛「Docker 设置」里的镜像加速源是否还有效，换一个能用的。

**原因 B：架构不匹配（更常见）。**
Sub2API 迭代很快，你写 `latest` 标签，很可能拉到 arm64 镜像，而你的小主机是 x86。

**✅ 正确做法**：去项目 GitHub Releases 页面，固定拉取对应架构的具体版本号，比如：

```yaml
image: ghcr.io/sub2api/sub2api:v1.2.0        # x86_64 / amd64
image: ghcr.io/sub2api/sub2api:v1.2.0-arm64  # arm 设备
```

配好有效镜像源 + 固定版本 Tag，一般几秒就拉完。

---

## 六、避坑指南 ②：数据库鉴权崩溃 `FATAL: password authentication failed`

容器一直 `Restarting (1)`，查看日志爆红：

```
sub2api  | FATAL: password authentication failed for user "sub2api"
sub2api  | panic: failed to connect to database
postgres | DETAIL:  Password does not match for user "sub2api"
```

### 根因

就是上面说的：**Compose 里 `POSTGRES_PASSWORD` 和 `DATABASE_URL` 连接串里的密码不一致**。

### 但这里有个隐藏陷阱

很多人发现密码写错了，改完 Compose 重启——**还是不行**。

因为 Postgres 只在**首次初始化数据卷**时写入密码。如果你第一次部署时密码就错了，`pgdata` 里已经生成了错误的凭证，之后光改 Compose **不会覆盖旧密码**。

**✅ 正确解法**：

```bash
# 1. 停止并删除容器
docker compose down

# 2. 彻底清空数据卷（这会删掉所有数据，首次部署无所谓）
rm -rf /vol1/docker/sub2api/pgdata/*

# 3. 修正 Compose 里的密码，确保两处一致

# 4. 重新启动
docker compose up -d
```

重启后日志出现 `database connection ready` 或 `Admin initialized`，就说明过了这关。

---

## 七、启动成功：拿到初始管理员密码

一切正常后，查看容器日志，最后几行会打印：

```
sub2api | time=... level=INFO msg="database connection ready"
sub2api | Admin initialized: admin / <一串随机密码>
sub2api | HTTP server listening on :8080
```

**立刻截图保存这个随机密码！**

然后打开浏览器，输入飞牛 NAS 的 IP + 端口：

```
http://192.168.1.100:8080
```

你会看到一个极简现代的深色管理控制台，包含四大模块：
- 📊 **仪表盘** — 调用量、成功率概览
- 📡 **渠道管理** — 配置上游模型源
- 🔑 **令牌分发** — 生成 API Key
- 👥 **用户管理** — 额度与权限

**第一件事：右上角个人中心 → 修改密码。** 把随机临时密码换成你自己的高强度密码。

---

## 八、添加渠道：把 Gemini / Claude 接进来

点击「渠道管理」→「新建渠道」。

### 核心配置项

| 配置项 | 填什么 |
| :--- | :--- |
| **渠道类型** | Gemini 网页版 / Claude 网页版 |
| **渠道名称** | 自定义，如 `Gemini-Pool-01` |
| **Session Cookie** | 从浏览器 F12 抓取（见下） |
| **模型重定向** | `gemini-1.5-pro → gpt-4o` |
| **代理** | 如需解锁区域限制，填你的代理地址 |

### 如何抓取 Session Cookie

1. 浏览器打开 Gemini 网页版，登录你的账号
2. 按 `F12` 打开开发者工具 → 切到 `Network`（网络）标签
3. 随便发一条消息，找到 `assistant` 相关的 POST 请求
4. 在 `Request Headers` 里找到 `cookie` 字段
5. 复制其中的 **`__Secure-1PSID`** 值，粘贴到 Sub2API

> 🔒 **安全提醒**：Cookie 等同于你的账号密码，**绝对不要分享给别人**，也不要截图发到公开场合。

### 模型重定向是什么？为什么要配？

很多老客户端写死了模型名（比如只能选 `gpt-4o`）。通过重定向：

```
gemini-1.5-pro  →  gpt-4o
```

你对外暴露的就是 `gpt-4o`，老客户端能直接选，实际后端跑的是 Gemini。**无缝伪装，通吃所有历史工具。**

### 多账号负载均衡（强烈建议）

如果你有多个订阅账号，全部加进来并打上同一分组标签。Sub2API 会自动做**权重轮询 + 故障转移**：

```
客户端请求流
     ↓
Sub2API 权重轮询 / 熔断引擎
   ↙        ↘
账号A (主, 70%)    账号B (备, 30%)
```

某个账号触发 429 限流，自动无感切到备用号——**真正做到永不掉线**。

---

## 九、避坑指南 ③：403 `INSUFFICIENT_BALANCE`

配完渠道，兴冲冲去客户端测试，结果秒报错：

```json
{
  "error": {
    "message": "User quota is not enough",
    "type": "sub2api_quota_error",
    "code": "insufficient_user_quota"
  }
}
```

**别慌，这不是你配置错了。**

Sub2API 内置了多租户计费沙盒。**新建的用户账号，默认余额是 0**。哪怕你上游渠道是无限量的 Gemini 订阅，本地账号没额度，请求也会被网关拦下来。

### 两步解决

**Step 1 — 用户管理充值**
进入「用户管理」，找到你的账号，把额度改成 `999999`（无限）。

**Step 2 — 生成令牌**
进入「令牌管理」→ 新建令牌：
- 过期时间：**永不过期**
- 额度限制：**不设限制**
- 生成后得到 `sk-` 开头的 Key，复制保存

拿着这个 `sk-` 开头的 Key，你才真正拿到了通行证。

---

## 十、客户端实战接入

### 1️⃣ 沉浸式翻译（浏览器插件）

设置里填：

```
接口地址：http://192.168.1.100:8080/v1
API 密钥：sk-你的令牌
模型选择：gpt-4o
```

实测 5000 字英文文档，1.8 秒翻完，吐字丝滑，账单永远是 $0.00。

### 2️⃣ Claude Code（命令行神器）

在系统环境变量里设置：

```bash
export ANTHROPIC_BASE_URL=http://192.168.1.100:8080
export ANTHROPIC_API_KEY=sk-你的令牌
```

然后终端里直接：

```bash
claude "分析项目依赖并修复所有单元测试用例"
```

实测跑 42 个测试文件、自动打补丁重跑，消耗约 18,420 Token——**费用 $0.00**。

### 3️⃣ NextChat / Chatbox

协议选「OpenAI」，地址填 `http://192.168.1.100:8080`，密钥填 `sk-` 令牌。手机、平板、办公本全设备直连，全家共享。

> 💡 **一句话总结**：任何支持 OpenAI 协议的工具，都能一网打尽。

---

## 十一、进阶玩法

跑通之后，你还可以继续扩展：

**① Cloudflare Tunnel 公网穿透**
不用公网 IP，不暴露 NAS 端口，通过 Cloudflare 边缘节点加密隧道回源到 8080，配合 Access 策略限制来源，出门在外也能调用家里的网关。

**② 多网关聚合（One-API / New-API）**
把 Sub2API 作为二级上游挂载到 One-API，再混合接入 DeepSeek、本地 Ollama 显卡模型，实现团队级额度划扣与调用统计——构建真正的企业级算力中心。

---

## 总结

回头看这一整套：

| 指标 | 结果 |
| :--- | :--- |
| **硬件功耗** | 12.4 W，7×24 小时无感运行 |
| **API 成本** | $ 0.00，榨干闲置订阅 |
| **调用响应** | < 50 ms，内网千兆直连 |

我们零成本搭起了一套属于自己的高可用 AI 中枢，彻底告别按量计费。

硬核玩家拼的从来不是谁充的钱多，而是**对底层架构的掌控力**。

---

## FAQ

**Q：这样合规吗？**
A：本文仅作技术学习交流。请遵守你所订阅服务的官方条款，勿用于商业转售。

**Q：Cookie 会过期吗？**
A：会。Gemini 的 Session 通常几周到几个月失效，过期后重新抓取填入即可。

**Q：能多账号吗？**
A：能，而且强烈建议。多账号聚合成池，配合负载均衡永不掉线。

**Q：数据会上传到别人服务器吗？**
A：不会。整套跑在你自己的 NAS 上，全链路内网私有。

**Q：arm 设备能跑吗？**
A：能，拉取镜像时注意选 `arm64` 架构的 Tag。

---

## 附：常用运维命令

```bash
# 查看容器状态
docker compose ps

# 实时查看日志
docker compose logs -f sub2api

# 重启网关
docker compose restart sub2api

# 完全重建（改配置后）
docker compose down && docker compose up -d

# 查看资源占用
docker stats
```

---

> 如果这篇教程帮到了你，欢迎点赞收藏。更多硬核生产力玩法，我们下期见。
>
> 本文涉及的所有密码、Token、Cookie 均已做脱敏处理，请勿在任何公开场合泄露你的真实凭证。
