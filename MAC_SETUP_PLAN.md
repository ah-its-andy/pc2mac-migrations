# MacBook Pro 侧配置与安装执行计划

> **适用主机**：`Andy-Tsui.local` (Apple Silicon `arm64`, macOS 27.0)
> **架构策略**：macOS 宿主保持极简，使用 **OrbStack Ubuntu (ARM64)** 作为类 WSL2 隔离开发环境，IDE 通过 `Remote-SSH` 连入开发。
> **关联文档**：完整迁移总览见 [MIGRATION_PLAN.md](./MIGRATION_PLAN.md)，进度看板见 [MIGRATION_STATUS.md](./MIGRATION_STATUS.md)。

---

## Step 1：macOS 宿主基础与软件安装

### 1.1 安装 Xcode CLI 工具、Rosetta 2 与 Homebrew
在 Mac 终端依次执行：
```bash
# 1. Xcode Command Line Tools
xcode-select --install

# 2. Rosetta 2（用于兼容少数 x86_64 Docker 镜像或工具）
softwareupdate --install-rosetta --agree-to-license

# 3. 安装 Homebrew (ARM64)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```
安装完 Homebrew 后，按终端提示把 `/opt/homebrew/bin/brew shellenv` 写入 `~/.zprofile` 并 `source ~/.zprofile`。

### 1.2 安装 Mac 端基础 CLI 与桌面应用
```bash
# 基础 CLI（用于拉取仓库、传输与日常操作）
brew install git gh rsync jq ripgrep

# 核心容器与隔离 VM 引擎（替代 WSL2 + Rancher Desktop）
brew install --cask orbstack

# 桌面开发与协作软件
brew install --cask \
  1password \
  cursor \
  visual-studio-code \
  slack \
  lark \
  telegram \
  tailscale
```
> **说明**：`Antigravity` 请直接从官方安装包安装并登录账号；`Unity Hub` 已确认不迁移。

---

## Step 2：创建 OrbStack Ubuntu 隔离开发环境

1. 打开 **OrbStack.app**，完成初始化引导。
2. 在 OrbStack 设置中确认开启 **Rosetta (Use Rosetta to run x86_64/amd64 binaries)**，以便后续兼容部分仅提供 `linux/amd64` 的私有镜像（如 `csn_api` / `csn_baseimage:dev`）。
3. 创建一个 Ubuntu 虚拟机（建议命名为 `dev` 或 `ubuntu`，用户名 `andi` 或默认 `ubuntu`）：
   ```bash
   orb create ubuntu dev
   ```
4. 验证隔离环境与双向挂载：
   ```bash
   # 进入隔离 Ubuntu 终端
   orb -m dev

   # 在 macOS 中访问 VM 磁盘
   ls ~/OrbStack/dev

   # 在 Mac 上测试 SSH 直连 VM（OrbStack 会自动配置 ~/.ssh/config）
   ssh dev@orb "uname -a"
   ```

---

## Step 3：Ubuntu 隔离环境内工具链安装 (`linux/arm64`)

> [!IMPORTANT]
> 请勿从旧 Windows WSL2 直接拷贝 `linux/amd64` 二进制工具；请在 **Ubuntu VM 内部** 执行以下安装（均为 `arm64` 原生版本）。

进入 VM（`orb -m dev`）后执行：

### 3.1 基础构建工具
```bash
sudo apt update && sudo apt install -y \
  git curl wget build-essential jq ripgrep unzip rsync ca-certificates gnupg
```

### 3.2 语言运行时（对照原 WSL2 环境）
- **Node.js (nvm + Node v24)**：
  ```bash
  curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
  source ~/.bashrc
  nvm install 24
  ```
- **Python 3 & uv**：
  ```bash
  sudo apt install -y python3 python3-pip python3-venv
  curl -LsSf https://astral.sh/uv/install.sh | sh
  ```
- **Go (1.25.x `linux-arm64`)**：
  ```bash
  # 按需调整具体小版本号，注意使用 linux-arm64
  curl -LO https://go.dev/dl/go1.25.1.linux-arm64.tar.gz
  mkdir -p ~/.local && tar -C ~/.local -xzf go1.25.1.linux-arm64.tar.gz
  echo 'export PATH="$HOME/.local/go/bin:$HOME/go/bin:$PATH"' >> ~/.bashrc
  ```
- **Java (JDK 21)**：
  ```bash
  sudo apt install -y openjdk-21-jdk
  ```
- **.NET SDK (arm64)**：
  ```bash
  curl -sSL https://dot.net/v1/dotnet-install.sh | bash /dev/stdin --channel LTS
  echo 'export PATH="$HOME/.dotnet:$PATH"' >> ~/.bashrc
  ```
- **AWS CLI v2 (`linux-aarch64`)**：
  ```bash
  curl "https://awscli.amazonaws.com/awscli-exe-linux-aarch64.zip" -o "awscliv2.zip"
  unzip awscliv2.zip && sudo ./aws/install
  rm -rf awscliv2.zip aws
  ```
- **Docker CLI 验证**：
  ```bash
  docker version
  docker compose version
  ```
  （OrbStack 会自动将 Docker CLI 与容器引擎透传进 `dev` 虚拟机，无需在 VM 内单独装 Docker Daemon。）

---

## Step 4：Mac 侧 IDE 扩展与 Antigravity 配置

### 4.1 批量安装 VS Code / Cursor 扩展
扩展清单已随本计划一起复制到 Mac（`vscode-extensions.txt` 和 `cursor-extensions.txt`）：
```bash
# 确保 code / cursor CLI 已加入 PATH（可在各自 IDE 内按 Cmd+Shift+P 执行 "Install 'code' command in PATH"）
cat vscode-extensions.txt | xargs -L 1 code --install-extension
cat cursor-extensions.txt | xargs -L 1 cursor --install-extension
```
并安装 `ms-vscode-remote.remote-ssh` / `anysphere.remote-ssh`，之后即可在 Mac 上通过 **Remote-SSH** 连入 `dev@orb` 开发。

### 4.2 Antigravity 与日常账号登录
- 登录 **1Password**、**Tailscale**、**Slack**、**Lark**、**Telegram**、浏览器账号。
- 登录 **Antigravity**，打开复制过来的 `~/pc2mac-migrations` 目录，即可让 Mac 侧 Antigravity 直接接手后续迁移与验收。

---

## Step 5：接收 Windows 侧数据后的恢复操作（数据推过来后执行）

1. **修复 SSH 密钥权限**（在 Ubuntu VM 内，若 macOS 宿主也需要则同步一份）：
   ```bash
   chmod 700 ~/.ssh
   chmod 600 ~/.ssh/*.pem ~/.ssh/config 2>/dev/null || true
   ```
2. **验证 AWS 与 Git 连通性**：
   ```bash
   aws sts get-caller-identity
   ssh -T git@github.com
   ```
3. **启动容器并导入数据库 `.sql` 备份**（`mysql` 的 `app` 库、`gambleguard-mysql` 的 `gambleguard` / `gambleguard-internal` 库）：
   - 先按 `~/stacks` / `~/csn-compose-wsl` 拉起 `mysql`、`gambleguard-mysql`、`redis`、`kafka-dev`。
   - 私有 `amd64` 镜像（如 `csn_api`、`csn_baseimage:dev`）在 `compose.yml` 中确认 `platform: linux/amd64`（OrbStack 将通过 Rosetta 2 运行）。
   - 将 Windows 侧导出的 `mysql_app.sql` 与 `gambleguard_mysql.sql` 导入对应容器并核对表数据。
