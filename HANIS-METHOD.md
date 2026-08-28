# Hanis Method — AI 操作入口

> **用途**：这是 Hanis 方法的主入口文档，优先给 AI / Agent 阅读。
> 
> 任何接手本项目的 AI，应先读本文件，再按其中阶段顺序执行。不要从历史日志、旧 Tailscale 配置或聊天记录反推当前流程。

---

# 0. Hanis 方法要解决什么

Hanis 的目标不是“再造一个 AI”。

Hanis 是一个本地 MCP Bridge，使网页版 ChatGPT 或其他支持 MCP 的 AI 能够：

1. 安全连接用户自己的电脑；
2. 只访问用户显式开放的 workspace；
3. 读取、修改、执行、测试本地项目；
4. 通过 `AGENTS.md + MEMORY.md` 继续以前 AI 的工作；
5. 在服务端权限边界内执行，而不是仅依赖模型自律；
6. 未来可以低成本切换 GPT、Codex、Claude、本地 AI 等执行端，而不更换项目工作方式。

核心结构：

```text
AI / ChatGPT / Codex / Local AI
              ↓
        Transport Plane
              ↓
         Hanis Runtime
              ↓
          Policy Engine
              ↓
 Workspace / Files / Git / Runner
              ↓
       Project Context
 AGENTS.md + MEMORY.md + Task
```

---

# 1. 总流程

严格按以下阶段推进：

```text
PHASE 1  网页版 ChatGPT 能连接本机
   ↓
PHASE 2  注册并限制 Workspace
   ↓
PHASE 3  建立 AGENTS / MEMORY 上下文协议
   ↓
PHASE 4  配置权限 Profile / Risk / Approval
   ↓
PHASE 5  验证读、写、执行、恢复、续接
```

**当前第一要务是 PHASE 1。**

在 PHASE 1 没有完成以前，不要先做复杂的 Agent 迁移、权限 UI、Journal 重构或多模型适配。

---

# PHASE 1 — 让网页版 ChatGPT 连接本地 Hanis MCP

## 1.1 目标状态

最终链路必须收敛为：

```text
ChatGPT Web
     ↓
ChatGPT Custom MCP App
     ↓
OpenAI Secure MCP Tunnel
     ↓
local tunnel-client
     ↓
http://127.0.0.1:<HANIS_PORT>/mcp
     ↓
Hanis MCP Server
     ↓
MCP Tools
```

默认不要使用：

- Tailscale Funnel；
- 公网端口映射；
- 自建反向代理；
- `/mcp-stateless`；
- 为了连接 ChatGPT 而额外暴露本机公网 HTTPS。

这些只属于 legacy / optional transport。

---

## 1.2 AI 开始工作前必须先检测

AI 不得假定 Hanis 已安装、Tunnel 已运行或 ChatGPT 已连接。

依次确认：

### A. 本地 MCP Server 是否存在

检查：

```text
http://127.0.0.1:<port>/healthz
http://127.0.0.1:<port>/readyz
http://127.0.0.1:<port>/mcp
```

当前生产实例默认参考：

```text
http://127.0.0.1:7677/mcp
```

但开源版不得把 `7677` 当成不可修改的硬编码。

成功条件：

```text
/healthz -> HTTP 200
/readyz  -> HTTP 200
MCP initialize -> success
```

如果本地 MCP Server 不存在：

```text
STOP PHASE 1B
先安装或启动 Hanis MCP Server
```

不能用 Tailscale、公网代理或 Tunnel 去掩盖“本地服务根本没启动”的问题。

---

## 1.3 Hanis 本地服务的安全默认值

本地 MCP Server 应满足：

```text
host = 127.0.0.1
transport = MCP Streamable HTTP
path = /mcp
serverInfo.name = ASCII
```

原则：

- 默认只监听 loopback；
- 不要求入站公网访问；
- Secure MCP Tunnel 运行在能访问这个本地地址的同一台电脑或同一私有网络中；
- Workspace 权限由 Hanis 自己控制，不由 Tunnel 决定。

---

