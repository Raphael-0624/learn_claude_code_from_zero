# Claude Code Memory 记忆系统深度解析

> Claude Code 的记忆系统是其最独特的能力之一。它让 Claude 能够跨会话学习、积累知识、持续进化——就像一个真正的团队成员，而不是每次都从零开始的工具。

---

## 一、记忆系统架构

Claude Code 实现了四层记忆架构，从短期到长期，从会话内到跨会话：

```
┌─────────────────────────────────────────────────────────────┐
│                    记忆系统全景                              │
│                                                             │
│  Layer 4: CLAUDE.md（静态项目指令）                         │
│  ├── 永久存在，手动维护                                      │
│  ├── 每次会话自动加载                                        │
│  └── 包含：规范、约束、架构决策                              │
│                                                             │
│  Layer 3: MEMORY.md（动态记忆索引）                         │
│  ├── 自动更新，AI 维护                                       │
│  ├── 前 200 行/25KB 自动加载                                 │
│  └── 指向详细 topic files                                   │
│                                                             │
│  Layer 2: Session Memory（会话摘要）                        │
│  ├── 会话结束时自动生成                                      │
│  ├── 保存关键决策和发现                                      │
│  └── 供下次会话参考                                          │
│                                                             │
│  Layer 1: Chat Memory（24小时合成）                         │
│  ├── Auto Dream 机制处理                                     │
│  ├── 跨会话知识整合                                          │
│  └── 长期知识库的基础                                        │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、CLAUDE.md：静态项目指令

### 2.1 文件层级与加载顺序

```
项目目录结构：
~/
├── .claude/
│   └── CLAUDE.md          # 全局指令（所有项目共享）
│
└── my-project/
    ├── CLAUDE.md           # 项目根指令（最重要）
    ├── src/
    │   └── CLAUDE.md       # 模块级指令（可选）
    └── tests/
        └── CLAUDE.md       # 测试目录指令（可选）

加载顺序：全局 → 项目根 → 当前目录
优先级：当前目录 > 项目根 > 全局
```

### 2.2 CLAUDE.md 最佳结构

```markdown
# 项目名称 - Claude 工作指南

## 🚨 绝对约束（必须遵守）
- 数据库：PostgreSQL 14+，禁止使用其他数据库
- 语言：TypeScript 5.x，严格模式，禁止 any 类型
- 框架：Next.js 14 App Router，禁止使用 Pages Router

## 🏗️ 架构概览
- 前端：Next.js + Tailwind CSS
- 后端：Node.js + Prisma ORM
- 部署：Vercel + Supabase

## 📋 代码规范
- 使用 ESLint + Prettier（配置已在 .eslintrc 中）
- 组件命名：PascalCase
- 函数命名：camelCase
- 文件命名：kebab-case

## 🔧 常用命令
- 开发：`npm run dev`
- 测试：`npm test`
- 构建：`npm run build`
- 数据库迁移：`npx prisma migrate dev`

## 📁 目录结构
- `src/app/` - Next.js App Router 页面
- `src/components/` - 可复用组件
- `src/lib/` - 工具函数和配置
- `src/types/` - TypeScript 类型定义

## ⚠️ 注意事项
- 不要修改 `src/lib/auth.ts`（认证核心，需要专门审查）
- 环境变量在 `.env.local` 中，不要提交到 git
- API 路由必须包含错误处理和日志记录
```

### 2.3 动态更新 CLAUDE.md

```bash
# 让 Claude 自动更新 CLAUDE.md
claude "将我们刚才确定的 API 设计原则添加到 CLAUDE.md"

# 查看 CLAUDE.md 当前内容
claude "显示当前 CLAUDE.md 的内容"

# 让 Claude 优化 CLAUDE.md 结构
claude "重新组织 CLAUDE.md，把最重要的约束放在最前面"
```

---

## 三、MEMORY.md：动态记忆索引

### 3.1 MEMORY.md 的工作机制

```
MEMORY.md 是 Claude 的"工作记忆索引"：

自动加载规则：
├── 前 200 行：每次会话自动加载到上下文
├── 25KB 限制：超过后只加载前 25KB
└── 索引结构：指向详细的 topic files

目录结构：
~/.claude/
├── MEMORY.md              # 主索引（自动加载）
├── memories/
│   ├── debugging.md       # 调试经验记录
│   ├── api-conventions.md # API 设计约定
│   ├── performance.md     # 性能优化记录
│   └── decisions.md       # 重要决策记录
```

### 3.2 MEMORY.md 示例结构

```markdown
# Claude 记忆索引
*最后更新：2026-04-07 by Auto Dream*

