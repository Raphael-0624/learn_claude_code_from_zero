# Claude Code Multi-Agent 多智能体协作深度解析

> 多 Agent 协作是 Claude Code 最前沿的能力。它让多个 Claude 实例并行工作、相互协调，将原本需要数小时的任务压缩到分钟级别完成。

---

## 一、多 Agent 架构概述

### 1.1 为什么需要多 Agent？

```
单 Agent 的局限性：
├── 上下文窗口有限（200K tokens）
├── 顺序执行，无法并行
├── 一个复杂任务可能耗尽整个上下文
└── 单点故障风险

多 Agent 的优势：
├── 并行执行，大幅提速
├── 每个 Agent 有独立的上下文窗口
├── 专业化分工，提高质量
└── 容错性更强
```

### 1.2 多 Agent 架构全景

```
┌─────────────────────────────────────────────────────────────┐
│                    多 Agent 系统                             │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Coordinator（协调器）                   │   │
│  │  ├── 任务分解                                        │   │
│  │  ├── 子任务分配                                      │   │
│  │  ├── 生命周期管理                                    │   │
│  │  └── 结果合并                                        │   │
│  └──────────────────────┬──────────────────────────────┘   │
│                         │                                   │
│         ┌───────────────┼───────────────┐                  │
│         ▼               ▼               ▼                  │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐           │
│  │  SubAgent  │  │  SubAgent  │  │  SubAgent  │           │
│  │  (前端)    │  │  (后端)    │  │  (测试)    │           │
│  └────────────┘  └────────────┘  └────────────┘           │
│         │               │               │                  │
│  ┌──────▼───────────────▼───────────────▼──────┐          │
│  │              Git Worktree（隔离沙箱）         │          │
│  │  worktree/frontend  worktree/backend  worktree/tests    │
│  └─────────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、SubAgent：单会话内的隔离实例

### 2.1 SubAgent 的特性

```python
# SubAgent 的核心属性
class SubAgent:
    # 独立的上下文窗口（全新的 200K tokens）
    context_window: int = 200_000
    
    # 专门为子任务定制的系统提示
    system_prompt: str
    
    # 只访问需要的工具（最小权限原则）
    tool_access: list[str]
    
    # 从父 Agent 传入的精简上下文
    parent_context: str
    
    # 独立的工作目录（通过 Git Worktree 隔离）
    working_dir: str
    
    def execute(self, task: str) -> str:
        result = self.run_agent_loop(task)
        # 只返回结果摘要，不返回完整上下文
        return self.summarize(result)
```

### 2.2 创建 SubAgent

```bash
# 方式 1：通过 Task 工具创建
claude "创建一个子 Agent 来处理数据库迁移，
        告诉它：使用 Prisma，目标是添加用户角色字段"

# 方式 2：通过 Bash 工具启动新的 Claude 进程
claude "在后台启动一个 Claude 实例来运行测试，
        完成后将结果写入 test-results.json"

# 方式 3：使用 --subagent 标志（实验性）
claude --subagent "分析 src/auth/ 目录的安全漏洞"
```

### 2.3 SubAgent 的上下文隔离

```
主 Agent 上下文（已使用 80%）
├── 大量对话历史
├── 多个文件内容
└── 工具调用历史

                    ↓ 只传递精简摘要

SubAgent 上下文（全新，0% 使用）
├── 精简的任务描述
├── 关键约束（来自父 Agent）
└── 需要处理的具体文件
```

---

## 三、Agent Teams：独立会话协作（实验性）

### 3.1 Agent Teams 概述

Agent Teams 是比 SubAgent 更高级的协作模式，需要 Claude Opus 4.6：

```
SubAgent vs Agent Teams：

SubAgent：
├── 在同一个 Claude Code 进程内
├── 由父 Agent 直接控制
├── 生命周期与父 Agent 绑定
└── 适合：简单的子任务