# 2. 创建 OpenAI Secure MCP Tunnel

网页版 ChatGPT **不能直接访问 localhost MCP**。

标准做法：使用 OpenAI Secure MCP Tunnel。

工作机制：

```text
本机 tunnel-client
       │
       │ outbound HTTPS
       ▼
api.openai.com:443
       │
       ▼
OpenAI-hosted tunnel endpoint
       │
       ▼
ChatGPT
```

没有公网入站端口。

## 2.1 前置条件

需要：

1. 一个 OpenAI `tunnel_id`；
2. 一个只用于运行 tunnel-client 的 runtime API key；
3. 本机可运行的 `tunnel-client`；
4. `tunnel-client` 能访问本地 Hanis `/mcp`；
5. 本机能出站访问 `api.openai.com:443`；
6. ChatGPT 账号 / workspace 具有相应 developer-mode / MCP app 权限。

**AI 不得把 API key 写入 Git、README、AGENTS.md、MEMORY.md、policy.json 或日志。**

---

## 2.2 获取 tunnel-client

优先使用：

```text
OpenAI Platform -> Tunnel settings -> Download
```

或 OpenAI 官方 `tunnel-client` 最新 release。

不要在项目脚本中永久写死某一个旧版本下载 URL。

---

## 2.3 配置 Windows 环境变量

PowerShell 当前会话示例：

```powershell
$env:CONTROL_PLANE_API_KEY="<RUNTIME_API_KEY>"
```

只把 secret 放进运行环境或安全 secret store。

不要写：

```text
.env committed to Git
AGENTS.md
MEMORY.md
workspaces.json
README.md
```

---

## 2.4 初始化 Hanis Tunnel Profile

对于 Hanis HTTP MCP：

```powershell
tunnel-client init `
  --profile hanis `
  --tunnel-id <TUNNEL_ID> `
  --mcp-server-url http://127.0.0.1:7677/mcp
```

如果 Hanis 使用其他端口，则替换 URL。

这里不要使用 `--mcp-command`，除非未来 Hanis 改成由 tunnel-client 直接启动的 stdio MCP。

---

## 2.5 必须先运行 Doctor

```powershell
tunnel-client doctor --profile hanis --explain
```

在进入 ChatGPT 配置前，应先确认：

```text
Tunnel identity      OK
Control plane        OK
Outbound HTTPS       OK
Local MCP reachable  OK
MCP handshake        OK
```

如果这里失败，不要去 ChatGPT UI 反复“创建连接器”。

应先诊断本地链路。

---

# 3. 网络代理处理

`tunnel-client` 默认需要访问：

```text
api.openai.com:443
```

如果用户所在网络无法直接访问 OpenAI，而机器已有 HTTP/HTTPS/SOCKS 代理：

1. 先检测真实代理监听端口；
2. 再把 tunnel-client 的出站流量指向该代理；
3. 不要假定常见端口一定存在；
4. 验证代理能真正访问 `api.openai.com:443`。

常见故障示例：

```text
错误：直接照抄一个“常见代理端口”
正确：先检测本机真实监听端口，再配置 tunnel-client
```

因此其他 AI 必须遵守：

> **检测实际运行状态，不复制某台机器的历史代理端口。**

---

# 4. 启动 Secure MCP Tunnel

Doctor 通过后：

```powershell
tunnel-client run --profile hanis
```

运行期间必须保持 tunnel-client 存活。

ChatGPT 的 app discovery、`tools/list` 和实际 MCP tool call 都依赖这个进程。

可使用 tunnel-client 提供的本地健康界面 / health endpoint 验证其：

```text
healthy
ready
connected
polling
```

如果 Tunnel 停止，ChatGPT 不可能继续访问本机。

---

# 5. 在网页版 ChatGPT 创建 Hanis App

## 5.1 ChatGPT 端

确保当前账号具有 developer mode / custom MCP app 权限。

然后在 ChatGPT Web 中：

```text
Apps / Plugins
   ↓
Create developer-mode app
   ↓
Connection = Tunnel
   ↓
