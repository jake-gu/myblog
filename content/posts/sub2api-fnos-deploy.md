---
title: "【飞牛OS 实战】Token 用不起？部署 Sub2API 把闲鱼 5 元 Gemini Pro 变成无限量私有网关"
date: 2026-09-25T02:20:00-07:00
draft: false
tags: ["飞牛OS", "Sub2API", "Docker", "Gemini", "反重力", "NAS", "AI网关"]
summary: "保姆级图文全流程：从闲鱼 5 元淘 Gemini Pro 账号、飞牛 OS (fnOS) 目录赋权与 Docker Compose 容器编排部署，到 Sub2API 控制台初始化、Jack 分组挂载 Antigravity 反重力节点、令牌签发及客户端（沉浸式翻译/Claude Code/NextChat）无缝调用全实操。"
---

> 保姆级全流程实操教程 · 5 元淘号到私有模型网关落地 · 飞牛 OS 容器编排 · 附完整配置与多端联调指南

---

## 写在前面：普通人实现"顶级算力自由"的最优解

先说说为什么大家需要这套方案：

1. **官方 API 太贵**：按量计费调几个自动化 Agent 或者给 IDE 挂个代码助手，一个月几百刀就没了。
2. **三方中转水太深**：掺水降智常态化，经常用小模型冒充大模型，还动不动就跑路封号。
3. **闲鱼白菜价资源用不起来**：闲鱼上 **5 块钱就能买到 Gemini Pro 现成账号**，但网页端只能自己打字聊，根本没法接入沉浸式翻译、Cursor、VS Code 或手机端。

**Sub2API + 反重力节点（Antigravity）+ 飞牛 OS（fnOS）**，就是专门解决这个问题的：
把几块钱买来的网页端会话通道，通过个人 NAS 容器逆向洗成**标准的 OpenAI `/v1` 接口**。全天 24 小时低功耗运行，全家全设备免费调用！

下面老顾一步一步带大家从零完整走通。

---

## 一、架构原理：Sub2API 是怎么工作的？

简单讲就是**协议逆向与标准化封装**：

```
闲鱼 5 元买号 / 官方订阅
        │
        ▼
   Antigravity 反重力节点（抹平风控与网络跳板）
        │
        ▼
 ┌─────────────────────────────────────────────┐
 │   飞牛 OS 本地部署的 Sub2API 网关 (8080)    │
 │  • 会话保持、Header 伪装与反向代理          │
 │  • PostgreSQL 存储用户凭证                  │
 │  • Redis 负责高并发流控与限速               │
 │  • 多租户沙盒、分组路由与故障自动转移       │
 └─────────────────────────────────────────────┘
        │
        ▼
标准 OpenAI 规范：http://10.0.0.56:8080/v1/chat/completions
        │
        ▼
沉浸式翻译 / Claude Code / NextChat / Chatbox / VS Code
```

经过网关这一层包装，对外吐出的全部是标准的 OpenAI 格式，本地调用延迟一般只有 30~50ms，性能损耗近乎为零。

---

## 二、准备工作：闲鱼 5 元买号与避坑指南

### 1. 账号挑选诀窍

