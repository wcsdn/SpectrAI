# SpectrAI 架构说明

## 系统架构

```
┌─────────────────────────────────────────┐
│         SpectrAI (Electron 应用)         │
│  - 会话管理                              │
│  - 多 AI 工具启动器                      │
│  - UI 界面                               │
└──────────────┬──────────────────────────┘
               │ 启动脚本
               │ start-minimax.sh
               ↓
┌─────────────────────────────────────────┐
│      Claude Code (AI 编程引擎)           │
│  - MCP 工具系统                          │
│  - 代码编辑逻辑                          │
│  - 文件操作能力                          │
│  - AI 对话流程                           │
└──────────────┬──────────────────────────┘
               │ API 调用
               │ (已修改为 MiniMax)
               ↓
┌─────────────────────────────────────────┐
│         MiniMax API                      │
│  - 大语言模型服务                        │
└──────────────┬──────────────────────────┘
               │ 返回响应
               ↓
┌─────────────────────────────────────────┐
│      工作目录 (你的项目)                 │
│  - SpectrAI 项目文件                     │
│  - 或其他任意项目                        │
└─────────────────────────────────────────┘
```

## 为什么需要 Claude Code？

### SpectrAI 的定位
- **不是** AI 编程助手本身
- **是** 多 AI 工具的管理器和启动器
- 提供统一的界面和会话管理

### Claude Code 的作用
- 提供完整的 AI 编程能力
- 实现 MCP (Model Context Protocol) 工具系统
- 处理代码编辑、文件操作等复杂逻辑
- 管理 AI 对话流程

### 为什么不直接接 MiniMax？
如果 SpectrAI 直接接 MiniMax，需要：
1. 重新实现整套 MCP 工具系统
2. 重写代码编辑和文件操作逻辑
3. 实现 AI 对话流程管理
4. 维护大量底层代码

**现在的方案**：借用 Claude Code 的能力，只替换 API 层，工作量小得多。

## 工作流程

### 1. 启动流程

1. 用户在 SpectrAI 中创建会话
2. 选择工作目录（如 `/Users/a12345/all-workspace/SpectrAI`）
3. SpectrAI 调用 `start-minimax.sh` 脚本
4. 脚本启动 Claude Code，传递工作目录参数
5. Claude Code 在指定工作目录下运行

### 2. 执行流程

用户输入 → SpectrAI 界面 → Claude Code 处理 → 调用 MiniMax API → 返回结果 → Claude Code 执行工具操作 → 修改工作目录文件

### 3. 工具执行

当 AI 需要操作文件时：
- MCP filesystem 工具在**工作目录**下执行
- 操作的是 SpectrAI 项目（或你选择的项目）的文件
- **不是** Claude Code 自己的代码

## 关键配置

### start-minimax.sh 脚本

**重要**：脚本不能切换工作目录，否则会覆盖 SpectrAI 传递的工作目录参数。

正确写法：
```bash
#!/bin/bash

# 获取项目根目录路径（但不切换）
CLAUDE_CODE_DIR="$(cd "$(dirname "$0")/.." && pwd)"

# 加载 MiniMax 配置
if [ -f "$CLAUDE_CODE_DIR/.env.minimax" ]; then
  export $(cat "$CLAUDE_CODE_DIR/.env.minimax" | grep -v '^#' | xargs)
fi

# 启动 Claude Code（保持当前工作目录）
exec bun "$CLAUDE_CODE_DIR/src/entrypoints/dev-cli.tsx" "$@"
```

错误写法（会导致工作目录错误）：
```bash
cd "$(dirname "$0")/.."  # ❌ 不要这样做！
```

### SpectrAI 数据库

会话配置存储在：
```
~/Library/Application Support/spectrai/claudeops.db
```

包含：
- 工作目录路径
- 启动脚本路径
- 会话配置

## 常见问题

### Q: 为什么 pwd 显示错误的目录？
A: 检查 `start-minimax.sh` 脚本是否有 `cd` 命令，删除它。

