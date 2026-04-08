# MemPalace Examples - AI 上下文文档

> [根目录](../CLAUDE.md) > **examples/**

**最后更新**：2026-04-08 22:47:00

## 模块概述

`examples/` 目录包含 MemPalace 的使用示例和集成指南。这些文件展示了如何将 MemPalace 集成到各种 AI 工具和工作流中。

**包含内容**：
- 基础使用示例（Python 脚本）
- CLI 工具集成指南（Gemini CLI、Claude Code）
- MCP 服务器配置
- Hook 自动保存教程

---

## 文件结构

```
examples/
├── basic_mining.py          # 基础项目采集示例
├── convo_import.py          # 对话导入示例
├── gemini_cli_setup.md      # Gemini CLI 集成指南
├── mcp_setup.md             # MCP 服务器配置
└── HOOKS_TUTORIAL.md        # Hook 自动保存教程
```

---

## Python 示例脚本

### basic_mining.py

**用途**：展示如何将项目文件夹采集到记忆宫殿

**使用场景**：
- 首次使用 MemPalace
- 想让 AI 学习项目代码库
- 建立项目知识基础

**命令流程**：
```bash
# 步骤 1：从文件夹结构初始化房间
mempalace init ~/projects/my_app

# 步骤 2：采集所有内容
mempalace mine ~/projects/my_app

# 步骤 3：搜索
mempalace search '为什么选择这种方法'
```

### convo_import.py

**用途**：展示如何导入 Claude Code / ChatGPT 对话

**使用场景**：
- 迁移历史对话到 MemPalace
- 让 AI 学习过去的讨论
- 建立对话历史知识库

**命令示例**：
```bash
# 导入 Claude Code 会话
mempalace mine ~/claude-sessions/ --mode convos --wing my_project

# 导入 ChatGPT 导出
mempalace mine ~/chatgpt-exports/ --mode convos

# 使用通用提取器进行更丰富的提取
mempalace mine ~/chats/ --mode convos --extract general
```

---

## 集成指南

### gemini_cli_setup.md

**用途**：完整的 Gemini CLI 集成指南

**包含内容**：
- 前置要求（Python 3.9+、Gemini CLI）
- 安装步骤（虚拟环境、包安装）
- 初始化配置（Palace 设置、Identity 和 Wings）
- MCP 连接配置
- Hook 自动保存设置
- 使用示例和验证方法

**适用场景**：
- 使用 Gemini CLI 作为主要 AI 工具
- 需要持久化记忆的 Gemini CLI 用户
- 想在 Gemini CLI 中使用 MemPalace MCP 工具

**关键命令**：
```bash
# MCP 注册
gemini mcp add mempalace /absolute/path/to/.venv/bin/python3 -m mempalace.mcp_server --scope user

# 验证连接
/mcp list  # 在 Gemini CLI 会话中
/hooks panel  # 验证 Hook 激活
```

### mcp_setup.md

**用途**：MCP 服务器快速配置

**包含内容**：
- MCP 服务器启动方法
- Claude Code 集成命令
- 可用工具列表（status、search、list_wings）
- Claude Code 中的使用说明

**适用场景**：
- 快速集成 MCP 到 Claude Code
- 了解可用的 MCP 工具
- 验证 MCP 连接

**关键命令**：
```bash
# 启动 MCP 服务器
python -m mempalace.mcp_server

# 添加到 Claude Code
claude mcp add mempalace -- python -m mempalace.mcp_server
```

**可用工具**：
- `mempalace_status` — 宫殿统计（翅膀、房间、抽屉数量）
- `mempalace_search` — 所有记忆的语义搜索
- `mempalace_list_wings` — 列出宫殿中的所有项目

### HOOKS_TUTORIAL.md

**用途**：Hook 自动保存功能简明教程

**包含内容**：
- Hook 类型说明（Save Hook、PreCompact Hook）
- Claude Code 配置示例
- JSON 配置格式

**适用场景**：
- 快速了解 Hook 功能
- 配置自动保存
- 防止对话上下文丢失

**配置示例**：
```json
{
  "hooks": {
    "Stop": [
      {
        "matcher": "",
        "hooks": [{"type": "command", "command": "./hooks/mempal_save_hook.sh"}]
      }
    ],
    "PreCompact": [
      {
        "matcher": "",
        "hooks": [{"type": "command", "command": "./hooks/mempal_precompact_hook.sh"}]
      }
    ]
  }
}
```

---

## 使用场景

### 场景 1：首次设置 MemPalace

1. 阅读 `gemini_cli_setup.md` 或 `mcp_setup.md`（根据您的 AI 工具）
2. 运行 `basic_mining.py` 中的命令初始化宫殿
3. 采集您的项目到宫殿
4. 配置 Hook 自动保存

### 场景 2：导入历史对话

1. 使用 `convo_import.py` 中的命令
2. 导入 Claude Code 或 ChatGPT 导出
3. 指定正确的 `--wing` 组织对话
4. 使用 `--extract general` 获得更丰富的提取

### 场景 3：集成到 AI 工具

1. **Gemini CLI**：按照 `gemini_cli_setup.md` 完整指南
2. **Claude Code**：使用 `mcp_setup.md` 快速配置
3. **其他工具**：参考 MCP 集成模式

### 场景 4：配置自动保存

1. 阅读 `HOOKS_TUTORIAL.md` 了解 Hook 功能
2. 配置 Save Hook（每 N 条消息保存）
3. 配置 PreCompact Hook（压缩前保存）
4. 测试 Hook 是否正常工作

---

## 相关文档

- [hooks/CLAUDE.md](../hooks/CLAUDE.md) - Hook 详细技术文档
- [mempalace/CLAUDE.md](../mempalace/CLAUDE.md) - 核心包文档
- [CLAUDE.md](../CLAUDE.md) - 项目根文档

---

## 快速参考

| 任务 | 示例文件 | 关键命令 |
|------|----------|----------|
| 初始化宫殿 | basic_mining.py | `mempalace init <dir>` |
| 采集项目 | basic_mining.py | `mempalace mine <dir>` |
| 导入对话 | convo_import.py | `mempalace mine <dir> --mode convos` |
| Gemini CLI | gemini_cli_setup.md | `gemini mcp add mempalace ...` |
| Claude Code | mcp_setup.md | `claude mcp add mempalace ...` |
| 自动保存 | HOOKS_TUTORIAL.md | 配置 hooks/ 目录中的脚本 |
