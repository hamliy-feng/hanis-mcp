# Minimal Open-Source Architecture

## Goal

把 Hanis 从“某一台机器上长期演化出来的生产 Bridge”收敛成任何人都能理解和部署的本地 MCP 工作区桥。

## 1. 默认架构

```text
┌──────────────┐
│   ChatGPT    │
└──────┬───────┘
       │ MCP
       ▼
┌────────────────────────┐
│ OpenAI Secure MCP      │
│ Tunnel                 │
└────────┬───────────────┘
         │ private tunnel
         ▼
┌────────────────────────┐
│ Hanis MCP Server       │
│ 127.0.0.1:<port>/mcp   │
└────────┬───────────────┘
         │
         ▼
┌────────────────────────┐
│ Workspace Registry     │
└───────┬─────────┬──────┘
        │         │
        ▼         ▼
   File Tools   Runner
        │         │
        └────┬────┘
             ▼
      Project Workspace
```

## 2. 五个模块

### Module A — MCP Server

只负责：

- 启动 Streamable HTTP MCP server
- `initialize`
- session 生命周期
- 注册工具
- health/ready endpoint

不要在 `server.ts` 里塞 workspace、runner、journal 的业务细节。

### Module B — Workspace Registry

负责：

- workspace id → root path
- read-only / read-write 权限
- runner profile
- 路径越界保护

示例配置：

```json
{
  "workspaces": [
    {
      "id": "my-project",
      "root": "D:\\Projects\\my-project",
      "access": "read-write",
      "runner": "docker"
    }
  ]
}
```

开源默认必须显式登记项目目录，不允许默认 `D:\`、`/` 这种全盘根目录。

### Module C — Tools

按能力拆文件，而不是一个巨大 `tools.ts`：

```text
src/tools/
├─ workspace.ts
├─ files.ts
├─ git.ts
├─ runner.ts
└─ tasks.ts
```

第一版只实现最有价值且容易解释的工具。

### Module D — Runner

统一接口：

```text
Runner
├─ DockerRunner   default
└─ NativeRunner   optional
```

默认使用 Docker/隔离 runner。

Native runner 明确标记为高权限高级功能，不在首次安装时启用。

### Module E — Continuity

任务续接分两层：

```text
Journal    机器可查询的任务状态
Memory     人/AI 可读的项目事实
```

Journal 可以继续 SQLite。

Memory 采用简单 Markdown：

```text
.agent_memory/MEMORY.md
```

只记录：

- 当前目标
- 硬约束
- 已完成
- 当前状态
- 验证结果
- blocker
- next action

不记录 token、密码、私钥、完整环境变量。

## 3. Transport 不进入核心代码

Hanis 本质上是 MCP server，不应该把“公网怎么进来”写死在核心。

默认文档采用：

```text
Secure MCP Tunnel -> localhost Hanis
```

可选 transport 单独放 docs：

```text
docs/transports/
├─ secure-mcp-tunnel.md
├─ tailscale.md
└─ reverse-proxy.md
```

这样未来 OpenAI tunnel client 更新，也不需要改 Hanis 核心。

## 4. 认证策略

当 Secure MCP Tunnel 作为默认接入层时，Hanis 本地默认只监听回环地址。

开源第一版不建议自己实现一套完整 OAuth server。

如果未来用户确实需要公网部署，再提供 optional auth adapter。

## 5. 推荐配置收敛

生产版多个 config 文件可以先收敛为两个：

```text
config/
├─ hanis.json
└─ workspaces.json
```

`hanis.json`：

```json
{
  "server": {
    "host": "127.0.0.1",
    "port": 7677,
    "path": "/mcp"
  },
  "runner": {
    "default": "docker"
  },
  "safety": {
    "snapshotBeforeWrite": true,
    "allowNativeRunner": false
  }
}
```

## 6. 安装体验目标

理想流程应该压缩成：

```text
1. clone
2. npm install
3. copy example config
4. register one workspace
5. npm run doctor
6. npm start
7. point Secure MCP Tunnel to localhost MCP URL
8. connect in ChatGPT
```

而不是让新用户理解 Tailscale、反代、OAuth、JWT、sessionful/stateless、多个端口和多个守护脚本。

## 7. Windows 一键化

第一版优先 Windows，因为当前生产环境已经验证。

`scripts/install-windows.ps1` 只做：

- 检查 Node
- 检查 Docker（可选但推荐）
- 安装依赖
- 生成 example config
- 询问 workspace root
- 写入 workspace 配置
- 注册开机自启（可选）

`scripts/doctor.ps1` 检查：

- 配置合法
- workspace root 存在
- MCP 端口可用
- Docker 是否可用
- server 可启动
- `/healthz`
- `/mcp` initialize

Tunnel 的创建和凭据不应该被安装脚本偷偷持久化到仓库目录。

## 8. 第一版不做什么

v0.1 不追求：

- 多云 tunnel 抽象
- Web 管理后台
- UI
- 复杂 RBAC
- 多租户
- 自建 OAuth provider
- 默认公网暴露
- 插件市场
- 自动上传 secret

先把“ChatGPT 能稳定、安全地在一个或多个本地项目中执行工作”做好。

## 9. 推荐版本路线

### v0.1 — Local Bridge

- MCP server
- workspace registry
- file tools
- git status/diff
- Docker shell runner
- snapshot
- project memory
- Secure MCP Tunnel 文档

### v0.2 — Task Continuity

- SQLite journal
- task create/resume/handoff/complete
- lease

### v0.3 — Advanced Runtime

- native runner
- runner profiles
- approval policy
- job queue

### v0.4 — Alternate Transports

- Tailscale
- reverse proxy
- optional OAuth

这样开源仓库从第一天就是清晰的，而不是把生产环境所有历史包袱一起公开。
