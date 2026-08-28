# Current Production Path — 2026-08-25

本文记录当前生产态 Hanis MCP 的真实链路，并区分“当前主线”和“历史兼容线”。

## A. 当前主线

```text
ChatGPT
  ↓
OpenAI MCP App / Connector
  ↓
OpenAI Secure MCP Tunnel
  ↓
local tunnel client
  ↓
127.0.0.1:7677/mcp
  ↓
Hanis MCP Server
  ↓
MCP Streamable HTTP session
  ↓
Tools
  ↓
Workspace Registry
  ↓
File / Git / Runner / Journal / Snapshot
  ↓
Registered local workspace
```

### 关键事实

- 本地服务：`127.0.0.1:7677`
- 当前使用 MCP endpoint：`/mcp`
- 当前是 sessionful Streamable HTTP。
- 每次 initialize 建立 MCP session，并通过 `Mcp-Session-Id` 维持后续请求。
- 当前 2026-08-25 trace 已确认 ChatGPT 请求实际进入 `/mcp`。
- Secure MCP Tunnel 让本地 MCP 不需要直接暴露公网。
- tunnel client 的控制面需要访问 OpenAI API；本机若必须走代理，应使用实际可工作的本地代理端口，而不是写死示例端口。

## B. 本地服务启动链

```text
Windows startup
  ↓
HanisWatchdog.vbs
  ↓
daemon/supervisor.js
  ↓
node --experimental-sqlite dist/server.js
  ↓
127.0.0.1:7677
```

supervisor 负责：

- 防止重复实例
- 拉起 MCP server
- 注入运行环境
- 监测端口
- 进程异常后自动重启

## C. MCP Server 内部结构

```text
server.ts
  ├─ config loader
  ├─ auth
  ├─ workspace registry
  ├─ runner
  ├─ snapshots
  ├─ journal.sqlite
  └─ tool registration
```

当前生产版包含 20+ MCP 工具，覆盖：

- workspace discovery
- file read/write/search
- patch
- git status/diff
- shell/script execution
- jobs
- snapshots
- task journal
- lease
- handoff/resume

## D. 多 workspace 模型

生产版不是“一项目一 Bridge”，而是：

```text
One Hanis Bridge
  ├─ workspace A
  ├─ workspace B
  ├─ workspace C
  └─ ...
```

这应继续保留，因为它能避免每增加一个项目都重复安装 MCP server 和 tunnel。

开源默认不应直接暴露整块磁盘；应要求用户显式配置允许访问的项目根目录。

## E. 项目连续性

Hanis 不应只依赖聊天上下文。项目状态通过本地文件记忆续接：

```text
project/
  ├─ AGENTS.md                # 长期规则，可选
  └─ .agent_memory/
      └─ MEMORY.md            # 当前项目状态
```

若已有项目使用其他约定，可通过 adapter 支持，但开源默认应该只保留一种简单格式。

## F. 历史兼容线

2026-08-23 以前主要使用过：

```text
ChatGPT
  ↓
public HTTPS
  ↓
Tailscale Funnel
  ↓
127.0.0.1:7677
  ↓
/mcp-stateless
```

生产代码中因此还保留：

- Tailscale Funnel 脚本
- public base URL
- OAuth/JWT
- `/mcp-stateless`
- `server/discover` 兼容处理

这些不是当前开源主线的必要组成。

## G. 当前最值得删除的复杂度

对开源版而言，优先从默认路径中移除：

1. Tailscale 必装依赖
2. 公网 HTTPS 配置
3. 自建 OAuth token endpoint
4. stateless MCP endpoint
5. 与历史 ChatGPT 探测行为绑定的兼容代码
6. 机器绑定的硬编码路径
7. 默认全盘 workspace
8. 真实日志/数据库/快照状态

保留它们作为 legacy/optional 文档即可。