在闲鱼搜索 `gemini pro`（例如：[闲鱼 Gemini 搜索入口](https://www.goofish.com/search?q=gemini)）：

- **价格区间**：3 元 ~ 5 元/号。
- **挑选原则**：优先买**带独立邮箱的一手新号**或教育优惠直接激活号，不要买多人共享的号。
- **无痕验号**：商家发货后，先在电脑浏览器的**无痕窗口（Incognito）**登录并尝试发起一条对话，确认能正常使用。

### 2. 获取凭据与接入点

- **方案 A（反重力节点）**：商家直接提供挂载反重力中转的凭证（Token 或 BaseURL）。
- **方案 B（提取 Cookie）**：登录网页版后按 `F12` 打开开发者工具，切到「网络（Network）」，随便发一句话，找到带 `assistant` 的请求，在 Request Headers 中复制 `__Secure-1PSID` 的 Cookie 值备用。

> 🔒 **安全提醒**：Cookie 与 Token 等同于账号最高控制权，请勿分享或上传到公开仓库。

---

## 三、飞牛 OS 部署步骤一：目录规划与提前赋权

**⚠️ 这一步不做对，后续数据库挂载必定报错闪退！**

### 1. 新建存储目录

打开飞牛 OS 的「文件管理器」，进入你的主力存储池（如 `/vol1/`），在 `docker` 目录下新建 `sub2api` 文件夹，并在其内部建立 3 个子目录：

```
/vol1/docker/sub2api/
├── pgdata/     # PostgreSQL 16 数据库持久化目录
├── redis/      # Redis 7 缓存与流控数据目录
└── config/     # Sub2API 核心配置文件目录
```

### 2. 终端赋权（解决权限隔离导致闪退）

在电脑上打开终端（PowerShell 或 CMD），通过 SSH 登录你的飞牛 OS：

```bash
ssh jack@10.0.0.56
```

登录后执行全量赋权命令：

```bash
cd /vol1/docker
sudo chmod -R 777 ./sub2api
```

> **为什么要赋 777 权限？**
> 因为 Postgres 容器内部默认是以 `postgres` 系统用户（UID 999 左右）启动的。如果宿主机的 `pgdata` 目录是普通 NAS 用户权限，Postgres 初始化写入时会直接报 `Permission denied: could not create directory`，并导致容器陷入死循环重启。

---

## 四、飞牛 OS 部署步骤二：Docker Compose 容器编排

打开飞牛 OS 网页端，点击进入**「Docker 应用中心」** → 点击右上角**「新建项目 / 编排」**：

- **项目名称**：填入 `sub2api`
- **项目路径**：选择刚才创建的 `/vol1/docker/sub2api`
- **编排代码**：直接完整粘贴以下配置

```yaml
version: '3.8'

services:
  sub2api:
    image: ghcr.io/sub2api/sub2api:v1.2.0   # 固定版本 Tag，避免 latest 导致架构不匹配
    container_name: sub2api
    restart: unless-stopped
    ports:
      - "8080:8080"          # 飞牛 NAS 局域网对外唯一暴露的访问端口
    environment:
      - DATABASE_URL=postgres://sub2api:Sub2Api_StrongPassword_2026@postgres:5432/sub2api
      - REDIS_URL=redis://redis:6379
      - TZ=Asia/Shanghai
    depends_on:
      - postgres
      - redis
    networks:
      - sub2api-net

  postgres:
    image: postgres:16-alpine
    container_name: sub2api-db
    restart: unless-stopped
    environment:
      - POSTGRES_USER=sub2api
      - POSTGRES_PASSWORD=Sub2Api_StrongPassword_2026  # ⚠️ 必须与上方 DATABASE_URL 密码完全一致
      - POSTGRES_DB=sub2api
      - TZ=Asia/Shanghai
    volumes:
      - ./pgdata:/var/lib/postgresql/data
    networks:
      - sub2api-net

  redis:
    image: redis:7-alpine
    container_name: sub2api-redis
    restart: unless-stopped
    volumes:
      - ./redis:/data
    networks:
      - sub2api-net

networks:
  sub2api-net:
    driver: bridge
```

### 编写要点提醒

1. **密码一致性**：`POSTGRES_PASSWORD` 和 `DATABASE_URL` 里的密码务必完全一致。如果初次填写错误，光修改 compose 无法覆盖数据卷，必须运行 `rm -rf ./pgdata/*` 清空后再拉起。
2. **网络隔离安全**：数据库与 Redis 不要映射宿主机端口，全部封装在 `sub2api-net` 桥接网络中内部互联，极大保障安全。

粘贴完成后，点击**「构建并启动」**。飞牛会自动拉取镜像并启动 3 个容器，状态全部显示为绿色的**「运行中」**。

---

## 五、Sub2API 管理后台初始设置

容器就绪后，打开浏览器，访问你的飞牛 NAS IP 地址加 8080 端口：

```
http://10.0.0.56:8080/admin/dashboard
```

![Sub2API 真实仪表盘](/images/sub2api_dashboard_real.png)

### 步骤 1：管理员初次登录与改密

- 初次安装后，Sub2API 默认管理员账号为 `admin`，初始密码可在飞牛 Docker 应用中心点击 `sub2api` 容器日志查看（搜索关键字 `Admin initialized`）。
- 登录后第一件事：点击右上角管理员头像 → **「个人中心 / 账号设置」**，立即修改为自己的强密码。

### 步骤 2：分配账户无限额度（关键）

Sub2API 默认启用沙盒计量，初始用户额度是 0，不改的话后面调用会秒报 `403 INSUFFICIENT_BALANCE`：

1. 点击左侧导航栏的 **「用户管理」**；
2. 找到 `admin` 账号（或者你新建的使用者账号），点击 **「编辑」**；
3. 将账户余额修改为一个大数（比如输入 `9999990.00`），点击保存。此时右上角会显示绿色的丰厚余额标签。

---

## 六、进阶实操：挂载 Antigravity 反重力节点与 Jack 分组配置

这是整套系统最精妙的一步——把闲鱼买来的廉价号真正注入网关：

![反重力分组管理](/images/sub2api_antigravity_group.png)

### 步骤 1：创建专属分组（以 Jack 分组为例）

1. 在左侧菜单点击 **「分组管理」**；
2. 点击右上角青绿色的 **「+ 创建分组」** 按钮；
3. **配置参数表单**：
   - **分组名称**：填入 `Jack`（可自定义，用于客户端路由）
   - **平台选择**：选择带有紫色云朵图标的 **`Antigravity`**（反重力）平台
   - **计费类型**：选择 **`标准 (余额)`**
   - **费率倍数**：保持默认的 **`1x`**
   - **访问类型**：选择 **`公开`**
4. 点击保存，表格中就会出现一条高亮的分组记录。

### 步骤 2：在渠道管理中添加上游账号

1. 点击左侧菜单 **「渠道管理」** → **「新建渠道」**；
2. **渠道名称**：例如 `Gemini-Pro-Goofish-01`；
3. **上游类型**：选择 `Antigravity` / `Gemini Web`；
4. **填入凭据**：把从闲鱼商家处获取的授权凭据、或 F12 抓取的 Cookie 粘贴到对应框内；
5. **绑定分组**：将该渠道勾选绑定到刚刚创建的 **`Jack` 分组** 中；
6. **模型重定向（推荐）**：

   ```text
   gemini-3.1-pro-high -> gpt-4o
   ```

   *作用：让只认 GPT-4 的老客户端直接选 gpt-4o，实际后端无缝调用 Gemini 3.1 Pro。*

7. 点击测试并保存，显示「渠道正常」即可。

### 步骤 2.5：当前可用的模型清单

在「分组管理」或「渠道管理」的模型列表中，可以直接看到上游透传的真实模型名。老顾这套环境里挂载的最新主力模型如下：

| 模型名称 | 定位与特点 | 推荐场景 |
| :--- | :--- | :--- |
| **`gemini-3.1-pro-high`** | 谷歌旗舰推理模型，高算力档位，长上下文与复杂逻辑最强 | 深度代码重构、长文档分析、复杂推理 |
| **`gemini-3.8-flash-high`** | 高速高并发档位，响应极快、成本极低 | 沉浸式翻译、批量摘要、日常对话 |
| **`DeepSeek-V4-Flash`** | 国产旗舰高速版，中文理解与代码能力出色 | 中文写作、Agent 自动化、代码补全 |

> 💡 **实战建议**：日常翻译和高频轻量任务走 `gemini-3.8-flash-high`，复杂推理和代码任务切到 `gemini-3.1-pro-high`，中文场景优先 `DeepSeek-V4-Flash`，三条线路互为备份。

> **多账号高可用玩法**：如果你闲鱼上买了 2~3 个账号，全部添加进来并绑定在 `Jack` 分组下。Sub2API 会自动按权重轮询。某一个账号被限流时会自动无感切换到下一个账号，保证 7×24 小时永不断线！

### 步骤 3：签发调用令牌（API Key）

1. 点击左侧菜单 **「令牌管理 / API 密钥」**；
2. 点击 **「+ 添加新令牌」**；
3. **名称**：例如 `Jack-Universal-Key`；
4. **绑定分组**：选择 **`Jack`** 分组；
5. **过期时间**：设置为 **永不过期**；
6. **额度限制**：设置为 **不限额度**；
7. 点击创建，系统会生成一个以 **`sk-`** 开头的长字符串秘钥，点击复制并保存好。

---

## 七、多端客户端实战接入配置

拿着刚才生成的 `sk-` 令牌和飞牛网关地址，你所有的常用工具都可以零成本起飞了：

### 1. 沉浸式翻译（浏览器插件）

- **翻译服务**：选择 **OpenAI**
- **接口地址**：`http://10.0.0.56:8080/v1/chat/completions`（或者 `http://10.0.0.56:8080/v1`）
- **API 密钥**：粘贴你的 `sk-xxxxxxxx`
- **自定义模型**：填 `gemini-3.8-flash-high`（翻译这种高频任务用 Flash 档最快最省）
- **实测效果**：上万字英文长文 2 秒完成双语排版，完全不需要充钱。

### 2. Claude Code（命令行终极代码助手）

在你的 Mac/Windows 开发机环境变量中配置：

```bash
export ANTHROPIC_BASE_URL="http://10.0.0.56:8080"
export ANTHROPIC_API_KEY="sk-你的网关令牌"
export ANTHROPIC_MODEL="gemini-3.1-pro-high"
```

打开终端直接跑：

```bash
claude "分析当前工程的全部报错并生成修复补丁"
```

实测重构整个项目、跑完几十个测试用例，消耗几十万 Token，全由家里的飞牛小主机默默抗下，费用永远是 $0.00！

### 3. NextChat / Chatbox / 移动端客户端

- **API 协议**：选择 **OpenAI**
- **API 地址**：`http://10.0.0.56:8080`
- **API Key**：填入你的 `sk-` 令牌
- **模型**：直接填 `gemini-3.1-pro-high` 或 `DeepSeek-V4-Flash`

全家在局域网内任意手机、平板、笔电都能畅享。

---

## 八、总结与日常运维速查

| 常用操作 | 命令 / 入口 |
| :--- | :--- |
| **飞牛后台访问** | `http://<飞牛IP>:8080/admin/dashboard` |
| **容器状态查看** | 飞牛 OS「Docker 应用中心」或 SSH 执行 `docker compose ps` |
| **实时查看网关日志** | `docker logs -f sub2api` |
| **重置数据库数据卷** | 停止项目后执行 `sudo rm -rf /vol1/docker/sub2api/pgdata/*` |
| **更新网关版本** | 修改 compose 镜像 tag 后点击应用中心「重新构建」 |

只要花 5 块钱买个号，配上飞牛 OS 稳定的 Docker 底座，你就拥有了一套属于自己的私人算力中心。

赶紧动手折腾起来吧！

---

> 本文涉及的所有密码、Token、Cookie 均已做脱敏处理，请勿在任何公开场合泄露你的真实凭证。
