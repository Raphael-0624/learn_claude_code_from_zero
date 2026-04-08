# Claude Code 架构深度拆解与开源生态全景报告

> 基于 2026年3月31日 源码泄漏事件的技术分析  
> 报告日期：2026年4月7日

---

## 目录

1. [泄漏事件背景](#1-泄漏事件背景)
2. [整体架构概览](#2-整体架构概览)
3. [核心模块深度拆解](#3-核心模块深度拆解)
   - 3.1 [Query Engine — 会话心脏](#31-query-engine--会话心脏)
   - 3.2 [Agent 执行框架 (Harness)](#32-agent-执行框架-harness)
   - 3.3 [工具系统 (Tool System)](#33-工具系统-tool-system)
   - 3.4 [记忆系统 (Memory System)](#34-记忆系统-memory-system)
   - 3.5 [权限与安全系统](#35-权限与安全系统)
   - 3.6 [多 Agent 编排](#36-多-agent-编排)
4. [隐藏特性与内部彩蛋](#4-隐藏特性与内部彩蛋)
5. [开源改造项目全景](#5-开源改造项目全景)
6. [技术借鉴与设计启示](#6-技术借鉴与设计启示)

---

## 1. 泄漏事件背景

### 1.1 事件时间线

| 时间 | 事件 |
|------|------|
| 2025年2月 | 第一次小规模泄漏（部分源码暴露） |
| 2026年3月31日 | 主要泄漏事件：npm 包 `@anthropic-ai/claude-code` v2.1.88 意外包含 59.8MB 的 `.map` 源码映射文件 |
| 2026年3月31日（同日） | 安全研究员 Chaofan Shou 在 X 上公开披露，代码迅速在 GitHub 上被镜像传播 |
| 2026年4月初 | Anthropic 发出版权下架请求（DMCA），但代码已广泛传播；多个 clean-room 重实现项目涌现 |

### 1.2 泄漏规模

- **文件数量**：1,906 个 TypeScript 文件
- **代码行数**：约 513,000 行（未混淆）
- **文件大小**：59.8 MB（`.map` 源码映射文件）
- **泄漏原因**：发布打包流程中的人为失误（非安全漏洞）

### 1.3 Anthropic 官方声明

> "这是一次由人为错误导致的发布打包问题，不是安全漏洞，没有敏感客户数据或凭证泄漏。"

---

## 2. 整体架构概览

Claude Code 是一个基于 **TypeScript + Bun 运行时** 的终端 AI 编程 Agent，采用分层模块化架构：

```
┌─────────────────────────────────────────────────────────┐
│                    Presentation Layer                    │
│         Terminal UI (React + Ink 框架渲染)               │
├─────────────────────────────────────────────────────────┤
│                   Application Layer                      │
│         业务规则 / 命令系统 / 配置管理                    │
├─────────────────────────────────────────────────────────┤
│                  Orchestration Layer                     │
│    Query Engine | Multi-Agent Router | Hook System       │
├─────────────────────────────────────────────────────────┤
│                      AI Layer                            │
│    Claude API 调用 | 工具调用循环 | 上下文压缩            │
├─────────────────────────────────────────────────────────┤
│                    Tool Registry                         │
│    38~60+ 工具 | MCP 集成 | 权限分层                     │
├─────────────────────────────────────────────────────────┤
│                   Memory System                          │
│  CLAUDE.md | Auto Memory | Session Memory | Auto Dream   │
└─────────────────────────────────────────────────────────┘
```

### 技术栈

| 组件 | 技术选型 | 说明 |
|------|----------|------|
| 运行时 | **Bun** | 替代 Node.js，更快的启动速度 |
| 终端 UI | **React + Ink** | 在终端中渲染 React 组件 |
| Schema 验证 | **Zod v4** | 工具输入/输出的类型安全 |
| 语言 | **TypeScript** | 全量类型覆盖 |
| 核心文件 | `QueryEngine.ts` | 约 46,000 行，系统核心 |

---

## 3. 核心模块深度拆解

### 3.1 Query Engine — 会话心脏

`QueryEngine.ts` 是整个系统最核心的文件，约 **46,000 行**，负责管理单次对话的完整生命周期。

#### 核心职责

```
QueryEngine 职责图
├── 流式响应管理 (Streaming Response Management)
│   └── 处理 SSE 流，实时渲染 token
├── 工具调用循环 (Tool-Call Loop)
│   └── 检测 tool_use → 执行工具 → 注入结果 → 继续推理
├── Thinking Mode 预算管理
│   └── 控制 extended thinking 的 token 预算
├── API 失败自动重试 (Auto Retry)
│   └── 指数退避 + 错误分类处理
├── Token 计数与追踪
│   └── 实时统计输入/输出 token 消耗
└── 上下文窗口管理 (Context Budget)
    └── 触发压缩策略，防止超出限制
```

#### 工具调用循环（核心 Agentic Loop）

```
用户输入
    ↓
[QueryEngine] 构建消息列表
    ↓
调用 Claude API（流式）
    ↓
解析响应
    ├── 纯文本 → 渲染输出，结束
    └── tool_use → 
            ↓
        [Tool Registry] 查找工具
            ↓
        [Permission System] 权限检查
            ↓
        执行工具（BashTool / FileReadTool / ...）
            ↓
        将 tool_result 注入消息列表
            ↓
        回到「调用 Claude API」步骤（循环）
```

---

### 3.2 Agent 执行框架 (Harness)

这是 Claude Code 最核心的设计哲学：**以工具为中心的 Agentic 执行框架**。

#### 框架层次结构

```
Agent Harness
├── 单 Agent 模式（默认）
│   ├── 接收用户任务
│   ├── 规划执行步骤（内部 CoT）
│   ├── 循环调用工具
│   └── 汇报结果
│
├── 子 Agent 模式（Subagent）
│   ├── 父 Agent 通过 AgentTool 内联生成子 Agent
│   ├── 子 Agent 在独立上下文中运行
│   ├── 运行至完成后返回结果给父 Agent
│   └── 适用于：并行化子任务、隔离危险操作
│
└── Agent 团队模式（Agent Teams）
    ├── 命名 Agent 组，共享目标
    ├── Lead Agent 协调工作分配
    ├── Teammate Agent 各自拥有独立 context window
    └── Agent 间可直接通信
```

#### 执行控制机制

**Hooks 系统**（用户自定义 Shell 命令，在 Agent 生命周期特定节点触发）：

```yaml
# 示例 hooks 配置
hooks:
  pre_tool_use:
    - command: "safety-check.sh"    # 工具执行前安全检查
  post_tool_use:
    - command: "audit-log.sh"       # 工具执行后审计日志
  pre_bash:
    - command: "sandbox-validate.sh" # Bash 执行前沙箱验证
```

Hooks 的关键特性：
- **确定性**：无论 Agent 推理结果如何，Hooks 必然执行
- **阻断能力**：可在执行前阻止危险操作
- **防注入**：抵御 Prompt Injection 攻击的重要防线

---

### 3.3 工具系统 (Tool System)

#### 工具注册表结构

核心文件：`tools.ts`（工具注册表）

每个工具包含：
- **JSON Schema 定义**（输入参数类型）
- **权限声明**（所属 Tier）
- **并发声明**（是否可并行执行）
- **执行逻辑**（具体实现）

#### 工具分层（三层权限模型）

```
Tier 1 — 自动批准（Auto-approved）
├── FileReadTool      读取文件
├── DirectoryListTool 列出目录
├── WebSearchTool     网络搜索
└── WebFetchTool      获取网页内容

Tier 2 — 提示确认（Prompt for confirmation）
├── FileWriteTool     写入文件
├── FileEditTool      编辑文件
└── CreateFileTool    创建文件

Tier 3 — 需要明确批准或阻断（Require explicit approval）
├── BashTool          执行 Shell 命令（最高风险）
├── DeleteFileTool    删除文件
└── NetworkTool       网络操作（受限）
```

#### 工具激活策略

```
工具激活模式
├── 始终激活（Always Active）
│   └── 核心工具，每次会话默认加载
├── 条件激活（Conditionally Active）
│   └── 基于环境或配置动态启用
└── 特性门控（Feature-gated）
    └── 通过 Feature Flag 控制，44个隐藏 Flag
```

#### 工具搜索工具（Tool Search Tool）

一个元工具，允许 Agent 在运行时**按需发现和加载工具**，避免将所有工具定义塞入上下文窗口，显著节省 token。

#### 程序化工具调用（Programmatic Tool Calling）

Agent 可以通过生成 Python/Shell 代码来编排多个工具，而非顺序 API 调用：

```python
# Agent 生成的编排代码示例
import subprocess
files = list_directory("./src")
for f in files:
    content = read_file(f)
    if "TODO" in content:
        edit_file(f, fix_todos(content))
```

这允许：循环、条件分支、并行执行等复杂控制流。

#### BashTool 安全设计

BashTool 是最高风险工具，内置了大量防护：
- **Shell 注入防御**：确定性解析逻辑，在执行前对命令分类
- **OS 级沙箱**：限制文件系统和网络访问范围
- **命令白名单/黑名单**：可配置的允许/拒绝规则

---

### 3.4 记忆系统 (Memory System)

Claude Code 实现了一套**四层记忆架构**，解决 LLM 天然无状态的问题：

```
记忆层次结构（从短期到长期）

Layer 0: Context Window（即时工作记忆）
    ├── 容量：200K tokens（Claude 3.7 Sonnet）
    ├── 内容：当前对话、工具输出、文件内容
    └── 生命周期：单次会话

Layer 1: Session Memory（会话记忆）
    ├── 机制：后台自动提取对话关键信息
    ├── 触发：达到 token 阈值或工具调用次数
    ├── 存储：结构化摘要写入磁盘
    └── 生命周期：跨会话持久化

Layer 2: Auto Memory / MEMORY.md（项目记忆）
    ├── 机制：Agent 自动生成并维护笔记
    ├── 内容：错误修复、命令参数、架构决策、用户偏好
    ├── 结构：MEMORY.md 作为索引，指向细分领域文件
    └── 生命周期：项目级持久化

Layer 3: CLAUDE.md（人工注入记忆）
    ├── 机制：用户手动维护的 Markdown 文件
    ├── 内容：构建命令、代码规范、架构说明
    ├── 加载时机：每次会话开始时自动注入
    └── 生命周期：永久（手动维护）
```

#### 上下文压缩策略（三层压缩）

当 Context Window 接近上限时，触发分级压缩：

```
压缩触发链

Context 使用率 < 64%
    → 正常运行，无压缩

Context 使用率 64%~75%
    → MicroCompact 触发
    → 本地裁剪旧工具输出（无 API 调用，零成本）

Context 使用率 75%~95%
    → AutoCompact 触发
    → 生成结构化会话摘要
    → 保留关键信息，丢弃冗余内容
    → 预留 token buffer 供后续推理

Context 使用率 > 95%
    → Full Compact（紧急压缩）
    → 全面压缩 + 选择性重注入关键信息
```

#### Auto Dream — 记忆整合系统

Auto Dream 是最独特的设计，类比人类 **REM 睡眠**的记忆巩固机制：

```
Auto Dream 触发条件
├── 自动触发：空闲 24 小时 + 累计 5 次会话后
└── 手动触发：用户显式调用

Auto Dream 四阶段流程
├── Phase 1: Orientation（定向）
│   └── 扫描所有记忆文件，建立全局视图
├── Phase 2: Gather Signal（信号收集）
│   └── 识别高频访问、近期变更、冲突条目
├── Phase 3: Consolidation（整合）
│   └── 合并相关信息，更新过时内容
└── Phase 4: Prune & Index（剪枝与索引）
    └── 删除陈旧条目，重建 MEMORY.md 索引
```

**设计价值**：防止记忆文件随时间膨胀退化，保持长期项目中的记忆质量。

---

### 3.5 权限与安全系统

#### AI 分类器（Auto Mode）

在 Auto Mode 下，一个 AI 分类器实时评估每个拟执行操作的风险等级：

```
操作风险评估流程

Agent 提出操作请求
    ↓
AI 分类器评估
    ├── 低风险 → 自动执行（减少审批疲劳）
    ├── 中风险 → 提示用户确认
    └── 高风险 → 阻断 / 要求明确批准
```

#### 权限规则优先级

```
deny rules（拒绝规则）
    > ask rules（询问规则）
    > allow rules（允许规则）
```

#### 子 Agent 权限继承

子 Agent 会话自动继承父 Agent 的权限配置，避免重复审批提示。

---

### 3.6 多 Agent 编排

#### 两种协作模式对比

| 特性 | Subagent（子 Agent） | Agent Teams（Agent 团队） |
|------|---------------------|--------------------------|
| 创建方式 | 父 Agent 通过 AgentTool 内联创建 | 预定义命名 Agent 组 |
| 上下文 | 独立 context window | 各自独立 context window |
| 通信方式 | 结果返回给父 Agent | Agent 间可直接通信 |
| 生命周期 | 运行至完成即销毁 | 持续协作 |
| 适用场景 | 并行子任务、隔离操作 | 复杂长期项目协作 |

#### 编排模式

```
支持的编排模式
├── Sequential（顺序）：Agent A → Agent B → Agent C
├── Concurrent（并发）：Agent A + Agent B 同时运行
├── Handoff（交接）：Agent A 完成后移交给 Agent B
└── Group Chat（群聊）：多 Agent 协商决策
```

---

## 4. 隐藏特性与内部彩蛋

泄漏代码中发现了 **44 个隐藏 Feature Flag**，揭示了多个未发布功能：

### 4.1 KAIROS — 常驻自主 Agent

- **描述**：一个"always-on"的自主 Agent 模式
- **推测功能**：后台持续运行，主动监控代码库变化并提出建议
- **状态**：未发布，仅存在于 Feature Flag 中

### 4.2 Undercover Mode — 卧底模式

- **触发条件**：检测到用户为 Anthropic 员工
- **功能**：自动注入系统提示，要求 AI 从 git commit 和 PR 中移除所有 Anthropic 内部信息
- **移除内容**：AI 归因、内部代号（如 Tengu）、内部工具名称
- **争议**：开源社区对此透明度问题提出强烈批评

### 4.3 内部模型代号

| 代号 | 对应模型 | 备注 |
|------|----------|------|
| **Capybara** | Claude 4.6 变体（也称 Mythos） | 内部注释显示已到第 8 版，1M token 上下文 |
| **Tengu** | 另一 Claude 变体 | 在 Undercover Mode 中被列为需要隐藏的词汇 |

---

## 5. 开源改造项目全景

### 5.1 项目概览矩阵

| 项目名 | 语言 | 定位 | 主要改造点 | GitHub 热度 |
|--------|------|------|-----------|------------|
| **OpenCode** | Go | 生产级替代品 | 多模型支持、LSP 集成、隐私优先 | ⭐⭐⭐⭐⭐ 数万 Star |
| **claw-code** | Python | 架构镜像研究 | 完整架构重写、研究导向 | ⭐⭐⭐⭐ 快速增长 |
| **nano-claude-code** | Python | 极简可运行版 | 最小化实现，教学用途 | ⭐⭐⭐ |
| **open-claude-code** (ruvnet) | TypeScript | Clean-room CLI | 基于反编译分析重建 | ⭐⭐⭐ |
| **Claurst** | Rust | 高性能重实现 | 多 Provider、token 高效模式 | ⭐⭐⭐ |
| **Aider** | Python | AI 结对编程 | Git 集成、多 LLM 支持 | ⭐⭐⭐⭐⭐ 成熟项目 |
| **Cline** (原 Claude Dev) | TypeScript | VS Code 插件 | IDE 集成、MCP 支持 | ⭐⭐⭐⭐⭐ |

---

### 5.2 OpenCode — 最具影响力的开源替代

**GitHub**：`opencode-ai/opencode`  
**定位**：生产级、Provider 无关的终端 AI 编程 Agent

#### 核心改造点

**① Provider 无关化**
- 原版：强绑定 Anthropic Claude API
- 改造：支持 75+ LLM Provider（OpenAI、Google Gemini、AWS Bedrock、Groq、Azure、本地模型等）
- 实现：统一 Provider 抽象层，动态切换模型

**② 双 Agent 架构**
- **Plan Agent**（只读分析）：负责理解代码库、制定计划，不执行写操作
- **Build Agent**（执行变更）：负责实际修改文件和执行命令
- 设计价值：分离分析与执行，降低误操作风险

**③ LSP 集成（Language Server Protocol）**
- 原版：无原生 LSP 支持
- 改造：集成 LSP 服务器，提供诊断、错误检查、代码补全等代码智能功能
- 效果：AI 生成的代码质量更高，减少语法错误

**④ 隐私优先架构**
- 代码和数据不经过第三方服务器
- 支持完全本地化部署（配合本地模型）

**⑤ 客户端/服务器架构**
- 支持远程操作模式
- 多会话并发支持

#### 重要事件
2026年1月，Anthropic 实施技术封锁，阻止第三方应用通过消费者订阅认证访问 Claude，导致 OpenCode 部分用户被切断。

---

### 5.3 claw-code — 架构镜像研究项目

**语言**：Python  
**定位**：基于泄漏架构的学术研究级重实现

#### 核心改造点

**① 完整架构镜像**
- 按照泄漏的 TypeScript 架构，用 Python 重新实现所有核心模块
- 保留 QueryEngine、Tool Registry、Memory System 的设计哲学

**② 研究导向扩展**
- 添加了详细的执行日志和可观测性接口
- 便于研究者分析 Agent 决策过程

**③ 模块化解耦**
- 将原版紧耦合的模块拆分为独立可替换组件
- 每个模块可单独测试和替换

---

### 5.4 Claurst — Rust 高性能重实现

**语言**：Rust  
**定位**：追求完整行为覆盖的高性能版本

#### 核心改造点

**① 多 Provider 支持**
- 不绑定 Anthropic，支持任意兼容 OpenAI API 格式的模型

**② Token 高效语音模式（Token-efficient Speech Modes）**
- 实验性功能：优化 token 使用效率的特殊交互模式

**③ Rust 性能优势**
- 更低的内存占用
- 更快的启动速度
- 更好的并发性能

---

### 5.5 Aider — 成熟的 AI 结对编程工具

**GitHub**：`paul-gauthier/aider`  
**语言**：Python  
**定位**：AI 结对编程工具（早于 Claude Code 泄漏，独立发展）

#### 与 Claude Code 的差异化改造

**① Git 深度集成**
- 自动生成有意义的 commit message
- 支持 git blame 感知的代码修改
- 变更历史追踪

**② 多 LLM 支持**
- 支持 Claude、GPT-4、Gemini、本地模型等
- 可配置不同任务使用不同模型

**③ 架构图感知**
- 自动分析代码库结构，生成 repo map
- 帮助 AI 理解大型代码库的整体架构

---

### 5.6 Cline（原 Claude Dev）— VS Code 生态最强

**GitHub**：`cline/cline`  
**语言**：TypeScript  
**定位**：VS Code 插件形态的自主编程 Agent

#### 核心改造点

**① IDE 深度集成**
- 原版：纯终端 CLI
- 改造：VS Code 插件，利用 IDE 的文件树、终端、调试器

**② MCP 生态扩展**
- 完整支持 Model Context Protocol
- 可接入 Notion、Slack、数据库等外部工具

**③ 多 LLM Provider**
- 支持 Anthropic、OpenAI、Google、本地模型
- 用户可自由切换

**④ 可视化审批界面**
- 在 VS Code 侧边栏中展示 Agent 的每一步操作
- 用户可逐步审批或拒绝

---

### 5.7 nano-claude-code — 教学级极简实现

**语言**：Python  
**定位**：最小化可运行实现，用于教学和理解架构

#### 核心价值

- 用最少的代码实现 Claude Code 的核心 Agentic Loop
- 适合学习 Agent 框架设计原理
- 代码量极小，易于阅读和修改

---

## 6. 技术借鉴与设计启示

### 6.1 对企业级 Agent 产品的核心启示

#### ① 工具分层权限模型
**启示**：不要让 Agent 拥有无限权限。三层权限模型（自动/询问/阻断）是平衡效率与安全的最佳实践。

#### ② 记忆系统的分层设计
**启示**：
- 短期（Context Window）+ 中期（Session Memory）+ 长期（CLAUDE.md/MEMORY.md）的分层设计
- Auto Dream 式的定期整合机制，防止记忆退化
- 对于企业差旅 Agent：可设计「行程记忆」「用户偏好记忆」「企业政策记忆」三层

#### ③ Hooks 系统的确定性控制
**启示**：在 Agent 推理之外，保留确定性的控制层。无论 AI 怎么推理，某些安全检查必须执行。

#### ④ 上下文压缩的分级策略
**启示**：不要等到 context 满了才处理。分级压缩（MicroCompact → AutoCompact → Full Compact）是长会话 Agent 的必备设计。

#### ⑤ 双 Agent 分离模式（来自 OpenCode）
**启示**：将「分析规划」和「执行变更」分离到不同 Agent，降低误操作风险。对于差旅 Agent，可设计「需求理解 Agent」和「预订执行 Agent」的分离。

### 6.2 值得警惕的设计问题

| 问题 | Claude Code 的做法 | 更好的做法 |
|------|-------------------|-----------|
| 透明度 | Undercover Mode 隐藏 AI 身份 | 明确披露 AI 参与 |
| 供应商锁定 | 强绑定 Anthropic API | Provider 无关化设计 |
| 权限过度 | BashTool 可执行任意命令 | 最小权限原则 |
| 记忆隐私 | 记忆文件明文存储 | 加密存储 + 访问控制 |

---

*报告生成时间：2026年4月7日*  
*数据来源：泄漏源码分析、开源社区讨论、技术博客、GitHub 项目*
