# MacBook Pro 侧配置与安装执行计划

> **适用主机**：`Andy-Tsui.local` (Apple Silicon `arm64`, macOS 27.0)
> **架构策略**：直接在 **macOS 宿主环境** 原生配置全部开发工具链（Homebrew + Apple Silicon 原生运行时），容器仍使用 **OrbStack**（仅用于运行 Docker 容器，不再创建单独的 Ubuntu 虚拟机）。
> **关联文档**：完整迁移总览见 [MIGRATION_PLAN.md](./MIGRATION_PLAN.md)，进度看板见 [MIGRATION_STATUS.md](./MIGRATION_STATUS.md)。

---

## Step 1：macOS 宿主基础与 Homebrew 安装

在 Mac 终端依次执行：
```bash
# 1. Xcode Command Line Tools
xcode-select --install

# 2. Rosetta 2（用于兼容少数 x86_64 Docker 镜像或工具）
softwareupdate --install-rosetta --agree-to-license

# 3. 安装 Homebrew (ARM64)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```
安装完 Homebrew 后，按终端提示将 Homebrew 加入 `~/.zprofile` 并生效：
```bash
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv)"
```

---

## Step 2：安装 CLI 工具链、运行时与桌面应用（macOS 原生 ARM64）

> [!IMPORTANT]
> 请勿从旧 Windows WSL2 拷贝 `linux/amd64` 二进制工具；全部通过 Homebrew 安装 macOS Apple Silicon 原生版本。

### 2.1 基础 CLI 与语言运行时（对照原 WSL2 环境）
```bash
# 基础 CLI
brew install git gh rsync jq ripgrep wget unzip

# 语言运行时与云工具
brew install nvm uv python@3.12 go openjdk@21 awscli
brew install --cask dotnet-sdk
```

安装完成后，将环境变量写入 `~/.zshrc`：
```bash
cat <<'EOF' >> ~/.zshrc

# Homebrew & Runtime Paths
export NVM_DIR="$HOME/.nvm"
[ -s "/opt/homebrew/opt/nvm/nvm.sh" ] && \. "/opt/homebrew/opt/nvm/nvm.sh"
export PATH="/opt/homebrew/opt/openjdk@21/bin:$HOME/go/bin:$HOME/.local/bin:$PATH"
EOF
source ~/.zshrc

# 安装 Node.js v24
nvm install 24
nvm alias default 24
```

### 2.2 容器运行时与桌面应用
```bash
# 容器引擎（仅用于跑 Docker / Compose 容器，内存占用低且支持 Rosetta 模拟 amd64）
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
> **说明**：
> - `Antigravity` 请直接从官方安装包安装并登录账号；`Unity Hub` 已确认不迁移。
> - 打开 **OrbStack.app** 完成初始化，并在设置中开启 **Rosetta**（用于兼容部分仅提供 `linux/amd64` 的私有容器镜像，如 `csn_api`、`csn_baseimage:dev`），之后验证 `docker version` 与 `docker compose version`。

---

## Step 3：Mac 侧 IDE 扩展与 Antigravity 配置

### 3.1 批量安装 VS Code / Cursor 扩展
扩展清单已随本计划一起复制到 Mac（`vscode-extensions.txt` 和 `cursor-extensions.txt`）：
```bash
cd ~/pc2mac-migrations
# 确保 code / cursor CLI 已加入 PATH（可在各自 IDE 内按 Cmd+Shift+P 执行 "Install 'code' command in PATH"）
cat vscode-extensions.txt | xargs -L 1 code --install-extension
cat cursor-extensions.txt | xargs -L 1 cursor --install-extension
```

### 3.2 Antigravity 与日常账号登录
- 登录 **1Password**、**Tailscale**、**Slack**、**Lark**、**Telegram**、浏览器账号。
- 登录 **Antigravity**，打开 `~/pc2mac-migrations` 目录，即可让 Mac 侧 Antigravity 直接接手后续迁移与验收。

---

## Step 4：接收 Windows 侧数据后的恢复操作（数据推过来后执行）

1. **修复 SSH 密钥权限**：
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
