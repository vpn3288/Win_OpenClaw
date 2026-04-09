# 🦞 OpenClaw Windows 11 LTSC 完整安装与运维手册

> **适用环境：** Windows 11 LTSC（无商店、无 winget、无任何预装工具的全新系统）  
> **固定版本：** `openclaw@2026.3.28`（此版本为已知最后一个无 Windows ESM Bug 的稳定版）  
> **局域网架构：** 支持多台设备（Windows 主机 + R4S + N5105 小主机）各自独立端口访问  
> **最后更新：** 2026-04-09

---

## 📋 目录

- [背景知识：为什么固定 2026.3.28](#背景知识为什么固定-20263-28)
- [端口机制说明（必读）](#端口机制说明必读)
- [局域网多机部署规划](#局域网多机部署规划)
- [阶段零：打开正确的终端](#阶段零打开正确的终端)
- [阶段一：安装 PowerShell 7.6.0](#阶段一安装-powershell-760)
- [阶段二：安装 winget（LTSC 专项）](#阶段二安装-wingetltsc-专项)
- [阶段三：安装所有依赖](#阶段三安装所有依赖)
- [阶段四：安装 OpenClaw 固定版本](#阶段四安装-openclaw-固定版本)
- [阶段五：初始化配置](#阶段五初始化配置)
- [阶段六：端口配置（真正生效的方法）](#阶段六端口配置真正生效的方法)
- [阶段七：局域网访问配置](#阶段七局域网访问配置)
- [阶段八：安装 PCClaw Windows 控制技能包](#阶段八安装-pcclaw-windows-控制技能包)
- [Linux 设备安装（R4S / N5105）](#linux-设备安装r4s--n5105)
- [后期维护命令速查表](#后期维护命令速查表)
- [排错手册](#排错手册)
- [安全注意事项](#安全注意事项)

---

## 背景知识：为什么固定 2026.3.28

| 版本 | 状态 | Windows 兼容性 |
|---|---|---|
| `2026.3.28` | ✅ 稳定推荐 | 完全正常 |
| `2026.4.5` | ❌ 有 Bug | 交互式 onboard 崩溃（ESM URL scheme 错误） |
| `2026.4.6+` | ⚠️ 修复中 | 补丁已合并但尚不稳定 |

**Bug 说明：** `2026.4.5` 版本在 Windows 上运行交互式 onboard 时，动态 import 传入了裸 Windows 路径（`C:\...`）而非 `file:///C:/...`，导致 Node.js ESM 加载器报错 `ERR_UNSUPPORTED_ESM_URL_SCHEME`。该 bug 为 GitHub Issue #61810，已确认为回归 bug。

---

## 端口机制说明（必读）

OpenClaw 的端口按以下**严格优先级**决定，**从高到低**：

```
1. CLI 参数 --port XXXXX         ← 最高优先级，覆盖一切
2. 环境变量 OPENCLAW_GATEWAY_PORT ← 覆盖配置文件
3. 配置文件 gateway.port          ← 仅在无环境变量时生效
4. 默认值 18789                   ← 什么都没设时使用
```

> ⚠️ **常见陷阱：** 如果你在配置文件写了端口 18888，但存在环境变量 `OPENCLAW_GATEWAY_PORT=18789`，实际运行的是 18789。改配置文件无效！必须同时处理环境变量。

**排查端口冲突的第一步：**

```powershell
# Windows：查看是否有环境变量覆盖
Get-ChildItem Env: | Where-Object { $_.Name -like "*OPENCLAW*" }

# 查看 .env 文件
Get-Content "$env:USERPROFILE\.openclaw\.env" -ErrorAction SilentlyContinue
```

---

## 局域网多机部署规划

本教程支持以下三机架构，每台分配独立端口：

| 设备 | 系统 | 分配端口 | Web UI 地址 |
|---|---|---|---|
| Windows 主机 | Windows 11 LTSC | `18789` | `http://主机IP:18789` |
| R4S 路由器/小主机 | OpenWRT/Linux | `18790` | `http://R4S的IP:18790` |
| N5105 小主机 | Linux (Debian/Ubuntu) | `18791` | `http://N5105的IP:18791` |

> **端口间距建议至少 20**，因为 Browser CDP 端口会从 `gateway.port + 9` 开始自动分配，避免冲突。

---

## 阶段零：打开正确的终端

1. 按 `Win + X`
2. 点「**终端（管理员）**」或「**Windows PowerShell（管理员）**」
3. 粘贴以下两条，防止脚本执行被拦截：

```powershell
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser
```

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
```

> 出现 `Y/N` 提示时输入 `Y` 回车。看到 "Bypass" 说明权限已够，继续即可。

---

## 阶段一：安装 PowerShell 7.6.0

### 1-1 下载 MSI 安装包

```powershell
$url = "https://github.com/PowerShell/PowerShell/releases/download/v7.6.0/PowerShell-7.6.0-win-x64.msi"
$output = "$env:TEMP\PowerShell-7.6.0-win-x64.msi"
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12, [Net.SecurityProtocolType]::Tls13
Invoke-WebRequest -Uri $url -OutFile $output -UseBasicParsing
Write-Host "下载完成，文件大小：$((Get-Item $output).Length / 1MB) MB"
```

### 1-2 静默安装

```powershell
Start-Process msiexec.exe -Wait -ArgumentList "/I `"$output`" /quiet ADD_EXPLORER_CONTEXT_MENU_OPENPOWERSHELL=1 ADD_FILE_CONTEXT_MENU_RUNPOWERSHELL=1 ENABLE_PSREMOTING=0 REGISTER_MANIFEST=1 USE_MU=1 ENABLE_MU=1 ADD_PATH=1"
Write-Host "PowerShell 7.6.0 安装完成"
```

### 1-3 重启终端，切换到 PowerShell 7

搜索栏输入 `pwsh`，右键「以管理员身份运行」

### 1-4 重新设置执行策略（新窗口必须重做）

```powershell
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
```

### 1-5 验证

```powershell
$PSVersionTable.PSVersion
# 期望输出：Major: 7, Minor: 6
```

---

## 阶段二：安装 winget（LTSC 专项）

> VCLibs 和 UIXaml 报"已有更高版本"是正常的，忽略即可。

### 2-1 修复 TLS（防止 GitHub 下载中断）

```powershell
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12, [Net.SecurityProtocolType]::Tls13
```

### 2-2 下载三个依赖文件

```powershell
$vcLibsUrl = "https://aka.ms/Microsoft.VCLibs.x64.14.00.Desktop.appx"
$vcLibsPath = "$env:TEMP\VCLibs.appx"
Invoke-WebRequest -Uri $vcLibsUrl -OutFile $vcLibsPath -UseBasicParsing
Write-Host "VCLibs 完成：$((Get-Item $vcLibsPath).Length / 1KB) KB"
```

```powershell
$uiXamlUrl = "https://github.com/microsoft/microsoft-ui-xaml/releases/download/v2.8.6/Microsoft.UI.Xaml.2.8.x64.appx"
$uiXamlPath = "$env:TEMP\UIXaml.appx"
Invoke-WebRequest -Uri $uiXamlUrl -OutFile $uiXamlPath -UseBasicParsing
Write-Host "UIXaml 完成：$((Get-Item $uiXamlPath).Length / 1KB) KB"
```

```powershell
# 使用固定版本直链，不走 latest 重定向（防 EOF 断流）
$wingetUrl = "https://github.com/microsoft/winget-cli/releases/download/v1.10.340/Microsoft.DesktopAppInstaller_8wekyb3d8bbwe.msixbundle"
$wingetPath = "$env:TEMP\winget.msixbundle"
Invoke-WebRequest -Uri $wingetUrl -OutFile $wingetPath -UseBasicParsing
Write-Host "winget 包完成：$((Get-Item $wingetPath).Length / 1MB) MB"
```

> ⚠️ winget 包必须 **20MB 以上**，否则是下载中断，重新执行上一条。

### 2-3 安装

```powershell
Add-AppxPackage -Path $vcLibsPath -ErrorAction SilentlyContinue
Add-AppxPackage -Path $uiXamlPath -ErrorAction SilentlyContinue
Add-AppxPackage -Path $wingetPath
```

### 2-4 验证

```powershell
winget --version
# 期望：v1.x.x
```

---

## 阶段三：安装所有依赖

### 3-1 安装 Git

```powershell
winget install --id Git.Git -e --source winget --silent
```

### 3-2 安装 Node.js LTS（稳定版，非 Current）

```powershell
winget install --id OpenJS.NodeJS.LTS -e --source winget --silent
```

### 3-3 安装 Visual C++ 运行库

```powershell
winget install --id Microsoft.VCRedist.2015+.x64 -e --source winget --silent
```

### 3-4 刷新 PATH（让刚装的命令立刻可用）

```powershell
$env:Path = [System.Environment]::GetEnvironmentVariable("Path", "Machine") + ";" + [System.Environment]::GetEnvironmentVariable("Path", "User")
```

### 3-5 验证三件套

```powershell
node --version   # 期望：v22.x.x 或 v24.x.x
npm --version    # 期望：10.x.x 或以上
git --version    # 期望：git version 2.x.x
```

> 如果 `node` 仍然找不到，关闭终端重新打开，再执行 3-4。

### 3-6 更新 npm

```powershell
npm install -g npm@latest
npm --version
```

### 3-7 配置 npm 全局路径（永久修复"命令找不到"）

```powershell
$npmGlobalBin = "$(npm prefix -g)"
$currentPath = [System.Environment]::GetEnvironmentVariable("Path", "Machine")
if ($currentPath -notlike "*$npmGlobalBin*") {
    [System.Environment]::SetEnvironmentVariable("Path", "$currentPath;$npmGlobalBin", "Machine")
    Write-Host "✅ PATH 已更新：$npmGlobalBin"
} else {
    Write-Host "✅ PATH 已包含 npm 全局路径"
}
$env:Path = [System.Environment]::GetEnvironmentVariable("Path", "Machine") + ";" + [System.Environment]::GetEnvironmentVariable("Path", "User")
```

---

## 阶段四：安装 OpenClaw 固定版本

### 4-1 安装 2026.3.28（固定稳定版）

```powershell
npm install -g openclaw@2026.3.28
```

### 4-2 刷新 PATH

```powershell
$env:Path = [System.Environment]::GetEnvironmentVariable("Path", "Machine") + ";" + [System.Environment]::GetEnvironmentVariable("Path", "User")
```

### 4-3 验证

```powershell
openclaw --version
# 期望输出包含：2026.3.28
```

---

## 阶段五：初始化配置

### 5-1 清理旧配置（全新安装时执行）

```powershell
Remove-Item -Recurse -Force "$env:USERPROFILE\.openclaw" -ErrorAction SilentlyContinue
Write-Host "旧配置已清理"
```

### 5-2 非交互式初始化（绕过 Windows ESM bug）

```powershell
openclaw onboard --non-interactive --accept-risk --mode local --flow quickstart --auth-choice skip --skip-channels --skip-skills --skip-search --skip-health --skip-ui
```

### 5-3 配置 API Key

**Anthropic Claude（推荐，效果最好）**  
→ 去 https://console.anthropic.com/ 注册，格式：`sk-ant-...`

```powershell
openclaw config set providers.anthropic.apiKey "sk-ant-你的key"
openclaw config set agents.defaults.provider "anthropic"
openclaw config set agents.defaults.model "claude-sonnet-4-5"
```

**OpenRouter（有免费额度，适合先试用）**  
→ 去 https://openrouter.ai/ 注册，格式：`sk-or-...`

```powershell
openclaw config set providers.openrouter.apiKey "sk-or-你的key"
openclaw config set agents.defaults.provider "openrouter"
openclaw config set agents.defaults.model "anthropic/claude-3.5-sonnet"
```

---

## 阶段六：端口配置（真正生效的方法）

> 本阶段解决"配置文件写了端口但实际不生效"的问题。

### 6-1 查看当前实际运行端口

```powershell
# 查看所有 OPENCLAW 相关环境变量
Get-ChildItem Env: | Where-Object { $_.Name -like "*OPENCLAW*" }

# 查看 .env 文件
Get-Content "$env:USERPROFILE\.openclaw\.env" -ErrorAction SilentlyContinue

# 查看当前配置文件里的端口
openclaw config get gateway.port
```

### 6-2 彻底修改端口（三步缺一不可）

以下以 Windows 主机改为 `18789` 为例：

**步骤一：删除环境变量覆盖（如果存在）**

```powershell
# 删除当前会话的环境变量
Remove-Item Env:OPENCLAW_GATEWAY_PORT -ErrorAction SilentlyContinue

# 删除系统级永久环境变量
[System.Environment]::SetEnvironmentVariable("OPENCLAW_GATEWAY_PORT", $null, "Machine")
[System.Environment]::SetEnvironmentVariable("OPENCLAW_GATEWAY_PORT", $null, "User")
Write-Host "环境变量已清除"
```

**步骤二：修改 .env 文件（如果存在覆盖）**

```powershell
$envFile = "$env:USERPROFILE\.openclaw\.env"
if (Test-Path $envFile) {
    $content = Get-Content $envFile
    $content = $content | Where-Object { $_ -notmatch "OPENCLAW_GATEWAY_PORT" }
    Set-Content $envFile $content
    Write-Host ".env 文件已更新"
}
```

**步骤三：修改配置文件端口**

```powershell
# Windows 主机用 18789
openclaw config set gateway.port 18789
```

**步骤四：重启 Gateway 使配置生效**

```powershell
openclaw gateway stop
Start-Sleep -Seconds 2
openclaw gateway start
Start-Sleep -Seconds 3
openclaw gateway status
```

**步骤五：验证端口确实生效**

```powershell
# 查看实际监听端口
netstat -ano | Select-String ":18789"
```

### 6-3 三台设备端口配置命令（分别在对应设备上执行）

**Windows 主机 → 端口 18789：**

```powershell
openclaw config set gateway.port 18789
```

**R4S（SSH 进入后）→ 端口 18790：**

```bash
openclaw config set gateway.port 18790
```

**N5105（SSH 进入后）→ 端口 18791：**

```bash
openclaw config set gateway.port 18791
```

---

## 阶段七：局域网访问配置

> ⚠️ **安全前提：** 局域网绑定必须先设置 auth token，否则 Gateway 拒绝启动。

### 7-1 生成并设置认证 Token

```powershell
# 自动生成安全 Token
openclaw doctor --generate-gateway-token

# 查看生成的 Token（保存好，访问 Web UI 时需要）
openclaw config get gateway.auth.token
```

### 7-2 绑定到局域网（所有设备通用）

```powershell
openclaw config set gateway.bind "lan"
```

### 7-3 开放 Windows 防火墙

```powershell
# Windows 主机：开放 18789 端口
New-NetFirewallRule -DisplayName "OpenClaw Gateway 18789" -Direction Inbound -Protocol TCP -LocalPort 18789 -Action Allow

# 验证规则
Get-NetFirewallRule -DisplayName "OpenClaw Gateway*"
```

### 7-4 重启 Gateway

```powershell
openclaw gateway restart
Start-Sleep -Seconds 3
openclaw gateway status --json
```

### 7-5 验证局域网可达

```powershell
# 查看本机 IP
ipconfig | Select-String "IPv4"

# 测试本机访问
Invoke-WebRequest -Uri "http://127.0.0.1:18789" -UseBasicParsing -TimeoutSec 5
```

在局域网内其他设备浏览器访问：`http://[Windows主机IP]:18789`

---

## 阶段八：安装 PCClaw Windows 控制技能包

PCClaw 提供 14 个 Windows 原生技能（截图、OCR、UI 自动化、剪贴板、语音识别/合成、系统通知等），让 OpenClaw 真正能控制你的电脑。

```powershell
# 打开 PCClaw 官网查看最新安装命令
Start-Process "https://pcclaw.ai/"
```

按官网页面上的一键安装命令执行，会自动注册所有技能到 OpenClaw。

---

## Linux 设备安装（R4S / N5105）

> 以下命令在 SSH 连接到 R4S 或 N5105 后执行。

### 安装 Node.js 22 LTS

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt-get install -y nodejs
node --version
npm --version
```

### 安装 OpenClaw 固定版本

```bash
npm install -g openclaw@2026.3.28
```

### 初始化配置

```bash
openclaw onboard --non-interactive --accept-risk --mode local --flow quickstart \
  --auth-choice skip --skip-channels --skip-skills --skip-search --skip-health --skip-ui
```

### 配置端口（R4S 示例，N5105 改为 18791）

```bash
# 删除可能存在的环境变量覆盖
unset OPENCLAW_GATEWAY_PORT
sed -i '/OPENCLAW_GATEWAY_PORT/d' ~/.openclaw/.env 2>/dev/null

# 设置端口
openclaw config set gateway.port 18790   # R4S
# openclaw config set gateway.port 18791 # N5105

# 设置 API Key
openclaw config set providers.anthropic.apiKey "sk-ant-你的key"
openclaw config set agents.defaults.provider "anthropic"
openclaw config set agents.defaults.model "claude-sonnet-4-5"
```

### 配置局域网访问

```bash
# 生成 Token
openclaw doctor --generate-gateway-token
openclaw config get gateway.auth.token

# 绑定局域网
openclaw config set gateway.bind "lan"

# 开放防火墙（Ubuntu/Debian）
sudo ufw allow 18790/tcp   # R4S
# sudo ufw allow 18791/tcp # N5105
sudo ufw status
```

### 安装为系统服务（开机自启）

```bash
# 启用 systemd 用户服务
loginctl enable-linger "$(whoami)"

# 安装 Gateway 服务
openclaw gateway install

# 启动
openclaw gateway start
systemctl --user status openclaw-gateway.service
```

### 验证

```bash
openclaw gateway status
# 浏览器访问：http://[设备IP]:18790（或 18791）
```

---

## 后期维护命令速查表

### 日常操作

```powershell
# 打开 Web UI 控制面板
openclaw dashboard

# 查看 Gateway 运行状态
openclaw gateway status

# 启动 / 停止 / 重启 Gateway
openclaw gateway start
openclaw gateway stop
openclaw gateway restart

# 实时查看日志
openclaw logs --follow

# 查看完整状态（含所有频道）
openclaw status --all
```

### 诊断与修复

```powershell
# 自动诊断（第一个排查命令）
openclaw doctor

# 自动修复配置问题
openclaw doctor --fix

# 查看频道连接状态
openclaw channels status
openclaw channels status --probe

# 查看已安装插件
openclaw plugins list --json

# 查看模型状态
openclaw models status
```

### 配置管理

```powershell
# 查看所有配置
openclaw config get

# 查看单项配置
openclaw config get gateway.port
openclaw config get gateway.bind
openclaw config get gateway.auth.token
openclaw config get agents.defaults.model

# 修改配置
openclaw config set gateway.port 18789
openclaw config set gateway.bind "lan"
openclaw config set agents.defaults.model "claude-opus-4-5"

# 备份配置（升级前必做！）
$date = Get-Date -Format "yyyyMMdd-HHmm"
Copy-Item "$env:USERPROFILE\.openclaw" "$env:USERPROFILE\.openclaw-backup-$date" -Recurse
Write-Host "备份完成：.openclaw-backup-$date"
```

### 版本管理

```powershell
# 查看当前版本
openclaw --version

# 固定安装稳定版（推荐）
npm install -g openclaw@2026.3.28

# 升级到最新版（注意：可能有新 bug）
npm install -g openclaw@latest

# 降级回稳定版
npm install -g openclaw@2026.3.28
openclaw gateway restart
```

### 端口排查专用

```powershell
# 查看所有 OPENCLAW 环境变量
Get-ChildItem Env: | Where-Object { $_.Name -like "*OPENCLAW*" }

# 查看 .env 文件内容
Get-Content "$env:USERPROFILE\.openclaw\.env" -ErrorAction SilentlyContinue

# 查看配置文件端口
openclaw config get gateway.port

# 查看实际监听端口
netstat -ano | Select-String ":18789"
netstat -ano | Select-String ":18790"
netstat -ano | Select-String ":18791"

# 测试 Gateway 是否响应
Invoke-WebRequest -Uri "http://127.0.0.1:18789" -UseBasicParsing -TimeoutSec 5
```

### Token 管理

```powershell
# 查看当前 Token
openclaw config get gateway.auth.token

# 重新生成 Token
openclaw doctor --generate-gateway-token

# 手动设置 Token
openclaw config set gateway.auth.token "你的长随机字符串"
```

---

## 排错手册

| 错误 / 症状 | 原因 | 解决方法 |
|---|---|---|
| `openclaw: command not found` | PATH 未刷新 | 重新执行阶段三 3-4，或重开终端 |
| `ERR_UNSUPPORTED_ESM_URL_SCHEME` | 版本 >= 2026.4.5 的 Windows bug | `npm install -g openclaw@2026.3.28` |
| 配置文件改了端口不生效 | 环境变量优先级更高 | 见阶段六 6-2，清除环境变量 |
| `refusing to bind gateway without auth` | 开了 lan 绑定但没设 Token | 先执行 `openclaw doctor --generate-gateway-token` |
| `EADDRINUSE` 端口占用 | 上次 Gateway 未正常关闭 | `openclaw gateway stop --force` |
| Web UI 局域网无法访问 | 防火墙未开放端口 | 见阶段七 7-3，添加防火墙规则 |
| `schtasks` 相关错误 | 权限不足，无法创建计划任务 | 正常现象，自动降级为启动文件夹模式 |
| `sharp` 构建失败 | libvips 冲突 | `$env:SHARP_IGNORE_GLOBAL_LIBVIPS=1` 后重装 |
| winget 下载 EOF 错误 | 网络中断/GitHub CDN 问题 | 加 TLS 修复后用固定版本直链重试 |
| Gateway 启动后立刻崩溃 | 配置文件损坏 | `openclaw doctor --fix`，或删除重建 |

### 完全重装（核武器）

```powershell
# 1. 停止 Gateway
openclaw gateway stop

# 2. 备份配置
$date = Get-Date -Format "yyyyMMdd-HHmm"
Copy-Item "$env:USERPROFILE\.openclaw" "$env:USERPROFILE\.openclaw-backup-$date" -Recurse -ErrorAction SilentlyContinue

# 3. 卸载
npm uninstall -g openclaw

# 4. 删除配置
Remove-Item -Recurse -Force "$env:USERPROFILE\.openclaw" -ErrorAction SilentlyContinue

# 5. 重装稳定版
npm install -g openclaw@2026.3.28

# 6. 刷新 PATH
$env:Path = [System.Environment]::GetEnvironmentVariable("Path","Machine") + ";" + [System.Environment]::GetEnvironmentVariable("Path","User")

# 7. 重新初始化
openclaw onboard --non-interactive --accept-risk --mode local --flow quickstart --auth-choice skip --skip-channels --skip-skills --skip-search --skip-health --skip-ui
```

---

## 安全注意事项

- **不要将 Gateway 暴露到公网**，局域网访问已经是权限很大的操作
- **API Key 不要泄露**，不要截图，不要提交到 Git 仓库
- 在 `openclaw.json` 里设置 `channels.whatsapp.allowFrom` / `channels.telegram.allowFrom` 白名单，只允许你自己的账号控制
- 定期运行 `openclaw doctor` 检查安全配置
- 已知 CVE：CVE-2026-25253（CVSS 8.8），一个已修复的 WebSocket RCE 漏洞，请确保使用 2026.3.28 及以上版本

---

## 参考资料

- [OpenClaw 官方文档](https://docs.openclaw.ai/)
- [OpenClaw 安装指南](https://docs.openclaw.ai/install)
- [Windows 平台说明](https://docs.openclaw.ai/platforms/windows)
- [Gateway 配置参考](https://docs.openclaw.ai/gateway/configuration)
- [GitHub Issue #61810 - Windows ESM Bug](https://github.com/openclaw/openclaw/issues/61810)
- [PCClaw Windows 技能包](https://pcclaw.ai/)
- [npm 包页面](https://www.npmjs.com/package/openclaw)

---

*本文档基于 OpenClaw 2026.3.28，结合官方文档、GitHub Issues 及社区经验整理。*
