# Claude Code Context 上下文系统深度解析

> **Context Window** 是 Claude Code 的工作记忆。理解它如何工作、如何压缩、如何腐烂，是高效使用 Claude Code 的关键。

---

## 一、什么是上下文窗口？

上下文窗口（Context Window）是 Claude 在单次会话中能够"看到"的全部信息。它是一个有限的 token 空间，包含：

```
┌─────────────────────────────────────────────────────────┐
│                    上下文窗口（200K tokens）              │
│                                                         │
│  ┌─────────────────┐  ┌─────────────────────────────┐  │
│  │   系统提示       │  │        对话历史              │  │
│  │  (~5K tokens)   │  │   用户消息 + Claude 回复     │  │
│  └─────────────────┘  └─────────────────────────────┘  │
│                                                         │
│  ┌─────────────────┐  ┌─────────────────────────────┐  │
│  │   文件内容       │  │        工具输出              │  │
│  │  (读取的代码)    │  │   bash/grep/read 结果       │  │
│  └─────────────────┘  └─────────────────────────────┘  │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │              CLAUDE.md / MEMORY.md               │   │
│  │              (项目指令 + 记忆文件)                │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

---

## 二、上下文窗口规格

### 2.1 不同模型的窗口大小

| 模型 | 上下文窗口 | 压缩触发阈值 | 压缩缓冲区 |
|------|-----------|------------|----------|
| Claude 3.5 Sonnet | 200K tokens | ~128K (64%) | ~33K tokens |
| Claude 3.7 Sonnet | 200K tokens | ~150K (75%) | ~33K tokens |
| Claude 3 Opus | 200K tokens | ~128K (64%) | ~33K tokens |
| Claude 3.5 Haiku | 200K tokens | ~128K (64%) | ~20K tokens |
| 未来模型（预期） | 1M tokens | ~420K (42%) | ~100K tokens |

### 2.2 Token 消耗分布

```
典型项目会话的 token 分布：

系统提示          ████░░░░░░░░░░░░░░░░  5%  (~10K)
CLAUDE.md         ████░░░░░░░░░░░░░░░░  5%  (~10K)
对话历史          ████████░░░░░░░░░░░░  20% (~40K)
文件内容          ████████████░░░░░░░░  30% (~60K)
工具输出          ████████████████░░░░  40% (~80K)
```

---

## 三、上下文压缩机制

### 3.1 三级压缩策略

Claude Code 实现了三级渐进式压缩：

```
Level 1: MicroCompact（微压缩）
├── 触发条件：上下文使用率 > 50%
├── 操作：裁剪冗余工具输出、截断过长响应
├── Token 节省：10-20%
└── 信息损失：极低

Level 2: AutoCompact（自动压缩）
├── 触发条件：上下文使用率 > 64-75%
├── 操作：生成对话摘要，替换原始历史
├── Token 节省：40-60%
└── 信息损失：低（关键信息保留）

Level 3: Full Compact（完全压缩）
├── 触发条件：上下文使用率 > 90%（紧急）
├── 操作：激进压缩，只保留最关键信息
├── Token 节省：70-80%
└── 信息损失：中（部分细节丢失）
```

### 3.2 AutoCompact 工作原理

```python
# AutoCompact 的内部逻辑（伪代码）
def auto_compact(context):
    # 1. 识别可压缩内容
    compressible = [
        msg for msg in context.history
        if msg.age > RECENT_THRESHOLD  # 非最近消息
        and not msg.is_critical        # 非关键信息
    ]
    
    # 2. 生成摘要
    summary = claude.summarize(
        compressible,
        preserve=["decisions", "errors", "key_facts"],
        compress=["verbose_output", "repeated_content"]
    )
    
    # 3. 替换原始内容
    context.replace(compressible, summary)
    
    # 4. 保留最近 N 轮完整对话
    context.keep_recent(turns=5)
    
    return context
