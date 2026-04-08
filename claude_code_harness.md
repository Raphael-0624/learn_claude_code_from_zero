# Claude Code Harness 执行引擎深度解析

> **Harness** 是 Claude Code 的核心基础设施层，充当 AI Agent 与真实世界之间的操作系统。它负责沙箱执行、权限控制、工具调用编排和会话状态管理。

---

## 一、什么是 Harness？

Harness（内部代号：**Tengu**）是 Claude Code 的执行引擎，位于 Claude LLM 与真实计算环境之间。它不是一个单一组件，而是一套完整的基础设施层，包含：

- **沙箱执行环境**：安全隔离地运行 Python/Bash 代码
- **权限控制系统**：三层权限模型，精细管控工具访问
- **工具调用编排器**：将 LLM 的意图转化为真实操作
- **会话状态管理器**：跨工具调用维护上下文一致性

```
┌─────────────────────────────────────────────────────┐
│                   Claude LLM                        │
│              (意图 / 推理 / 规划)                    │
└─────────────────────┬───────────────────────────────┘
                      │ 工具调用请求
                      ▼
┌─────────────────────────────────────────────────────┐
│                  Harness 层                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │
│  │ 权限控制 │  │ 工具注册 │  │   沙箱执行环境   │  │
│  │  系统    │  │  中心    │  │  (Bash/Python)   │  │
│  └──────────┘  └──────────┘  └──────────────────┘  │
│  ┌──────────────────────────────────────────────┐   │
│  │           会话状态管理器                      │   │
│  └──────────────────────────────────────────────┘   │
└─────────────────────┬───────────────────────────────┘
                      │ 执行结果
                      ▼
┌─────────────────────────────────────────────────────┐
│              真实计算环境                            │
│   文件系统 / 终端 / 网络 / 外部 API / MCP 服务      │
└─────────────────────────────────────────────────────┘
```

---

## 二、沙箱执行环境

### 2.1 执行隔离机制

Harness 为每个工具调用创建隔离的执行上下文：

```bash
# Bash 工具执行示例
# Harness 会在受控环境中执行，并捕获所有输出
$ claude -p "运行测试并报告结果"

# 内部执行流程：
# 1. 解析意图 → 确定需要 bash 工具
# 2. 权限检查 → 是否允许执行 bash
# 3. 沙箱执行 → 在隔离环境中运行命令
# 4. 结果捕获 → stdout/stderr/exit code
# 5. 返回 LLM → 将结果注入上下文
```

### 2.2 执行超时与资源限制

| 参数 | 默认值 | 说明 |
|------|--------|------|
| 命令超时 | 120 秒 | 单个 bash 命令最长执行时间 |
| 输出截断 | ~100KB | 超长输出自动截断 |
| 并发工具 | 受限 | 防止资源竞争 |
| 网络访问 | 可配置 | 默认允许，可通过权限控制 |

### 2.3 环境变量传递

```bash
# Harness 会将以下环境变量注入执行环境：
CLAUDE_CODE_SESSION_ID=<session-id>
CLAUDE_CODE_WORKING_DIR=<project-root>
CLAUDE_CODE_TOOL_TIMEOUT=120

# 用户可以通过 .env 文件或 shell 配置传递自定义变量
export MY_API_KEY="..."
claude  # Harness 会继承父进程的环境变量
```

---

## 三、权限控制系统

### 3.1 三层权限模型

Harness 实现了精细的三层权限控制：

```
Layer 1: Always Deny（永远拒绝）
├── 删除系统文件
├── 修改 /etc、/sys 等系统目录
├── 访问其他用户的私有数据
└── 执行已知危险命令（rm -rf /）

Layer 2: Always Allow（永远允许）
├── 读取项目文件
├── 运行测试命令
├── 查看 git 状态
└── 搜索代码库

Layer 3: User-Gated（用户审批）
├── 写入/修改文件
├── 执行网络请求
├── 安装依赖包
└── 运行构建脚本
```

### 3.2 Auto Mode 的 AI 分类器

在 `--dangerously-skip-permissions` 或 Auto Mode 下，Harness 使用内置 AI 分类器自动判断操作风险：

```python
# 伪代码：AI 分类器逻辑
def classify_tool_call(tool_name, tool_args):
    risk_factors = [
        is_destructive(tool_args),      # 是否破坏性操作
        affects_system_files(tool_args), # 是否影响系统文件
        requires_network(tool_args),     # 是否需要网络
        modifies_state(tool_args),       # 是否修改状态
    ]
    
    risk_score = calculate_risk(risk_factors)
    
    if risk_score > THRESHOLD_HIGH:
        return "DENY"
    elif risk_score > THRESHOLD_MEDIUM:
        return "REQUEST_APPROVAL"
    else:
        return "ALLOW"
```

### 3.3 权限配置文件

```json
// .claude/settings.json
{
  "permissions": {
    "allow": [
      "bash(npm test)",
      "bash(git *)",
      "read(*)",
      "write(src/**)"
    ],
    "deny": [
      "bash(rm -rf *)",
      "bash(sudo *)",
      "write(/etc/**)"
    ]
  }
}
```

---

## 四、Programmatic Tool Calling（PTC）

### 4.1 什么是 PTC？

PTC 是 Harness 的高级特性，允许 Claude 编写代码来**编排多个工具调用**，而不是逐个调用工具：

