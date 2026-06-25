---
name: setup-feishu-hooks
description: >
  设置 Claude Code 飞书（Feishu/Lark）群机器人 Hook 通知，在 Stop、Notification、TaskCreated、TaskCompleted 事件触发后向飞书群发送卡片消息提醒。支持焦点检测——仅当 Claude Code 窗口不处于前台时才发送通知，避免在主动使用时被打扰。同时支持 **Windows（PowerShell）** 和 **macOS（bash + python3）** 平台，自动适配。当用户提及"飞书通知"、"Feishu hook"、"飞书提醒"、"Lark webhook"、"Hook 通知"、"后台通知"、"webhook 通知"等关键词时，应该使用此技能。
compatibility: |
  Windows: PowerShell 5.1+（系统自带）
  macOS: python3（系统自带）、curl、osascript
---

# 飞书 Hook 通知设置技能（跨平台）

## 概述

在 Claude Code 的用户级配置（`~/.claude/settings.json`）中注册 Hook，当 `Stop`、`Notification`、`TaskCreated`、`TaskCompleted` 事件触发时，通过脚本向飞书群机器人 Webhook 发送交互式卡片消息。

### 特性

- **跨平台**：同时支持 Windows（PowerShell）和 macOS/Linux（bash + python3）
- **四种事件通知**：Stop（红色）、Notification（蓝色）、TaskCreated（绿色）、TaskCompleted（绿色）
- **焦点检测**：自动检测 Claude Code 终端是否为前台窗口，主动操作时不发送通知
  - Windows：通过 `user32.dll` Win32 API 检测前台窗口进程链
  - macOS：通过 `osascript` + `ps` 命令检测前台 App 进程链
- **编码**：显式 UTF-8 编码，中文字符正常显示
- **静默失败**：超时 4 秒，不影响 Claude Code

## 工作流程

### Step 1: 获取飞书 Webhook URL

引导用户创建飞书群机器人并获取 Webhook URL：

1. 打开飞书 → 进入一个群聊（或创建新群，如 "Claude Code 通知"）
2. 群设置 → 机器人 → 添加机器人
3. 选择**自定义机器人（Webhook）**
4. 设置机器人名称（如 "Claude Code Hook"）
5. 复制 Webhook URL（格式：`https://open.feishu.cn/open-apis/bot/v2/hook/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`）

> **安全提示**: Webhook URL 包含群聊标识，泄露后任何人可向该群发消息。不要提交到公开仓库。

### Step 2: 检测平台

```bash
uname -s
# Darwin → macOS, Linux → Linux, MINGW*/MSYS* → Windows (Git Bash)
```

根据平台选择对应的脚本和命令路径格式。

### Step 3: 创建脚本目录

```bash
mkdir -p ~/.claude/scripts
```

### Step 4: 写入通知脚本

#### 方案 A：macOS / Linux → `feishu-notify.sh`

创建 `~/.claude/scripts/feishu-notify.sh`：

