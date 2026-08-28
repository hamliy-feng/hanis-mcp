# Hanis Runtime Contract

## 目标

Hanis 的开源设计不绑定 GPT、Codex、Claude、本地模型或某一个 IDE。核心目标是：

> 同一个本地工作区、同一份规则、同一份记忆、同一套权限，在不同 AI 执行端之间无痛切换。

因此 Hanis 分成四个彼此独立的平面：

```text
Transport Plane     ChatGPT / MCP Tunnel / other MCP clients
Runtime Plane       Hanis MCP Server / Workspace / Runner
Context Plane       AGENTS.md / MEMORY.md / Journal
Policy Plane        Capabilities / Risk / Approval / Scope
```

任何 AI 都只是 Transport/Client 一侧的“执行者”。

---

## 1. 稳定核心

默认主链：

```text
AI Client
  ↓ MCP
Secure Transport
  ↓
Hanis MCP Server
  ↓
Policy Engine
  ↓
Workspace Registry
  ↓
Tools / Runner / Journal
```

Hanis 核心只负责五件事：

1. MCP 会话与工具暴露
2. Workspace 边界
3. 权限与风险判定
4. 执行与回滚/快照
5. 项目上下文与续接

Tunnel、Tailscale、具体模型、具体 IDE 都不进入核心。

---

## 2. Context Plane：规则与记忆

建议使用以下可迁移结构：

```text
project/
├─ AGENTS.md
└─ .hanis/
   ├─ MEMORY.md
   └─ state.json        # 可选，机器状态/版本/哈希，不给人手写
```

### AGENTS.md

长期、稳定、人工可读的项目规则。

只写：

- 项目目标与边界
- 目录/架构约束
- 编码/设计风格
- 权威数据源
- 构建/测试入口
- 明确的 Do / Don't
- 记忆更新约定

不写：

- token / 密码 / 私钥
- 动态任务进度
- 临时日志
- 权限授予

### MEMORY.md

动态、可覆盖更新的“当前项目事实”。

只记录：

- Current objective
- Hard constraints
- Current architecture
- Durable decisions
- Completed work
- Active blockers
- Validation status
- Exact next actions

它不是聊天记录，也不是 append-only 日志。

### Journal

任务级历史继续放 SQLite/结构化日志：

- task id
- start/end
- tool/action
- result
- lease
- handoff

因此三者职责严格区分：

```text
AGENTS.md = 长期规则
MEMORY.md = 当前事实
Journal   = 历史记录
```

---

## 3. AI 切换协议

Hanis 不要求每个 AI 都原生支持同一种 agent 文件。

首次进入项目时执行 deterministic discovery：

```text
发现：
- AGENTS.md
- CLAUDE.md
- GEMINI.md
- .cursorrules / rules
- 其他已知 agent instruction
```

然后只问用户一次：

```text
发现已有本地 Agent 规则，如何使用？

A. 继承现有规则并转换为 Hanis 标准（推荐）
B. Hanis 标准 + 参考现有风格
C. 只读参考，不写入
D. 忽略，使用全新 Hanis 标准
```

这个选择持久化到 `.hanis/state.json`，不需要每次询问。

---

## 4. 本地 AI 的角色

本地 AI 可以使用，而且非常适合，但它应该是 Optional Context Adapter，而不是 Hanis 的强依赖。

### 本地 AI 可以做

- 汇总多个旧 agent 文件
- 把 vendor-specific 规则转换为 `AGENTS.md`
- 将超长 MEMORY 压缩成稳定事实
- 生成 session handoff
- 比较两个 Agent 风格的冲突
- 给用户解释“继承后会发生什么”

### 本地 AI 不应该做

- 决定权限
- 绕过 Policy Engine
- 自动读取 secret
- 独立判断高风险命令是否可执行
- 成为唯一的项目事实来源

也就是说：

```text
Filesystem scanner / parser = 确定性发现
Local AI                 = 语义整理
Hanis Policy Engine      = 最终权限裁决
```

这样可以做到“API 成本接近 0 的上下文迁移”，但不会因为本地小模型误判而破坏权限边界。

---

## 5. 兼容旧项目

开源版 canonical 路径建议逐步统一到：

```text
AGENTS.md
.hanis/MEMORY.md
```

同时提供兼容读取：

```text
.agent_memory/MEMORY.md
.agent_index/MEMORY.md
.codex-memory/
```

迁移时不直接覆盖旧文件；先生成对比，再由用户选择是否切换。

---

## 6. 每次会话的固定生命周期

### Session Start

1. resolve workspace
2. load nearest `AGENTS.md`
3. load `.hanis/MEMORY.md`
4. load active task/journal
5. evaluate permission profile
6. start work

### During Work

仅当 durable fact 变化时更新 MEMORY。

### Session End

1. finish/cancel outstanding jobs
2. update MEMORY
3. write validation + blockers + next actions
4. close/handoff task

本地 AI 可以参与 2/3 的内容压缩，但不改变事实来源。

---

## 7. 多 AI 并发

多个 AI 同时操作时：

- workspace task 使用 lease
- 文件写入支持 expected SHA / snapshot
- MEMORY 更新使用 atomic replace
- Journal 作为冲突追踪源
- 第二个 Agent 可以读，但写入前必须拿 lease 或明确 takeover

避免“两个 AI 同时改 MEMORY，最后谁覆盖谁”。

---

## 8. 最终用户体验

用户切换模型时不需要重新解释项目：

```text
GPT → Hanis → 同一 Workspace
Codex → Hanis → 同一 Workspace
Local AI → Hanis → 同一 Workspace
Other MCP Client → Hanis → 同一 Workspace
```

AI 更换，Context 和 Policy 不更换。

这才是 Hanis 真正的可迁移层。
