# Hanis Permission Model

## 目标

Hanis 不能只有“全开”与“全关”。不同用户、不同 workspace、不同任务需要完全不同的权限。

权限模型必须满足：

- 默认安全
- 能力可组合
- 高风险操作存在显式允许空间
- AI 无法靠提示词绕过权限
- 权限由 Hanis 服务端强制执行
- 权限可以按 workspace / session / action 临时升级

---

## 1. 权限与 Agent 指令彻底分离

```text
AGENTS.md
  = AI 应该怎么做

Hanis Policy
  = AI 实际能做什么
```

即使 AGENTS.md 写着“可以删除整个项目”，只要 Policy 没有 `filesystem.delete`，工具调用就必须被拒绝。

反过来，Policy 即使允许删除，Agent 也仍然应该服从项目规则与用户目标。

---

## 2. 使用 Capability，而不是固定角色

Hanis 内部权限使用原子能力：

```text
workspace.read
filesystem.read
filesystem.write
filesystem.delete
filesystem.move

git.read
git.commit
git.push
git.force

shell.container
shell.native
process.manage
package.install
network.outbound

docker.read
docker.manage

ssh.connect
remote.execute

secrets.use
secrets.read

system.service
system.settings
system.admin
```

角色只是这些 capability 的 preset，不是底层实现。

---

## 3. 风险等级

### R0 — Observe

典型操作：

- list workspace/files
- read/search
- git status/diff
- read logs

默认可允许。

### R1 — Scoped Edit

- create/write/patch file
- rename/move within workspace
- snapshot/restore

默认 Developer 可以允许。

### R2 — Isolated Execute

- Docker/container shell
- test/build/lint
- package install inside sandbox
- limited outbound network

建议默认 ask 或按 profile allow。

### R3 — Host Mutation

- native shell
- delete files
- process kill/start
- Docker create/stop/remove
- git commit

默认 ask。

### R4 — External / Privileged

- git push
- SSH remote execution
- cloud/deploy actions
- system service changes
- admin/elevated command
- secret injection into external process

默认 deny 或 ask-every-time。

### R5 — Destructive / Credential Exposure

- force push
- recursive deletion outside normal build output
- destructive database operation
- disk/partition operation
- security setting changes
- reading/exporting raw secrets

默认 deny。

但 Hanis 不需要把它们“写死为永远不可能”。Owner 可以显式开启 dangerous capability；一旦开启，仍要求动作级确认、审计，并在可行时先快照/备份。

---

## 4. 四维判定

一次工具调用是否能执行，不只看 tool name：

```text
Decision = Capability × Scope × Risk × Approval
```

例如：

```text
filesystem.write
scope = D:\Projects\demo
risk = R1
approval = allow
```

而：

```text
filesystem.delete
scope = D:\Projects\demo
risk = R3
approval = ask
```

即使都是 file tool，权限也不同。

---

## 5. Approval 模式

每个 capability 支持：

```text
deny
ask
auto_allow
```

用户批准时可选择有效期：

```text
allow_once
allow_session
allow_task
allow_workspace
always
```

推荐默认 UI：

```text
允许这次
允许本任务
始终允许此工作区
拒绝
```

对于 R4/R5，不建议出现无提示 `always`，除非用户开启 Expert Mode。

---

## 6. 权限组与首次默认组合

Hanis 对用户提供 5 个权限组。用户侧说明采用正向清单，只显示该组实际开放的能力。

### Reader

开放：

- Workspace / 文件 / 文档 / 代码读取
- 搜索
- Git status / diff
- AGENTS / MEMORY 读取
- 项目分析、Plan、TODO 输出

### Editor

开放：

- 创建 / 修改 / patch / 移动 Workspace 内文件
- 写入前快照与恢复
- MEMORY 更新

### Developer

开放：

- 隔离 Shell / 项目脚本
- build / test / lint
- 容器运行
- 容器内依赖安装与受控网络
- Git commit 等开发动作

### Operator

开放：

- Native Shell
- 进程管理
- Docker 管理
- Git push
- SSH / remote execute
- 系统服务操作
- `secrets.use` 外部任务注入

### Owner

开放：

- Hanis 完整 Capability 集
- 删除、force push、系统管理、原始 Secret 等高风险能力的显式授权 / elevation

首次默认组合：

```text
Reader + Editor
```

用户可以在首次问询中直接选择更高权限组或组合。Hanis 应计算并反馈：

```text
开放 Workspace 数量
开放权限组
开放 Capability 数量
开放 Capability 清单
```

正常状态反馈始终采用正向能力清单：展示已开放的 Workspace、权限组、Capability 数量和 Capability 名称。策略判断只在相关操作实际触发时反馈该次结果。

---

## 7. Workspace 自己有权限边界

Workspace 范围由用户首次输入显式提供，可以是整个盘符、指定目录、单个项目或多个路径。每个 Workspace 都以用户提供的 root 作为 Scope。

示例：

```json
{
  "id": "my-app",
  "root": "D:\\Projects\\my-app",
  "groups": ["reader", "editor"],
  "enabledCapabilities": [
    "workspace.read",
    "filesystem.read",
    "filesystem.write",
    "filesystem.move",
    "git.read"
  ]
}
```

用户可以有多个 workspace：

```text
work-project      Developer
archive           Reader
home-lab          Operator
server-prod       Custom high-risk
```

---

## 8. 高风险操作如何“允许但不失控”

高风险能力不应消失，而应采用 Elevation：

```text
Normal Session
   ↓ 请求 R4/R5
Policy Engine
   ↓
Elevation Required
   ↓
用户明确批准
   ↓
临时 capability token
   ↓
执行单次/本任务
   ↓
自动失效
```

建议 elevation token 只存在内存，不写到 AGENTS.md 或 MEMORY.md。

---

## 9. Secret 权限特殊处理

优先提供：

```text
secrets.use
```

而不是：

```text
secrets.read
```

例如部署时，Hanis 可以把 GitHub token 注入子进程环境，但模型看不到 token 明文。

```text
AI asks: deploy
Hanis injects secret → process
process returns status
AI sees result, not secret
```

只有极少数 Owner 场景才开放 raw secret read。

---

## 10. Shell 命令不能只靠字符串白名单

生产版白名单可以继续作为基础，但开源版应增加命令风险分类：

```text
command
  ↓ parse
executable + args + paths + network intent
  ↓
risk classifier
  ↓
policy decision
```

例如：

```text
npm test                 R2
npm install              R2 + network
rm build/output.tmp      R3 scoped
rm -rf project-root      R5
ssh user@host            R4
powershell as admin      R4/R5
```

不要只因为 executable 是 `powershell` 就全部允许。

---

## 11. 权限配置文件

建议：

```text
.hanis/policy.json
```

但它属于本机/工作区运行策略，不是 Agent 指令。

推荐默认不提交 Git；若团队确实希望共享，可以只提交 `policy.example.json`。

---

## 12. Policy Engine 最终顺序

```text
Tool Call
  ↓
Resolve workspace
  ↓
Resolve requested operation
  ↓
Check path/scope
  ↓
Map capability
  ↓
Calculate risk
  ↓
Read profile + overrides
  ↓
ALLOW / ASK / DENY
  ↓
If ALLOW → snapshot when applicable
  ↓
Execute
  ↓
Audit
```

权限规则必须在执行器之前发生，不能交给模型自己决定。
