# claude-code-hooks

Claude Code 完成任务后自动通过 Telegram 通知 OpenClaw 主 agent 的回调机制。

## 工作原理

通过 Claude Code 的 Stop Hook 机制，在每次任务完成时自动触发通知脚本，将任务结果推送给 OpenClaw 主 agent。

```
dispatch-claude-code.sh
       │
       ├─ 写入 task-meta.json（任务名、目标群组）
       ├─ 启动 Claude Code（via claude_code_run.py）
       │   └─ Claude Code 执行任务
       │
       └─ 任务完成 → Stop Hook 自动触发
               │
               └─ notify-agi.sh 执行：
                   ├─ 读取 task-meta.json + task-output.txt
                   ├─ 写入 latest.json（完整结果）
                   ├─ openclaw message send → Telegram 群
                   └─ 写入 pending-wake.json（供 AGI 心跳读取）
```

### 防重复机制

Stop 和 SessionEnd 事件均会触发 Hook。脚本使用 `.hook-lock` 文件去重：30 秒内重复触发自动跳过，只处理第一个事件（通常是 Stop）。

## 安装步骤

### 1. 复制 Hook 脚本

```bash
mkdir -p ~/.claude/hooks/
cp hooks/notify-agi.sh ~/.claude/hooks/
chmod +x ~/.claude/hooks/notify-agi.sh
```

### 2. 配置 Claude Code Settings

将 `hooks/claude-settings.json` 的内容合并到 `~/.claude/settings.json`（或直接覆盖）：

```bash
cp hooks/claude-settings.json ~/.claude/settings.json
```

`settings.json` 注册的 Hook 配置如下：

```json
{
  "hooks": {
    "Stop": [{"hooks": [{"type": "command", "command": "~/.claude/hooks/notify-agi.sh", "timeout": 10}]}],
    "SessionEnd": [{"hooks": [{"type": "command", "command": "~/.claude/hooks/notify-agi.sh", "timeout": 10}]}]
  }
}
```

### 3. 确认路径（dash001 用户）

脚本已适配 `dash001` 用户，以下路径已预设：

| 路径 | 说明 |
|------|------|
| `~/.claude/hooks/notify-agi.sh` | Stop Hook 脚本 |
| `/home/dash001/.openclaw/workspace/data/claude-code-results/` | 结果输出目录 |
| `/home/dash001/.npm-global/bin/openclaw` | openclaw CLI 路径 |

如需修改，编辑 `hooks/notify-agi.sh` 顶部的变量定义。

## 使用方法

### 基础任务派发

```bash
bash scripts/dispatch-claude-code.sh \
  -p "实现一个 Python 爬虫" \
  -n "my-scraper" \
  -g "目标TelegramGroupID" \
  --permission-mode "bypassPermissions" \
  --workdir "/path/to/project"
```

### 参数说明

| 参数 | 说明 |
|------|------|
| `-p, --prompt` | 任务提示（必填）|
| `-n, --name` | 任务名称（用于跟踪）|
| `-g, --group` | Telegram 群组 ID（结果自动发送）|
| `-w, --workdir` | Claude Code 工作目录 |
| `--permission-mode` | 权限模式（如 `bypassPermissions`）|
| `--allowed-tools` | 允许的工具列表 |
| `--model` | 覆盖默认模型 |

### 结果文件

任务完成后，结果写入：

```
/home/dash001/.openclaw/workspace/data/claude-code-results/latest.json
```

文件格式：

```json
{
  "session_id": "...",
  "timestamp": "2026-03-06T12:00:00+11:00",
  "task_name": "my-scraper",
  "telegram_group": "...",
  "output": "...",
  "status": "done"
}
```

## 目录结构

```
claude-code-hooks/
├── README.md                          # 本文件
├── claude-settings.json               # Claude Code 配置（备用）
├── hooks/
│   ├── notify-agi.sh                  # Stop Hook 脚本 → 复制到 ~/.claude/hooks/
│   └── claude-settings.json           # Claude Code settings → 复制到 ~/.claude/
└── scripts/
    ├── dispatch-claude-code.sh        # 一键派发任务脚本
    ├── claude_code_run.py             # Claude Code PTY 运行器
    └── run-claude-code.sh             # 备用运行脚本
```

## 注意事项

- **不要在脚本中硬编码 token 或群组 ID**：群组 ID 通过 `-g` 参数传入，bot token 由 openclaw 配置文件管理。
- **防重复锁文件**：`.hook-lock` 文件位于结果目录，30 秒内重复触发自动跳过；如需强制重新触发，删除该文件即可。
- **openclaw CLI 路径**：脚本默认使用 `/home/dash001/.npm-global/bin/openclaw`，若 openclaw 安装位置不同，请修改 `notify-agi.sh` 中的 `OPENCLAW_BIN` 变量。
- **结果目录自动创建**：`notify-agi.sh` 会自动创建 `data/claude-code-results/`，无需手动创建。
