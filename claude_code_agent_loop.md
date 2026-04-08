# Claude Code Agent Loop 执行循环深度解析

> Agent Loop 是 Claude Code 的心脏。它定义了 Claude 如何感知环境、制定计划、采取行动、观察结果，并在循环中不断推进任务直到完成。

---

## 一、Agent Loop 概述

### 1.1 核心循环

Claude Code 的 Agent Loop 遵循经典的 **Think → Act → Observe → Decide** 循环，内部代号为 `nO`：

```
┌─────────────────────────────────────────────────────────────┐
│                    Agent Loop (nO)                          │
│                                                             │
│    ┌──────────┐                                             │
│    │  Think   │ ← 分析当前状态，决定下一步行动              │
│    └────┬─────┘                                             │
│         │                                                   │
│         ▼                                                   │
│    ┌──────────┐                                             │
│    │   Act    │ → 调用工具（bash/read/write/grep...）        │
│    └────┬─────┘                                             │
│         │                                                   │
│         ▼                                                   │
│    ┌──────────┐                                             │
│    │ Observe  │ ← 接收工具执行结果                          │
│    └────┬─────┘                                             │
│         │                                                   │
│         ▼                                                   │
│    ┌──────────┐                                             │
│    │  Decide  │ → 继续循环 or 完成任务 or 请求帮助          │
│    └──────────┘                                             │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 ReAct 模式

Agent Loop 实现了 ReAct（Reasoning + Acting）模式，推理与行动交织进行：

```python
# ReAct 模式伪代码
def agent_loop(task: str) -> str:
    context = [task]
    
    while True:
        # Think: 推理当前状态
        thought = claude.think(context)
        context.append(f"Thought: {thought}")
        
        # Act: 决定行动
        action = claude.decide_action(context)
        
        if action.type == "FINISH":
            return action.result
        
        if action.type == "TOOL_CALL":
            # Execute: 执行工具
            result = harness.execute(action.tool, action.args)
            context.append(f"Observation: {result}")
        
        if action.type == "ASK_USER":
            # 请求用户输入
            user_input = get_user_input(action.question)
            context.append(f"User: {user_input}")
```

---

## 二、Plan Mode：先规划，再执行

### 2.1 Plan Mode 的工作原理

Plan Mode 是 2026 年 1 月引入的重要特性，让 Claude 在执行前先制定完整计划：

```bash
# 启用 Plan Mode
claude --plan "重构整个认证模块"

# Plan Mode 的执行流程：
# Phase 1: 只读探索（不修改任何文件）
#   ├── 读取相关文件
#   ├── 分析代码结构
#   └── 识别依赖关系
#
# Phase 2: 生成计划
#   ├── 列出所有需要修改的文件
#   ├── 描述每个修改的内容
#   └── 估计工作量和风险
#
# Phase 3: 用户确认
#   └── 展示计划，等待用户批准
#
# Phase 4: 执行（用户批准后）
#   └── 按计划逐步执行
```

### 2.2 Plan Mode 示例

```
用户：claude --plan "将所有 API 路由从 Express 迁移到 Fastify"

Claude 的计划输出：
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📋 迁移计划：Express → Fastify

分析结果：
- 找到 23 个 API 路由文件
- 找到 5 个中间件文件
- 找到 2 个错误处理文件

计划步骤：
1. 安装 Fastify 依赖（npm install fastify @fastify/cors）
2. 创建 Fastify 应用入口（src/app.ts）
3. 迁移 23 个路由文件（逐个转换语法）
4. 迁移 5 个中间件（转换为 Fastify 插件）
5. 更新错误处理（适配 Fastify 错误格式）
6. 更新测试文件（更新 supertest 配置）
7. 运行测试验证

预计修改：31 个文件
风险评估：中等（需要测试验证）

是否继续执行？[y/N]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### 2.3 Tasks：持久化任务（2026年1月新增）

```bash
# 创建持久化任务
claude /task create "重构认证模块"

# 任务会被保存到 .claude/tasks/ 目录
# 即使会话中断，任务状态也会保留

# 查看任务列表
claude /tasks

# 继续未完成的任务
claude /task resume <task-id>

# 任务状态：
# PENDING → IN_PROGRESS → COMPLETED / FAILED
```

---

## 三、执行阶段详解

### 3.1 阶段一：收集上下文

```python
# 执行前的上下文收集
def collect_context(task: str) -> Context:
    context = Context()
    
    # 1. 加载项目配置
    context.load_claude_md()
    context.load_memory_md()
    
    # 2. 分析任务，确定需要读取的文件
    relevant_files = analyze_task(task)
    for file in relevant_files:
        context.add_file(read(file))
    
    # 3. 收集环境信息
    context.add_env_info({
        "cwd": os.getcwd(),
        "git_status": git_status(),
        "node_version": node_version(),
    })
    
    return context
```

### 3.2 阶段二：采取行动

