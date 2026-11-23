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

### Phase 1：MacBook Pro 基础环境初始化（Mac 侧负责）
- [x] 安装 Xcode Command Line Tools、Rosetta 2、Homebrew
- [x] 安装 **OrbStack**（仅作为 Docker 容器运行时，开启 Rosetta 兼容）
- [x] 在 macOS 宿主通过 Homebrew 安装原生 ARM64 工具链
- [x] 安装桌面应用（Antigravity、Cursor、VS Code 等）

### Phase 2：局域网 / Tailscale 传输（双侧协同）
- [x] 同步基础非敏感配置（Mac `~/.gitconfig` 设为 `osxkeychain`、VS Code & Cursor `settings.json`、WSL `.bashrc` 参考 `wsl-bashrc.ref`）
- [x] 同步 Antigravity 配置与本地对话历史（`~/.gemini/config/config.json`、`mcp_config.json`、`mcp/`，以及 `conversations`、`brain`、`annotations`、`chapter_titles` 增量合并）
- [x] 同步代码项目：`~/source/csn-workspace`（完整目录）、`~/source/framework`、`~/source/GambleGuard-Workspace` 与 `~/source/monorepo`
- [x] 在 Mac Rancher Desktop 上重新构建 `linux/arm64` 镜像：`csn_baseimage:dev` 与 `csn_bo_consumer:dev`
- [ ] 传输其余凭证与密钥（`~/.ssh/*.pem`、`~/.aws` 等 —— **用户要求暂不同步 AWS/instabox 等访问密钥**）
- [ ] 按需传输其余代码目录（`~/source/csnremaster-workspace` 等）
- [x] 旧机 MySQL 容器与数据已确认无需迁移（待 Windows Navicat 导出 `.ncx` 连接配置）

### Phase 3：容器与数据库重建（Mac 侧负责）
- [x] 批量替换 `stacks/*/compose.yml` 挂载路径为 `/Users/andytsui/source/...` 并拉起 `mysql`、`redis`、`kafka-dev`、`kafka-init`、`ap-csn-api`、`ap-csn-bo`、`ap-csn-bo-consumer`、`ap-frontend`
- [ ] 启动 `gambleguard-mysql` 并导入 `mysql` (`app`) 与 `gambleguard-mysql` 的 `.sql` 备份

### Phase 4 & 5：IDE 恢复与整体验收（Mac 侧负责）
- [ ] 批量安装 VS Code / Cursor 扩展并恢复配置
- [ ] 验证 Antigravity（含 `meegle`、`slack` MCP）
- [ ] 完成 SSH、AWS/ECR、数据库、代码编译全项验收

---

## 交接日志（Handoff Log）
- **2026-11-23 (monorepo 同步)**：已将 `~/source/monorepo` 同步至 Mac `~/source/monorepo`；Mac 端 `tailscale-app` 因 `.pkg` 需管理员权限，待用户在 Mac 本地执行 `brew reinstall --cask tailscale-app`。
- **2026-11-23 (framework & stacks 容器拉起)**：已同步 `~/source/framework` 至 Mac；构建 `linux/arm64` 版 `csn_bo_consumer:dev`；批量替换 `stacks/*/compose.yml` 路径为 `/Users/andytsui/source/...` 并成功拉起 `mysql`、`redis`、`kafka-dev`、`ap-csn-api`、`ap-csn-bo`、`ap-csn-bo-consumer`、`ap-frontend`。待 Windows 侧启动 Rancher Desktop 后导出并导入 MySQL 备份。
- **2026-11-23 (Antigravity 对话历史)**：已增量合并同步 Antigravity 本地对话历史（`conversations` 共 20 个会话、`brain` 共 19 个目录、`annotations`、`chapter_titles`），保留了 Mac 端原有新建会话。
- **2026-11-23 (csn-workspace & ARM64 镜像)**：已完整同步 `csn-workspace` 至 Mac `~/source/csn-workspace/`，并在 Mac Rancher Desktop 上使用 `Dockerfile.arm64`（切换至 `archive.debian.org`）成功构建 `linux/arm64` 架构的 `csn_baseimage:dev` 镜像。
- **2026-11-23 (Antigravity 与代码同步)**：已同步 Antigravity 配置（`config.json`、`mcp_config.json`、`mcp/`），并已将 `csn-workspace/csn_bo` 与 `GambleGuard-Workspace` 传输至 Mac `~/source/`。
- **2026-11-23 (配置同步)**：已将 `~/.gitconfig`（适配为 `osxkeychain`）、VS Code / Cursor `settings.json`、WSL `wsl-bashrc.ref` 同步至 Mac，并核对安装了非 WSL 扩展（已按要求跳过 AWS、instabox、`.ssh/*.pem` 及含 token 的 `mcp.json`）。
- **2026-11-23 (更新)**：调整方案为直接在 macOS 宿主配置原生 ARM64 开发环境（不使用 OrbStack Ubuntu VM，OrbStack 仅用于 Docker 容器），已同步更新 `MAC_SETUP_PLAN.md` 并重新复制至 Mac。
- **2026-11-23**：打通免密 SSH (`andytsui@192.168.0.21`)，导出 `vscode-extensions.txt` 与 `cursor-extensions.txt`，生成 Mac 侧独立安装执行计划 `MAC_SETUP_PLAN.md` 并复制至 Mac `~/pc2mac-migrations/`。
- **2026-11-20**：初始化 `pc2mac-migrations` 仓库，完成 `MIGRATION_PLAN.md`、`MIGRATION_STATUS.md` 与 `.gitignore`。