### Q: 修改脚本后需要重新编译吗？
A: 不需要，只需在 SpectrAI 中重新创建会话。

### Q: MCP 工具操作的是哪个项目？
A: 操作的是你在 SpectrAI 中选择的工作目录，不是 Claude Code 项目本身。

### Q: 能否让 SpectrAI 直接接 MiniMax？
A: 理论上可以，但需要重新实现 Claude Code 的所有功能，工作量巨大。现在的方案是"借壳"，更高效。

## 配置隔离

### SpectrAI 独立配置目录

SpectrAI 使用自己独立的配置目录，不干扰系统 Claude：

```
~/.spectrai/
├── projects/           # 项目对话数据
│   └── <projectHash>/
│       └── <conversationId>.jsonl
├── mcp/               # MCP 配置文件
│   └── mcp-config-<sessionId>.json
└── custom-rules.json  # 自定义解析规则
```

数据库位置：
```
~/Library/Application Support/spectrai/claudeops.db
```

环境变量：
```bash
CLAUDE_HOME=~/.spectrai  # SpectrAI 启动 Claude Code 时自动设置
```

### 数据隔离对比

| 组件 | SpectrAI | 系统 Claude |
|------|----------|-------------|
| 配置目录 | `~/.spectrai/` | `~/.claude/` |
| 项目数据 | `~/.spectrai/projects/` | `~/.claude/projects/` |
| 数据库 | `~/Library/Application Support/spectrai/` | `~/Library/Application Support/claude/` |
| MCP 配置 | `~/.spectrai/mcp/` | `~/.claude/mcp/` |

✅ 完全隔离，互不干扰！

## 提供者对比

### Claude Code 提供者

**特点**：
- ✅ 完整的 AI 编程助手
- ✅ 可以读写文件
- ✅ 可以执行命令
- ✅ 可以使用 MCP 工具
- ✅ 有自己的 workspace 概念

**配置**：
- 配置目录：`~/.spectrai/`（不是 `~/.claude/`）
- 项目数据：`~/.spectrai/projects/`
- 环境变量：`CLAUDE_HOME=~/.spectrai`

**使用场景**：
- 需要 AI 帮你写代码
- 需要 AI 操作文件系统
- 需要 AI 执行命令
- 需要完整的开发助手功能

### MiniMax 提供者

**特点**：
- ✅ 纯对话 API
- ✅ 快速响应
- ✅ 完全独立
- ❌ **不能读写文件**
- ❌ **不能执行命令**
- ❌ **没有 workspace 概念**

**配置**：
- 配置文件：`.env.minimax`
- API 调用：直接 HTTPS 请求
- 无需 CLI：不依赖任何命令行工具

**使用场景**：
- 纯对话交流
- 快速问答
- 不需要文件操作
- 不需要命令执行

### 如何选择？

| 需求 | 推荐提供者 |
|------|-----------|
| 写代码、改文件 | Claude Code |
| 执行命令 | Claude Code |
| 使用 MCP 工具 | Claude Code |
| 纯对话交流 | MiniMax 或 Claude Code |
| 快速响应 | MiniMax |

## 测试验证

### 验证配置隔离

```bash
# SpectrAI 的配置目录（应该存在）
ls -la ~/.spectrai/

# 系统 Claude 的配置目录（不应该被 SpectrAI 修改）
ls -la ~/.claude/
```

### 验证工作目录

在 SpectrAI 的 Claude Code 会话中执行：

```bash
echo $CLAUDE_HOME  # 应该输出：/Users/a12345/.spectrai
pwd                # 应该输出：你选择的工作目录
```

### 验证数据隔离

创建会话后检查：

```bash
# SpectrAI 的项目数据（应该有新文件）
ls -la ~/.spectrai/projects/

# 系统 Claude 的项目数据（不应该有新文件）
ls -la ~/.claude/projects/
```
