# Open-Source Release Checklist

## 1. Repository boundary

- [ ] 新建独立 `hanis-mcp` 仓库，不直接把生产目录原样 `git init`。
- [ ] 只复制经过审查的源码与 example config。
- [ ] 生产目录继续保持私有和可运行状态。

## 2. Secret removal

发布前确认仓库中不存在：

- [ ] `.secrets/`
- [ ] `bridge.token`
- [ ] OpenAI tunnel token / client credential
- [ ] tunnel id / client instance id
- [ ] OpenAI organization id / workspace id
- [ ] API key
- [ ] OAuth secret
- [ ] JWT signing secret
- [ ] SSH private key
- [ ] proxy credential
- [ ] cookies / browser profile
- [ ] `.env`

## 3. Local identity removal

- [ ] 用户名
- [ ] 邮箱
- [ ] Windows account name
- [ ] hostname
- [ ] Tailscale device name
- [ ] MagicDNS domain
- [ ] 真实项目名称
- [ ] 真实磁盘根路径
- [ ] 私有 IP / 内网地址

所有示例统一替换为类似：

```text
D:\Projects\example
127.0.0.1:7677
my-project
```

## 4. Runtime state removal

不得提交：

- [ ] `journal.sqlite`
- [ ] `journal.sqlite-wal`
- [ ] `journal.sqlite-shm`
- [ ] `logs/`
- [ ] `snapshots/`
- [ ] `jobs/`
- [ ] `*.lock`
- [ ] build cache
- [ ] local node logs

## 5. Safe defaults

- [ ] 默认 host = `127.0.0.1`
- [ ] 默认不开放整块磁盘
- [ ] 默认显式登记 workspace
- [ ] 默认 native runner = off
- [ ] 默认 shell 网络能力最小化
- [ ] 默认写入前 snapshot = on
- [ ] 默认路径越界保护 = on
- [ ] 默认日志不记录 secret / file full contents
- [ ] `serverInfo.name` 使用 ASCII

## 6. Minimal end-to-end test

至少验证：

- [ ] server starts
- [ ] `/healthz` = 200
- [ ] `/readyz` = 200
- [ ] MCP initialize success
- [ ] tools/list success
- [ ] list_workspaces
- [ ] list_files
- [ ] read_file
- [ ] write temp file
- [ ] snapshot before write
- [ ] restore snapshot
- [ ] git_status
- [ ] run isolated shell command
- [ ] restart server and reconnect
- [ ] Secure MCP Tunnel can reach local `/mcp`
- [ ] ChatGPT can invoke at least one read tool
- [ ] ChatGPT can invoke a write tool with expected safety behavior

## 7. Documentation

- [ ] README: 5 分钟理解项目
- [ ] Quick Start
- [ ] Architecture
- [ ] Security model
- [ ] Workspace config
- [ ] Runner model
- [ ] Memory protocol
- [ ] Secure MCP Tunnel setup
- [ ] Troubleshooting
- [ ] Legacy Tailscale guide（optional）

## 8. Git hygiene

推荐 `.gitignore` 至少覆盖：

```gitignore
node_modules/
dist/
.env
.env.*
!.env.example
.secrets/
logs/
jobs/
snapshots/
*.sqlite
*.sqlite-wal
*.sqlite-shm
*.lock
*.log
config/hanis.json
config/workspaces.json
```

真实配置不提交，只提交：

```text
config/hanis.example.json
config/workspaces.example.json
```

## 9. License / project governance

发布前由仓库所有者明确选择：

- [ ] 开源许可证
- [ ] repository owner / organization
- [ ] contribution policy
- [ ] security reporting channel

许可证不要在未确认的情况下替用户决定。

## 10. Final pre-publish gate

- [ ] 对整个准备发布目录做一次 secret scan
- [ ] 搜索 `C:\Users\`
- [ ] 搜索真实用户名/邮箱
- [ ] 搜索 `api.openai.com` 周边是否意外记录凭据
- [ ] 搜索 `token`
- [ ] 搜索 `secret`
- [ ] 搜索 `Authorization`
- [ ] 搜索真实公网 URL
- [ ] 从一台“干净环境”的视角重新走 Quick Start
- [ ] 确认不依赖生产目录任何文件

通过后再创建公开仓库。
