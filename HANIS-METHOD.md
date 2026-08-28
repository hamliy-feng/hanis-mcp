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

在 PHASE 1 中，任何外部 Agent 读完本文件后的第一项动作，是一次性向用户收集构建所需输入，再自动完成可自动完成的安装、检测、配置和验证。

---

# PHASE 1 — 让网页版 ChatGPT 连接本地 Hanis MCP

## 1.0 外部 Agent 读完本文后的首次回复（必须一次性询问）

不要把配置拆成多轮追问。第一次回复应一次性向用户收集以下 4 项：

### A. Workspace 开放范围

询问用户希望网页版 GPT 可以访问哪些本地范围。支持一个或多个：

```text
整个盘符      例如 D:\
指定目录      例如 D:\Projects
单个项目      例如 D:\Projects\my-app
多个目录/项目 可同时提供
```

用户提供的范围就是 Hanis 后续注册 Workspace 和计算 Scope 的依据。

### B. Tunnel ID

用户需要提供自己的 OpenAI Secure MCP Tunnel ID：

```text
Tunnel ID: <TUNNEL_ID>
```

### C. Runtime API Key

用户需要提供 tunnel-client 使用的 Runtime API Key。

推荐两种输入方式：

```text
方式 1：用户已注入 CONTROL_PLANE_API_KEY 环境变量
方式 2：用户临时提供 Runtime API Key，由 Agent 写入安全运行环境 / secret store
```

Agent 只确认凭据“已收到 / 已注入”，不在回复、日志、README、AGENTS.md、MEMORY.md 中回显明文。

### D. 权限组

权限组按需要组合。首次默认启用：

```text
Reader + Editor
```

用户可以选择：

```text
Reader
Editor
Developer
Operator
Owner
```

每个权限组只按“开放了什么”向用户说明：

```text
Reader
- 读取 Workspace / 文件 / 文档 / 代码
- 搜索内容
- 查看 Git 状态与 diff
- 读取 AGENTS / MEMORY 并输出分析、Plan、TODO

Editor
- 创建 / 修改 / patch / 移动 Workspace 内文件
- 创建写入前快照并恢复
- 更新项目 MEMORY

Developer
- 运行隔离 Shell / 项目脚本
- build / test / lint
- 容器内安装依赖与使用网络
- Git commit 等开发动作

Operator
- 运行 Native Shell
- 管理进程与 Docker
- Git push
- SSH / 远程执行 / 服务操作
- 使用受保护 Secret 执行外部任务

Owner
- 启用 Hanis 提供的完整 Capability 集
- 对删除、force push、系统管理、原始 Secret 等高风险能力进行显式授权 / 临时提权
```

`Reader + Editor` 是默认组合；用户说“默认”即可采用这两个权限组。

### 首次标准问询模板

外部 Agent 应直接向用户发送类似以下内容：

```text
我已经读完 HANIS-METHOD.md。请一次性提供以下信息，我收到后会继续自动检测和搭建：

1. Workspace 开放范围：
   （盘符 / 目录 / 项目路径，可提供多个）

2. Tunnel ID：

3. Runtime API Key：
   （可直接临时提供，或回复“已注入 CONTROL_PLANE_API_KEY”）

4. 权限组：
   Reader / Editor / Developer / Operator / Owner
   默认：Reader + Editor
```

### 用户回复后，Agent 必须先做一次正向确认

开始修改本机配置前，先反馈实际将启用的内容，例如：

```text
Hanis 构建配置已确认

Workspace：2 个
- D:\Projects\project-a
- E:\research

权限组：Reader + Editor
开放 Capability：<根据当前 policy 计算数量>
- <只列实际开放的 capability>

Tunnel ID：已收到
Runtime API Key：已收到（不回显）

开始进行本地 MCP、网络、Tunnel 与 ChatGPT 连接检测。
```

用户侧状态报告只列：

```text
开放了哪些范围
开放了哪些能力
开放能力数量
当前连接到哪一步
```

内部 Policy Engine 负责完整的风险判断；正常构建反馈始终采用正向清单，只展示已开放的范围、权限组、Capability 数量和 Capability 名称。

---

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

当前标准 Transport 为：

```text
Secure MCP Tunnel + local tunnel-client + sessionful /mcp
```

Hanis MCP 保持在本地地址，由 tunnel-client 通过出站 HTTPS 与 OpenAI 建立连接。

---

## 1.2 收到用户输入后的自动检测

收到 Workspace、Tunnel ID、Runtime API Key 和权限组以后，Agent 开始自动检测环境。

网络代理不属于首次必填项；先检测当前网络是否可以直接完成连接。

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