## 活跃项目
- **my-saas-app**：SaaS 应用，PostgreSQL + Next.js，见 memories/my-saas-app.md
- **data-pipeline**：数据管道项目，Python + Airflow，见 memories/data-pipeline.md

## 重要发现
- PostgreSQL 连接池配置：最大 20 连接，见 memories/db-config.md
- Next.js 14 App Router 的 ISR 配置方法，见 memories/nextjs-tips.md

## 用户偏好
- 代码风格：函数式编程优先，避免类
- 注释风格：只注释"为什么"，不注释"是什么"
- 测试框架：Vitest（不是 Jest）

## 最近会话摘要
- 2026-04-06：完成认证模块重构，使用 JWT + Refresh Token
- 2026-04-05：优化数据库查询，性能提升 3x
- 2026-04-04：部署到 Vercel，配置了 CI/CD
```

### 3.3 手动管理记忆

```bash
# 让 Claude 记住重要信息
claude "记住：我们的 API 使用 snake_case 命名，不是 camelCase"

# 查看当前记忆
claude "显示你记住的关于这个项目的所有信息"

# 清除特定记忆
claude "忘记关于旧版 API 设计的记忆，我们已经重构了"

# 手动编辑记忆文件
vim ~/.claude/MEMORY.md
```

---

## 四、Auto Dream：后台记忆整合机制

### 4.1 什么是 Auto Dream？

Auto Dream 是 Claude Code 最神秘的特性之一——一个在后台运行的记忆整合机制，类比人脑在睡眠时处理和整合记忆：

```
人脑睡眠记忆处理          Claude Auto Dream
─────────────────         ─────────────────
白天积累经验      →       会话积累记忆
睡眠时整合记忆    →       后台触发整合
剪枝无用记忆      →       删除过时条目
强化重要记忆      →       合并相关信息
重构记忆结构      →       优化索引结构
```

### 4.2 触发条件

```python
# Auto Dream 触发逻辑（伪代码）
def should_trigger_auto_dream():
    conditions = [
        # 条件 1：时间条件
        hours_since_last_dream() >= 24,
        
        # 条件 2：会话数量条件
        sessions_since_last_dream() >= 5,
        
        # 条件 3：记忆文件大小条件
        memory_file_size() > SIZE_THRESHOLD,
        
        # 条件 4：手动触发
        user_requested_dream(),
    ]
    
    # 满足任一条件即触发
    return any(conditions)
```

### 4.3 Auto Dream 的三个操作

```
操作 1: 剪枝（Pruning）
├── 删除过时的信息（已解决的 bug、废弃的 API）
├── 移除重复的条目
└── 清理临时性的记录

操作 2: 合并（Merging）
├── 将相关信息整合到同一主题
├── 消除矛盾的记录
└── 更新过时的信息

操作 3: 重构（Restructuring）
├── 优化 MEMORY.md 的索引结构
├── 将频繁访问的信息移到前面
└── 创建新的 topic files 分类存储
```

### 4.4 Auto Dream 的限制

```bash
# ⚠️ 重要限制：Auto Dream 只能修改记忆文件
# 它不能修改项目源代码
# 它不能执行任何命令
# 它不能访问网络