选择对应 tunnel
或填写 tunnel_id
   ↓
Scan Tools
   ↓
Create
```

如果 Tunnel 不出现在 ChatGPT：

优先检查：

```text
Tunnel 是否关联目标 ChatGPT workspace
当前操作用户是否具有 Tunnels Read + Use
ChatGPT developer mode 是否开启
Tunnel client 是否仍然在线
```

不要先修改 Hanis MCP Server。

---

# 6. PHASE 1 验收标准

只有以下全部完成，才算“网页版 GPT 已连接本地”。

## Transport

```text
[ ] tunnel-client healthy
[ ] tunnel-client ready
[ ] tunnel-client connected
[ ] outbound OpenAI control plane reachable
```

## Hanis MCP

```text
[ ] /healthz = 200
[ ] /readyz = 200
[ ] initialize success
[ ] tools/list success
```

## ChatGPT

```text
[ ] Hanis app 可在 ChatGPT 中被选择
[ ] Scan Tools 成功
[ ] ChatGPT 能调用至少一个只读工具
[ ] ChatGPT 能获得真实本机 workspace 结果
```

建议第一条实际测试：

```text
使用 Hanis 列出当前可访问的 workspaces。
```

如果能返回本机真实 workspace 列表，则 PHASE 1 基础链路成立。

---

# 7. PHASE 1 故障定位顺序

必须按由近到远排查：

```text
① Hanis process
   ↓
② localhost /healthz
   ↓
③ localhost /mcp handshake
   ↓
④ tunnel-client doctor
   ↓
⑤ outbound api.openai.com:443
   ↓
⑥ tunnel workspace association / permissions
   ↓
⑦ ChatGPT app Scan Tools
   ↓
⑧ actual tool call
```

禁止一看到 ChatGPT 报错就同时改：

```text
Hanis
Tunnel
代理
OAuth
Workspace
工具定义
```

一次只确认一层。

---

# 8. PHASE 1 完成后才进入 Workspace

下一阶段目标：

```text
One Hanis Runtime
       ↓
Workspace Registry
   ├── project-a
   ├── project-b
   └── project-c
```

默认原则：

- 用户显式选择 workspace；
- 不默认开放 `D:\`、`C:\`、`/` 等整个磁盘；
- 每个 workspace 具有独立 root、access、runner、policy；
- 路径不可逃逸 workspace root；
- 一台 Hanis 可以管理多个项目，不需要每个项目重新安装一个 MCP Server。

详细设计见：

```text
docs/runtime-contract.md
docs/minimal-open-source-architecture.md
```

---

# 9. PHASE 3 — AGENTS / MEMORY

连接和 workspace 成立后，再处理 AI 连续性。

Hanis canonical context：

```text
project/
├── AGENTS.md
└── .hanis/
    ├── MEMORY.md
    ├── state.json
    └── policy.json
```

职责：

```text
AGENTS.md          长期规则 / 架构 / 风格 / 验证方式
.hanis/MEMORY.md   当前 durable facts / progress / blocker / next action
state.json         Hanis 机器状态 / adapter 状态
policy.json        权限策略
```

已有项目第一次接入时，应扫描：

```text
AGENTS.md
CLAUDE.md
GEMINI.md
.cursor/
.cursorrules
.agent_memory/
.agent_index/
.codex-memory/
HANDOFF.md
```

然后只询问一次用户：

```text
1. 继承并转换
2. Hanis + 参考现有 Agent 风格（推荐）
3. 只读参考
4. 忽略旧规则
```

本地 AI 可以作为 Optional Context Adapter 来总结、迁移和转换这些规则，但不能成为权限系统。

详细见：

```text
docs/agent-memory-protocol.md
```

---

# 10. PHASE 4 — 权限

**文本 Agent 规则 ≠ 系统权限。**

即使 `AGENTS.md` 写着“允许删除全部文件”，如果 Hanis Policy Engine 不允许，则工具调用必须被拒绝。

权限模型：

```text
Capability × Scope × Risk × Approval
```

例：

```text
filesystem.read
filesystem.write
filesystem.delete