如果本地 MCP Server 尚未运行，Agent 进入 Hanis 本地安装 / 启动步骤，并在本地 health、ready 与 MCP initialize 成功后继续 Tunnel 配置。

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

网页版 ChatGPT 与本地 Hanis 的标准连接方式是 OpenAI Secure MCP Tunnel。

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

这条链路使用本机主动发起的出站 HTTPS；Hanis MCP 继续运行在本地地址。

## 2.1 用户输入与 Agent 自动准备

用户已在 `1.0` 一次性提供：

```text
Tunnel ID
Runtime API Key
```

Agent 自动负责：

```text
检查 / 安装 tunnel-client
检查本地 Hanis /mcp
检查网络
初始化 tunnel profile
运行 doctor
启动 tunnel
验证工具发现
```

Runtime API Key 仅进入运行环境或安全 secret store，不进入 Git、README、AGENTS.md、MEMORY.md、policy.json 或普通日志。

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

# 3. 网络连接与代理处理

`tunnel-client` 需要通过 HTTPS 连接 OpenAI control plane。

网络处理固定为三层，不在首次问询中要求用户填写代理：

## 3.1 第一层：直接连接

Agent 首先测试当前网络：

```text
DNS resolution
TCP 443
TLS / HTTPS
api.openai.com control plane
```

如果直连成功，直接继续 `tunnel-client doctor` 和 Tunnel 启动。

## 3.2 第二层：自动使用本机代理

如果直连失败，Agent 自动检测机器上已有的代理能力，包括：

```text
HTTP_PROXY / HTTPS_PROXY / ALL_PROXY
Windows 系统代理
常见本地 HTTP / HTTPS / SOCKS 监听
正在运行的代理进程及其实际监听端口
```

检测到候选代理后，应逐个验证：

```text
代理端口可连接
通过代理可完成 DNS / TLS / HTTPS
通过代理可访问 OpenAI control plane
```

选取通过验证的代理配置 tunnel-client，然后重新运行：

```powershell
tunnel-client doctor --profile hanis --explain
```

## 3.3 第三层：按 Hanis 方法逐层排查

如果自动代理仍无法走通，按以下顺序定位：

```text
① DNS
   ↓
② 本机 TCP 443 / TLS
   ↓
③ 代理进程是否运行
   ↓
④ 代理实际监听地址 / 端口
   ↓
⑤ 代理到 api.openai.com 的 HTTPS
   ↓
⑥ tunnel-client 环境变量 / profile
   ↓
⑦ OpenAI control plane
   ↓
⑧ local Hanis /mcp
   ↓
⑨ tunnel-client doctor
```

只有自动检测无法确定有效路径，或者代理本身需要用户账号 / 凭据时，再向用户询问对应网络信息。

原则：

> **先直连；直连不通就自动找可用代理；仍不通再按层排查。代理不是首次安装表单的一部分。**

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

Workspace 规则：

- 用户在首次问询中显式提供开放范围；
- 开放范围可以是整个盘符、目录、单个项目或多个路径；
- 每个 workspace 具有独立 root、access、runner、policy；
- 路径访问以用户提供的 workspace root 为 Scope；
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

权限组：

```text
Reader
Editor
Developer
Operator
Owner
```

首次默认组合：

```text
Reader + Editor
```

权限组的用户侧说明只列该组实际开放的 Capability；Hanis 在确认配置时同时给出开放 Capability 数量。

高风险权限支持临时 elevation：

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
1. 一次性询问用户：Workspace 范围 + Tunnel ID + Runtime API Key + 权限组
2. 正向反馈：将开放的 Workspace / 权限组 / Capability 数量与清单
3. 检查 Hanis 是否已经安装/运行
4. 检查 localhost MCP health / initialize / tools
5. 网络先直连；失败后自动代理；仍失败则逐层排查
6. 初始化并检查 Secure MCP Tunnel
7. 确认 ChatGPT Web 能看到并调用 Hanis
8. 注册 Workspace Registry
9. 读取已有 AGENTS / MEMORY / Agent 风格
10. 读取并应用 policy
11. 创建或恢复 Task
12. 执行实际项目工作
13. 验证
14. 更新 MEMORY
15. 完成 / handoff
```

首次用户问询必须合并成一轮；网络代理由 Agent 在后续自动检测流程中处理。

---

# 12. 不变量

任何未来版本都尽量保持以下不变量：

1. **本机优先**：MCP Server 使用本地地址，由 Secure MCP Tunnel 连接网页版 GPT。
2. **Tunnel 与 Runtime 解耦**：Secure MCP Tunnel 只是 Transport。
3. **一台 Runtime，多 Workspace**。
4. **Workspace 范围由用户首次输入显式确定**，支持盘符、目录、项目和多路径。
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
