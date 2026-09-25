---
title: "【飞牛OS 实战】Token 用不起？部署 Sub2API 把订阅变私有网关（含应用卡死/自启/权限排查实录）"
date: 2026-09-24T22:45:00-07:00
draft: false
tags: ["飞牛OS", "Sub2API", "Docker", "AI", "NAS", "Gemini", "DeepSeek", "系统运维", "故障排查"]
summary: "硬核保姆级实战：在飞牛 OS (fnOS) 上用 Docker 部署 Sub2API 网关，把网页端 Gemini / Claude 订阅逆向封装为标准 OpenAI /v1 接口；并附赠飞牛 OS 应用中心实操排障全记录（进程强杀、状态锁解除、禁止开机自启、mode 777 凭据权限修复）。"
---

> 保姆级实操教程 · 订阅转私有网关 · 飞牛 OS 底层运维排障实录 · 附完整 docker-compose 配置

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

这篇博客，我带你从零开始，在**飞牛 OS（fnOS）**上把整套东西搭起来；文末还整理了飞牛系统深度排查与权限避坑指南（涵盖进程管理、应用卡死解除、自启控制与凭据权限修复）。

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

接着打开 SSH 终端，连接你的飞牛 OS（例如 `ssh jack@10.0.0.56`），执行：

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

然后容器直接 `Exit(1)` 卡死，你还查不出原因。提前赋权，是最省事的避坑方式。

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
      - DATABASE_URL=postgres://sub2api:你的强密码@postgres:5432/sub2api
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
      - POSTGRES_PASSWORD=你的强密码        # ⚠️ 必须与上面 DATABASE_URL 里的完全一致
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

1. **数据库密码必须两处一致**：`POSTGRES_PASSWORD` 和 `DATABASE_URL` 里的密码要一字不差。
2. **只暴露 8080**：Postgres 和 Redis **绝对不要映射宿主机端口**。它们只在 `sub2api-net` 内网互通，不对外暴露，防止被扫到爆破。
3. **三个容器必须同一网络**：`sub2api-net` 让它们能用服务名互相解析（`postgres:5432`、`redis:6379`）。

---

## 五、Sub2API 部署三大避坑指南

### 避坑 ①：镜像拉取失败 `manifest unknown` 或超时

点启动就报错：
```
Error response from daemon: manifest unknown
Error response from daemon: Get "https://ghcr.io/v2/": context deadline exceeded
```

**原因分析**：
- 国内网络访问 ghcr.io / docker.io 不稳定。
- 架构不匹配：Sub2API 迭代频繁，若写 `latest` 标签极易拉到 arm64 镜像，导致在 x86 机器上无法运行。

**✅ 正确解法**：
1. 在飞牛「Docker 设置」中配置有效的国内镜像加速源。
2. 固定拉取与自己 CPU 架构匹配的具体 Tag：
```yaml
image: ghcr.io/sub2api/sub2api:v1.2.0        # x86_64 / amd64
image: ghcr.io/sub2api/sub2api:v1.2.0-arm64  # arm64 设备
```

---

### 避坑 ②：数据库鉴权崩溃 `FATAL: password authentication failed`

容器反复 `Restarting (1)`，日志爆红：
```
sub2api  | FATAL: password authentication failed for user "sub2api"
sub2api  | panic: failed to connect to database
```

**隐藏陷阱**：
Postgres **只在初次初始化数据卷**时创建数据库并写入密码。如果第一次启动时密码配置有误，`pgdata` 已经生成了旧数据，之后即便修改 compose 文件也无法覆盖！

**✅ 正确解法**：
```bash
# 1. 停止并移除容器
docker compose down

# 2. 彻底清空持久化卷（初次部署无重要数据时）
rm -rf /vol1/docker/sub2api/pgdata/*

# 3. 检查并确保两处密码完全一致后重新拉起
docker compose up -d
```

---

### 避坑 ③：客户端请求 403 `INSUFFICIENT_BALANCE`

接入客户端测试时返回错误：
```json
{
  "error": {
    "message": "User quota is not enough",
    "type": "sub2api_quota_error",
    "code": "insufficient_user_quota"
  }
}
```

**原因分析**：
Sub2API 内置多租户计量沙盒。新创建的账户默认额度为 0。即使上游订阅无限制，没有本地网关额度仍会被拦截。