```bash
#!/bin/bash
# feishu-notify.sh — macOS/Linux Feishu webhook notifier for Claude Code hooks
# Requires: python3 (pre-installed on macOS), curl

WEBHOOK_URL="__WEBHOOK_URL__"

/usr/bin/python3 -c "
import json, os, urllib.request, subprocess, sys
from datetime import datetime

url = '$WEBHOOK_URL'
event = os.environ.get('CLAUDE_HOOK_EVENT', '')
text = os.environ.get('CLAUDE_HOOK_TEXT', '')
ts = datetime.now().strftime('%Y-%m-%d %H:%M:%S')

# Focus detection (macOS: osascript, Linux: /proc)
try:
    if sys.platform == 'darwin':
        fg_pid = int(subprocess.check_output(
            ['osascript', '-e',
             'tell application \"System Events\" to get unix id of first application process whose frontmost is true']
        ).strip())
    elif sys.platform == 'linux':
        import re
        # xdotool or wmctrl for Linux
        try:
            fg_pid = int(subprocess.check_output(['xdotool', 'getactivewindow', 'getwindowpid']).strip())
        except:
            fg_pid = 0
    else:
        fg_pid = 0

    cursor = os.getpid()
    for _ in range(20):
        if sys.platform == 'linux':
            try:
                with open(f'/proc/{cursor}/status') as f:
                    for line in f:
                        if line.startswith('PPid:'):
                            ppid = int(line.split()[1])
                            break
            except:
                break
        else:
            try:
                ppid = int(subprocess.check_output(['ps', '-o', 'ppid=', '-p', str(cursor)]).strip())
            except:
                break
        if ppid in (0, cursor): break
        cursor = ppid
        if fg_pid and cursor == fg_pid: sys.exit(0)
except:
    pass

# Build card content
titles_map = {
    'Stop': ('Claude Code 已停止', 'red'),
    'Notification': ('Claude Code 通知', 'blue'),
    'TaskCreated': ('Claude Code 任务已创建', 'green'),
    'TaskCompleted': ('Claude Code 任务已完成', 'green'),
}
header_title, template = titles_map.get(event, ('Claude Code Hook', 'grey'))
label = event or '未知'

detail = f'**事件**: {label}\\n**时间**: {ts}'
if text:
    key = {'Stop': '原因', 'Notification': '内容', 'TaskCreated': '描述', 'TaskCompleted': '结果'}.get(event, '内容')
    detail += f'\\n**{key}**: {text}'

payload = json.dumps({
    'msg_type': 'interactive',
    'card': {
        'config': {'wide_screen_mode': True},
        'header': {'title': {'tag': 'plain_text', 'content': header_title}, 'template': template},
        'elements': [{'tag': 'div', 'text': {'tag': 'lark_md', 'content': detail}}]
    }
}).encode('utf-8')

req = urllib.request.Request(url, data=payload,
    headers={'Content-Type': 'application/json; charset=utf-8'}, method='POST')
urllib.request.urlopen(req, timeout=4).close()
" 2>/dev/null
```

然后赋予执行权限并替换 URL：

```bash
chmod +x ~/.claude/scripts/feishu-notify.sh
# 编辑替换 __WEBHOOK_URL__ 为实际的 Webhook URL
sed -i '' 's|__WEBHOOK_URL__|https://open.feishu.cn/open-apis/bot/v2/hook/你的URL|' ~/.claude/scripts/feishu-notify.sh
```

#### 方案 B：Windows → `feishu-notify.ps1`

创建 `~/.claude/scripts/feishu-notify.ps1`：

```powershell
param()

$webhookUrl = "WEBHOOK_URL_HERE"
$eventName = $env:CLAUDE_HOOK_EVENT
$hookText = $env:CLAUDE_HOOK_TEXT
$timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"

# Focus detection: skip if Claude Code terminal is foreground window
Add-Type -Name Win32 -Namespace Focus -MemberDefinition @'
[DllImport("user32.dll")]
public static extern IntPtr GetForegroundWindow();
[DllImport("user32.dll")]
public static extern uint GetWindowThreadProcessId(IntPtr hWnd, out uint pid);
'@ -ErrorAction SilentlyContinue

$fgPid = 0
try {
    $fgWindow = [Focus.Win32]::GetForegroundWindow()
    [Focus.Win32]::GetWindowThreadProcessId($fgWindow, [ref]$fgPid) | Out-Null
    $cursorPid = $pid
    $isFocused = $false
    for ($i = 0; $i -lt 15; $i++) {
        $proc = Get-CimInstance Win32_Process -Filter "ProcessId = $cursorPid" -ErrorAction SilentlyContinue
        if (-not $proc) { break }
        $parentPid = $proc.ParentProcessId
        if ($parentPid -eq 0 -or $parentPid -eq $cursorPid) { break }
        $cursorPid = $parentPid
        if ($cursorPid -eq $fgPid) { $isFocused = $true; break }
    }
    if ($isFocused) { exit 0 }
} catch { /* fail open: send notification */ }

# Collect all available CLAUDE_HOOK_* environment variables
$envVars = Get-ChildItem Env: | Where-Object { $_.Name -like "CLAUDE_HOOK_*" }
$extraDetails = ""
foreach ($var in $envVars) {
    if ($var.Name -ne "CLAUDE_HOOK_EVENT" -and $var.Value) {
        $extraDetails += "`n**$($var.Name)**: $($var.Value)"
    }
}

$eventLabel = if ($eventName) { $eventName } else { "未知" }

