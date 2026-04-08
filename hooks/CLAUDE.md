# MemPalace Hooks - AI 上下文文档

> [根目录](../CLAUDE.md) > **hooks/**

**最后更新**：2026-04-08 22:46:00

## 模块概述

`hooks/` 目录包含两个 Shell 脚本，为 AI 终端工具（Claude Code、Codex CLI）提供自动保存功能。这些 Hook 让 MemPalace 在对话过程中自动捕获重要信息，无需手动保存命令。

**核心价值**：
- **零手动干预**：AI 自动保存关键话题、决策、引用和代码
- **智能分类**：AI 理解对话上下文，将记忆组织到合适的翅膀/大厅/衣柜
- **安全网**：PreCompact Hook 确保上下文压缩前保存所有内容

---

## Hook 类型

| Hook | 触发时机 | 行为 |
|------|----------|------|
| **Save Hook** | 每 N 条用户消息（默认 15） | 阻止 AI 停止，要求保存关键内容 |
| **PreCompact Hook** | 上下文压缩前 | 强制保存所有内容，防止上下文丢失 |

---

## 文件结构

```
hooks/
├── mempal_save_hook.sh       # Save Hook - 定期自动保存
├── mempal_precompact_hook.sh # PreCompact Hook - 压缩前紧急保存
└── README.md                 # 人类可读的详细文档
```

---

## 工作原理

### Save Hook（Stop 事件）

```
用户发送消息 → AI 响应 → Claude Code 触发 Stop Hook
                                            ↓
                        Hook 统计 JSONL 转录中的人类消息数
                                            ↓
              ┌─── < 15 条（自上次保存）──→ echo "{}"（让 AI 停止）
              │
              └─── ≥ 15 条 ──→ {"decision": "block", "reason": "save..."}
                                                ↓
                                        AI 保存到记忆宫殿
                                                ↓
                                        AI 尝试再次停止
                                                ↓
                                        stop_hook_active = true
                                                ↓
                                    Hook 看到标志 → echo "{}"（放行）
```

**`stop_hook_active` 标志**：防止无限循环。阻止一次 → AI 保存 → 尝试停止 → 标志为 true → 放行。

### PreCompact Hook

```
上下文窗口即将满 → Claude Code 触发 PreCompact
                            ↓
                    Hook 总是阻止
                            ↓
                    AI 保存所有内容
                            ↓
                    压缩继续进行
```

**无需计数**：压缩总是值得保存。

---

## 配置

### 可调参数

编辑 `mempal_save_hook.sh`：

- **`SAVE_INTERVAL=15`** — 保存间隔（用户消息数）。越低 = 保存越频繁，越高 = 打扰越少
- **`STATE_DIR`** — Hook 状态存储位置（默认 `~/.mempalace/hook_state/`）
- **`MEMPAL_DIR`** — 可选。设置为对话目录以在每次触发时自动运行 `mempalace mine <dir>`。留空（默认）让 AI 通过阻止消息处理保存

### 安装 - Claude Code

添加到 `.claude/settings.local.json`：

```json
{
  "hooks": {
    "Stop": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "/absolute/path/to/hooks/mempal_save_hook.sh",
        "timeout": 30
      }]
    }],
    "PreCompact": [{
      "hooks": [{
        "type": "command",
        "command": "/absolute/path/to/hooks/mempal_precompact_hook.sh",
        "timeout": 30
      }]
    }]
  }
}
```

设置可执行权限：
```bash
chmod +x hooks/mempal_save_hook.sh hooks/mempal_precompact_hook.sh
```

### 安装 - Codex CLI (OpenAI)

添加到 `.codex/hooks.json`：

```json
{
  "Stop": [{
    "type": "command",
    "command": "/absolute/path/to/hooks/mempal_save_hook.sh",
    "timeout": 30
  }],
  "PreCompact": [{
    "type": "command",
    "command": "/absolute/path/to/hooks/mempal_precompact_hook.sh",
    "timeout": 30
  }]
}
```

---

## MemPalace CLI 集成

相关命令：

```bash
mempalace mine <dir>               # 采集目录中的所有文件
mempalace mine <dir> --mode convos # 仅采集对话转录
```

Hook 自动从其自身路径解析仓库根目录，因此无论您将仓库安装在何处都能工作。

---

## 调试

检查 Hook 日志：
```bash
cat ~/.mempalace/hook_state/hook.log
```

示例输出：
```
[14:30:15] Session abc123: 12 次交互，12 次自上次保存
[14:35:22] Session abc123: 15 次交互，15 次自上次保存
[14:35:22] 在第 15 次交互时触发保存
[14:40:01] Session abc123: 18 次交互，3 次自上次保存
```

---

## 成本

**零额外 Token**。Hook 是本地运行的 Bash 脚本，不调用任何 API。唯一的"成本"是 AI 在每个检查点花费几秒钟组织记忆——而且它使用已经加载的上下文来执行此操作。

---

## 技术细节

### Claude Code JSON 输入格式

**Save Hook 输入**：
```json
{
  "session_id": "abc123",
  "stop_hook_active": false,
  "transcript_path": "/path/to/session.jsonl"
}
```

**PreCompact Hook 输入**：
```json
{
  "session_id": "abc123"
}
```

### Hook 输出格式

**放行**：
```json
{}
```

**阻止**：
```json
{
  "decision": "block",
  "reason": "AUTO-SAVE checkpoint. Save key topics, decisions, quotes, and code from this session to your memory system..."
}
```

### 安全措施

- **路径清理**：SESSION_ID 经过清理，仅允许字母数字、连字符和下划线
- **路径展开**：`~` 在路径中被正确展开为 $HOME
- **错误处理**：Python 解析失败时有后备默认值

---

## 故障排除

| 问题 | 解决方案 |
|------|----------|
| Hook 未触发 | 检查 `.claude/settings.local.json` 或 `.codex/hooks.json` 中的路径是否正确 |
| Hook 无限阻止 | 检查 `stop_hook_active` 逻辑，确保第二次调用时放行 |
| 日志未写入 | 确保 `STATE_DIR` 目录可写 |
| AI 不保存 | 检查 `reason` 消息是否清晰，AI 是否理解保存指令 |

---

## 相关文档

- [README.md](./README.md) - 人类可读的详细文档
- [mempalace/CLAUDE.md](../mempalace/CLAUDE.md) - 核心包文档
- [examples/CLAUDE.md](../examples/CLAUDE.md) - 使用示例文档