git.read
git.commit
git.push
git.force

shell.container
shell.native

network.outbound
package.install

docker.read
docker.manage

ssh.connect
remote.execute

secrets.use
secrets.read

system.service
system.admin
```

默认提供 Profile：

```text
Reader
Editor
Developer  ← 默认推荐
Operator
Owner
```

高风险权限应支持临时 elevation：

```text
allow_once
allow_session
allow_task
allow_workspace
always
```

详细见：

```text
docs/permission-model.md
```

---

# 11. AI 接手 Hanis 项目时的固定执行顺序

任何 AI 读取本文件后，应按这个顺序：

```text
1. 确认用户目标和实际项目路径
2. 检查 Hanis 是否已经安装/运行
3. 检查 localhost MCP health
4. 检查 Secure MCP Tunnel
5. 确认 ChatGPT Web 能看到并调用 Hanis
6. 确认 Workspace Registry
7. 读取 AGENTS.md
8. 读取 MEMORY.md
9. 读取 policy / 当前权限
10. 创建或恢复 Task
11. 执行实际项目工作
12. 验证
13. 更新 MEMORY
14. 完成 / handoff
```

如果第 3–5 步尚未完成，而用户当前目标是“建立网页版 GPT 与本机连接”，不要跳到第 7–14 步。

---

# 12. 不变量

任何未来版本都尽量保持以下不变量：

1. **本机优先**：MCP Server 默认不公开暴露。
2. **Tunnel 与 Runtime 解耦**：Secure MCP Tunnel 只是 Transport。
3. **一台 Runtime，多 Workspace**。
4. **Workspace 显式授权**，不默认全盘。
5. **Agent Context 与 Permission 分离**。
6. **权限服务端强制执行**，不能依赖模型自律。
7. **Memory 文件化**，不能只依赖某一个聊天窗口。
8. **AI 可替换**：GPT、Codex、本地 AI 等可读取同一套项目事实。
9. **Secret 不进入模型可读记忆**；优先 `secrets.use` 而不是 `secrets.read`。
10. **先建立最小稳定链路，再加高级能力**。

---

# 13. 当前实施优先级

```text
P0  Web ChatGPT ↔ Local Hanis MCP 稳定连接
P1  Workspace Registry + 路径边界
P2  AGENTS / MEMORY canonical protocol
P3  Capability Policy Engine
P4  Snapshot / Runner / Git / Task Continuity
P5  Local AI Adapter / legacy adapters / alternate transports
```

如果开发资源有限，严格按这个顺序。

---

# 14. 官方依据与版本注意

本方法当前默认采用 OpenAI Secure MCP Tunnel，因为 OpenAI 官方文档明确支持使用 `tunnel-client` 将私有、本地或开发机上的 MCP Server 通过**仅出站 HTTPS**连接到受支持 OpenAI 产品，而无需暴露公网入站端口。

OpenAI 产品权限、Developer Mode UI、套餐和 MCP 写权限可能变化，因此实现时：

- 不要把 ChatGPT UI 路径写死进程序逻辑；
- 不要假设每个套餐都有完整 write/modify MCP 权限；
- Tunnel CLI 以官方当前版本为准；
- `tunnel-client help quickstart` 与 `tunnel-client doctor` 应作为实际环境的最终依据。

---

# 15. 本项目其他权威文档

```text
HANIS-METHOD.md                         ← AI 第一入口，本文件
README.md                               ← 项目概览
docs/current-production-path.md        ← 当前生产链路
docs/minimal-open-source-architecture.md
docs/runtime-contract.md
docs/agent-memory-protocol.md
docs/permission-model.md
docs/release-checklist.md
.agent_memory/MEMORY.md                 ← 当前整理项目自己的阶段记忆
```

出现冲突时：

```text
用户最新明确要求
   > HANIS-METHOD.md 中的不变量 / 当前阶段
   > 专项设计文档
   > MEMORY.md 中记录的当前事实
   > 历史生产文档 / 日志
```

不要把历史实现自动当成未来开源标准。
