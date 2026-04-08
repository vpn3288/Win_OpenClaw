
OpenClaw Debian 12 安装终极教程 v3.0

从零开始，每条命令独立，复制粘贴即可执行，不需要任何基础。

══════════════════════════════
阶段零：打开正确的终端
══════════════════════════════

1. 按下 Ctrl + Alt + T 打开终端。
2. 更新系统并安装必备工具：
   输入以下命令来更新您的系统：
   ```
   sudo apt update
   sudo apt upgrade -y
   sudo apt install -y curl wget unzip git
   ```

══════════════════════════════
阶段一：安装 PowerShell 7.6.0
══════════════════════════════

1. **下载并安装 PowerShell 7.6.0：**
   复制并粘贴以下命令，安装 PowerShell 7.6.0：
   ```
   curl -L https://github.com/PowerShell/PowerShell/releases/download/v7.6.0/powershell-7.6.0-linux-x64.tar.gz -o powershell-7.6.0-linux-x64.tar.gz
   mkdir ~/powershell
   tar -xvf powershell-7.6.0-linux-x64.tar.gz -C ~/powershell
   sudo ln -s ~/powershell/pwsh /usr/local/bin/pwsh
   ```

2. **验证 PowerShell 安装：**
   输入以下命令确认安装成功：
   ```
   pwsh --version
   ```
   如果显示版本 `7.6`，说明安装成功。

══════════════════════════════
阶段二：安装必备工具
══════════════════════════════

1. **安装 Git：**
   如果还没有安装 Git，可以通过以下命令安装：
   ```
   sudo apt install git -y
   ```

2. **安装 Node.js LTS：**
   安装 Node.js 的 LTS 版本，确保稳定性：
   ```
   curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
   sudo apt install -y nodejs
   ```

3. **安装 Visual C++ 运行库：**
   ```
   sudo apt install -y libunwind8
   ```

4. **确保所需工具已经安装：**
   输入以下命令检查：
   ```
   git --version
   node --version
   npm --version
   ```
   如果都能正确显示版本号，说明安装成功。

══════════════════════════════
阶段三：安装 OpenClaw 2026.3.28（稳定版）
══════════════════════════════

1. **通过 npm 安装 OpenClaw：**
   复制并执行以下命令：
   ```
   npm install -g openclaw@2026.3.28
   ```

2. **刷新系统 PATH，确保 openclaw 命令生效：**
   ```
   export PATH=$PATH:/usr/local/bin
   ```

3. **验证安装：**
   输入以下命令，确认安装版本：
   ```
   openclaw --version
   ```
   如果显示 `2026.3.28`，说明安装成功。

══════════════════════════════
阶段四：配置 OpenClaw
══════════════════════════════

1. **清理可能存在的旧配置：**
   如果您以前使用过 OpenClaw，可以先清理旧的配置文件：
   ```
   rm -rf ~/.openclaw
   echo "旧配置已清理（没有也没关系）"
   ```

2. **用非交互模式完成初始化：**
   通过以下命令绕过可能出现的 ESM bug 并进行初始化：
   ```
   openclaw onboard --non-interactive --accept-risk --mode local --flow quickstart --auth-choice skip --skip-channels --skip-skills --skip-search --skip-health --skip-ui
   ```

3. **配置 API Key：**
   选择其中一个 API 服务：
   - **选项 A：Anthropic Claude（推荐）**
     注册并获取您的 API Key，输入以下命令：
     ```
     openclaw config set providers.anthropic.apiKey "sk-ant-您的APIKey"
     openclaw config set agents.defaults.provider "anthropic"
     openclaw config set agents.defaults.model "claude-sonnet-4-5"
     ```
   - **选项 B：OpenRouter（免费额度，适合试用）**
     注册并获取 API Key，输入以下命令：
     ```
     openclaw config set providers.openrouter.apiKey "sk-or-您的APIKey"
     openclaw config set agents.defaults.provider "openrouter"
     openclaw config set agents.defaults.model "anthropic/claude-3.5-sonnet"
     ```

4. **启动 OpenClaw Gateway：**
   输入以下命令来安装并启动 OpenClaw Gateway 服务：
   ```
   openclaw gateway install
   openclaw gateway start
   ```

5. **访问控制面板：**
   打开浏览器，访问以下链接：
   ```
   http://127.0.0.1:18789
   ```

══════════════════════════════
后期维护命令速查表
══════════════════════════════

1. **打开控制面板：**
   ```
   openclaw dashboard
   ```

2. **查看运行状态：**
   ```
   openclaw gateway status
   ```

3. **启动/停止 Gateway：**
   启动：
   ```
   openclaw gateway start
   ```
   停止：
   ```
   openclaw gateway stop
   ```

4. **查看日志：**
   ```
   openclaw logs --follow
   ```

5. **诊断修复：**
   自动诊断：
   ```
   openclaw doctor
   ```

   修复配置问题：
   ```
   openclaw doctor --fix
   ```

6. **版本管理：**
   查看当前版本：
   ```
   openclaw --version
   ```
   升级到最新版：
   ```
   npm install -g openclaw@latest
   ```
   升级到指定稳定版：
   ```
   npm install -g openclaw@2026.3.28
   ```

7. **配置管理：**
   查看当前配置：
   ```
   openclaw config get
   ```

   修改配置：
   ```
   openclaw config set agents.defaults.model "claude-opus-4-5"
   ```

8. **备份配置：**
   ```
   cp -r ~/.openclaw ~/.openclaw-backup-$(date +%Y%m%d)
   ```

9. **完全重装：**
   如果需要完全重新安装：
   ```
   openclaw gateway stop
   rm -rf ~/.openclaw
   npm uninstall -g openclaw
   npm install -g openclaw@2026.3.28
   openclaw onboard --non-interactive --accept-risk --mode local --flow quickstart --auth-choice skip --skip-channels --skip-skills --skip-search --skip-health --skip-ui
   ```

关键注意事项：
- 如果出现 `openclaw: command not found` 错误，执行阶段二和阶段三的 PATH 刷新命令，或者重新打开终端。
- 如果 Gateway 启动后无法访问，手动访问 `http://127.0.0.1:18789`。
- 升级时先备份配置，确保升级后能恢复到稳定版本。