```python
# 行动执行循环
def execute_actions(plan: Plan, context: Context):
    for step in plan.steps:
        # 执行前检查
        if not can_execute(step, context):
            request_user_approval(step)
        
        # 执行工具调用
        result = harness.execute(step.tool, step.args)
        
        # 更新上下文
        context.add_result(result)
        
        # 验证结果
        if not verify_result(result, step.expected):
            handle_failure(step, result)
```

### 3.3 阶段三：验证结果

```bash
# Claude 会自动验证执行结果
# 示例：修改代码后自动运行测试

claude "修复登录 bug"
# 内部流程：
# 1. 读取相关文件
# 2. 修改代码
# 3. 自动运行测试（验证修复有效）
# 4. 如果测试失败，自动回滚并重试
```

---

## 四、错误恢复机制

### 4.1 Self-Healing Query Loop

Claude Code 实现了自愈查询循环，能够自动从错误中恢复：

```python
# 自愈循环（伪代码）
def self_healing_loop(task: str, max_retries: int = 3):
    for attempt in range(max_retries):
        try:
            result = execute_task(task)
            return result
            
        except ContextOverflowError:
            # 上下文溢出：触发压缩
            compact_context()
            continue
            
        except TokenLimitError:
            # Token 超限：升级到更大模型
            escalate_to_larger_model()
            continue
            
        except ToolExecutionError as e:
            # 工具执行失败：分析错误，调整策略
            new_strategy = analyze_error(e)
            task = apply_strategy(task, new_strategy)
            continue
            
        except ModelError:
            # 模型错误：降级到备用模型
            fallback_to_backup_model()
            continue
    
    raise MaxRetriesExceeded(f"任务在 {max_retries} 次尝试后仍然失败")
```

### 4.2 错误类型与恢复策略

| 错误类型 | 触发条件 | 恢复策略 |
|---------|---------|---------|
| ContextOverflowError | 上下文 > 90% | 触发 Full Compact |
| TokenLimitError | 单次请求超限 | 升级到 Opus 模型 |
| ToolExecutionError | 工具调用失败 | 分析错误，调整参数 |
| ModelError | API 错误 | 指数退避重试 |
| PermissionError | 权限不足 | 请求用户授权 |
| TimeoutError | 执行超时 | 分解任务，逐步执行 |

### 4.3 指数退避重试

```python
# 指数退避重试策略
def retry_with_backoff(func, max_retries=5):
    for attempt in range(max_retries):
        try:
            return func()
        except RetryableError as e:
            if attempt == max_retries - 1:
                raise
            
            # 指数退避：1s, 2s, 4s, 8s, 16s
            wait_time = 2 ** attempt
            # 加入随机抖动，避免雷群效应
            jitter = random.uniform(0, wait_time * 0.1)
            time.sleep(wait_time + jitter)
```

---

## 五、自动检查点

### 5.1 检查点触发时机

```python
# 自动检查点创建逻辑
CHECKPOINT_TRIGGERS = [
    "before_file_write",      # 写入文件前
    "before_destructive_cmd", # 执行破坏性命令前
    "before_context_compact", # 上下文压缩前
    "after_major_milestone",  # 完成重要里程碑后
    "on_session_timeout",     # 会话超时前
]

def create_checkpoint(trigger: str, state: AgentState):
    checkpoint = {
        "id": generate_id(),
        "trigger": trigger,
        "timestamp": now(),
        "state": serialize(state),
        "context_snapshot": state.context.snapshot(),
    }
    save_checkpoint(checkpoint)
```

### 5.2 检查点恢复

```bash
# 查看可用检查点
claude --list-checkpoints

# 输出示例：
# ID          时间                触发原因              状态
# cp_001      2026-04-07 14:30   before_file_write     可用
# cp_002      2026-04-07 14:45   after_major_milestone 可用
# cp_003      2026-04-07 15:00   before_context_compact 可用

# 恢复到特定检查点
claude --resume cp_002

# 恢复到最近的检查点
claude --resume latest
```

---

## 六、并发与并行执行

### 6.1 并行工具调用

```python
# Agent Loop 支持并行工具调用
# 当多个工具调用相互独立时，并行执行

# 示例：并行读取多个文件
parallel_calls = [
    ToolCall("read", {"path": "src/auth.ts"}),
    ToolCall("read", {"path": "src/user.ts"}),
    ToolCall("read", {"path": "src/db.ts"}),
]

# 并行执行，等待所有结果
results = await asyncio.gather(*[
    harness.execute(call) for call in parallel_calls
])
```

### 6.2 任务分解与并行

```bash
# 对于大型任务，Claude 会自动分解并并行处理
claude "分析整个代码库的安全漏洞"

# 内部：
# 1. 分解为子任务：
#    - 分析认证模块
#    - 分析 API 路由
#    - 分析数据库查询
#    - 分析依赖包
# 2. 并行执行子任务（使用多个子 Agent）
# 3. 合并结果，生成综合报告
```