**✅ 正确解法**：
1. 登录 Web 管理后台（`http://飞牛IP:8080`），进入「用户管理」，将账户额度修改为无限（如 `999999`）。
2. 在「令牌管理」中创建新 Token：设置永不过期、额度不限，生成得到 `sk-` 开头的 API 密钥。

---

## 六、渠道接入与高可用配置

在 Sub2API 仪表盘中，点击「渠道管理」→「新建渠道」：

| 配置项 | 推荐填法 | 说明 |
| :--- | :--- | :--- |
| **渠道类型** | Gemini 网页版 / Claude 网页版 | 根据你持有的订阅类型选择 |
| **渠道名称** | `Gemini-Pro-Pool` | 自定义标识 |
| **Session Cookie** | 填入 `__Secure-1PSID` 等凭据 | 网页登录后 F12 抓包获取 |
| **模型重定向** | `gemini-1.5-pro -> gpt-4o` | 让只认 GPT 的老客户端直接调用 |

> 🔒 **安全守则**：Cookie 与 API Token 等同于账号最高控制权，全流程本地保存，严禁分享或上传公开仓库。

### 多账号轮询与故障转移（推荐）
如果你有多个订阅账号，建议全部录入并划分至同一渠道分组：
```
客户端请求流 ──> Sub2API 轮询引擎 ──┬──> 账号 A (主用 70% 权重)
                                    └──> 账号 B (备用 30% 权重)
```
当主力账号遭遇风控或触发 429 限流时，网关自动将请求无缝熔断切换至备用账号，保证服务不中断。

---

## 七、客户端实战接入示例

### 1. 沉浸式翻译（浏览器插件）
- **接口地址**：`http://10.0.0.56:8080/v1`
- **API 密钥**：`sk-你的令牌`
- **模型**：`gpt-4o`（通过模型重定向实际映射到 Gemini 1.5 Pro）
- **效果**：长文档数秒翻完，零账单支出。

### 2. Claude Code 命令行编程
```bash
export ANTHROPIC_BASE_URL=http://10.0.0.56:8080
export ANTHROPIC_API_KEY=sk-你的令牌
claude "分析仓库结构并补充单元测试"
```

### 3. NextChat / Chatbox 客户端
协议选择 **OpenAI**，API 域名填写 `http://10.0.0.56:8080`，输入 `sk-` Key，手机、平板和内网设备皆可畅享。

---

## 八、飞牛 OS 进阶实操与排障手册（系统底层运维实录）

在飞牛 OS 上运行 AI 工具与应用中心程序时，你可能会遇到**应用卡在启用中无法停止、开机自动拉起、或者是像 DeepSeek Harness / DSH 等应用由于凭据权限严格断言（mode 777）而秒退**的情况。以下整理自真实实机排查经验，供所有飞牛玩家参考：

### 1. 应用中心卡在“启用中”转圈，界面无法点击停止？

飞牛 OS 应用中心界面有时因启动钩子未返回而卡在“启用中”，此时前端按钮不可点。

**排查与解决**：
1. **SSH 登录飞牛**：`ssh jack@10.0.0.56`
2. **检查 systemd 服务**：
   ```bash
   systemctl list-units --type=service | grep -iE "harness|sub2api|app"
   sudo systemctl stop <服务名>
   ```
3. **若提示 `Unit not loaded`（非标准服务托管）**：说明是由飞牛框架（`trim_app`）直接派生的后台任务，直接按关键词抓取并强杀进程：
   ```bash
   ps aux | grep -E "harness|dsh|sub2api" | grep -v grep
   sudo pkill -9 -f harness
   sudo pkill -9 -f dsh
   ```
4. **后台无进程但网页仍在转圈（清除状态锁）**：
   飞牛应用守护进程在等待超时或残留了状态锁文件，执行重启管理服务并清理锁：
   ```bash
   sudo systemctl restart trim_appcenter trim_app_daemon 2>/dev/null || sudo systemctl restart trim*app* 2>/dev/null
   sudo rm -f /var/run/trim*app*.lock /tmp/trim*app*.lock 2>/dev/null
   ```
   刷新浏览器（Ctrl + F5），前端即可恢复正常控制状态。

---

### 2. 飞牛 OS 应用目录结构与禁止开机自启

