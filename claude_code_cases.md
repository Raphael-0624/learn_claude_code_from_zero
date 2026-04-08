# Claude Code 实战案例库

> 真实场景的完整操作流程与最佳实践  
> 基于开发者社区真实案例整合  
> 更新日期：2026年4月7日

---

## 目录

1. [案例一：接手遗留代码库](#案例一接手遗留代码库)
2. [案例二：从零搭建全栈项目](#案例二从零搭建全栈项目)
3. [案例三：大规模代码重构](#案例三大规模代码重构)
4. [案例四：性能瓶颈定位与优化](#案例四性能瓶颈定位与优化)
5. [案例五：安全漏洞审查与修复](#案例五安全漏洞审查与修复)
6. [案例六：自动化测试体系建设](#案例六自动化测试体系建设)
7. [案例七：CI/CD 流水线搭建](#案例七cicd-流水线搭建)
8. [案例八：遗留系统技术栈迁移](#案例八遗留系统技术栈迁移)
9. [案例九：多 Agent 并行开发](#案例九多-agent-并行开发)
10. [案例十：API 文档自动化生成](#案例十api-文档自动化生成)

---

## 案例一：接手遗留代码库

**难度**：⭐⭐ 初级  
**耗时**：约 2 小时（传统方式需 2 天）  
**场景**：新加入团队，需要快速理解一个 3 年历史的 Node.js 项目

### 背景

项目有 200+ 文件，原开发者已离职，文档几乎为零，需要在 1 周内接手并开始开发新功能。

### 操作流程

**Step 1：生成项目地图**

```
> 请帮我理解这个项目，按以下顺序进行：

1. 先读取 package.json，了解技术栈和脚本
2. 读取项目根目录结构
3. 找到入口文件，追踪主要模块
4. 识别核心业务逻辑所在位置

不要修改任何文件，只读取和分析。
完成后生成一份 docs/project-map.md，包含：
- 技术栈总结
- 目录结构说明
- 核心模块列表（每个模块一句话说明）
- 数据流向图（用 Mermaid）
```

**Step 2：识别技术债务**

```
> 基于你对项目的理解，请识别技术债务：

分析维度：
1. 代码质量问题（重复代码、过长函数、命名混乱）
2. 安全隐患（硬编码密钥、SQL 拼接、缺少输入验证）
3. 性能问题（N+1 查询、缺少缓存、同步阻塞）
4. 测试覆盖（哪些模块没有测试）
5. 依赖风险（过时的依赖、已知漏洞）

输出格式：按严重程度排序的问题列表，每个问题包含：
- 位置（文件:行号）
- 问题描述
- 风险等级（高/中/低）
- 建议修复方案
```

**Step 3：建立 CLAUDE.md**

```
> 基于你对项目的分析，帮我生成一份 CLAUDE.md 文件：

包含：
1. 项目概述（2-3句话）
2. 技术栈
3. 常用命令
4. 重要约束（特别是那些有技术债务的模块，标注"暂时不要修改"）
5. 已知问题

保存到项目根目录的 CLAUDE.md
```

### 结果

- 2 小时内生成了完整的项目文档
- 识别出 23 个技术债务问题，按优先级排序
- 建立了 CLAUDE.md，后续所有会话自动获得项目上下文

### 关键 CLAUDE.md 片段

```markdown
# 项目约束

## 暂时不要修改的模块
- src/payment/legacy-gateway.js：老支付网关，有外部依赖，修改需要通知运营
- src/report/excel-generator.js：已知内存泄漏，正在重写，不要在此基础上开发

## 数据库注意事项
- 使用原始 SQL（不是 ORM），所有查询在 src/db/queries/ 目录
- 生产数据库有 500 万用户数据，任何 schema 变更需要评估影响
```

---

## 案例二：从零搭建全栈项目

**难度**：⭐⭐⭐ 中级  
**耗时**：约 4 小时（传统方式需 2 周）  
**场景**：从零搭建一个任务管理 SaaS 应用（Next.js + NestJS + PostgreSQL）

### 操作流程

**Step 1：架构设计**

```
> 我要搭建一个任务管理 SaaS 应用，请帮我设计技术架构：

需求：
- 多租户（每个公司独立数据）
- 用户认证（邮箱+密码，支持 Google OAuth）
- 任务管理（创建/分配/状态跟踪/截止日期）
- 实时通知（任务状态变更时通知相关人员）
- 数据导出（Excel/CSV）

技术偏好：
- 前端：Next.js 14（App Router）
- 后端：NestJS
- 数据库：PostgreSQL
- 实时：WebSocket 或 SSE

请输出：
1. 系统架构图（Mermaid）
2. 数据库 ER 图（Mermaid）
3. 目录结构
4. 技术选型说明（每个选择的理由）

先只做设计，不写代码。
```

**Step 2：分阶段实现**

```
> 架构设计已确认，现在开始实现。

第一阶段：项目脚手架（今天完成）
1. 初始化 monorepo（使用 pnpm workspaces）
2. 创建 apps/web（Next.js 14）
3. 创建 apps/api（NestJS）
4. 创建 packages/shared（共享类型定义）
5. 配置 TypeScript、ESLint、Prettier
6. 配置 Docker Compose（PostgreSQL + Redis）

请按顺序执行，每步完成后告诉我，遇到问题立即停下来问我。
```

**Step 3：核心功能实现**

```
> 脚手架已就绪，开始实现用户认证模块：

后端（NestJS）：
- AuthModule：JWT 认证 + Refresh Token
- UsersModule：用户 CRUD
- 数据库迁移：users 表

前端（Next.js）：
- 登录页面（/login）
- 注册页面（/register）
- 认证中间件（保护需要登录的路由）
- 用户状态管理（Zustand）

要求：
- 前后端类型共享（通过 packages/shared）
- 错误处理统一（前端 toast 提示）
- 表单验证（Zod）

先实现后端，再实现前端。
```

### 关键技巧

**使用 Git Worktree 并行开发前后端**：

```bash
git worktree add ../task-app-frontend main
git worktree add ../task-app-backend main

# 终端 1：前端
cd ../task-app-frontend && claude
> 专注实现前端部分，后端 API 已在 localhost:3001

# 终端 2：后端
cd ../task-app-backend && claude
> 专注实现后端 API，前端在 localhost:3000
```

### 结果

- 4 小时内完成了可运行的 MVP
- 包含用户认证、任务 CRUD、基础 UI
- 代码质量高，有完整的类型定义和测试

---

## 案例三：大规模代码重构

**难度**：⭐⭐⭐⭐ 高级  
**耗时**：约 6 小时（传统方式需 2 周）  
**场景**：将 50 个 React class 组件迁移为函数组件

### 背景

项目有 50 个 class 组件，需要迁移到函数组件以支持新的 Hooks 功能，同时不能影响现有功能。

### 操作流程

**Step 1：分析与规划**

```
> 请分析 src/components/ 目录下所有 class 组件：

1. 列出所有 class 组件（文件路径）
2. 按复杂度分类：
   - 简单（只有 render，无生命周期）
   - 中等（有 1-2 个生命周期方法）
   - 复杂（有多个生命周期、内部状态、refs）
3. 识别迁移风险点（如使用了 getDerivedStateFromProps 等复杂 API）
4. 制定迁移顺序（从简单到复杂）

输出：迁移计划表（CSV 格式）
列：文件路径, 复杂度, 风险点, 建议迁移顺序
```

**Step 2：建立迁移规范**

```
> 在开始迁移前，请帮我建立迁移规范：

1. 生命周期方法对应关系：
   - componentDidMount → useEffect(fn, [])
   - componentDidUpdate → useEffect(fn, [deps])
   - componentWillUnmount → useEffect(() => { return cleanup }, [])
   - shouldComponentUpdate → React.memo + useMemo

2. 创建一个迁移示例：
   - 选择一个中等复杂度的组件
   - 展示完整的迁移过程（before/after）
   - 说明每个决策的理由

3. 生成迁移 checklist（.claude/migration-checklist.md）
```

**Step 3：批量迁移（使用子 Agent）**

```
> 开始批量迁移简单组件（复杂度：简单，共 20 个）：

迁移规则：
- 遵循 .claude/migration-checklist.md 中的规范
- 每个组件迁移后立即运行对应测试
- 如果测试失败，回滚并标记为"需要人工处理"
- 迁移完成后更新 .claude/migration-progress.md

请并行处理，每次处理 5 个组件，完成后报告结果。
```

**Step 4：验证与收尾**

```
> 所有简单组件已迁移，请进行验证：

1. 运行完整测试套件：npm test
2. 运行 TypeScript 检查：npm run type-check
3. 运行 ESLint：npm run lint
4. 生成迁移报告：
   - 成功迁移数量
   - 失败/跳过数量
   - 测试覆盖率变化
   - 代码行数变化

如果有失败的测试，列出来，我来决定如何处理。
```

### 关键配置（CLAUDE.md 片段）

```markdown
## 重构进行中

当前正在进行 class → 函数组件迁移（进度见 .claude/migration-progress.md）

迁移规范：
- 参考 .claude/migration-checklist.md
- 每次迁移后必须运行测试
- 不要修改组件的 props 接口
- 不要修改组件的对外行为
```

### 结果

- 6 小时内完成 50 个组件迁移
- 测试覆盖率从 67% 提升到 82%（迁移过程中补充了测试）
- 零功能回归

---

## 案例四：性能瓶颈定位与优化

**难度**：⭐⭐⭐ 中级  
**耗时**：约 3 小时  
**场景**：API 接口响应时间从 200ms 劣化到 2000ms，需要定位并修复

### 操作流程

**Step 1：系统性分析**

```
> 我们的 /api/orders 接口响应时间从 200ms 劣化到 2000ms。

请帮我系统性地分析可能的原因：

1. 读取 src/order/order.service.ts 中的 getOrders 方法
2. 读取对应的数据库查询（src/db/queries/orders.sql 或 ORM 查询）
3. 检查是否有 N+1 查询问题
4. 检查是否有缺少索引的查询
5. 检查是否有不必要的数据加载（加载了不需要的字段/关联）

不要修改代码，先给我分析报告。
```

**Step 2：数据库查询优化**

```
> 分析确认是 N+1 查询问题，每个订单都单独查询用户信息。

请优化：
1. 将 N+1 查询改为 JOIN 或批量查询
2. 添加必要的数据库索引（生成迁移文件）
3. 添加查询结果缓存（Redis，TTL 5分钟）

优化后：
- 运行现有测试确保功能正确
- 添加性能测试（使用 k6 或 autocannon）
- 生成性能对比报告
```

**Step 3：应用层优化**

```
> 数据库优化后响应时间降到 500ms，还需要继续优化到 200ms。

请分析应用层的优化空间：
1. 检查是否有不必要的数据序列化/反序列化
2. 检查是否可以使用流式响应
3. 检查是否有可以并行的操作（目前是串行的）
4. 检查响应数据是否可以压缩

对每个优化点：说明预期收益，实现后测量实际效果。
```

### 结果

| 优化阶段 | 响应时间 | 优化内容 |
|----------|----------|----------|
| 优化前 | 2000ms | - |
| 数据库优化 | 500ms | 消除 N+1，添加索引 |
| 缓存优化 | 150ms | Redis 缓存热点数据 |
| 并行优化 | 120ms | 并行查询独立数据 |

---

## 案例五：安全漏洞审查与修复

**难度**：⭐⭐⭐ 中级  
**耗时**：约 4 小时  
**场景**：上线前的安全审查，发现并修复安全漏洞

### 操作流程

**Step 1：全面安全扫描**

```
> 请对整个项目进行安全审查，重点关注 OWASP Top 10：

1. 注入攻击（SQL 注入、命令注入、LDAP 注入）
2. 身份认证问题（弱密码策略、会话管理）
3. 敏感数据暴露（日志中的密码、API Key 硬编码）
4. XML 外部实体（XXE）
5. 访问控制缺陷（越权访问）
6. 安全配置错误（默认密码、不必要的功能开启）
7. XSS（跨站脚本）
8. 不安全的反序列化
9. 使用含已知漏洞的组件
10. 日志和监控不足

输出格式：
每个漏洞：
- 漏洞类型
- 位置（文件:行号）
- 风险等级（严重/高/中/低）
- 漏洞描述
- 修复方案
- 修复优先级
```

**Step 2：修复高危漏洞**

```
> 安全报告已确认，请按优先级修复高危漏洞：

严重漏洞（立即修复）：
1. src/user/user.controller.ts:45 - SQL 注入风险
2. src/auth/auth.service.ts:23 - JWT 密钥硬编码
3. src/upload/upload.controller.ts:67 - 文件上传无类型验证

修复要求：
- 每个修复必须有对应的安全测试
- 修复后不能影响现有功能
- 添加注释说明安全考虑

请逐一修复，每个修复后告诉我，等我确认再继续。
```

**Step 3：建立安全基线**

```
> 漏洞已修复，请帮我建立安全开发基线：

1. 创建安全编码规范（docs/security-guidelines.md）
2. 在 CLAUDE.md 中添加安全约束
3. 配置 npm audit 自动检查（在 CI 中）
4. 添加 Helmet.js 安全头配置
5. 配置 rate limiting（防止暴力破解）
```

### 发现的典型漏洞

```javascript
// ❌ 漏洞：SQL 注入
const users = await db.query(
  `SELECT * FROM users WHERE email = '${email}'`
);

// ✅ 修复：参数化查询
const users = await db.query(
  'SELECT * FROM users WHERE email = $1',
  [email]
);

// ❌ 漏洞：JWT 密钥硬编码
const token = jwt.sign(payload, 'my-secret-key');

// ✅ 修复：环境变量
const token = jwt.sign(payload, process.env.JWT_SECRET);
```

---

## 案例六：自动化测试体系建设

**难度**：⭐⭐⭐ 中级  
**耗时**：约 5 小时  
**场景**：项目测试覆盖率为 0，需要从零建立测试体系

### 操作流程

**Step 1：测试策略制定**

```
> 项目目前没有任何测试，请帮我制定测试策略：

项目情况：
- NestJS 后端，50 个 Service，30 个 Controller
- 核心业务：用户管理、订单处理、支付集成
- 数据库：PostgreSQL（使用 TypeORM）

请制定：
1. 测试分层策略（单元/集成/E2E 各占比例）
2. 优先测试哪些模块（按业务重要性）
3. Mock 策略（数据库、外部 API 如何 Mock）
4. 测试数据管理方案（测试数据库、Fixtures）
5. 覆盖率目标（分阶段）

输出：测试策略文档（docs/testing-strategy.md）
```

**Step 2：建立测试基础设施**

```
> 测试策略已确认，请建立测试基础设施：

1. 配置 Jest（jest.config.ts）
   - 支持 TypeScript
   - 配置路径别名
   - 覆盖率报告（lcov + html）

2. 创建测试工具库（test/helpers/）
   - 数据库测试工具（连接测试数据库、清理数据）
   - Mock 工厂（生成测试数据）
   - 请求测试工具（封装 supertest）

3. 配置测试数据库
   - Docker Compose 添加测试数据库
   - 测试前自动运行迁移
   - 每个测试后清理数据

4. 创建第一个测试作为示例（UserService 的 createUser 方法）
```

**Step 3：批量生成测试**

```
> 基础设施已就绪，请为核心模块生成测试：

优先级 1（本周完成）：
- UserService：所有 public 方法
- AuthService：登录、注册、Token 刷新
- OrderService：创建订单、更新状态

要求：
- 每个方法至少 3 个测试用例（正常、边界、错误）
- 使用工厂函数生成测试数据
- Mock 外部依赖（数据库、邮件服务）
- 覆盖率目标：> 80%

请逐个模块处理，完成一个告诉我。
```

### 结果

| 阶段 | 覆盖率 | 测试数量 | 耗时 |
|------|--------|----------|------|
| 开始前 | 0% | 0 | - |
| 第一周 | 45% | 120 | 5小时 |
| 第二周 | 72% | 280 | 8小时 |
| 第三周 | 85% | 380 | 6小时 |

---

## 案例七：CI/CD 流水线搭建

**难度**：⭐⭐ 初级  
**耗时**：约 2 小时  
**场景**：为 GitHub 项目搭建完整的 CI/CD 流水线

### 操作流程

**Step 1：设计流水线**

```
> 请为我们的项目设计 GitHub Actions CI/CD 流水线：

项目情况：
- monorepo（前端 Next.js + 后端 NestJS）
- 部署目标：AWS（前端 Vercel，后端 ECS）
- 分支策略：main（生产）、develop（测试）、feature/*（开发）

流水线要求：
- PR 时：代码检查 + 测试 + 构建验证
- 合并到 develop：自动部署到测试环境
- 合并到 main：手动审批后部署到生产

请设计流水线图（Mermaid）和各 Job 的职责。
```

**Step 2：实现 CI 流水线**

```
> 请实现 CI 流水线（.github/workflows/ci.yml）：

包含以下 Jobs：
1. lint：ESLint + TypeScript 检查
2. test：单元测试 + 集成测试（并行）
3. build：构建前后端（验证构建成功）
4. security：npm audit + 依赖漏洞扫描

优化要求：
- 使用缓存（node_modules、构建缓存）
- 并行执行独立 Job
- 失败时发送 Slack 通知
- 目标：总耗时 < 5 分钟
```

**Step 3：实现 CD 流水线**

```
> 请实现 CD 流水线：

测试环境（.github/workflows/deploy-staging.yml）：
- 触发：push 到 develop 分支
- 步骤：构建 Docker 镜像 → 推送 ECR → 更新 ECS 服务
- 部署后：运行 smoke test

生产环境（.github/workflows/deploy-production.yml）：
- 触发：手动触发（workflow_dispatch）+ 需要审批
- 步骤：同测试环境 + 蓝绿部署
- 部署后：运行 E2E 测试 + 监控告警检查
- 失败时：自动回滚

请使用 GitHub Environments 实现审批流程。
```

---

## 案例八：遗留系统技术栈迁移

**难度**：⭐⭐⭐⭐⭐ 专家级  
**耗时**：约 2 周（传统方式需 3 个月）  
**场景**：将 PHP 5.6 + MySQL 的遗留系统迁移到 Node.js + PostgreSQL

### 操作流程

**Step 1：逆向工程**

```
> 请分析这个 PHP 项目，帮我理解业务逻辑：

1. 扫描所有 PHP 文件，识别：
   - 数据库表结构（从 SQL 文件或 ORM 定义）
   - 核心业务逻辑（不是框架代码）
   - API 接口（路由定义）
   - 外部依赖（第三方 API、服务）

2. 生成业务逻辑文档（docs/business-logic.md）
   - 用中文描述每个核心功能
   - 不要描述技术实现，只描述业务规则

3. 生成数据库 ER 图（docs/database-er.md）
```

**Step 2：制定迁移策略**

```
> 基于分析，请制定迁移策略：

约束：
- 迁移期间新旧系统并行运行
- 不能有数据丢失
- 用户无感知（不能有停机时间）

请设计：
1. 数据迁移方案（如何从 MySQL 迁移到 PostgreSQL）
2. 流量切换方案（如何逐步将流量从旧系统切到新系统）
3. 回滚方案（如果新系统出问题如何快速回滚）
4. 迁移阶段划分（每个阶段的目标和验收标准）
```

**Step 3：分阶段实现**

```
> 开始第一阶段：核心用户模块迁移

目标：
- 实现用户注册、登录、个人信息管理
- 与旧系统数据库同步（双写）
- 通过所有现有功能测试

技术栈：
- NestJS + TypeORM + PostgreSQL
- 参考 docs/business-logic.md 中的业务规则
- 参考 docs/database-er.md 中的数据结构

注意：
- 新系统的 API 路径与旧系统保持一致（方便切流量）
- 密码迁移：旧系统用 MD5，新系统用 bcrypt，需要兼容处理
```

---

## 案例九：多 Agent 并行开发

**难度**：⭐⭐⭐⭐ 高级  
**耗时**：约 3 小时（传统方式需 2 天）  
**场景**：同时开发 3 个独立功能模块，使用多 Agent 并行

### 操作流程

**准备工作**

```bash
# 创建三个独立的工作目录
git worktree add ../feature-auth feature/auth
git worktree add ../feature-payment feature/payment
git worktree add ../feature-notification feature/notification
```

**终端 1：认证模块**

```
# cd ../feature-auth && claude

> 你负责实现用户认证模块：
  - JWT 认证（access + refresh token）
  - 邮箱验证
  - 密码重置
  
  接口规范在 docs/api-spec.md 中，请严格遵守。
  完成后在 feature/auth 分支提交，PR 标题：feat: user authentication module
```

**终端 2：支付模块**

```
# cd ../feature-payment && claude

> 你负责实现支付模块：
  - 接入 Stripe（使用 stripe-node SDK）
  - 支持信用卡和支付宝
  - 支付成功/失败 Webhook 处理
  
  接口规范在 docs/api-spec.md 中，请严格遵守。
  完成后在 feature/payment 分支提交，PR 标题：feat: payment module
```

**终端 3：通知模块**

```
# cd ../feature-notification && claude

> 你负责实现通知模块：
  - 邮件通知（使用 SendGrid）
  - 站内通知（WebSocket）
  - 通知模板管理
  
  接口规范在 docs/api-spec.md 中，请严格遵守。
  完成后在 feature/notification 分支提交，PR 标题：feat: notification module
```

**合并与集成**

```
> 三个模块已完成，请帮我进行集成：

1. 检查三个模块的接口是否符合 docs/api-spec.md
2. 检查是否有命名冲突或依赖冲突
3. 按顺序合并：notification → auth → payment
4. 运行完整测试套件
5. 生成集成测试报告
```

### 结果

- 3 个模块并行开发，总耗时 3 小时
- 相当于串行开发的 1/3 时间
- 集成时发现 2 个接口命名冲突，30 分钟内解决

---

## 案例十：API 文档自动化生成

**难度**：⭐ 入门  
**耗时**：约 1 小时  
**场景**：为现有 API 自动生成完整文档

### 操作流程

**Step 1：生成 OpenAPI 规范**

```
> 请为整个 API 生成 OpenAPI 3.0 规范：

扫描范围：src/controllers/ 目录下所有控制器

提取信息：
- 路由路径和 HTTP 方法
- 请求参数（路径参数、查询参数、请求体）
- 响应格式（成功和错误）
- 认证要求（哪些接口需要 JWT）
- 从 JSDoc 注释中提取描述

输出：docs/openapi.yaml（OpenAPI 3.0 格式）

注意：
- 如果某个接口信息不完整，用 TODO 标注
- 不要编造不存在的接口
```

**Step 2：生成开发者文档**

```
> 基于 OpenAPI 规范，生成面向前端开发者的 Markdown 文档：

要求：
- 语言友好，避免后端术语
- 每个接口包含：描述、请求示例（curl）、响应示例
- 按功能模块分组（用户、订单、支付等）
- 包含认证说明（如何获取和使用 Token）
- 包含错误码说明

保存到：docs/api-guide.md
```

**Step 3：配置 Swagger UI**

```
> 请在 NestJS 项目中配置 Swagger UI：

1. 安装 @nestjs/swagger
2. 在 main.ts 中配置 Swagger
3. 为所有 DTO 添加 @ApiProperty 装饰器
4. 为所有 Controller 添加 @ApiTags 装饰器
5. 配置认证（Bearer Token）

Swagger UI 访问地址：/api/docs
只在非生产环境开启
```

---

## 效率数据汇总

| 案例 | 传统耗时 | Claude Code 耗时 | 提速倍数 |
|------|----------|-----------------|----------|
| 接手遗留代码库 | 2天 | 2小时 | 8x |
| 从零搭建全栈项目 | 2周 | 4小时 | 21x |
| 大规模代码重构 | 2周 | 6小时 | 14x |
| 性能瓶颈优化 | 3天 | 3小时 | 8x |
| 安全漏洞审查 | 1周 | 4小时 | 10x |
| 测试体系建设 | 2周 | 5小时 | 19x |
| CI/CD 搭建 | 3天 | 2小时 | 12x |
| 技术栈迁移 | 3个月 | 2周 | 6x |
| 多 Agent 并行 | 2天 | 3小时 | 5x |
| API 文档生成 | 1天 | 1小时 | 8x |

> 数据来源：开发者社区调研 + Anthropic 官方案例研究（2026年Q1）

---

*Claude Code 实战案例库 · 2026年4月7日*