# 受 Auto Dream 管理的文件：
~/.claude/MEMORY.md
~/.claude/memories/*.md

# 不受 Auto Dream 影响的文件：
src/           # 项目源代码
CLAUDE.md      # 项目指令（需要手动维护）
.env           # 环境变量
```

### 4.5 手动触发 Auto Dream

```bash
# 手动触发记忆整合
claude /dream

# 查看 Auto Dream 日志
claude /dream --log

# 预览 Auto Dream 将做什么（不实际执行）
claude /dream --dry-run

# 禁用 Auto Dream
# 在 ~/.claude/settings.json 中：
{
  "autoDream": {
    "enabled": false
  }
}
```

---

## 五、Session Memory：会话摘要

### 5.1 会话结束时的记忆生成

```python
# 会话结束时，Claude 自动生成摘要
def generate_session_memory(session):
    return {
        "date": session.date,
        "duration": session.duration,
        "project": session.project,
        
        # 关键决策
        "decisions": extract_decisions(session),
        
        # 重要发现
        "discoveries": extract_discoveries(session),
        
        # 完成的任务
        "completed": extract_completed_tasks(session),
        
        # 未完成的任务
        "pending": extract_pending_tasks(session),
        
        # 遇到的问题
        "issues": extract_issues(session),
    }
```

### 5.2 会话摘要示例

```markdown
## 会话摘要 - 2026-04-07 14:30

**项目**：my-saas-app
**时长**：2小时15分钟
**工具调用**：47次

### 完成的任务
- ✅ 重构用户认证模块（JWT + Refresh Token）
- ✅ 修复了 3 个数据库查询 N+1 问题
- ✅ 添加了 API 速率限制（100 req/min）

### 重要决策
- 选择 Redis 作为 Session 存储（而非数据库）
- API 版本策略：URL 路径版本（/api/v1/）

### 待处理
- ⏳ 邮件通知系统（下次会话继续）
- ⏳ 管理员权限系统

### 发现的问题
- PostgreSQL 连接池在高并发下有瓶颈，需要调优
```

---

## 六、RAG 支持：搜索历史对话

### 6.1 RAG 记忆检索

付费用户可以使用 RAG（检索增强生成）搜索历史对话：

```bash
# 搜索历史对话中的相关信息
claude "搜索我们之前讨论过的数据库优化方案"

# 搜索特定时间范围
claude "找出上周关于 API 设计的所有讨论"

# 搜索特定项目的历史
claude "显示 my-saas-app 项目的所有历史决策"
```

### 6.2 RAG 的工作原理

```
历史对话 → 向量化 → 存储在向量数据库
                              │
用户查询 → 向量化 → 相似度搜索 ┘
                              │
                    检索相关片段
                              │
                    注入当前上下文
                              │
                    Claude 生成回答
```

---

## 七、记忆系统最佳实践

### 7.1 CLAUDE.md 维护策略

```bash
# 定期审查和更新 CLAUDE.md
# 建议每周或每个重要里程碑后更新

# 让 Claude 帮你维护
claude "审查当前 CLAUDE.md，删除过时的信息，添加我们最近确定的新约束"

# 版本控制 CLAUDE.md
git add CLAUDE.md
git commit -m "docs: 更新 Claude 工作指南"
```

### 7.2 记忆分层策略

```
永久记忆（CLAUDE.md）：
├── 技术栈选择
├── 代码规范
├── 架构决策
└── 绝对约束

动态记忆（MEMORY.md）：
├── 最近的发现
├── 临时的约定
├── 进行中的任务
└── 已知的问题

会话记忆（Session Memory）：
├── 本次会话的进展
├── 临时的上下文
└── 短期任务状态
```

### 7.3 团队协作中的记忆管理

```bash
# 团队共享 CLAUDE.md（提交到 git）
git add CLAUDE.md
git commit -m "docs: 团队 Claude 工作指南"

# 个人记忆不提交（在 .gitignore 中）
echo "MEMORY.md" >> .gitignore
echo ".claude/memories/" >> .gitignore

# 团队级别的记忆可以放在专门的文件中
echo "TEAM_MEMORY.md" >> .gitignore  # 或者提交共享
```

---

## 八、常见问题

### Q: Claude 真的能记住跨会话的信息吗？

是的，通过 MEMORY.md 和 CLAUDE.md 机制。但这不是"真正的"记忆——它是通过文件系统实现的持久化，每次会话开始时这些文件会被加载到上下文中。

### Q: Auto Dream 会修改我的代码吗？

**不会**。Auto Dream 有严格的限制，只能修改 `~/.claude/memories/` 目录下的记忆文件，不能修改任何项目源代码。

### Q: 如何防止 Claude 记住不想要的信息？

```bash
# 明确告诉 Claude 不要记住某些信息
claude "不要将这次讨论的内容写入记忆"

# 或者在会话结束后清理
claude "清除今天会话中关于 [主题] 的所有记忆"

# 直接编辑记忆文件
vim ~/.claude/MEMORY.md
```

### Q: MEMORY.md 文件太大怎么办？

```bash
# 让 Auto Dream 整理
claude /dream

# 手动清理
claude "整理 MEMORY.md，删除超过 30 天的旧记录"

# 归档旧记忆
mv ~/.claude/MEMORY.md ~/.claude/memories/archive-2026-Q1.md
touch ~/.claude/MEMORY.md
```

---

*本文档基于 Claude Code 泄漏源码分析及官方文档整理，部分内部实现细节可能随版本更新而变化。*