```

### 3.3 压缩触发时机

```bash
# 查看当前上下文使用情况
claude /status

# 输出示例：
# Context: 142,847 / 200,000 tokens (71.4%)
# ⚠️  AutoCompact 将在 ~5,000 tokens 后触发

# 手动触发压缩
claude /compact

# 强制完整压缩
claude /compact --full
```

---

## 四、"中间遗失"问题

### 4.1 注意力分布不均

研究表明，Claude 对上下文的注意力分布呈 U 形曲线：

```
注意力强度
    │
100%│ ████                              ████
 80%│ ████                              ████
 60%│ ████ ████                    ████ ████
 40%│ ████ ████ ████          ████ ████ ████
 20%│ ████ ████ ████ ████ ████ ████ ████ ████
  0%└─────────────────────────────────────────
    开头                                结尾
    (系统提示)    中间内容容易遗忘    (最近对话)
```

### 4.2 实际影响

```bash
# 问题场景：长会话中的信息遗忘
# 第 1 轮：用户说"使用 PostgreSQL 数据库"
# 第 50 轮：Claude 可能忘记这个约束，建议使用 SQLite

# 解决方案 1：将关键约束写入 CLAUDE.md
echo "## 技术约束\n- 数据库：PostgreSQL（不要使用其他数据库）" >> CLAUDE.md

# 解决方案 2：在关键决策时重申约束
claude "记住我们使用 PostgreSQL，现在帮我优化这个查询..."

# 解决方案 3：使用 /compact 重置上下文，保留关键信息
claude /compact --preserve "数据库使用 PostgreSQL"
```

### 4.3 最佳实践：关键信息置顶

```markdown
<!-- CLAUDE.md 结构建议 -->
# 项目关键约束（最重要的信息放最前面）

## 绝对约束
- 数据库：PostgreSQL 14+
- 语言：TypeScript（严格模式）
- 框架：Next.js 14（App Router）

## 代码规范
...（次要信息）

## 背景信息
...（可以被遗忘的信息）
```

---

## 五、上下文腐烂（Context Rot）

### 5.1 什么是上下文腐烂？

随着会话进行，上下文中积累了大量历史信息，导致：

1. **信噪比下降**：有用信息被大量历史淹没
2. **注意力分散**：Claude 难以聚焦当前任务
3. **矛盾信息**：早期决策与后期修改产生冲突
4. **性能下降**：响应质量和速度都会降低

### 5.2 腐烂症状识别

```bash
# 上下文腐烂的典型症状：
# ❌ Claude 开始重复之前已经讨论过的内容
# ❌ Claude 忘记了之前确认的技术决策
# ❌ 响应变得更加保守和不确定
# ❌ Claude 开始询问已经提供过的信息
# ❌ 代码质量下降，出现之前已修复的错误
```

### 5.3 对抗腐烂的策略

```bash
# 策略 1：定期压缩（推荐每 30-50 轮）
claude /compact

# 策略 2：新建会话处理独立任务
# 不要在一个会话中处理多个不相关的任务

# 策略 3：使用子 Agent 隔离复杂子任务
claude "使用子 Agent 处理数据库迁移，保持主上下文干净"

# 策略 4：将重要决策写入 CLAUDE.md
claude "将我们刚才确定的 API 设计原则写入 CLAUDE.md"
```

---

## 六、子 Agent 的上下文隔离

### 6.1 为什么需要上下文隔离？

```
主 Agent 上下文（已使用 80%）
├── 系统提示
├── 大量对话历史
├── 多个文件内容
└── 工具调用历史

问题：主 Agent 上下文快满了，但还需要处理复杂子任务

解决方案：启动子 Agent，使用全新的上下文窗口
```

### 6.2 子 Agent 上下文架构

```python
# 子 Agent 拥有独立的上下文窗口
class SubAgent:
    context_window: int = 200_000  # 全新的 200K 窗口
    system_prompt: str             # 专门为子任务定制
    tool_access: list              # 只访问需要的工具
    parent_context: str            # 从父 Agent 传入的精简上下文
    
    def execute(self, task: str) -> str:
        # 在隔离环境中执行，不影响主 Agent
        result = self.run(task)
        # 只返回结果摘要给主 Agent
        return self.summarize(result)
