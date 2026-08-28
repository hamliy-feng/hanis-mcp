# Hanis MCP — Local Workspace Bridge

Hanis MCP 的目标很简单：让 ChatGPT 通过 MCP 安全地访问你自己电脑上的工作区，完成读文件、写文件、搜索、运行命令、查看 Git 状态和跨会话续接任务，而不需要把本地服务直接暴露到公网。

> 当前目录是“开源整理区”，不是生产实例。生产版保持在原位置，不在这里直接修改。

## 1. 最小链路

```text
ChatGPT
  ↓
OpenAI Secure MCP Tunnel
  ↓
Local tunnel client
  ↓
http://127.0.0.1:<port>/mcp
  ↓
Hanis MCP Server
  ↓
Workspace Registry
  ↓
File tools / Shell runner / Git / Journal
```

这应该是开源版唯一的默认主线。

## 2. 为什么要简化

当前生产版是在多轮调试中逐渐长出来的，包含：

- Secure MCP Tunnel
- 历史 Tailscale Funnel 公网入口
- `/mcp` sessionful MCP
- `/mcp-stateless` 兼容端点
- OAuth / JWT
- 多 workspace
- Docker runner / native runner
- snapshot
- SQLite journal
- task lease / handoff
- audit / trace
- 多套 runner profile

这些能力不是都应该成为“第一次安装就必须理解”的东西。

开源版应把它们分层：

```text
Core        MCP + workspace + file tools
Safety      path confinement + snapshot + write checks
Continuity  task journal + project memory protocol
Runner      shell / Docker / native execution
Transport   Secure MCP Tunnel（外部接入层）
Legacy      Tailscale / public HTTPS / stateless compatibility
```

## 3. 第一版建议只保留的工具

### Core

- `list_workspaces`
- `workspace_status`
- `list_files`
- `read_file`
- `search_text`
- `write_file`
- `apply_patch`
- `git_status`
- `git_diff`

### Execution

- `run_shell`
- `run_workspace_script`

### Continuity

- `task_create`
- `workspace_resume`
- `task_status`
- `task_handoff`
- `task_complete`

### Safety

- `create_snapshot`
- `restore_snapshot`

不是第一版必需的能力先不要进入默认安装流程。

## 4. 建议的开源仓库结构

```text
hanis-mcp/
├─ src/
│  ├─ server.ts
│  ├─ workspace/
│  │  ├─ registry.ts
│  │  └─ paths.ts
│  ├─ tools/
│  │  ├─ files.ts
│  │  ├─ git.ts
│  │  ├─ runner.ts
│  │  └─ tasks.ts
│  ├─ runner/
│  │  ├─ interface.ts
│  │  ├─ docker.ts
│  │  └─ native.ts
│  ├─ journal/
│  │  └─ sqlite.ts
│  └─ safety/
│     └─ snapshots.ts
├─ config/
│  ├─ hanis.example.json
│  └─ workspaces.example.json
├─ scripts/
│  ├─ install-windows.ps1
│  ├─ doctor.ps1
│  └─ start.ps1
├─ docs/
│  ├─ architecture.md
│  ├─ security.md
│  ├─ memory-protocol.md
│  └─ legacy-tailscale.md
├─ tests/
├─ .env.example
├─ .gitignore
├─ package.json
└─ README.md
```

## 5. 默认安全原则

1. MCP Server 默认只监听 `127.0.0.1`。
2. 默认通过 Secure MCP Tunnel 接入，不要求公网端口。
3. workspace 必须显式登记；不默认开放整块磁盘。
4. 路径解析必须阻止越界和符号链接逃逸。
5. 写操作默认先快照或支持 SHA 校验。
6. shell runner 与文件工具分开授权。
7. `.env`、token、日志、journal、snapshot、SSH 文件全部禁止进入 Git。
8. `serverInfo.name` 使用 ASCII。
9. 项目记忆只能存状态与决策，不能存密码、token、私钥。

## 6. 生产版与开源版的边界

生产版保留现有能力，作为参考实现。

开源版不应直接复制以下内容：

- `.secrets/`
- `journal.sqlite*`
- `logs/`
- `snapshots/`
- 真实 `workspaces.json`
- 真实用户路径
- tunnel id / client instance id
- 真实 OpenAI organization/workspace 信息
- Tailscale 私有设备信息
- SSH key 或挂载配置

## 7. 当前整理状态

- 已确认当前实际主链路是 Secure MCP Tunnel → 本地 `/mcp`。
- 已确认一台 Bridge 可以承载多个 workspace。
- 已把 Tailscale / stateless / OAuth 从“默认主线”降级为兼容层。
- 下一步是从生产代码中抽取最小 Core，而不是整仓复制。

更多细节见：

- `docs/current-production-path.md`
- `docs/minimal-open-source-architecture.md`
- `docs/release-checklist.md`