```python
# 传统方式：逐个工具调用（高延迟，高 token 消耗）
# 调用 1: read("src/main.py")
# 调用 2: read("src/utils.py")  
# 调用 3: grep("TODO", "src/")
# 调用 4: write("report.md", ...)

# PTC 方式：一次性编排（低延迟，低 token 消耗）
files = [read(f) for f in glob("src/**/*.py")]
todos = grep("TODO", files)
write("report.md", format_report(todos))
```

### 4.2 PTC 的性能优势

| 指标 | 传统方式 | PTC 方式 | 提升 |
|------|---------|---------|------|
| 工具调用次数 | N 次 | 1 次 | N倍 |
| 往返延迟 | N × RTT | 1 × RTT | N倍 |
| Token 消耗 | 高（每次调用都有开销） | 低 | 30-60% |
| 并行执行 | 不支持 | 支持 | 显著 |

### 4.3 PTC 的限制

- 只能在 Harness 支持的工具上使用
- 代码必须是确定性的（不能有随机行为）
- 执行时间受沙箱超时限制
- 不能访问 Harness 外部的资源

---

## 五、会话状态管理

### 5.1 状态持久化机制

```
会话状态存储位置：
~/.claude/
├── sessions/
│   ├── <session-id>/
│   │   ├── context.json      # 当前上下文快照
│   │   ├── tool_history.json # 工具调用历史
│   │   └── checkpoints/      # 自动检查点
│   └── ...
├── settings.json             # 全局配置
└── MEMORY.md                 # 跨会话记忆
```

### 5.2 自动检查点

Harness 在关键操作前自动创建检查点：

```bash
# 检查点触发条件：
# 1. 写入文件前
# 2. 执行破坏性命令前
# 3. 上下文压缩前
# 4. 会话超时前

# 恢复检查点：
claude --resume <checkpoint-id>

# 查看可用检查点：
claude --list-checkpoints
```

### 5.3 跨工具状态共享

```python
# Harness 维护工具间的共享状态
class HarnessState:
    working_directory: str      # 当前工作目录
    environment_vars: dict      # 环境变量
    file_cache: dict            # 已读取文件缓存
    tool_results: list          # 工具调用结果历史
    pending_approvals: list     # 待审批操作队列
```

---

## 六、Harness 与 Agent Loop 的关系

```
Agent Loop（高层）
    │
    ├── Think: Claude 分析情况，决定下一步
    │
    ├── Act: 向 Harness 发送工具调用请求
    │         │
    │         ├── Harness 权限检查
    │         ├── Harness 沙箱执行
    │         └── Harness 返回结果
    │
    ├── Observe: Claude 接收 Harness 返回的结果
    │
    └── Decide: 继续循环 or 完成任务
```

Harness 是 Agent Loop 的执行层，Agent Loop 是 Harness 的调用者。两者共同构成 Claude Code 的核心运行机制。

---

## 七、调试与监控

### 7.1 Harness 调试模式

```bash
# 启用详细日志
claude --verbose

# 查看工具调用详情
claude --debug-tools

# 输出 Harness 执行追踪
CLAUDE_DEBUG=harness claude

# 查看权限决策日志
CLAUDE_DEBUG=permissions claude
```

### 7.2 关键指标监控

```bash
# 查看当前会话统计
claude /status

# 输出示例：
# Session ID: abc123
# Tools called: 47
# Tokens used: 85,432 / 200,000
# Permissions granted: 12
# Permissions denied: 2
# Checkpoints: 5
```

---

## 八、最佳实践

### 8.1 权限最小化原则

```json
// 推荐：精确指定允许的操作
{
  "permissions": {
    "allow": [
      "bash(npm test)",
      "bash(npm run lint)",
      "write(src/**/*.ts)"
    ]
  }
}

// 不推荐：过于宽泛的权限
{
  "permissions": {
    "allow": ["bash(*)", "write(**)"]
  }
}
```

### 8.2 利用 PTC 提升效率

```bash
# 对于批量操作，明确告诉 Claude 使用并行处理
claude "并行读取所有测试文件，找出失败的测试用例"

# 对于复杂工作流，使用 Plan Mode 先规划
claude --plan "重构整个认证模块"
```

### 8.3 检查点策略

```bash
# 在重要操作前手动创建检查点
claude /checkpoint "重构前的状态"

# 定期清理旧检查点
claude /cleanup-checkpoints --older-than 7d
```

---

## 九、常见问题

### Q: Harness 如何防止 Claude 执行危险命令？

Harness 使用多层防护：
1. **静态规则**：预定义的危险命令黑名单
2. **AI 分类器**：动态评估命令风险
3. **用户审批**：高风险操作需要明确确认
4. **沙箱隔离**：即使执行也在受控环境中

### Q: 为什么有时候 Claude 会拒绝执行某些操作？

可能原因：
- 操作被权限系统标记为高风险
- 当前处于 Plan Mode（只读状态）
- 操作超出了配置的权限范围
- Harness 检测到潜在的破坏性影响

### Q: 如何在 CI/CD 中使用 Harness？

```bash
# CI/CD 推荐配置
claude --dangerously-skip-permissions \
       --output-format json \
       --max-turns 10 \
       -p "运行完整测试套件并生成报告"
```

---

*本文档基于 Claude Code 泄漏源码分析及官方文档整理，部分内部实现细节可能随版本更新而变化。*
