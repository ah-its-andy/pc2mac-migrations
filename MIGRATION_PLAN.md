# Windows (WSL2) → MacBook Pro 工作负载迁移方案

> **目标**：将当前 Windows 11 (x86_64) + WSL2 (`Ubuntu`) + Rancher Desktop 上的开发环境、容器与数据库、凭证配置、AI/IDE 工具链及个人数据，安全、完整地迁移至全新的 MacBook Pro (Apple Silicon / ARM64)。
> **Mac 开发架构**：macOS 宿主保持极简，使用 **OrbStack Linux Machine (`Ubuntu`)**（或开源方案 **Lima**）构建类似 WSL2 的隔离 Linux 开发环境，IDE 通过 `Remote-SSH` 连入开发。
> **传输方式**：局域网直连传输（`rsync` over SSH / SMB 共享）。
> **盘点日期**：2026-11-18

---

## 一、当前环境盘点摘要

### 1.1 系统与硬件基线
- **当前主机**：Microsoft Windows 11 专业版 (64 位, Build 26100)，RAM 31.1 GB
- **磁盘使用**：`C:\` 已用 888.9 GB / 可用 2835.2 GB；WSL2 (`Ubuntu`) 挂载为 `Z:\`
- **目标主机架构注意**：MacBook Pro 为 **Apple Silicon (ARM64)**，Linux x86_64 二进制工具链与部分 Docker 镜像不可直接跨架构硬拷贝，需使用原生 ARM64 版本或在容器运行时中启用 Rosetta 2 兼容。

### 1.2 代码仓库清单
> [!WARNING]
> 以下标记为 **有未提交修改** 的仓库，迁移前必须先 `commit`/`stash` 或打包保护，避免丢失工作进度。

#### WSL2 (`Ubuntu` `/home/andi/source`)
| 路径 | 远端仓库 | 状态 | 备注 |
| :--- | :--- | :--- | :--- |
| `~/source/monorepo` | `github.com/popodaoorg/monorepo.git` | **1 处未提交修改** | 需优先处理 |
| `~/source/GambleGuard-Workspace/states-md` | `github.com/ah-its-andy/states-md.git` | Clean | - |
| `~/source/GambleGuard-Workspace/backendserver` | `anointer4.poponft.com/cicc/backendserver.git` | Clean | Remote 含内嵌凭据，建议迁移后改用 Keychain/SSH |
| `~/source/GambleGuard-Workspace/backendserver-telemetry` | `anointer4.poponft.com/cicc/backendserver-telemetry.git` | Clean | 同上 |
| `~/source/GambleGuard-Workspace/backendserver-internal` | `anointer4.poponft.com/cicc/backendserver-internal.git` | Clean | 同上 |
| `~/source/framework` | `43.218.52.113/csn/framework.git` | Clean | 私有 Git 服务 |
| `~/source/csn-workspace/csn_api` | `43.218.52.113/csn/csn_api.git` | Clean | Remote 含内嵌凭据 |
| `~/source/csn-workspace/csn_bo` | `43.218.52.113/csn/csn_bo.git` | Clean | Remote 含内嵌凭据 |
| `~/source/csn-workspace/csn-states` | `github.com/ah-its-andy/states-md.git` | Clean | - |
| `~/source/csnremaster-workspace/stacks` | `github.com/firstly1mans/stacks.git` | Clean | - |
| `~/source/csnremaster-workspace/csn_bo_remaster_front` | `github.com/firstly1mans/bo_remaster_front.git` | Clean | - |
| `~/source/csnremaster-workspace/bo_remaster` | `github.com/firstly1mans/bo_remaster.git` | Clean | - |

> **已确认排除（不迁移）**：`~/source/tonepivot`（含 `src`、`ui-sketches`、`docs`）、`~/source/darktable-sync` 及相关测试/演示目录（`~/ProPhotoTestset`、`~/tonepivot-demo`）。

#### Windows 宿主 (`C:\Users\mykge`)
- `C:\Users\mykge\source\keyrep`
- `C:\Users\mykge\source\repos`
- `C:\Users\mykge\OneDrive\Documentos\source\{dbcat, navipuppy, watercups, pc2mac-migrations}`（已在 OneDrive 同步目录内）

### 1.3 容器、编排与本地数据库 (Rancher Desktop)
- **容器运行时**：Rancher Desktop (`docker` 29.9.3-rd, `kubectl`, `helm`)
- **当前运行/存在的容器**：
  - 业务服务：`ap-csn-api` (`739275455467.dkr.ecr.ap-east-1.amazonaws.com/csn_api:v1.0.146`)、`ap-csn-bo` (`csn_baseimage:dev`)
  - 基础设施：`mysql`、`gambleguard-mysql`、`redis`、`kafka-dev`、`kafka-init`（及历史容器 `redis-container`、`mysql-container`）
- **持久化数据卷与本地目录**：
  - Docker Volumes：`gambleguard_mysql-data`、`infra_mysql_data`（及若干匿名哈希卷）
  - WSL 本地挂载/编排目录：`~/csn-compose-wsl`、`~/csn-mysql-wsl`、`~/stacks`、`~/stacks-mac`

### 1.4 凭证、密钥与环境配置
- **SSH 密钥与配置**：
  - WSL `~/.ssh/`：`config` 及 8 个 `.pem` 私钥（`AP-Prod.pem`、`CICC-JumpServer-1.pem`、`CICC-JumpServer.pem`、`CICC-Server.pem`、`CICC.pem`、`PoPo-test.pem`、`ap-defender.pem`、`ec2common.pem`）
  - Windows `~/.ssh/`：`known_hosts`
- **云与容器凭证**：
  - WSL `~/.aws/`：`config`、`credentials`
  - WSL `~/.docker/`：`config.json`、`buildx`
- **Shell 与 Git**：WSL `~/.gitconfig`、`~/.bashrc`；Windows `~/.gitconfig`

### 1.5 开发运行时与桌面工具
- **WSL 运行时（在 Mac 的隔离 Ubuntu VM 中重装 ARM64 原生版）**：`git`、`python3`、`go` (`1.25.x`)、`java` (`jdk-21`)、`dotnet`、`.nvm` (Node v24)、`aws` CLI、`Android` SDK
- **Windows 桌面开发软件**：
  - **Visual Studio Community 2022**（macOS 无 VS，建议改用 **JetBrains Rider** 或 **VS Code + C# Dev Kit**；`Unity 2022.3.62f3` & `Unity Hub` 已确认不迁移）
- **AI 工具与 IDE 配置**：
  - **Antigravity**：`C:\Users\mykge\.gemini\antigravity`（含 MCP 配置 `meegle`、`slack`，以及 `brain`、`conversations`、`plugin_data`、`annotations` 等）
  - **VS Code** (`AppData\Roaming\Code\User`) & **Cursor** (`AppData\Roaming\Cursor\User`, `~/.cursor`)：含 `settings.json`、`snippets`、`mcp.json` 及已安装扩展（Avalonia、C# Dev Kit、Unity Tools 等）
  - **OpenCode**、`.copilot`、`.grokbot`、`antigravity-bot`、`antigravity-cli`
- **浏览器、密码与通讯软件**：
  - **1Password**（云同步账号密码）、**Slack**、**Lark (飞书)**、**Telegram Desktop**
  - **Tailscale**（可用于跨设备加密内网直连）
  - **Edge / Chrome / Firefox**（开启账号同步或导出书签）
- **其他本地目录**：WSL 下的 `~/seatunnel-sync`、`~/migration-setup` 等（已排除 `~/ProPhotoTestset`、`~/tonepivot-demo`）

---

## 二、分阶段迁移执行方案

### Phase 0：迁移前准备（在当前 Windows/WSL 上完成）
> 可由助手协助自动化执行。

1. **处理未提交的代码修改**：
   - 检查 `~/source/monorepo` 的 `git status` 与 `git diff`，决定提交推送或生成 patch 打包。
   - 检查所有待迁移仓库的本地分支是否有未 `push` 的 commit。
2. **数据库逻辑导出（最关键）**：
   - 对运行中的 `mysql` 和 `gambleguard-mysql` 执行 `mysqldump --all-databases --single-transaction --routines --triggers` 导出为 `.sql` 文件（存入临时迁移目录如 `~/mac-transfer/db-dumps/`）。
   - **切勿直接依赖跨架构/跨引擎拷贝 MySQL 底层二进制 data 目录**，逻辑 `.sql` 导出可 100% 保证 Mac 端重建成功。
3. **导出清单与配置快照**：
   - 导出 VS Code / Cursor 扩展列表：`code --list-extensions > vscode-extensions.txt`、`cursor --list-extensions > cursor-extensions.txt`。
   - 打包需要保留的敏感配置与目录（仅本地局域网传输，不上传公网）：`~/.ssh`、`~/.aws`、`~/.gitconfig`、Antigravity 配置目录、IDE `User` 配置。

---

### Phase 1：MacBook Pro 基础环境与隔离 Linux 环境初始化（新机到手后）

#### 1.1 隔离环境选型（macOS 上的 WSL2 替代方案）
| 方案 | 特点与优势 | 适用建议 |
| :--- | :--- | :--- |
| **OrbStack (Ubuntu Machine)** *(推荐)* | 单一软件同时提供 **轻量级 Ubuntu 虚拟机 + Docker/K8s**；秒级启动、动态内存、内置 Rosetta 2 跑 x86_64、无缝文件互通，天然支持 `ssh ubuntu@orb` 给 VS Code / Cursor `Remote-SSH` 使用 | 最接近 WSL2 + Rancher Desktop 的无缝体验，个人版免费 |
| **Lima (`lima-vm`)** *(开源备选)* | `brew install lima` 创建隔离 Ubuntu ARM64 虚拟机，基于 Apple Virtualization.framework，支持 Rosetta 与 `Remote-SSH` | 纯开源免费，适合配合 Rancher Desktop / Colima |

#### 1.2 macOS 宿主极简初始化
1. **系统基础与包管理**：
   - 安装 Xcode Command Line Tools：`xcode-select --install`
   - 安装 Rosetta 2（兼容少数 x86_64 容器或工具）：`softwareupdate --install-rosetta --agree-to-license`
   - 安装 **Homebrew**（ARM64 原生路径 `/opt/homebrew`），安装基础工具 `git`, `rsync`, `gh`, `jq`
2. **安装隔离环境与容器引擎**：
   - 安装 **OrbStack**（并创建一个 Ubuntu ARM64 Machine，如命名 `dev` 或 `ubuntu`，开启 Rosetta x86_64 兼容）。
3. **安装桌面应用**：
   - 1Password、Antigravity、Cursor、VS Code、Slack、Lark、Telegram、Tailscale

#### 1.3 Ubuntu 隔离环境内工具链初始化
进入隔离 Ubuntu 环境（如 `orb` 或 `ssh ubuntu@orb`）后，安装 `linux/arm64` 原生工具链（**不要**直接从旧机拷贝 `linux/amd64` 二进制）：
- 基础 CLI：`git`, `curl`, `build-essential`, `jq`, `ripgrep`, `unzip`
- 语言运行时：`nvm` (Node v24)、`python3` / `uv`、`go` (`1.25.x` arm64)、`jdk-21` (arm64)、`dotnet-sdk` (arm64)、`awscli` (aarch64)
- Docker CLI：OrbStack 会自动将宿主 Docker socket 与 `docker` / `docker compose` 透传进 Linux Machine，无需在 VM 内套娃安装 Docker Daemon。

---

### Phase 2：局域网直连传输（Windows/WSL → MacBook Pro）
> 推荐方式：**Mac 开启「远程登录 (SSH)」后，从 WSL2 使用 `rsync` 直接推送**（保留权限、支持断点续传、速度最快）。

1. **开启 Mac 接收端**：
   - Mac 上打开 `系统设置 → 通用 → 共享 → 开启「远程登录」`（记下局域网 IP，如 `192.168.x.x`）。
   - 确保两台机器在同一路由器（尽量使用 5GHz Wi-Fi 或网线/雷雳直连）。
2. **推送 WSL2 工作负载 → Mac 隔离 Ubuntu 环境**：
   - 可先推送到 Mac 宿主中转目录或直接通过 SSH 端口转发推送到 Ubuntu VM（OrbStack 亦支持从 Mac 宿主目录 `/Users/<name>/...` 直接在 VM 内无缝访问，非常方便）：
   - 凭证与配置：`~/.ssh`、`~/.aws`、`~/.gitconfig`、`~/.bashrc`（传输后在 Ubuntu VM 内执行 `chmod 700 ~/.ssh && chmod 600 ~/.ssh/*.pem`；如同时需要在 macOS 宿主使用 Git/SSH，可同步一份到 macOS `~/.ssh`）
   - 代码与编排目录：`~/source`（`rsync` 时加上 `--exclude='tonepivot' --exclude='darktable-sync' --exclude='node_modules' --exclude='bin' --exclude='obj' --exclude='.venv'`）、`~/stacks`、`~/stacks-mac`、`~/csn-compose-wsl`、`~/csn-mysql-wsl`
   - 数据库导出文件与大型数据集：`~/mac-transfer/db-dumps/`、`~/seatunnel-sync`
3. **传输 Windows 宿主侧文件**：
   - Antigravity 数据：`C:\Users\mykge\.gemini\antigravity`（重点迁移 `mcp/`、`brain/`、`plugin_data/`、`conversations/` 等）
   - IDE 配置：`AppData\Roaming\Code\User`、`AppData\Roaming\Cursor\User`、`~/.cursor`
   - 非 OneDrive 同步目录：`C:\Users\mykge\source\keyrep`、`C:\Users\mykge\source\repos`
   - （可通过 WSL 访问 `/mnt/c/Users/mykge/...` 一并 `rsync` 推送，或在 Mac Finder 使用 `smb://` 挂载拉取）

---

### Phase 3：容器与数据库重建（Mac 端）
1. **检查镜像架构兼容性**：
   - `mysql:8.0`、`redis`、`apache/kafka:3.8.1` 均有官方 `linux/arm64` 镜像，Mac 上直接拉取原生镜像即可。
   - 私有镜像如 `739275455467.dkr.ecr.ap-east-1.amazonaws.com/csn_api:v1.0.146` 与 `csn_baseimage:dev`：先用 `aws ecr get-login-password` 登录拉取，若仅提供 `amd64`，在 Compose 中指定 `platform: linux/amd64` 并开启容器引擎的 Rosetta 模拟；长期建议用 `docker buildx` 构建多架构镜像。
2. **启动容器并恢复数据库**：
   - 参考 `~/stacks` 与 `~/stacks-mac` 启动 `mysql`、`gambleguard-mysql`、`redis`、`kafka-dev`。
   - 将 Phase 0 导出的 `.sql` 备份导入新容器，验证表结构与业务数据完整性。

---

### Phase 4：AI 工具、IDE 与桌面应用恢复
1. **Antigravity**：登录账号，恢复 `~/.gemini/antigravity` 下的 `mcp/`、`plugin_data/`、`brain/` 等目录，检查 `meegle` 与 `slack` MCP 连接状态。
2. **VS Code / Cursor**：导入 `settings.json`、`snippets`、`mcp.json`，通过之前导出的扩展列表批量安装插件，并在 macOS `~/.ssh/config` 中配置好 `orb`（OrbStack 自动注入）后，使用 **Remote-SSH** 连入 Ubuntu 隔离环境开发（替代原 Windows 上的 `Remote-WSL`）：
   ```bash
   cat vscode-extensions.txt | xargs -L 1 code --install-extension
   ```
3. **.NET / C# 桌面开发**：如需在 macOS 宿主开发，配置 VS Code C# Dev Kit 或 JetBrains Rider。
4. **日常应用**：登录 1Password、Slack、Lark、Telegram、Tailscale、浏览器账号。

---

## 三、迁移完成验收清单（Checklist）

- [ ] **Git & SSH**：`ssh -T git@github.com` 及各跳板机（`CICC-JumpServer` 等 `.pem`）连接正常；所有仓库无遗漏本地改动
- [ ] **AWS & ECR**：`aws sts get-caller-identity` 与 `aws ecr get-login-password` 正常
- [ ] **容器与数据库**：`ap-csn-api`、`ap-csn-bo`、`mysql`、`gambleguard-mysql`、`redis`、`kafka-dev` 健康运行，业务数据查询正常
- [ ] **语言运行时**：`node`、`python3`、`go`、`java`、`dotnet` 均为 Apple Silicon 原生版本且能正常编译运行项目
- [ ] **AI & IDE**：Antigravity（含 `meegle`/`slack` MCP）、Cursor、VS Code 扩展与配置完整可用
- [ ] **旧机退场缓冲**：确认新 MacBook Pro 稳定工作 1–2 周后，再清理或重置旧 Windows 电脑
