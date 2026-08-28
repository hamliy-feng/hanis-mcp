# Agent & Memory Protocol

## 核心目标

让任何 AI 进入项目后，不依赖旧聊天，也能快速恢复：

- 这个项目是什么
- 应该按什么风格工作
- 哪些约束不能破坏
- 上一次做到哪里
- 当前什么是事实
- 下一步应该做什么

同时支持 GPT / Codex / 本地 AI / 其他 MCP Client 无痛切换。

---

## 1. Canonical 文件

推荐 Hanis 对外只定义两个核心文件：

```text
project/
├─ AGENTS.md
└─ .hanis/
   └─ MEMORY.md
```

机器状态另存：

```text
.hanis/state.json
```

Policy 另存：

```text
.hanis/policy.json
```

注意：Policy 永远不由 AGENTS.md 替代。

---

## 2. 为什么叫 AGENTS.md

优先使用 `AGENTS.md` 而不是自造 `agent.md`：

- 与现有 Codex 风格兼容
- 文件名清楚表达“给 Agent 的长期规则”
- 支持未来目录递归作用域
- 其他 AI 即使不原生认识，也可以由 Hanis 主动读取后注入上下文

Hanis 自己不依赖某家模型是否“原生支持”这个文件。

---

## 3. AGENTS.md 标准结构

建议模板：

```markdown
# Project Agent Rules

## Project
项目定位与目标。

## Scope
这些规则作用于哪些目录。

## Sources of truth
哪些文件/数据库/schema 是权威来源。

## Architecture constraints
不能随意改变的架构约束。

## Style
代码、设计、命名、文档风格。

## Build and validation
构建、测试、检查入口。

## Do
鼓励的操作。

## Don't
禁止或必须先确认的项目行为。

## Memory protocol
什么时候更新 MEMORY、哪些内容不得写入。
```

不要把临时任务进度塞进 AGENTS.md。

---

## 4. MEMORY.md 标准结构

```markdown
# Project Memory

## Project identity / root

## Current objective

## Hard constraints / invariants

## Current architecture / authoritative sources

## Decisions and rationale

## Completed work

## Active work / blockers

## Validation / known status

## Next actions

## Important commands / entrypoints

## Session handoff history

## Last updated
```

MEMORY 是“当前压缩状态”，不是完整历史。

完整历史交给 Journal。

---

## 5. 进入已有项目时的 Discovery

Hanis 先做确定性扫描，而不是先调用 AI：

```text
AGENTS.md
CLAUDE.md
GEMINI.md
.cursor/**
.cursorrules
.github/copilot-instructions.md
.agent_memory/**
.agent_index/**
.codex-memory/**
其他已配置 adapter
```

扫描结果只描述“发现了什么”，不自动解释含义。

---

## 6. 首次接入询问

如果发现旧 Agent 规则，Hanis 只在首次接入询问：

```text
检测到这个项目已有 AI 工作规范。

1. 继承并转换为 Hanis 标准
2. 使用 Hanis 标准，但参考原有风格/设计
3. 保持原文件，只在运行时读取
4. 忽略原有规则
```

推荐默认：`2`。

原因：

- 不破坏原项目
- 可以无痛体验 Hanis
- 逐步形成统一标准
- 避免第一次就重写成熟项目的规则

用户选择写入 `.hanis/state.json`：

```json
{
  "contextMode": "hanis-plus-local-style",
  "discoveryDone": true
}
```

以后不重复问。

---

## 7. 本地 AI Adapter

如果用户安装了本地模型，Hanis 可以提供：

```text
Context Adapter
```

职责：

### migrate
把多个旧 AI 指令文件整理为 Hanis `AGENTS.md` 草案。

### summarize
压缩超长 MEMORY。

### compare
指出两个 Agent 规范之间的冲突。

### handoff
将本次 task/journal 压缩为可读 handoff。

### style-inherit
提取原项目的命名、结构、交互、视觉或代码风格。

这些调用不要求云端 API，因此可做到“无额外 API 调用成本”；但仍消耗用户本机算力。

---

## 8. 本地 AI 输出必须可追溯

不能只保存本地 AI 的总结结果。

建议 state 中保存来源哈希：

```json
{
  "contextSources": [
    {
      "path": "CLAUDE.md",
      "sha256": "..."
    },
    {
      "path": ".cursor/rules/project.mdc",
      "sha256": "..."
    }
  ]
}
```

这样源文件变化后，Hanis 知道旧总结已经过期，需要重新比较。

---

## 9. 多 AI 切换

切换时不做“记忆搬家”，因为记忆本来就在项目里：

```text
AI A leaves
  ↓
MEMORY + Journal remain local
  ↓
AI B connects
  ↓
Hanis loads same AGENTS/MEMORY
  ↓
AI B continues
```

这就是 0 friction 的核心。

---

## 10. Agent 风格与模型风格分开

项目风格应保存在 `AGENTS.md`。

例如：

```text
- UI 设计遵循已有 Design System
- 文档先结论后依据
- 不硬编码业务数据
- 修改 API 前先查 contract
```

而“AI 回答更简洁/更激进/更保守”属于客户端模型偏好，不应该污染项目规则。

---

## 11. 冲突优先级

建议固定：

```text
用户当前明确指令
  ↓
Hanis enforced Policy
  ↓
nearest AGENTS.md
  ↓
parent AGENTS.md
  ↓
MEMORY current facts
  ↓
legacy agent files / style reference
```

其中 Policy 不是文本优先级，而是执行层硬边界。

---

## 12. Memory 写入规则

只在以下事实变化时更新：

- objective materially changed
- architecture decision made
- blocker found/cleared
- authoritative source changed
- important file added/removed
- validation status changed
- next safe action changed

不要写：

- 聊天逐句摘要
- chain-of-thought
- secret/token
- 大段日志
- 猜测

---

## 13. 自动化程度

推荐默认：

```text
读 AGENTS/MEMORY      自动
发现 legacy rules     自动
询问继承策略           首次一次
本地 AI 整理草案       可选
写 durable memory     自动/工具触发
高风险权限             永远交给 Policy/用户
```

这能在便利和可控之间保持清晰边界。