Agent Teams：
├── 独立的 Claude Code 会话
├── 可以直接相互通信
├── 独立的生命周期
└── 适合：复杂的并行工作流
```

### 3.2 Agent Teams 配置

```json
// .claude/teams.json（实验性配置）
{
  "team": {
    "name": "full-stack-team",
    "model": "claude-opus-4-6",
    "agents": [
      {
        "id": "frontend",
        "role": "前端开发",
        "tools": ["read", "write", "bash"],
        "workdir": "worktrees/frontend"
      },
      {
        "id": "backend",
        "role": "后端开发",
        "tools": ["read", "write", "bash"],
        "workdir": "worktrees/backend"
      },
      {
        "id": "qa",
        "role": "质量保证",
        "tools": ["read", "bash"],
        "workdir": "worktrees/qa"
      }
    ],
    "coordinator": {
      "id": "lead",
      "role": "技术负责人",
      "tools": ["all"]
    }
  }
}
```

### 3.3 Agent Teams 通信

```python
# Agent 间通信协议
class AgentMessage:
    sender: str          # 发送方 Agent ID
    receiver: str        # 接收方 Agent ID（或 "broadcast"）
    type: MessageType    # TASK / RESULT / STATUS / REQUEST
    content: str         # 消息内容
    timestamp: datetime  # 时间戳

# 通信方式 1：共享任务板
class SharedTaskBoard:
    tasks: list[Task]    # 任务列表
    results: dict        # 结果存储
    
    def post_task(self, task: Task):
        self.tasks.append(task)
    
    def claim_task(self, agent_id: str) -> Task:
        # Agent 认领任务
        available = [t for t in self.tasks if t.status == "PENDING"]
        if available:
            task = available[0]
            task.status = "IN_PROGRESS"
            task.assigned_to = agent_id
            return task

# 通信方式 2：直接消息传递
async def send_message(sender: str, receiver: str, content: str):
    message = AgentMessage(sender, receiver, content)
    await message_queue.put(message)
```

---

## 四、Git Worktree：并行执行的基础设施

### 4.1 为什么使用 Git Worktree？

```bash
# 问题：多个 Agent 同时修改同一个文件会产生冲突
# 解决方案：Git Worktree 为每个 Agent 提供独立的工作目录

# 传统方式（有冲突风险）：
# Agent 1 修改 src/auth.ts
# Agent 2 同时修改 src/auth.ts
# → 文件冲突！

# Git Worktree 方式（无冲突）：
# Agent 1 在 worktrees/feature-auth/ 修改 src/auth.ts
# Agent 2 在 worktrees/feature-api/ 修改 src/auth.ts
# → 各自独立，最后合并
```

### 4.2 Git Worktree 配置

```bash
# 为多 Agent 创建 Worktree
git worktree add worktrees/frontend -b feature/frontend
git worktree add worktrees/backend -b feature/backend
git worktree add worktrees/tests -b feature/tests

# 查看所有 Worktree
git worktree list
# 输出：
# /project                    abc1234 [main]
# /project/worktrees/frontend def5678 [feature/frontend]
# /project/worktrees/backend  ghi9012 [feature/backend]
# /project/worktrees/tests    jkl3456 [feature/tests]

# 清理 Worktree
git worktree remove worktrees/frontend
git worktree prune
```

### 4.3 自动化 Worktree 管理

```bash
# Claude 可以自动管理 Worktree
claude "创建三个并行 Agent：
        - Agent 1：在 worktrees/frontend 开发登录页面
        - Agent 2：在 worktrees/backend 开发认证 API
        - Agent 3：在 worktrees/tests 编写集成测试
        完成后合并所有分支"
```

---

## 五、Coordinator 模式

### 5.1 Coordinator 的职责

```python
class Coordinator:
    """主编排器，负责整个多 Agent 工作流"""
    
    def decompose_task(self, task: str) -> list[SubTask]:
        """将大任务分解为可并行的子任务"""
        # 分析任务依赖关系
        # 识别可并行的部分
        # 生成子任务列表
        pass
    
    def assign_tasks(self, subtasks: list[SubTask]) -> dict:
        """将子任务分配给合适的 Agent"""
        assignments = {}
        for task in subtasks:
            agent = self.select_best_agent(task)
            assignments[agent.id] = task
        return assignments
    
    def monitor_progress(self) -> dict:
        """监控所有 Agent 的执行进度"""
        return {
            agent.id: agent.status
            for agent in self.agents
        }
    
    def merge_results(self, results: dict) -> str:
        """合并所有 Agent 的结果"""
        # 解决冲突
        # 整合输出
        # 生成最终报告
        pass
```

### 5.2 任务分解策略

```
任务分解原则：

1. 独立性原则
   ├── 子任务之间尽量减少依赖
   ├── 避免共享可变状态
   └── 通过接口而非实现耦合

2. 粒度原则
   ├── 子任务不能太小（协调开销 > 执行收益）
   ├── 子任务不能太大（失去并行优势）
   └── 理想粒度：30分钟 ~ 2小时的工作量