飞牛 OS 应用通常存放于 `/vol1/` 主存储卷下：
- `@appmeta/`：存放应用元数据及配置声明（包含自启配置）
- `@appconf/`：应用运行时生成的配置
- `@apphome/` 或 `@appcenter/`：主程序和启动/停止控制脚本
- `@appshare/`：持久化数据与共享资源

**若图形界面关闭自启不生效，可在终端彻底封锁**：
```bash
# 1. 查找应用相关目录
find /vol* -maxdepth 3 -type d -name "*harness*" 2>/dev/null

# 2. 将元数据中的 autostart 参数设为 false
sed -i -E 's/"(auto_start|autostart)":\s*true/"": false/g' /vol1/@appmeta/*harness*/* /vol1/@appconf/*harness*/* 2>/dev/null

# 3. 封禁启动入口脚本（最稳妥）
for file in /vol1/@appcenter/deepseek-harness/*.sh; do
  [ -f "$file" ] && mv "$file" "${file}.disabled"
done

# 重启管理中心生效
systemctl restart trim_appcenter 2>/dev/null
```

---

### 3. 外部访问与安全可信白名单（Settings unavailable 报错）

在部分 AI 应用（如 DSH）的 Models 页面若提示 `settings are unavailable in this browser`：
- **原因**：程序默认仅监听 `127.0.0.1:3080`，由飞牛前端反向代理通过 iframe 嵌入；若未配置可信域名/IP 白名单，后端接口会直接拦截。
- **解决规则**：
  - 监听地址：保持 `127.0.0.1` 内部监听
  - **可信访问地址**：必须填入你平时在浏览器中访问飞牛的地址和端口（无需 `http://`，逗号分隔），例如：
    ```text
    10.0.0.56:5666, 10.0.0.56
    ```

---

### 4. 关键踩坑：凭据权限 `mode 777` 导致应用秒退（chmod 600 修复）

很多开发者习惯性给目录执行 `chmod -R 777`，但部分严格遵循 Linux 安全规范的 AI 工具（例如读取 `.credentials.yaml`、私钥、Token 的程序）在启动时会进行安全断言：

```
credentials-local: /vol1/@appshare/DeepSeekHarness/.dsh/.credentials.yaml is readable beyond its owner (mode 777); 
run "chmod 600 /vol1/@appshare/DeepSeekHarness/.dsh/.credentials.yaml" before starting again
[Runner] dsh 进程退出，退出码: 1, 信号: null
```

**原因分析**：
飞牛在 `@appshare` 下创建的文件如果开放了过大权限（777），程序会认为存在同主机其他用户窥探凭据的泄露风险，从而主动退出拒绝执行。

**✅ 正确修复步骤**：
```bash
# 1. 将凭据文件权限强制收紧为仅所有者可读写 (600)
chmod 600 /vol1/@appshare/DeepSeekHarness/.dsh/.credentials.yaml

# 2. 将配置目录权限收紧为 700
chmod 700 /vol1/@appshare/DeepSeekHarness/.dsh

# 3. 验证权限
ls -l /vol1/@appshare/DeepSeekHarness/.dsh/.credentials.yaml
# 输出必须为：-rw------- 1 ...

# 4. 恢复此前重命名的启动脚本并启动应用
for file in /vol1/@appcenter/deepseek-harness/*.disabled; do
  [ -f "$file" ] && mv "$file" "${file%.disabled}"
done
```
在飞牛网页中心点击启动，应用即可秒级恢复常驻运行（状态显示绿色运行中）。

---

## 九、总结与运维常用速查

| 场景 | 命令 / 操作 |
| :--- | :--- |
| **容器状态查看** | `docker compose ps` / `docker stats` |
| **实时日志跟踪** | `docker compose logs -f sub2api` |
| **强行终止卡死应用** | `pkill -9 -f <关键字>` |
| **释放前端状态死锁** | `systemctl restart trim_appcenter && rm -f /var/run/trim*app*.lock` |
| **凭据文件权限修复** | `chmod 600 <credentials_path>` |

无论是折腾 Docker 版 Sub2API 把网页订阅转化为源源不断的私有模型算力，还是深度调试飞牛 OS 原生应用中心的服务与权限体系，核心都是**弄清底层调用链与权限隔离机制**。

掌控自己的 NAS，掌控自己的私有 AI 基础设施！
