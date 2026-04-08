# Claude Code 用户完整使用手册

> 从用户视角出发的能力全景与操作说明  
> 基于官方文档 + 2026年3月31日源码泄漏内容整合  
> 更新日期：2026年4月7日（v2 — 补充 Feature Flag 完整版 + 重要文件说明）

---

## 目录

1. [快速上手](#1-快速上手)
2. [交互模式总览](#2-交互模式总览)
3. [Slash 命令完整参考](#3-slash-命令完整参考)
4. [键盘快捷键](#4-键盘快捷键)
5. [CLI 命令行参数](#5-cli-命令行参数)
6. [权限模式详解](#6-权限模式详解)
7. [CLAUDE.md 配置指南](#7-claudemd-配置指南)
8. [记忆系统使用指南](#8-记忆系统使用指南)
9. [会话管理](#9-会话管理)
10. [模型选择与别名](#10-模型选择与别名)
11. [MCP 集成与扩展](#11-mcp-集成与扩展)
12. [自定义 Slash 命令](#12-自定义-slash-命令)
13. [Hooks 自动化配置](#13-hooks-自动化配置)
14. [多 Agent 与并行工作流](#14-多-agent-与并行工作流)
15. [CI/CD 与自动化集成](#15-cicd-与自动化集成)
16. [44 个隐藏 Feature Flag 完整解析](#16-44-个隐藏-feature-flag-完整解析)
17. [重要文件与目录完整说明](#17-重要文件与目录完整说明)
18. [企业与团队配置](#18-企业与团队配置)
19. [常见使用场景示例](#19-常见使用场景示例)

---

## 1. 快速上手

### 安装

```bash
npm install -g @anthropic-ai/claude-code
```

### 首次启动

```bash
# 在项目目录中启动
cd your-project
claude
```

### 基本交互

启动后，直接用自然语言描述你想做的事：

```
> 帮我找出 src/auth.ts 中所有的 TODO 注释并修复它们
> 解释一下这个项目的整体架构
> 写一个针对 UserService 的单元测试
> 修复最近一次 git commit 引入的 bug
```

---

## 2. 交互模式总览

Claude Code 支持四种主要交互模式：

| 模式 | 启动方式 | 适用场景 |
|------|----------|----------|
| **交互式 REPL** | `claude` | 日常开发，多轮对话 |
| **单次查询（Headless）** | `claude -p "..."` | 脚本集成，CI/CD |
| **管道输入** | `cat file \| claude -p "..."` | 处理文件/命令输出 |
| **会话恢复** | `claude --continue` | 继续上次工作 |

---

## 3. Slash 命令完整参考

在交互式 REPL 中，输入 `/` 开头的命令来控制 Claude Code 的行为。

### 3.1 会话控制命令

| 命令 | 说明 | 示例 |
|------|------|------|
| `/clear` | 清除当前对话上下文，开始新会话 | `/clear` |
| `/compact [指令]` | 压缩上下文，可附加保留指令 | `/compact 保留所有关于认证模块的讨论` |
| `/exit` 或 `/quit` | 退出 Claude Code | `/exit` |

### 3.2 配置与设置命令

| 命令 | 说明 |
|------|------|
| `/config` | 打开配置界面，可视化修改所有设置项 |
| `/keybindings` | 管理自定义键盘快捷键，生成 `~/.claude/keybindings.json` |
| `/permissions` | 查看和管理当前会话的工具权限规则 |
| `/model` | 切换 AI 模型（opus / sonnet / haiku 及其变体） |

### 3.3 模型与能力命令

| 命令 | 说明 |
|------|------|
| `/model opus` | 切换到 Claude Opus（最强推理，适合复杂任务） |
| `/model sonnet` | 切换到 Claude Sonnet（均衡性能，日常首选） |
| `/model haiku` | 切换到 Claude Haiku（最快速度，简单任务） |
| `/model opusplan` | 规划用 Opus，执行用 Sonnet（成本优化模式） |
| `/model sonnet[1m]` | 使用 100万 token 上下文窗口版本 |

### 3.4 记忆与上下文命令

| 命令 | 说明 |
|------|------|
| `/memory` | 查看和编辑当前项目的 Auto Memory 内容 |
| `/compact` | 手动触发上下文压缩 |

### 3.5 工具与集成命令

| 命令 | 说明 |
|------|------|
| `/mcp` | 查看已连接的 MCP 服务器列表和状态 |
| `/tools` | 列出当前会话中可用的所有工具 |

### 3.6 会话导航命令

| 命令 | 说明 |
|------|------|
| `/teleport` | 在不同设备间传输会话状态 |
| `/remote-env` | 连接到远程开发环境 |
| `/voice` | 启用语音输入模式（2026年3月新增） |

### 3.7 其他命令

| 命令 | 说明 |
|------|------|
| `/help` | 显示帮助信息和命令列表 |
| `/status` | 显示当前会话状态（token 使用量、模型等） |
| `/bug` | 报告 Claude Code 的 bug |
| `/doctor` | 检查 Claude Code 安装和配置是否正常 |
| `/login` | 重新登录 Anthropic 账户 |
| `/logout` | 退出登录 |

---

## 4. 键盘快捷键

### 4.1 核心快捷键

| 快捷键 | 功能 |
|--------|------|
| `Esc` | 中断 Claude 当前执行 |
| `Esc` + `Esc`（双击） | 打开会话回溯菜单（Rewind） |
| `Shift` + `Tab` | 循环切换权限模式（default → acceptEdits → plan → auto） |
| `Ctrl` + `C` | 强制退出 |

### 4.2 文本编辑快捷键（Bash 风格）

| 快捷键 | 功能 |
|--------|------|
| `Ctrl` + `A` | 跳到行首 |
| `Ctrl` + `E` | 跳到行尾 |
| `Ctrl` + `W` | 删除前一个单词 |
| `Ctrl` + `U` | 删除到行首 |
| `Ctrl` + `K` | 删除到行尾 |
| `Option` + `F` | 向前移动一个单词 |
| `Option` + `B` | 向后移动一个单词 |

### 4.3 历史与搜索

| 快捷键 | 功能 |
|--------|------|
| `↑` / `↓` | 浏览命令历史 |
| `Ctrl` + `R` | 反向搜索命令历史 |

### 4.4 图片与多媒体

| 快捷键 | 平台 | 功能 |
|--------|------|------|
| `Ctrl` + `V` | 通用 | 粘贴图片（截图直接分析） |
| `Cmd` + `V` | iTerm2 | 粘贴图片 |
| `Alt` + `V` | Windows | 粘贴图片 |

### 4.5 自定义快捷键

通过 `/keybindings` 命令或直接编辑 `~/.claude/keybindings.json`：

```json
{
  "bindings": [
    {
      "key": "ctrl+shift+t",
      "command": "/compact 保留所有测试相关讨论"
    },
    {
      "key": "ctrl+shift+r",
      "command": "/model opusplan"
    }
  ]
}
```

---

## 5. CLI 命令行参数

### 5.1 基础参数

```bash
# 启动交互式会话
claude

# 单次查询（Headless 模式）
claude -p "解释这个函数的作用"
claude --print "解释这个函数的作用"

# 指定模型
claude --model sonnet
claude --model claude-opus-4-6
claude --model opus[1m]   # 使用 1M token 上下文

# 指定输出格式
claude -p "查询" --output-format text        # 默认纯文本
claude -p "查询" --output-format json        # JSON 格式
claude -p "查询" --output-format stream-json # 流式 JSON
```

### 5.2 会话管理参数

```bash
# 继续最近一次会话
claude --continue
claude -c

# 恢复指定会话
claude --resume <session-id>
claude --resume   # 显示会话选择器

# 从文件加载初始提示
claude --init-prompt-file ./prompt.md
```

### 5.3 权限控制参数

```bash
# 指定权限模式
claude --permission-mode default        # 默认：询问确认
claude --permission-mode acceptEdits    # 自动接受文件编辑
claude --permission-mode plan           # 只规划，不执行
claude --permission-mode auto           # AI 分类器自动判断

# 危险模式：跳过所有权限检查（仅在沙箱环境使用！）
claude --dangerously-skip-permissions

# 指定允许/禁止的工具
claude --allowedTools "Read,Edit,Bash"
claude --disallowedTools "Bash"
claude -p "任务" --allowedTools "Read,Write"
```

### 5.4 管道与自动化

```bash
# 管道输入
cat logs.txt | claude -p "解释这些错误"
git diff | claude -p "审查这些变更"
npm run build 2>&1 | claude -p "修复这个构建错误"
curl -s api.example.com/data | claude -p "为这个 JSON 生成 TypeScript 类型"

# 结合 jq 处理 JSON 输出
claude -p "列出所有函数" --output-format json | jq '.functions[]'

# 在 CI 中使用
claude -p "运行测试并修复失败的用例" \
  --allowedTools "Read,Edit,Bash" \
  --dangerously-skip-permissions \
  --output-format json
```

### 5.5 完整参数列表

| 参数 | 简写 | 说明 |
|------|------|------|
| `--print` | `-p` | 单次查询模式，输出后退出 |
| `--output-format` | | 输出格式：text / json / stream-json |
| `--model` | | 指定模型 |
| `--continue` | `-c` | 继续最近会话 |
| `--resume` | | 恢复指定会话 |
| `--permission-mode` | | 权限模式 |
| `--dangerously-skip-permissions` | | 跳过所有权限检查 |
| `--allowedTools` | | 允许使用的工具列表 |
| `--disallowedTools` | | 禁止使用的工具列表 |
| `--system-prompt` | | 自定义系统提示 |
| `--append-system-prompt` | | 追加到系统提示 |
| `--max-turns` | | 最大工具调用轮次 |
| `--verbose` | | 显示详细日志 |
| `--no-stream` | | 禁用流式输出 |
| `--version` | `-v` | 显示版本号 |
| `--help` | `-h` | 显示帮助 |

---

## 6. 权限模式详解

### 6.1 五种权限模式

#### 🔵 Default（默认模式）
- **行为**：执行文件编辑和 Shell 命令前询问用户确认
- **适用**：日常开发，需要保持对 Agent 行为的完整控制
- **切换**：`Shift+Tab` 或 `--permission-mode default`

#### 🟢 AcceptEdits（自动接受编辑）
- **行为**：自动接受所有文件编辑，但 Shell 命令仍需确认
- **适用**：信任 Claude 的代码修改，但不想让它随意执行命令
- **切换**：`Shift+Tab` 循环到此模式

#### 🟡 Plan（规划模式）
- **行为**：Claude 只分析和规划，不执行任何实际操作
- **适用**：先看 Claude 的方案，再决定是否执行
- **切换**：`Shift+Tab` 循环到此模式

#### 🟠 Auto（自动模式）
- **行为**：AI 分类器自动评估每个操作的风险，低风险自动执行
- **适用**：减少审批疲劳，适合熟悉 Claude Code 的用户
- **切换**：`--permission-mode auto`

#### 🔴 BypassPermissions（跳过所有权限）
- **行为**：跳过所有权限检查，完全自主执行
- **⚠️ 警告**：仅在完全隔离的沙箱环境中使用！
- **切换**：`--dangerously-skip-permissions`

### 6.2 细粒度权限配置

在 `.claude/settings.json` 中配置：

```json
{
  "permissions": {
    "allow": [
      "Read",
      "Edit",
      "Bash(git *)",
      "Bash(npm test)",
      "WebSearch"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(sudo *)",
      "Bash(curl * | bash)"
    ],
    "ask": [
      "Bash"
    ]
  }
}
```

**规则优先级**：`deny` > `ask` > `allow`

---

## 7. CLAUDE.md 配置指南

### 7.1 文件位置与优先级

```
优先级（高→低）
├── ~/.claude/CLAUDE.md          全局个人配置（所有项目生效）
├── /project-root/CLAUDE.md      项目根目录配置
└── /project-root/src/CLAUDE.md  子目录配置（仅对该目录生效）
```

### 7.2 推荐 CLAUDE.md 模板

```markdown
# 项目名称 - Claude Code 配置

## 项目概述
这是一个 [项目类型] 项目，使用 [技术栈]。

## 常用命令
- 启动开发服务器：`npm run dev`
- 运行测试：`npm test`
- 构建：`npm run build`
- 代码检查：`npm run lint`

## 代码规范
- 使用 TypeScript，严格模式
- 函数命名：camelCase
- 组件命名：PascalCase
- 注释语言：中文
- 缩进：2 空格

## 架构说明
- `src/` - 源代码
- `src/components/` - React 组件
- `src/services/` - 业务逻辑
- `src/utils/` - 工具函数
- `tests/` - 测试文件

## 重要约束
- 不要修改 `src/legacy/` 目录下的文件
- 所有 API 调用必须经过 `src/services/api.ts`
- 新功能必须包含单元测试

## 审查清单
在提交代码前，确保：
- [ ] 所有测试通过
- [ ] 没有 TypeScript 错误
- [ ] 没有 ESLint 警告
- [ ] 更新了相关文档
```

### 7.3 高级配置技巧

```markdown
## 禁止操作
永远不要：
- 删除任何 .env 文件
- 修改 package-lock.json
- 直接操作数据库（使用 ORM）

## 首选库
- HTTP 请求：axios（不要用 fetch）
- 状态管理：Zustand（不要用 Redux）
- 样式：Tailwind CSS

## 当前已知问题
- auth.ts 第 142 行有一个已知 bug，暂时不要修改
- 测试环境的 Redis 连接不稳定，失败时重试即可
```

> **最佳实践**：建议 CLAUDE.md 保持在 200 行以内，过长会降低 Claude 的遵循质量。

---

## 8. 记忆系统使用指南

### 8.1 四层记忆架构

```
Layer 3: CLAUDE.md（人工注入，永久）
    ↑ 每次会话开始时加载
Layer 2: Auto Memory / MEMORY.md（自动积累，项目级）
    ↑ Agent 自动写入，跨会话持久化
Layer 1: Session Memory（自动提取，跨会话）
    ↑ 后台自动保存关键信息
Layer 0: Context Window（即时，单次会话）
```

### 8.2 Auto Memory 使用

Claude Code 会自动在 `.claude/MEMORY.md` 中记录：
- 发现的 bug 和修复方法
- 项目特定的命令和参数
- 用户偏好和工作习惯
- 架构决策和重要约定

MEMORY.md 的前 200 行（或 25KB）会在每次对话开始时自动加载。

**查看记忆**：
```
/memory
```

**手动触发记忆整合（Auto Dream）**：
```
请整合并优化你的项目记忆
```

### 8.3 上下文压缩触发时机

| 使用率 | 触发动作 | 用户感知 |
|--------|----------|----------|
| < 64% | 无操作 | 正常 |
| 64%~75% | MicroCompact（本地裁剪） | 无感知 |
| 75%~95% | AutoCompact（生成摘要） | 可能看到压缩提示 |
| > 95% | Full Compact（紧急压缩） | 明显的压缩操作 |

**手动压缩并保留特定内容**：
```
/compact 保留所有关于数据库 schema 的讨论，以及 auth 模块的修改历史
```

---

## 9. 会话管理

### 9.1 继续与恢复会话

```bash
# 继续最近一次会话（最常用）
claude --continue
claude -c

# 显示会话选择器，选择要恢复的会话
claude --resume

# 恢复特定会话（通过 session ID）
claude --resume abc123def456
```

### 9.2 会话回溯（Rewind）

在交互式会话中：
1. 双击 `Esc` 打开回溯菜单
2. 选择要回到的历史节点
3. 从该节点继续，形成新的对话分支（Fork）

**使用场景**：Claude 走错了方向，想回到某个正确的节点重新尝试。

### 9.3 并行会话（Git Worktree）

```bash
# 创建新的 worktree 用于并行开发
git worktree add ../feature-auth feature/auth
git worktree add ../feature-payment feature/payment

# 在不同终端中分别启动 Claude Code
cd ../feature-auth && claude
cd ../feature-payment && claude
```

**效果**：两个 Claude Code 实例同时工作在不同分支，互不干扰。

### 9.4 会话存储位置

```
~/.claude/projects/<encoded-cwd>/
├── *.jsonl              # 会话消息记录（每条消息一行 JSON）
~/.claude/
├── settings.json        # 全局设置
├── keybindings.json     # 自定义快捷键
└── CLAUDE.md            # 全局记忆文件
```

---

## 10. 模型选择与别名

### 10.1 可用模型别名

| 别名 | 对应模型 | 特点 | 适用场景 |
|------|----------|------|----------|
| `opus` | Claude Opus 4.6（内部代号 Fennec） | 最强推理，最慢，最贵 | 复杂架构设计、难题攻关 |
| `sonnet` | Claude Sonnet 4.6 | 均衡性能，日常首选 | 日常编码、代码审查 |
| `haiku` | Claude Haiku 4.5 | 最快，最便宜 | 简单查询、快速补全 |
| `opusplan` | Opus 规划 + Sonnet 执行 | 成本优化 | 复杂任务的经济方案 |
| `opus[1m]` | Opus 4.6 + 1M 上下文 | 超大上下文 | 分析超大代码库 |
| `sonnet[1m]` | Sonnet 4.6 + 1M 上下文 | 均衡 + 超大上下文 | 大型项目日常开发 |

### 10.2 模型切换

```bash
# 启动时指定
claude --model opus
claude --model sonnet[1m]

# 会话中切换
/model haiku
/model opusplan
```

### 10.3 opusplan 模式详解

`opusplan` 是一个特殊的双模型策略：
1. **规划阶段**：使用 Opus 进行深度思考，制定详细方案
2. **执行阶段**：切换到 Sonnet 执行具体操作

**成本节省**：相比全程使用 Opus，成本降低约 60-70%，质量接近。

---

## 11. MCP 集成与扩展

### 11.1 什么是 MCP

Model Context Protocol（模型上下文协议）是 Anthropic 于 2025年12月发布的开放标准，允许 Claude Code 连接外部工具和数据源。目前支持 **6,000+** 应用集成。

### 11.2 配置 MCP 服务器

在 `~/.claude/settings.json` 或 `.claude/settings.json` 中：

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "your-token"
      }
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "DATABASE_URL": "postgresql://localhost/mydb"
      }
    }
  }
}
```

也可以使用项目级 `.mcp.json` 文件单独管理 MCP 配置（详见第 17 章）。

### 11.3 常用 MCP 服务器

| 服务器 | 功能 |
|--------|------|
| `@mcp/github` | GitHub Issues、PR、代码搜索 |
| `@mcp/postgres` | PostgreSQL 数据库查询 |
| `@mcp/slack` | Slack 消息和频道 |
| `@mcp/notion` | Notion 文档和数据库 |
| `@mcp/jira` | JIRA 任务管理 |
| `@mcp/browser` | Chrome 浏览器控制（Playwright） |
| `@mcp/filesystem` | 扩展文件系统访问 |

---

## 12. 自定义 Slash 命令

### 12.1 创建项目级自定义命令

在 `.claude/commands/` 目录下创建 Markdown 文件：

```bash
mkdir -p .claude/commands
```

**示例：创建代码审查命令**

`.claude/commands/review.md`：
```markdown
# 代码审查

请对以下内容进行全面的代码审查：

1. **安全性**：检查 SQL 注入、XSS、认证漏洞
2. **性能**：识别 N+1 查询、不必要的循环、内存泄漏
3. **可维护性**：评估代码复杂度、命名规范、注释质量
4. **测试覆盖**：检查是否有对应的单元测试

$ARGUMENTS
```

使用：`/review src/auth.ts`

### 12.2 Skills 目录（高级）

`.claude/skills/<name>/SKILL.md` 格式的命令可以被 Claude 自主调用：

```bash
mkdir -p .claude/skills/deploy
```

`.claude/skills/deploy/SKILL.md`：
```markdown
---
name: deploy
description: 部署应用到生产环境
triggers: [deploy, 部署, release]
---

# 部署技能

当用户要求部署时，执行以下步骤：
1. 运行测试：`npm test`
2. 构建项目：`npm run build`
3. 推送 Docker 镜像
4. 更新 K8s 配置
```

---

## 13. Hooks 自动化配置

### 13.1 Hooks 配置位置

在 `~/.claude/settings.json` 或 `.claude/settings.json` 中：

```json
{
  "hooks": {
    "pre_tool_use": [...],
    "post_tool_use": [...],
    "pre_bash": [...],
    "post_bash": [...],
    "on_error": [...]
  }
}
```

### 13.2 实用 Hooks 示例

#### 自动格式化代码

```json
{
  "hooks": {
    "post_tool_use": [
      {
        "matcher": "FileEdit|FileWrite",
        "command": "prettier --write $CLAUDE_TOOL_FILE 2>/dev/null || true"
      }
    ]
  }
}
```

#### 阻止危险命令

```json
{
  "hooks": {
    "pre_bash": [
      {
        "command": "echo $CLAUDE_BASH_COMMAND | grep -qE '(rm -rf|DROP TABLE|sudo rm)' && echo 'BLOCKED' && exit 1 || exit 0"
      }
    ]
  }
}
```

#### 操作审计日志

```json
{
  "hooks": {
    "post_tool_use": [
      {
        "command": "echo \"$(date): $CLAUDE_TOOL_NAME - $CLAUDE_TOOL_INPUT\" >> ~/.claude/audit.log"
      }
    ]
  }
}
```

### 13.3 Hooks 环境变量

| 变量 | 说明 |
|------|------|
| `$CLAUDE_TOOL_NAME` | 当前执行的工具名称 |
| `$CLAUDE_TOOL_INPUT` | 工具的输入参数（JSON） |
| `$CLAUDE_TOOL_OUTPUT` | 工具的输出结果 |
| `$CLAUDE_TOOL_FILE` | 被操作的文件路径 |
| `$CLAUDE_BASH_COMMAND` | 即将执行的 Bash 命令 |
| `$PROJECT_ROOT` | 项目根目录路径 |

---

## 14. 多 Agent 与并行工作流

### 14.1 子 Agent（Subagent）

在任务描述中明确要求并行执行：

```
请并行完成以下三个任务：
1. 为 UserService 编写单元测试
2. 为 PaymentService 编写单元测试
3. 为 AuthService 编写单元测试
```

### 14.2 Agent 团队模式

```
请组建一个 Agent 团队来完成这个功能：
- 架构师 Agent：设计整体方案
- 前端 Agent：实现 UI 组件
- 后端 Agent：实现 API 接口
- 测试 Agent：编写测试用例
```

### 14.3 Git Worktree 并行开发

```bash
git worktree add ../feat-search feature/search
git worktree add ../feat-payment feature/payment

# 两个终端同时运行
terminal1: cd ../feat-search && claude
terminal2: cd ../feat-payment && claude
```

---

## 15. CI/CD 与自动化集成

### 15.1 GitHub Actions 集成

```yaml
name: Claude Code Review
on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install Claude Code
        run: npm install -g @anthropic-ai/claude-code
      - name: Run Code Review
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          git diff origin/main...HEAD | claude -p \
            "审查这个 PR 的代码变更，重点关注安全性和性能问题" \
            --output-format json \
            --dangerously-skip-permissions
```

---

## 16. 44 个隐藏 Feature Flag 完整解析

> 以下内容来自 2026年3月31日源码泄漏，44 个 Flag 中约 20 个已发布，约 10 个 Beta，约 14 个未发布或内部专用。

### 16.1 已发布功能 Flag（~20 个）

| Flag 名称 | 功能 | 用户入口 |
|-----------|------|----------|
| `enable_auto_compact` | 自动上下文压缩 | 自动触发 / `/compact` |
| `enable_session_memory` | 跨会话记忆持久化 | 自动运行 |
| `enable_extended_thinking` | 扩展思考模式 | `/model opus` 时启用 |
| `enable_voice_input` | 语音输入（2026.03） | `/voice` |
| `enable_mcp_v2` | MCP 协议 v2，6000+ 集成 | `/mcp` 配置 |
| `enable_auto_mode` | AI 分类器自动权限判断 | `--permission-mode auto` |
| `enable_git_worktree` | Git Worktree 并行会话 | 多终端 + worktree |
| `enable_subagent` | 子 Agent 内联生成 | 自然语言描述并行任务 |
| `enable_agent_teams` | Agent 团队协作模式 | 自然语言描述团队任务 |
| `enable_hooks` | 用户自定义 Hooks 系统 | `settings.json` 配置 |
| `enable_custom_slash_commands` | 自定义 Slash 命令 | `.claude/commands/` 目录 |
| `enable_1m_context` | 100万 token 上下文（2026.02） | `--model opus[1m]` |
| `enable_opusplan_mode` | Opus 规划 + Sonnet 执行 | `/model opusplan` |
| `enable_image_paste` | 粘贴图片直接分析 | `Ctrl+V` 粘贴截图 |
| `enable_conversation_fork` | 会话分叉（Rewind + Fork） | `Esc` `Esc` |
| `enable_os_sandbox` | OS 级沙箱，限制 Bash 访问 | 自动启用 |
| `enable_ai_classifier` | AI 风险分类器 | `--permission-mode auto` |
| `enable_managed_settings` | 企业托管设置 | 系统管理员配置 |
| `enable_micro_compact` | MicroCompact：本地裁剪 | 自动触发（64%~75%） |
| `enable_thinking_budget` | 思考预算控制 | `/config` 设置 |

### 16.2 Beta / 部分发布 Flag（~10 个）

| Flag 名称 | 功能 | 状态 |
|-----------|------|------|
| `enable_buddy` | **Claude Buddy 虚拟伴侣**：终端内 ASCII 艺术宠物，18 种物种，根据账户 ID 确定性生成，观察用户编码并发表评论 | 2026年4月1日愚人节正式发布 |
| `enable_auto_dream` | Auto Dream 记忆整合：后台定期整合、剪枝和重组记忆文件 | 部分用户可用 |
| `enable_dev_team_mode` | 开发团队模式：Claude 将复杂任务分解，并行处理后合并结果 | Sonnet 5 特性，部分发布 |
| `enable_voice_dictation` | 语音听写：实时语音转文字输入 | Beta 测试中 |
| `enable_project_memory_index` | 项目记忆索引：MEMORY.md 作为索引，指向细分领域记忆文件 | Beta 测试中 |
| `enable_fast_mode` | 快速模式：牺牲部分质量换取更快响应速度 | 部分用户可用 |
| `enable_interleaved_thinking` | 交错思考：在工具调用之间插入思考步骤（Claude 4 特性） | Claude 4 模型可用 |
| `enable_mcp_oauth` | MCP OAuth 认证：MCP 服务器的 OAuth 2.0 认证支持 | 部分 MCP 服务器支持 |
| `enable_skills_directory` | Skills 目录：Agent 可自主调用的自定义技能 | 部分发布 |
| `enable_tool_search` | 工具按需发现和加载，节省 context 空间 | 部分发布 |

### 16.3 未发布 / 内部专用 Flag（~14 个）

#### 🔴 KAIROS — 常驻自主 Agent 守护进程
- **Flag**：`enable_kairos`
- **状态**：未发布
- **描述**：后台守护进程（Daemon），主动监控文件系统变化和 Git 事件，无需用户触发即可行动。包含 `autoDream` 夜间记忆巩固子系统，在用户空闲时自动合并观察结果、解决矛盾信息、将模糊笔记转化为已验证事实。

#### 🔴 ULTRAPLAN — 云端规划引擎
- **Flag**：`enable_ultraplan`
- **状态**：未发布（v2.1.92 中发现）
- **描述**：将复杂规划任务卸载到远程云容器中运行 Opus 4.6，支持长达 **30 分钟**的专项规划时间。用户可在 CLI 发起规划，在浏览器中查看并添加内联注释，然后在终端或远程执行。

#### 🔴 WEB_BROWSER_TOOL — 浏览器控制工具
- **Flag**：`enable_web_browser_tool`
- **状态**：未发布（通过「Claude in Chrome」扩展部分实现）
- **描述**：允许 Claude 直接控制 Chrome 浏览器，执行网页导航、表单填写、数据提取、多步骤工作流等操作，将 Agent 能力从终端延伸到浏览器。

#### 🔴 DAEMON — 后台进程模式
- **Flag**：`enable_daemon`
- **状态**：未发布
- **描述**：Claude Code 的后台常驻进程模式，与 KAIROS 协同工作。在用户不主动交互时持续运行，整合记忆、监控代码库变化、准备上下文，使下次交互更高效。

#### 🔴 AGENT_TRIGGERS — 事件驱动 Agent 激活
- **Flag**：`enable_agent_triggers`
- **状态**：未发布
- **描述**：基于事件的 Agent 自动激活机制（也称 Agent Skills）。可配置在特定文件变更、Git 事件、时间触发等条件下自动启动 Agent 执行预定义任务，无需用户手动触发。

#### 🔴 COORDINATOR_MODE — 多 Agent 总协调者
- **Flag**：`enable_coordinator_mode`
- **状态**：未发布
- **描述**：Claude 作为多 Agent 工作流的总协调者，管理多个并行 Worker Agent 的任务分配、进度追踪和结果整合。比现有 Agent Teams 更高级的编排层。

#### 🔴 KAIROS_AUTO_DREAM — KAIROS 专属记忆整合
- **Flag**：`enable_kairos_auto_dream`
- **状态**：未发布
- **描述**：KAIROS 模式下的专属夜间记忆巩固系统，在用户空闲时自动运行，合并观察结果、解决矛盾、将模糊笔记转化为已验证事实，并更新 MEMORY.md 索引。

#### 🔴 UNDERCOVER_MODE — 卧底模式
- **Flag**：`enable_undercover_mode`
- **状态**：内部专用（Anthropic 员工）
- **描述**：检测到 Anthropic 员工账户时自动激活。注入系统提示，要求 AI 从 git commit 和 PR 中移除所有 Anthropic 内部信息，包括 AI 归因、内部代号（Tengu、Capybara）、内部 Slack 频道名称、未发布模型版本号。**⚠️ 引发开源社区强烈批评**。

#### 🔴 ANTI_DISTILLATION — 反蒸馏保护
- **Flag**：`tengu_anti_distill_fake_tool_injection`
- **状态**：内部专用
- **描述**：向系统提示注入虚假工具定义，污染竞争对手通过抓取 API 流量收集的训练数据。配合 `CONNECTOR_TEXT` 机制将工具调用细节总结为模糊描述，防止竞争对手重建 Agent 的完整推理过程。

#### 🔴 FRUSTRATION_DETECTION — 用户情绪检测
- **Flag**：`enable_frustration_detection`
- **状态**：内部（部分生效）
- **描述**：通过正则表达式（如 `wtf|ffs|shit|damn` 等）监控用户消息中的情绪信号，检测到用户沮丧时自动调整响应策略，变得更有耐心和帮助性。

#### 🔴 未发布模型版本引用
- **Flag**：`enable_opus_4_7`、`enable_sonnet_4_8`
- **状态**：未发布
- **描述**：泄漏代码中发现的未发布模型版本引用。Opus 4.7 和 Sonnet 4.8 已在内部代码中出现，但尚未公开发布。

#### 🔴 内部模型代号
- **Capybara**（也称 Mythos）：Claude 4.6 变体的内部代号，内部注释显示已到第 8 版，1M token 上下文
- **Fennec**：Opus 4.6 的内部代号
- **Numbat**：仍在测试中的未发布模型代号，具体能力未知

---

## 17. 重要文件与目录完整说明

### 17.1 完整目录结构

```
~/.claude/                          # 全局用户目录
├── settings.json                   # 全局用户设置
├── keybindings.json                # 自定义键盘快捷键
├── CLAUDE.md                       # 全局个人记忆/指令
├── MEMORY.md                       # 全局 Auto Memory 索引
├── skills/                         # 全局自定义技能
│   └── <skill-name>/
│       └── SKILL.md
├── commands/                       # 全局自定义命令
│   └── <command-name>.md
└── projects/                       # 会话历史存储
    └── <encoded-project-path>/
        └── *.jsonl                 # 会话消息记录

~/.claude.json                      # 系统状态文件（OAuth、缓存等）

/project-root/
├── CLAUDE.md                       # 项目级指令（提交到 git）
├── .mcp.json                       # 项目级 MCP 服务器配置
└── .claude/
    ├── settings.json               # 项目共享设置（提交到 git）
    ├── settings.local.json         # 个人本地设置（git ignore）
    ├── MEMORY.md                   # 项目 Auto Memory 索引
    ├── commands/                   # 项目自定义命令
    │   └── <command-name>.md
    └── skills/                     # 项目自定义技能
        └── <skill-name>/
            └── SKILL.md

# 企业托管设置（系统级，用户无法覆盖）
Linux/WSL:  /etc/claude-code/managed-settings.json
macOS:      /Library/Application Support/ClaudeCode/managed-settings.json
Windows:    C:\Program Files\ClaudeCode\managed-settings.json
```

---

### 17.2 核心文件详解

#### 📋 CLAUDE.md — 项目指令文件

| 属性 | 说明 |
|------|------|
| **位置** | 项目根目录 / `~/.claude/` / 子目录 |
| **作用** | 每次会话开始时自动注入，提供持久化项目上下文 |
| **内容** | 常用命令、代码规范、架构说明、重要约束、禁止操作 |
| **建议长度** | 200 行以内（过长降低遵循质量） |
| **版本控制** | ✅ 建议提交到 git（团队共享） |
| **更新方式** | 手动编辑 |

#### 🧠 MEMORY.md — Auto Memory 索引

| 属性 | 说明 |
|------|------|
| **位置** | `.claude/MEMORY.md` 或 `~/.claude/MEMORY.md` |
| **作用** | Claude 自动写入的学习笔记索引，指向细分领域记忆文件 |
| **内容** | 错误修复记录、命令参数、架构决策、用户偏好 |
| **加载规则** | 前 200 行（或 25KB）在每次对话开始时自动加载 |
| **版本控制** | ⚠️ 可选（包含 AI 自动生成内容，可能频繁变更） |
| **更新方式** | Claude 自动维护，也可手动编辑 |

**MEMORY.md 典型结构**：
```markdown
# 项目记忆索引

## 快速参考
- 数据库：PostgreSQL 14，端口 5432
- 测试命令：`npm test -- --coverage`
- 部署：`./scripts/deploy.sh production`

## 详细记忆文件
- [调试记录](./memories/debugging.md)
- [API 约定](./memories/api-conventions.md)
- [已知问题](./memories/known-issues.md)
```

---

#### ⚙️ settings.json — 核心配置文件

**配置优先级（高→低）**：
```
managed-settings.json（企业，不可覆盖）
    ↓
命令行参数（临时覆盖）
    ↓
.claude/settings.local.json（个人本地，git ignore）
    ↓
.claude/settings.json（项目共享，提交 git）
    ↓
~/.claude/settings.json（全局用户默认）
    ↓
Claude Code 默认行为
```

**完整配置示例**：
```json
{
  "model": "sonnet",
  "permissions": {
    "allow": ["Read", "Edit", "Bash(git *)", "Bash(npm *)"],
    "deny": ["Bash(rm -rf *)", "Bash(sudo *)"],
    "ask": ["Bash"]
  },
  "hooks": {
    "post_tool_use": [
      {
        "matcher": "FileEdit|FileWrite",
        "command": "prettier --write $CLAUDE_TOOL_FILE 2>/dev/null || true"
      }
    ]
  },
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_TOKEN": "ghp_xxx" }
    }
  },
  "env": {
    "NODE_ENV": "development"
  }
}
```

---

#### 🔒 settings.local.json — 个人本地配置

| 属性 | 说明 |
|------|------|
| **位置** | `.claude/settings.local.json` |
| **作用** | 个人本地覆盖，不与团队共享 |
| **典型用途** | 个人 API Key、本地路径、个人偏好 |
| **版本控制** | ❌ 不提交（Claude Code 自动添加到 .gitignore） |

```json
{
  "model": "opus",
  "mcpServers": {
    "local-db": {
      "command": "npx",
      "args": ["-y", "@mcp/postgres"],
      "env": { "DATABASE_URL": "postgresql://localhost/mydev" }
    }
  }
}
```

---

#### 🏢 managed-settings.json — 企业托管配置

| 属性 | 说明 |
|------|------|
| **位置** | 系统级目录（见上方路径） |
| **作用** | 企业强制策略，用户无法覆盖 |
| **典型用途** | 禁止危险命令、限制可用模型、强制 MCP 服务器 |
| **版本控制** | 由 IT 管理员通过 MDM/配置管理工具部署 |

```json
{
  "managed": true,
  "permissions": {
    "deny": ["Bash(curl * | bash)", "Bash(wget * | sh)", "Bash(sudo *)"]
  },
  "allowedModels": ["sonnet", "haiku"],
  "mcpServers": {
    "company-jira": {
      "command": "npx",
      "args": ["-y", "@company/mcp-jira"]
    }
  }
}
```

---

#### 🔌 .mcp.json — 项目级 MCP 配置

| 属性 | 说明 |
|------|------|
| **位置** | 项目根目录 `.mcp.json` |
| **作用** | 项目专属 MCP 服务器配置，独立于 settings.json |
| **版本控制** | ✅ 建议提交（团队共享 MCP 配置） |

```json
{
  "mcpServers": {
    "project-db": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "DATABASE_URL": "${PROJECT_DB_URL}"
      }
    },
    "figma": {
      "command": "npx",
      "args": ["-y", "@mcp/figma"],
      "env": {
        "FIGMA_TOKEN": "${FIGMA_TOKEN}"
      }
    }
  }
}
```

---

#### 🗂️ ~/.claude.json — 系统状态文件

| 属性 | 说明 |
|------|------|
| **位置** | `~/.claude.json`（注意：在 `~/.claude/` 目录**外**） |
| **作用** | 系统自动管理的状态文件，存储运行时状态 |
| **内容** | OAuth 会话 Token、MCP 服务器配置缓存、每个项目的已允许工具列表、信任设置、各类缓存 |
| **版本控制** | ❌ 绝对不提交（含 OAuth Token 等敏感信息） |
| **更新方式** | Claude Code 自动维护，不建议手动编辑 |

> **注意**：`~/.claude.json` 与 `~/.claude/settings.json` 是两个不同的文件。前者是系统状态（短暂性），后者是用户策略（持久性）。

---

#### ⌨️ keybindings.json — 自定义快捷键

| 属性 | 说明 |
|------|------|
| **位置** | `~/.claude/keybindings.json` |
| **作用** | 自定义键盘快捷键映射 |
| **生成方式** | `/keybindings` 命令，或手动创建 |
| **生效方式** | 修改后自动生效，无需重启 |

```json
{
  "bindings": [
    { "key": "ctrl+shift+c", "command": "/compact" },
    { "key": "ctrl+shift+m", "command": "/model opusplan" },
    { "key": "ctrl+shift+r", "command": "/memory" }
  ]
}
```

---

#### 📁 commands/ 目录 — 自定义命令

| 属性 | 说明 |
|------|------|
| **位置** | `.claude/commands/`（项目级）或 `~/.claude/commands/`（全局） |
| **作用** | 定义可通过 `/命令名` 调用的自定义工作流 |
| **文件格式** | Markdown（文件名即命令名） |
| **调用方式** | `/review`、`/deploy` 等 |

---

#### 🎯 skills/ 目录 — 自定义技能

| 属性 | 说明 |
|------|------|
| **位置** | `.claude/skills/<name>/`（项目级）或 `~/.claude/skills/<name>/`（全局） |
| **作用** | 定义 Claude 可自主调用的技能（比 commands 更智能） |
| **文件格式** | `SKILL.md`（含 YAML frontmatter） |
| **调用方式** | 用户显式调用 `/skill-name`，或 Claude 根据上下文自动触发 |

---

#### 📊 sessions/ — 会话历史

| 属性 | 说明 |
|------|------|
| **位置** | `~/.claude/projects/<encoded-cwd>/*.jsonl` |
| **作用** | 存储所有会话的完整消息历史 |
| **格式** | JSONL（每行一条消息记录） |
| **用途** | 支持 `--continue`、`--resume`、Rewind、Fork 功能 |
| **清理** | 可手动删除旧会话文件释放空间 |

---

#### 🔌 插件（Plugins）生态

Claude Code 插件是将多种扩展打包在一起的可安装单元，包含：

| 插件组件 | 说明 |
|----------|------|
| **Skills** | Claude 自动使用的领域技能 |
| **Agents** | 有独立上下文和提示的专属 Agent |
| **Hooks** | 在 Claude 操作前后运行的脚本 |
| **MCP Servers** | 预配置的外部服务连接 |
| **Commands** | 可重复使用的工作流命令 |

**常见插件类型**：
- TypeScript 语言服务器集成
- Playwright 浏览器自动化
- 安全审查指导
- 代码审查工作流
- 数据库 Schema 管理

**企业私有插件市场**：Anthropic 于 2026年2月24日宣布为企业用户提供私有插件市场，支持内部插件的发布和管理。

---

## 18. 企业与团队配置

### 18.1 配置优先级层次

```
Managed Settings（最高，不可覆盖）
    ↓
Command Line Arguments（命令行参数）
    ↓
Local Project Settings（.claude/settings.local.json）
    ↓
Shared Project Settings（.claude/settings.json）
    ↓
User Settings（~/.claude/settings.json，最低）
```

### 18.2 团队共享配置

在项目仓库中提交 `.claude/settings.json`（团队共享）：

```json
{
  "permissions": {
    "allow": ["Read", "Edit", "Bash(npm *)", "Bash(git *)"],
    "deny": ["Bash(rm -rf *)"]
  },
  "hooks": {
    "post_tool_use": [
      {
        "matcher": "FileEdit",
        "command": "npm run lint --fix 2>/dev/null || true"
      }
    ]
  }
}
```

---

## 19. 常见使用场景示例

### 代码理解与探索
```
> 解释这个项目的整体架构，从入口文件开始
> 找出所有调用 UserService.create() 的地方
> 这个函数的时间复杂度是多少？有没有优化空间？
```

### Bug 修复
```bash
npm run dev 2>&1 | claude -p "分析这些错误并修复根本原因"
git log --oneline -10 | claude -p "找出最近哪个 commit 可能引入了内存泄漏"
```

### 功能开发
```
> 实现一个用户头像上传功能，支持 JPG/PNG，最大 5MB，存储到 S3
> 在现有的认证系统上添加 Google OAuth 登录
```

### 安全审查
```
> 检查这个项目中所有可能的 SQL 注入漏洞
> 审查 API 接口的认证和授权逻辑
> 检查是否有敏感信息被硬编码在代码中
```

### 性能优化
```
> 分析这个查询为什么很慢，并提出优化方案
> 找出前端代码中不必要的重渲染
```

---

*手册版本：v2.0 | 更新日期：2026年4月7日*  
*数据来源：Anthropic 官方文档 + 2026年3月31日源码泄漏分析*