3. 专业化原则
   ├── 按技术领域分工（前端/后端/测试）
   ├── 按功能模块分工（认证/支付/通知）
   └── 按工作类型分工（开发/测试/文档）
```

### 5.3 实际案例：全栈功能开发

```bash
# 用户请求：开发用户注册功能

# Coordinator 分解任务：
# ┌─────────────────────────────────────────────────────┐
# │ 任务：用户注册功能                                   │
# ├─────────────────────────────────────────────────────┤
# │ SubTask 1（后端 Agent）：                            │
# │   - 创建 POST /api/auth/register 接口               │
# │   - 实现邮箱验证逻辑                                 │
# │   - 密码哈希存储                                     │
# │                                                     │
# │ SubTask 2（前端 Agent）：                            │
# │   - 创建注册表单组件                                 │
# │   - 实现表单验证                                     │
# │   - 调用注册 API                                     │
# │                                                     │
# │ SubTask 3（测试 Agent）：                            │
# │   - 编写 API 单元测试                                │
# │   - 编写前端组件测试                                 │
# │   - 编写 E2E 集成测试                                │
# └─────────────────────────────────────────────────────┘

# 并行执行（3个 Agent 同时工作）
# 预计时间：30分钟（vs 单 Agent 的 90分钟）
```

---

## 六、状态同步机制

### 6.1 持久化检查点

```python
# 多 Agent 系统的检查点
class MultiAgentCheckpoint:
    coordinator_state: CoordinatorState
    agent_states: dict[str, AgentState]
    task_board: SharedTaskBoard
    timestamp: datetime
    
    def save(self):
        path = f".claude/checkpoints/multi-agent-{self.timestamp}.json"
        with open(path, 'w') as f:
            json.dump(self.to_dict(), f)
    
    @classmethod
    def restore(cls, checkpoint_id: str) -> 'MultiAgentCheckpoint':
        # 恢复整个多 Agent 系统的状态
        pass
```

### 6.2 结构化上下文交接

```python
# Agent 间的上下文交接协议
class ContextHandoff:
    """当一个 Agent 完成任务，将上下文传递给下一个 Agent"""
    
    completed_work: str      # 已完成的工作摘要
    key_decisions: list      # 关键决策
    artifacts: list[str]     # 产出物（文件路径）
    pending_issues: list     # 未解决的问题
    next_steps: list         # 建议的下一步
    
    def format_for_agent(self) -> str:
        """格式化为下一个 Agent 可以理解的上下文"""
        return f"""
## 上下文交接

### 已完成工作
{self.completed_work}

### 关键决策
{format_list(self.key_decisions)}

### 产出文件
{format_list(self.artifacts)}

### 待解决问题
{format_list(self.pending_issues)}

### 建议下一步
{format_list(self.next_steps)}
"""
```

### 6.3 共享任务板

```markdown
<!-- .claude/task-board.md（共享任务板示例）-->
# 任务板 - 用户注册功能

## 进行中
- [x] 后端 API 设计（后端 Agent，完成于 14:30）
- [ ] 前端表单组件（前端 Agent，进行中）
- [ ] 集成测试（测试 Agent，等待后端完成）

## 已完成
- [x] 数据库 Schema 设计
- [x] API 接口文档

## 阻塞
- ⚠️ 邮件服务配置（需要 DevOps 提供 SMTP 凭证）

## 决策记录
- 使用 bcrypt（cost factor 12）进行密码哈希
- 邮箱验证链接有效期 24 小时
- 注册成功后自动登录
```

---

## 七、操作分类：并发 vs 顺序

### 7.1 可并发操作

```python
# 只读操作：可以完全并发
CONCURRENT_SAFE = [
    "read_file",          # 读取文件
    "grep_search",        # 搜索代码
    "git_status",         # 查看 git 状态
    "run_tests",          # 运行测试（只读）
    "analyze_code",       # 代码分析
    "generate_docs",      # 生成文档
]

# 示例：并行分析多个模块
agents = [
    Agent("分析 src/auth/ 的安全漏洞"),
    Agent("分析 src/api/ 的性能问题"),
    Agent("分析 src/db/ 的查询效率"),
]
results = await asyncio.gather(*[a.run() for a in agents])
```

### 7.2 需要顺序执行的操作

```python
# 有状态操作：需要顺序执行或加锁
SEQUENTIAL_REQUIRED = [
    "write_file",         # 写入文件（可能冲突）
    "git_commit",         # git 提交（需要顺序）
    "database_migrate",   # 数据库迁移（有依赖）
    "deploy",             # 部署（需要等待构建）
    "install_deps",       # 安装依赖（可能冲突）
]

