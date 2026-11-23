# Migration Handoff & Status Board

> **用途**：Windows 侧与 MacBook Pro 侧 Antigravity 协同交接看板。每次完成阶段任务后更新本文件并 `git push`，另一侧 `git pull` 后即可无缝接力。

## 当前阶段：`Planning (待启动 Phase 0)`

### Phase 0：Windows / WSL2 迁移前准备（Windows 侧负责）
- [ ] 检查并处理 `~/source/monorepo` 未提交修改（1 处）
- [ ] 检查所有待迁移 Git 仓库是否有未 `push` 的本地分支/提交
- [ ] 导出 `mysql` 与 `gambleguard-mysql` 全库逻辑备份（`.sql`，仅存本地/局域网传输，**勿提交到 Git**）
- [x] 导出 VS Code / Cursor 扩展列表（`vscode-extensions.txt`、`cursor-extensions.txt`）
- [x] 生成 Mac 侧独立安装计划 `MAC_SETUP_PLAN.md` 并同步至 Mac (`~/pc2mac-migrations/`)
- [ ] 整理待传输配置清单（`~/.ssh`、`~/.aws`、`~/.gitconfig`、Antigravity `mcp/` & 配置）

### Phase 1：MacBook Pro 基础与隔离环境初始化（Mac 侧负责）
- [ ] 安装 Xcode Command Line Tools、Rosetta 2、Homebrew
- [ ] 安装 **OrbStack** 并创建 `Ubuntu` (ARM64) Linux Machine（开启 Rosetta 兼容）
- [ ] 在 Ubuntu VM 内安装 `linux/arm64` 工具链（`git`、`nvm`/Node、`python3`/`uv`、`go`、`jdk-21`、`dotnet`、`awscli`）
- [ ] 在 macOS 宿主安装桌面应用（1Password、Antigravity、Cursor、VS Code、Slack、Lark、Telegram、Tailscale）

### Phase 2：局域网 / Tailscale 传输（双侧协同）
- [ ] 传输 WSL2 凭证与配置（`~/.ssh`、`~/.aws`、`~/.gitconfig`、`~/.bashrc`）→ Ubuntu VM
- [ ] 传输代码与编排目录（`~/source` 排除 `tonepivot` & `darktable-sync`，`~/stacks`、`~/stacks-mac`、`~/csn-*`）→ Ubuntu VM
- [ ] 传输数据库 `.sql` 备份与 `~/seatunnel-sync` → Mac / Ubuntu VM
- [ ] 传输 Antigravity 配置与 IDE 用户配置 → macOS 宿主

### Phase 3：容器与数据库重建（Mac 侧负责）
- [ ] 启动 `mysql`、`gambleguard-mysql`、`redis`、`kafka-dev` 容器并导入 `.sql` 备份
- [ ] 验证 `ap-csn-api`、`ap-csn-bo` 镜像架构与运行状态（必要时开启 `linux/amd64` Rosetta 模拟）

### Phase 4 & 5：IDE 恢复与整体验收（Mac 侧负责）
- [ ] 批量安装 VS Code / Cursor 扩展并配置 `Remote-SSH` 连入 `ubuntu@orb`
- [ ] 验证 Antigravity（含 `meegle`、`slack` MCP）
- [ ] 完成 SSH、AWS/ECR、数据库、代码编译全项验收

---

## 交接日志（Handoff Log）
- **2026-11-23**：打通免密 SSH (`andytsui@192.168.0.21`)，导出 `vscode-extensions.txt` 与 `cursor-extensions.txt`，生成 Mac 侧独立安装执行计划 `MAC_SETUP_PLAN.md` 并复制至 Mac `~/pc2mac-migrations/`。
- **2026-11-20**：初始化 `pc2mac-migrations` 仓库，完成 `MIGRATION_PLAN.md`、`MIGRATION_STATUS.md` 与 `.gitignore`。