switch -Wildcard ($eventName) {
    "Stop" {
        $headerTitle = "Claude Code 已停止"
        $template = "red"
        $detail = "**事件**: $eventLabel`n**时间**: $timestamp"
        if ($hookText) { $detail += "`n**原因**: $hookText" }
    }
    "Notification" {
        $headerTitle = "Claude Code 通知"
        $template = "blue"
        $detail = "**事件**: $eventLabel`n**时间**: $timestamp"
        if ($hookText) { $detail += "`n**内容**: $hookText" }
    }
    "TaskCreated" {
        $headerTitle = "Claude Code 任务已创建"
        $template = "green"
        $detail = "**事件**: $eventLabel`n**时间**: $timestamp"
        if ($hookText) { $detail += "`n**描述**: $hookText" }
    }
    "TaskCompleted" {
        $headerTitle = "Claude Code 任务已完成"
        $template = "green"
        $detail = "**事件**: $eventLabel`n**时间**: $timestamp"
        if ($hookText) { $detail += "`n**结果**: $hookText" }
    }
    default {
        $headerTitle = "Claude Code Hook"
        $template = "grey"
        $detail = "**事件**: $eventLabel`n**时间**: $timestamp"
        if ($hookText) { $detail += "`n**内容**: $hookText" }
    }
}

if ($extraDetails) { $detail += $extraDetails }

$body = @{
    msg_type = "interactive"
    card = @{
        config = @{ wide_screen_mode = $true }
        header = @{
            title = @{ tag = "plain_text"; content = $headerTitle }
            template = $template
        }
        elements = @(
            @{ tag = "div"; text = @{ tag = "lark_md"; content = $detail } }
        )
    }
} | ConvertTo-Json -Depth 10

