# Hanis MCP

Hanis MCP 是一套让**网页版 ChatGPT 连接并操作本地电脑工作区**的方法与运行结构。

它通过 MCP 将 ChatGPT 与本地文件、Git、Shell、Docker、任务状态和项目记忆连接起来，同时把 Workspace、权限和高风险操作限制在本机 Hanis Runtime 中执行。

当前生产实例已经验证这条链路可以工作；本仓库目前主要整理并公开这套方法、约束与开源化设计。

---

## 1. Hanis 做了什么

Hanis 在网页版 ChatGPT 和本地电脑之间增加了一层本地 MCP Bridge。

标准链路为：

```text
ChatGPT Web
    ↓
OpenAI Secure MCP Tunnel
    ↓
local tunnel-client
    ↓
Hanis MCP Server
    ↓
Workspace / Files / Git / Runner / Memory
    ↓
本地项目
```

这套结构解决了四个问题：

- 网页版 ChatGPT 原本不能直接访问用户电脑上的 `localhost`、文件和开发环境；Hanis 通过 MCP 与 Secure MCP Tunnel 建立连接。
- 本地项目不需要为了让 ChatGPT 访问而直接暴露公网端口。
- 一台 Hanis 可以登记多个 Workspace，不需要每个项目单独部署一套 MCP 服务。
- Agent 规则、项目记忆和运行权限保存在本地，使不同 AI 可以基于同一套项目上下文继续工作。

Hanis 本身不是新的 AI 模型。它负责的是：**连接、工作区、工具、上下文和权限执行。**

---

## 2. 网页版 GPT 连接后，可以在本地做什么

连接成功后，ChatGPT 可以通过 Hanis 获得用户明确开放的本地能力。

目前生产结构已经覆盖以下类型的操作：

### 文件与项目

- 查看目录和文件结构
- 读取文件
- 搜索代码或文本
- 创建和修改文件
- 对指定文件应用 patch
- 在写入前创建快照并在需要时恢复

### Git

- 查看 Git 状态
- 查看 diff
- 配合权限策略扩展到 commit、push 等操作

### 本地执行

- 运行 Shell 命令
- 运行项目脚本
- 执行 build、test、lint
- 使用 Docker 隔离运行任务
- 在用户显式授权后执行 Native Runner 或更高权限操作

### 项目连续性

Hanis 不要求每次换一个 ChatGPT 对话都重新解释项目。

项目可以保存：

```text
AGENTS.md
.hanis/MEMORY.md
.hanis/state.json
.hanis/policy.json
```

其中：

- `AGENTS.md`：长期规则、架构和工作方式
- `MEMORY.md`：当前进度、决策、阻塞和下一步
- `state.json`：Hanis 的机器状态
- `policy.json`：本地权限策略

因此同一个本地项目可以被 ChatGPT、Codex、其他 MCP Agent 或本地 AI 接手，而不用只依赖上一段聊天记录。

---

## 3. 整个操作流程是什么

完整流程分为五个阶段。

### Phase 1 — 让网页版 ChatGPT 连接本机

```text
启动 Hanis MCP Server
        ↓
确认 localhost /mcp 可访问
        ↓
配置 OpenAI Secure MCP Tunnel
        ↓
启动 local tunnel-client
        ↓
在 ChatGPT Web 创建 MCP App
        ↓
扫描 Hanis Tools
        ↓
实际调用本地工具完成验收
```

默认连接目标类似：

```text
http://127.0.0.1:<port>/mcp
```

Hanis 默认只监听本机回环地址，由 Secure MCP Tunnel 主动向 OpenAI 建立出站连接。

### Phase 2 — 登记 Workspace

用户选择允许 AI 访问的本地目录，例如：

```text
D:\Projects\project-a
D:\Projects\project-b
```

Hanis 将这些目录注册为独立 Workspace，并为每个 Workspace 设置访问范围和 Runner。

开源默认不建议直接开放整个磁盘。

### Phase 3 — 读取 Agent 与项目记忆

进入已有项目时，Hanis 检测：

```text
AGENTS.md
CLAUDE.md
.cursor/
.agent_memory/
.agent_index/
.codex-memory/
其他已有 AI 规则
```

用户可以选择：

```text
继承转换
Hanis + 参考现有风格
只读参考
忽略
```

随后将项目上下文收敛到统一的 Agent / Memory 结构。

### Phase 4 — 设置权限

权限不是由 `AGENTS.md` 决定，而是由 Hanis 服务端 Policy Engine 强制执行。

权限按以下四个维度判断：

```text
Capability × Scope × Risk × Approval
```

例如：

```text
filesystem.read     → 自动允许
filesystem.write    → 当前 Workspace 内允许
shell.container     → 允许或询问
shell.native        → 高风险，询问
git.push            → 外部操作，询问
system.admin        → 默认禁止或临时提权
```

高风险操作可以使用：

```text
允许一次
允许本次会话
允许当前任务
允许当前 Workspace
长期允许
```

### Phase 5 — 正常工作与续接

连接、Workspace、上下文和权限建立后，ChatGPT 就可以像本地 Agent 一样工作：

```text
读取项目
    ↓
理解 AGENTS / MEMORY
    ↓
修改代码或文件
    ↓
运行测试 / build
    ↓
检查结果
    ↓
更新 MEMORY
    ↓
下一次对话继续
```

---

## 4. 现在已经实现了什么

Hanis 当前已经完成并实际验证的核心能力包括：

- **网页版 ChatGPT → 本地 MCP 的连接链路已经跑通。** 当前生产主线使用 Secure MCP Tunnel + sessionful MCP `/mcp`。
- **ChatGPT 可以调用本机 Hanis 工具。** 已实际用于读取文件、修改项目、运行命令、检查 Git、管理任务和访问多个 Workspace。
- **一台 Bridge 可以管理多个本地 Workspace。** 不需要为每个项目重新建立一条连接。
- **已经建立文件化项目记忆规则。** 新对话可以读取项目本地 Memory 后继续以前的工作。
- **已经形成 Agent / Memory 迁移方案。** 可以兼容已有 Codex、Claude、Cursor 等项目规则，并计划统一到 Hanis 的 canonical context。
- **已经形成分级权限模型。** 普通用户可以只开放读取/编辑，高级用户可以按风险显式开放 Shell、Docker、SSH、Git Push、系统操作等能力。
- **旧的公网暴露方案已经从默认结构中移除。** Tailscale Funnel、公开 HTTPS、stateless MCP 和自建 OAuth 被保留为 legacy / optional，而不是默认依赖。

当前仓库首先公开的是这套已经验证过的方法和设计约束。下一阶段是在这些约束下抽出干净的 `hanis-mcp` 最小实现和安装脚本，而不是直接复制带有个人机器配置的生产代码。

---

## 从哪里开始

如果是 AI / Agent 接手本项目，先阅读：

**[`HANIS-METHOD.md`](./HANIS-METHOD.md)**

它记录了从网页版 ChatGPT 连接本机开始的完整执行顺序。

进一步设计文档：

- [`docs/current-production-path.md`](./docs/current-production-path.md) — 当前实际链路
- [`docs/agent-memory-protocol.md`](./docs/agent-memory-protocol.md) — Agent / Memory 规则
- [`docs/permission-model.md`](./docs/permission-model.md) — 权限与风险模型
- [`docs/runtime-contract.md`](./docs/runtime-contract.md) — Hanis Runtime Contract
- [`docs/minimal-open-source-architecture.md`](./docs/minimal-open-source-architecture.md) — 最小开源结构