---

## 七、Agent Loop 的状态机

### 7.1 状态转换图

```
                    ┌─────────┐
                    │  IDLE   │
                    └────┬────┘
                         │ 接收任务
                         ▼
                    ┌─────────┐
                    │PLANNING │ ← Plan Mode
                    └────┬────┘
                         │ 计划完成
                         ▼
                    ┌─────────┐
              ┌────►│EXECUTING│
              │     └────┬────┘
              │          │ 工具调用
              │          ▼
              │     ┌─────────┐
              │     │OBSERVING│
              │     └────┬────┘
              │          │ 分析结果
              │          ▼
              │     ┌─────────┐
              │     │DECIDING │
              │     └────┬────┘
              │          │
              │    ┌─────┴──────┐
              │    │            │
              │ 继续执行    任务完成
              └────┘            │
                                ▼
                           ┌─────────┐
                           │COMPLETED│
                           └─────────┘
```

### 7.2 状态持久化

```python
# Agent 状态持久化到磁盘
class AgentState:
    status: AgentStatus          # 当前状态
    task: str                    # 原始任务
    plan: Plan                   # 执行计划
    completed_steps: list        # 已完成步骤
    pending_steps: list          # 待执行步骤
    context: Context             # 当前上下文
    checkpoints: list            # 检查点列表
    
    def save(self):
        # 序列化并保存到 .claude/sessions/<id>/state.json
        with open(self.state_file, 'w') as f:
            json.dump(self.to_dict(), f)
    
    @classmethod
    def load(cls, session_id: str) -> 'AgentState':
        # 从磁盘恢复状态
        with open(f'.claude/sessions/{session_id}/state.json') as f:
            return cls.from_dict(json.load(f))
```

---

## 八、调试 Agent Loop

### 8.1 详细日志模式

```bash
# 启用 Agent Loop 详细日志
claude --verbose

# 输出示例：
# [14:30:01] THINK: 分析任务：修复登录 bug
# [14:30:02] ACT: 调用工具 read(src/auth/login.ts)
# [14:30:02] OBSERVE: 读取成功，文件大小 2.3KB
# [14:30:03] THINK: 发现问题：密码比较使用了 == 而不是 bcrypt.compare
# [14:30:04] ACT: 调用工具 edit(src/auth/login.ts, ...)
# [14:30:04] OBSERVE: 文件修改成功
# [14:30:05] ACT: 调用工具 bash(npm test -- auth)
# [14:30:08] OBSERVE: 测试通过（3/3）
# [14:30:08] DECIDE: 任务完成
```

### 8.2 中断和恢复

```bash
# 中断正在执行的任务
Ctrl+C

# 恢复中断的任务
claude --resume

# 查看任务历史
claude /history
```

---

## 九、最佳实践

### 9.1 任务描述优化

```bash
# 不好：模糊的任务描述
claude "修复 bug"

# 好：具体的任务描述
claude "修复 src/auth/login.ts 中的密码验证 bug，
        确保使用 bcrypt.compare 而不是直接比较，
        并添加相应的单元测试"

# 最好：提供上下文和约束
claude "修复登录 bug（见 issue #123），
        使用 bcrypt.compare 进行密码验证，
        保持现有的 JWT 认证流程不变，
        添加测试覆盖正常和异常情况"
```

### 9.2 利用 Plan Mode

```bash
# 对于复杂任务，始终使用 Plan Mode
claude --plan "重构数据库层"

# 审查计划后再执行
# 这样可以：
# 1. 提前发现潜在问题
# 2. 调整执行策略
# 3. 避免不必要的修改
```

### 9.3 设置合理的 max-turns

```bash
# 对于简单任务，限制循环次数
claude --max-turns 5 "修复这个小 bug"

# 对于复杂任务，允许更多循环
claude --max-turns 50 "重构整个模块"

# 默认值通常是 10-20 轮
```

---

## 十、常见问题

### Q: Agent Loop 会无限循环吗？

不会。Claude Code 有多重保护机制：
1. `--max-turns` 限制最大循环次数
2. 上下文窗口限制（满了就必须压缩或结束）
3. 任务完成检测（Claude 会判断何时任务已完成）
4. 用户可以随时中断（Ctrl+C）

### Q: 为什么 Claude 有时候会"卡住"？

可能原因：
- 等待用户审批某个操作
- 工具执行时间较长（如运行测试）
- 上下文压缩正在进行
- 遇到了需要用户决策的情况

### Q: 如何让 Agent Loop 更高效？

1. 提供清晰的任务描述
2. 使用 Plan Mode 避免无效尝试
3. 将大任务分解为小任务
4. 使用 CLAUDE.md 提供项目上下文
5. 合理设置权限，减少审批中断

---

*本文档基于 Claude Code 泄漏源码分析及官方文档整理，部分内部实现细节可能随版本更新而变化。*