try {
    $utf8Bytes = [System.Text.Encoding]::UTF8.GetBytes($body)
    $webRequest = [System.Net.WebRequest]::Create($webhookUrl)
    $webRequest.Method = "POST"
    $webRequest.ContentType = "application/json; charset=utf-8"
    $webRequest.ContentLength = $utf8Bytes.Length
    $webRequest.Timeout = 4000
    $stream = $webRequest.GetRequestStream()
    $stream.Write($utf8Bytes, 0, $utf8Bytes.Length)
    $stream.Close()
    $webRequest.GetResponse() | Out-Null
} catch { /* silently fail */ }
```

### Step 5: 配置 settings.json hooks

#### macOS / Linux

在 `~/.claude/settings.json` 的 `hooks` 字段添加：

```json
{
  "hooks": {
    "Notification": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "bash ~/.claude/scripts/feishu-notify.sh",
            "timeout": 5
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "bash ~/.claude/scripts/feishu-notify.sh",
            "timeout": 5
          }
        ]
      }
    ],
    "TaskCreated": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "bash ~/.claude/scripts/feishu-notify.sh",
            "timeout": 5
          }
        ]
      }
    ],
    "TaskCompleted": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "bash ~/.claude/scripts/feishu-notify.sh",
            "timeout": 5
          }
        ]
      }
    ]
  }
}
```

#### Windows

```json
{
  "hooks": {
    "Notification": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "powershell -NoProfile -NonInteractive -File \"C:\\Users\\Admin\\.claude\\scripts\\feishu-notify.ps1\"",
            "timeout": 5
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "powershell -NoProfile -NonInteractive -File \"C:\\Users\\Admin\\.claude\\scripts\\feishu-notify.ps1\"",
            "timeout": 5
          }
        ]
      }
    ],
    "TaskCreated": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "powershell -NoProfile -NonInteractive -File \"C:\\Users\\Admin\\.claude\\scripts\\feishu-notify.ps1\"",
            "timeout": 5
          }
        ]
      }
    ],
    "TaskCompleted": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "powershell -NoProfile -NonInteractive -File \"C:\\Users\\Admin\\.claude\\scripts\\feishu-notify.ps1\"",
            "timeout": 5
          }
        ]
      }
    ]
  }
}
```

> **JSON 转义注意**: Windows 路径中的反斜杠在 JSON 中需写为 `\\`（如 `C:\\Users\\Admin\\.claude\\`）。也可用正斜杠 `/`（`C:/Users/Admin/.claude/`）。

#### hooks 配置结构说明

`settings.json` 中 `hooks` 的值结构如下：

```json
{
  "hooks": {
    "事件名": [
      {
        "matcher": "",    // 可选，工具名过滤（如 "Edit|Write"），事件级 Hook 留空
        "hooks": [
          {
            "type": "command",
            "command": "...",   // 要执行的命令
            "timeout": 5        // 超时秒数
          }
        ]
      }
    ]
  }
}
```

- 每个事件名映射到一个**数组**（不是对象）
- `matcher` 可选，用于工具类事件（如 `PostToolUse`），事件级 Hook 无需设置
- `timeout` 单位是**秒**（不是毫秒）

### Step 6: 测试通知

#### macOS / Linux

```bash
CLAUDE_HOOK_EVENT="Notification" CLAUDE_HOOK_TEXT="Hook 配置完成 - 测试消息" bash ~/.claude/scripts/feishu-notify.sh
```

#### Windows

```bash
CLAUDE_HOOK_EVENT="Notification" CLAUDE_HOOK_TEXT="Hook 配置完成 - 测试消息" powershell -NoProfile -NonInteractive -File ~/.claude/scripts/feishu-notify.ps1
```

### Step 7: 验证焦点检测

通过 `AskUserQuestion` 触发 `Stop` Hook 事件。在当前终端操作时不应收到通知，切换到其他窗口时才应收到。

## 事件与卡片样式对照

| 事件 | 触发时机 | 卡片颜色 | 中文标题 |
|------|---------|---------|---------|
| `Stop` | Claude Code 停止运行 | 红色 | Claude Code 已停止 |
| `Notification` | Claude 发送通知 | 蓝色 | Claude Code 通知 |
| `TaskCreated` | 任务被创建 | 绿色 | Claude Code 任务已创建 |
| `TaskCompleted` | 任务完成 | 绿色 | Claude Code 任务已完成 |
| 其他事件 | 未匹配 | 灰色 | Claude Code Hook |

## 焦点检测原理

### Windows

1. 通过 `user32.dll` 的 `GetForegroundWindow()` 获取当前前台窗口
2. 通过 `GetWindowThreadProcessId()` 获取前台窗口的进程 PID
3. 从 Hook 脚本（PowerShell）的 PID 向上遍历最多 15 层父进程链
4. 如果前台窗口 PID 匹配进程链中任意节点，判定为前台 → 跳过通知

### macOS / Linux

1. macOS 通过 `osascript` 获取前台 App 的 Unix PID
2. Linux 通过 `xdotool` 获取当前活跃窗口的进程 PID
3. 从脚本 PID 向上遍历最多 20 层父进程链（通过 `ps -o ppid=` 或 `/proc`）
4. 匹配则跳过通知
5. 失败时默认放行（发送通知）

## 常见问题

### 中文显示 `??`

**原因**：HTTP 请求未使用 UTF-8 编码。

**解决**：
- Windows：脚本已使用 `[System.Text.Encoding]::UTF8.GetBytes()` + `charset=utf-8`
- macOS/Linux：Python `json.dumps().encode('utf-8')` + header `Content-Type: application/json; charset=utf-8`

### 焦点检测不准确

**原因**：终端进程结构复杂（多标签、WSL、tmux 等）。

**解决**：
- 调整遍历深度（`for ($i = 0; $i -lt 15; $i++)` 或 `for _ in range(20)`）
- 或直接移除焦点检测：删除脚本中 `# Focus detection` 注释到 `exit 0` 之间的代码

### 脚本不执行 / 无反应

- 检查 `~/.claude/settings.json` 格式是否正确（可用 `jsonlint` 或 `python3 -m json.tool` 验证）
- 检查脚本路径是否存在、权限是否正确（macOS 需要 `chmod +x`）
- Hook `timeout` 是否足够（建议 5 秒）

### 消息杂乱 / 想减少通知事件

在 `settings.json` 中删除不需要的事件即可：

```json
"hooks": {
    "Stop": [...],          // 保留
    // "Notification": [],  // 删掉此行
    "TaskCompleted": [...]  // 保留
}
```

## 安全提示

- **Webhook URL 不要提交到公开仓库**
- 可在 `.gitignore` 中排除：`scripts/feishu-notify.*`
- 飞书自定义机器人 Webhook 没有加密，建议仅在内部群使用
- 需要更高的安全性可使用飞书开放 API（需 App ID + App Secret）

## 关键提醒

- 此配置作用于**用户级 scope**（`~/.claude/settings.json`），不影响项目级配置
- `disableAllHooks: true` 可一键禁用所有 Hook
- 四种 Hook 事件共用同一个脚本，也可为不同事件配置不同脚本
- 脚本修改后**立即生效**，无需重启 Claude Code