```

### 6.3 上下文传递最佳实践

```bash
# 好的做法：传递精简的上下文
claude "创建一个子 Agent，告诉它：
- 项目使用 TypeScript + PostgreSQL
- 任务：优化数据库查询性能
- 返回：优化后的查询和性能报告"

# 不好的做法：传递整个对话历史
# 这会浪费子 Agent 的上下文空间
```

---

## 七、上下文优化技巧

### 7.1 文件读取优化

```bash
# 不好：读取整个大文件
claude "读取 src/ 目录下所有文件"

# 好：只读取需要的部分
claude "只读取 src/auth/ 目录下的认证相关文件"

# 更好：使用 grep 先定位，再精确读取
claude "先用 grep 找到处理登录的函数，然后只读取那个函数"
```

### 7.2 工具输出控制

```bash
# 控制 bash 输出长度
claude "运行测试，只显示失败的测试用例，不要显示通过的"

# 使用结构化输出
claude "以 JSON 格式输出测试结果，只包含 failed 和 error 字段"
```

### 7.3 会话分割策略

```bash
# 大型项目建议按模块分割会话
# 会话 1：后端 API 开发
# 会话 2：前端组件开发  
# 会话 3：数据库设计
# 会话 4：测试和部署

# 使用 CLAUDE.md 在会话间共享关键信息
```

---

## 八、上下文使用量监控

### 8.1 实时监控

```bash
# 查看详细上下文统计
claude /status --verbose

# 输出示例：
# ┌─────────────────────────────────────┐
# │ Context Window Status               │
# ├─────────────────────────────────────┤
# │ Total:    200,000 tokens            │
# │ Used:     142,847 tokens (71.4%)    │
# │ Available: 57,153 tokens            │
# │                                     │
# │ Breakdown:                          │
# │   System prompt:    8,234 tokens    │
# │   CLAUDE.md:        3,421 tokens    │
# │   Conversation:    45,678 tokens    │
# │   File content:    52,341 tokens    │
# │   Tool output:     33,173 tokens    │
# │                                     │
# │ ⚠️  AutoCompact threshold: 75%      │
# │ 📊 Estimated turns remaining: ~8    │
# └─────────────────────────────────────┘
```

### 8.2 成本估算

```bash
# 上下文大小直接影响 API 成本
# Claude 3.5 Sonnet 定价（2026 Q2）：
# 输入：$3 / 1M tokens
# 输出：$15 / 1M tokens

# 一个 100K token 的上下文，每次调用成本：
# 输入成本：100K × $3/1M = $0.30
# 因此，减少上下文大小可以显著降低成本
```

---

## 九、常见问题

### Q: 上下文压缩会丢失重要信息吗？

AutoCompact 设计为保留关键决策、错误信息和重要事实，但可能丢失：
- 详细的中间步骤
- 冗余的工具输出
- 重复的讨论内容

建议将真正重要的信息写入 CLAUDE.md 以永久保存。

### Q: 如何知道上下文何时会被压缩？

```bash
# 方法 1：查看状态
claude /status

# 方法 2：观察 Claude 的行为
# 当 Claude 说"根据我们之前的讨论..."时，可能已经在使用压缩摘要

# 方法 3：启用详细日志
claude --verbose  # 会显示压缩事件
```

### Q: 1M token 窗口什么时候可用？

目前（2026 Q2）Claude Code 使用 200K token 窗口。1M token 窗口正在测试中，预计将在未来版本中推出，届时压缩触发阈值将调整为约 420K tokens。

---

*本文档基于 Claude Code 泄漏源码分析及官方文档整理，部分内部实现细节可能随版本更新而变化。*