# 使用锁机制避免冲突
async with file_lock("src/config.ts"):
    await agent.write_file("src/config.ts", content)
```

---

## 八、性能对比

### 8.1 单 Agent vs 多 Agent

| 任务类型 | 单 Agent | 多 Agent（3个） | 提速比 |
|---------|---------|----------------|--------|
| 全栈功能开发 | 90 分钟 | 35 分钟 | 2.6x |
| 代码库安全审计 | 120 分钟 | 40 分钟 | 3x |
| 大规模重构 | 180 分钟 | 60 分钟 | 3x |
| 测试套件编写 | 60 分钟 | 25 分钟 | 2.4x |
| 文档生成 | 45 分钟 | 15 分钟 | 3x |

### 8.2 成本分析

```
多 Agent 的成本考量：

优势：
├── 总时间减少（用户等待时间短）
├── 每个 Agent 上下文更小（token 效率高）
└── 并行执行减少总 API 调用时间

劣势：
├── 多个 Agent 同时消耗 token
├── 协调开销（Coordinator 的 token 消耗）
└── 上下文交接的 token 消耗

结论：
├── 对于大型任务（>2小时），多 Agent 更经济
└── 对于小型任务（<30分钟），单 Agent 更经济
```

---

## 九、最佳实践

### 9.1 何时使用多 Agent？

```bash
# 适合多 Agent 的场景：
# ✅ 任务可以明确分解为独立子任务
# ✅ 子任务之间依赖关系少
# ✅ 总工作量 > 2小时
# ✅ 需要专业化分工（前端/后端/测试）

# 不适合多 Agent 的场景：
# ❌ 任务高度耦合，无法并行
# ❌ 总工作量 < 30分钟
# ❌ 需要频繁的人工决策
# ❌ 资源受限（API 配额不足）
```

### 9.2 Coordinator 设计原则

```bash
# 好的 Coordinator 设计：
# 1. 清晰的任务分解（每个子任务有明确的输入/输出）
# 2. 最小化 Agent 间依赖
# 3. 定义清晰的接口（API 契约、文件格式）
# 4. 建立检查点，支持失败恢复

# 示例：好的任务分解
claude "将以下任务分配给三个并行 Agent：
        Agent 1（后端）：实现 POST /api/users 接口，
                         输出：API 文档 + 实现代码
        Agent 2（前端）：实现用户列表页面，
                         输入：Agent 1 的 API 文档
                         输出：React 组件
        Agent 3（测试）：编写测试，
                         输入：Agent 1 的 API 文档
                         输出：测试套件"
```

### 9.3 错误处理策略

```python
# 多 Agent 系统的错误处理
class MultiAgentErrorHandler:
    
    def handle_agent_failure(self, agent_id: str, error: Exception):
        # 策略 1：重试失败的 Agent
        if self.can_retry(agent_id):
            self.retry_agent(agent_id)
        
        # 策略 2：将任务重新分配给其他 Agent
        elif self.has_available_agent():
            self.reassign_task(agent_id)
        
        # 策略 3：降级为单 Agent 模式
        else:
            self.fallback_to_single_agent()
        
        # 策略 4：通知用户
        self.notify_user(f"Agent {agent_id} 失败，正在恢复...")
```

---

## 十、常见问题

### Q: 多 Agent 需要额外付费吗？

多 Agent 会消耗更多的 API token（因为多个 Agent 同时运行），但不需要额外的订阅费用。成本取决于总 token 消耗量。

### Q: Agent Teams 功能现在可以使用吗？

Agent Teams 目前是实验性功能，需要 Claude Opus 4.6 模型。普通用户可以通过 SubAgent 机制实现类似的并行工作流。

### Q: 如何防止多个 Agent 产生冲突？

1. 使用 Git Worktree 为每个 Agent 提供独立工作目录
2. 明确定义每个 Agent 的文件访问范围
3. 使用共享任务板协调工作
4. 在合并阶段使用 Coordinator 解决冲突

### Q: 多 Agent 系统崩溃后如何恢复？

```bash
# 查看多 Agent 检查点
claude /checkpoints --multi-agent

# 恢复到最近的检查点
claude --resume-multi-agent latest

# 查看各 Agent 的状态
claude /agent-status
```

---

*本文档基于 Claude Code 泄漏源码分析及官方文档整理，部分内部实现细节可能随版本更新而变化。*
