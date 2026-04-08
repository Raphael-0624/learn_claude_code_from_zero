# Claude Code Tool System 工具系统深度解析

> Claude Code 的工具系统是其能力的核心来源。约 40 个内置工具 + MCP 扩展协议，让 Claude 能够读写文件、执行命令、搜索网络、操控浏览器，甚至控制整个计算机。

---

## 一、工具系统架构

```
┌─────────────────────────────────────────────────────────────┐
│                    工具系统全景                              │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              工具调用协议层                          │   │
│  │   Typed Tool Dispatch Registry                      │   │
│  │   工具名 → 处理器 → 执行 → 结果                     │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │  内置工具    │  │  MCP 工具    │  │   Skills 工具    │  │
│  │  (~40个)     │  │  (6000+应用) │  │  (自定义能力)    │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              权限控制层                              │   │
│  │   Always Deny → Always Allow → User-Gated           │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、内置工具完整列表

### 2.1 文件系统工具

| 工具 | 功能 | 权限级别 |
|------|------|---------|
| `read` | 读取文件内容 | Always Allow |
| `write` | 写入/创建文件 | User-Gated |
| `edit` | 精确编辑文件（替换特定内容） | User-Gated |
| `multi_edit` | 批量编辑多个文件 | User-Gated |
| `glob` | 文件模式匹配（如 `src/**/*.ts`） | Always Allow |
| `ls` | 列出目录内容 | Always Allow |

```bash
# read 工具示例
claude "读取 src/auth/login.ts 文件"

# write 工具示例（需要确认）
claude "创建 src/utils/format.ts 文件"

# edit 工具示例（精确替换，避免重写整个文件）
claude "将 login.ts 中的 'password' 变量名改为 'hashedPassword'"

# glob 工具示例
claude "找出所有 .test.ts 文件"
```

### 2.2 搜索工具

| 工具 | 功能 | 权限级别 |
|------|------|---------|
| `grep` | 正则表达式搜索文件内容 | Always Allow |
| `search_files` | 语义搜索文件 | Always Allow |
| `find_definition` | 查找函数/类定义 | Always Allow |
| `find_references` | 查找引用位置 | Always Allow |

```bash
# grep 工具示例
claude "在 src/ 目录中搜索所有使用 'console.log' 的地方"

# 组合使用：先搜索，再读取
claude "找到所有处理认证的函数，然后读取它们"
```

### 2.3 执行工具

| 工具 | 功能 | 权限级别 |
|------|------|---------|
| `bash` | 执行 shell 命令 | User-Gated |
| `python` | 执行 Python 代码 | User-Gated |
| `node` | 执行 Node.js 代码 | User-Gated |

```bash
# bash 工具示例
claude "运行测试套件"
# 内部：bash("npm test")

# 复杂命令
claude "构建项目并检查输出大小"
# 内部：bash("npm run build && du -sh dist/")
```

### 2.4 网络工具

| 工具 | 功能 | 权限级别 |
|------|------|---------|
| `web_fetch` | 获取网页内容 | User-Gated |
| `web_search` | 搜索网络 | User-Gated |
| `http_request` | 发送 HTTP 请求 | User-Gated |

```bash
# web_search 工具示例
claude "搜索 Next.js 14 App Router 的最新文档"

# web_fetch 工具示例
claude "获取 https://api.example.com/docs 的内容"
```

### 2.5 Git 工具

| 工具 | 功能 | 权限级别 |
|------|------|---------|
| `git_status` | 查看 git 状态 | Always Allow |
| `git_diff` | 查看差异 | Always Allow |
| `git_log` | 查看提交历史 | Always Allow |
| `git_commit` | 创建提交 | User-Gated |
| `git_branch` | 管理分支 | User-Gated |
| `git_worktree` | 管理工作树（多 Agent 关键） | User-Gated |

### 2.6 Computer Use 工具（2026年3月新增）

| 工具 | 功能 | 权限级别 |
|------|------|---------|
| `computer_use` | 控制鼠标和键盘 | User-Gated |
| `screenshot` | 截取屏幕 | Always Allow |
| `browser_action` | 控制浏览器 | User-Gated |

```bash
# computer_use 工具示例
claude "打开浏览器，访问 localhost:3000，截图并检查 UI"

# 自动化测试
claude "运行 E2E 测试，如果有 UI 问题，截图并报告"
```

---

## 三、工具调用协议

### 3.1 Typed Tool Dispatch Registry

```typescript
// 工具注册表（简化版）
interface ToolRegistry {
  [toolName: string]: {
    handler: ToolHandler;
    schema: JSONSchema;
    permissions: PermissionLevel;
    timeout: number;
  }
}

// 工具调用流程
async function dispatchTool(
  toolName: string,
  args: unknown
): Promise<ToolResult> {
  // 1. 查找工具
  const tool = registry[toolName];
  if (!tool) throw new Error(`Unknown tool: ${toolName}`);
  
  // 2. 验证参数
  validate(args, tool.schema);
  
  // 3. 权限检查
  await checkPermission(tool.permissions, toolName, args);
  
  // 4. 执行工具
  const result = await tool.handler(args);
  
  // 5. 返回结果
  return result;
}
```

### 3.2 工具调用的 JSON 格式

```json
// Claude 发出的工具调用请求
{
  "type": "tool_use",
  "id": "toolu_01A09q90qw90lq917835lq9",
  "name": "bash",
  "input": {
    "command": "npm test -- --coverage",
    "timeout": 120
  }
}

// 工具执行结果
{
  "type": "tool_result",
  "tool_use_id": "toolu_01A09q90qw90lq917835lq9",
  "content": "Tests: 47 passed, 0 failed\nCoverage: 87.3%"
}
```

### 3.3 并行工具调用

```python
# Claude 可以并行调用多个工具（减少延迟）
# 示例：同时读取多个文件
[
  {"name": "read", "input": {"path": "src/auth.ts"}},
  {"name": "read", "input": {"path": "src/user.ts"}},
  {"name": "read", "input": {"path": "src/db.ts"}},
]
# 三个文件同时读取，而不是顺序读取
```

---

## 四、权限控制系统

### 4.1 三层权限模型

```
┌─────────────────────────────────────────────────────────┐
│  Layer 1: Always Deny（永远拒绝）                        │
│  ├── rm -rf /（删除根目录）                              │
│  ├── 修改 /etc/passwd（系统用户文件）                    │
│  ├── 访问其他用户的私有数据                              │
│  └── 执行已知恶意命令                                    │
├─────────────────────────────────────────────────────────┤
│  Layer 2: Always Allow（永远允许）                       │
│  ├── 读取项目文件（read, grep, glob）                    │
│  ├── 查看 git 状态（git_status, git_diff）               │
│  ├── 搜索代码库（search_files, find_definition）         │
│  └── 查看系统信息（ls, pwd, env）                        │
├─────────────────────────────────────────────────────────┤
│  Layer 3: User-Gated（用户审批）                         │
│  ├── 写入/修改文件（write, edit）                        │
│  ├── 执行命令（bash, python）                            │
│  ├── 网络请求（web_fetch, http_request）                 │
│  └── 安装依赖（bash("npm install")）                     │
└─────────────────────────────────────────────────────────┘
```

### 4.2 权限配置

```json
// .claude/settings.json
{
  "permissions": {
    "allow": [
      "bash(npm test)",
      "bash(npm run lint)",
      "bash(npm run build)",
      "bash(git add *)",
      "bash(git commit *)",
      "write(src/**)",
      "write(tests/**)"
    ],
    "deny": [
      "bash(rm -rf *)",
      "bash(sudo *)",
      "bash(curl * | bash)",
      "write(/etc/**)",
      "write(~/.ssh/**)"
    ]
  }
}
```

### 4.3 Auto Mode 权限

```bash
# Auto Mode：跳过用户审批（CI/CD 场景）
claude --dangerously-skip-permissions -p "运行完整测试"

# 在 Auto Mode 下，Harness 使用 AI 分类器判断风险
# 高风险操作仍然会被拒绝
```

---

## 五、MCP 扩展协议

### 5.1 什么是 MCP？

MCP（Model Context Protocol）是 Anthropic 开发的开放标准，允许 Claude 连接外部服务和工具：

```
Claude Code
    │
    └── MCP 客户端
            │
            ├── Google Drive MCP Server
            ├── GitHub MCP Server
            ├── Slack MCP Server
            ├── Jira MCP Server
            ├── Notion MCP Server
            ├── Stripe MCP Server
            ├── Figma MCP Server
            └── Zapier MCP Server（连接 6000+ 应用）
```

### 5.2 配置 MCP 服务器

```json
// .claude/mcp.json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    },
    "slack": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-slack"],
      "env": {
        "SLACK_BOT_TOKEN": "${SLACK_BOT_TOKEN}"
      }
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "DATABASE_URL": "${DATABASE_URL}"
      }
    }
  }
}
```

### 5.3 MCP 工具使用示例

```bash
# 使用 GitHub MCP
claude "在 GitHub 上创建一个 issue，标题：修复登录 bug，分配给 @alice"

# 使用 Slack MCP
claude "在 #dev-team 频道发送消息：部署已完成，版本 v2.1.0"

# 使用 Postgres MCP
claude "查询数据库，找出最近 7 天注册的用户数量"

# 使用 Figma MCP
claude "获取 Figma 设计稿中的颜色变量，更新到 tailwind.config.js"
```

### 5.4 自定义 MCP 服务器

```typescript
// 创建自定义 MCP 服务器
import { Server } from "@modelcontextprotocol/sdk/server/index.js";

const server = new Server({
  name: "my-custom-server",
  version: "1.0.0",
});

// 注册自定义工具
server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [{
    name: "deploy_to_staging",
    description: "部署到测试环境",
    inputSchema: {
      type: "object",
      properties: {
        version: { type: "string" }
      }
    }
  }]
}));

// 实现工具逻辑
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  if (request.params.name === "deploy_to_staging") {
    const { version } = request.params.arguments;
    // 执行部署逻辑
    return { content: [{ type: "text", text: `已部署版本 ${version}` }] };
  }
});
```

---

## 六、Skills：模块化能力

### 6.1 什么是 Skills？

Skills 是包含指令、脚本和资源的 Markdown 文件，可以被 Claude 加载和执行：

```markdown
<!-- .claude/skills/deploy.md -->
# 部署技能

## 描述
自动化部署流程，包含构建、测试、部署三个阶段

## 前置条件
- 已配置 AWS CLI
- 已设置 DEPLOY_KEY 环境变量

## 步骤
1. 运行测试：`npm test`
2. 构建项目：`npm run build`
3. 上传到 S3：`aws s3 sync dist/ s3://my-bucket/`
4. 清除 CDN 缓存：`aws cloudfront create-invalidation --distribution-id $CF_ID --paths "/*"`

## 回滚方案
如果部署失败，执行：`aws s3 sync s3://my-bucket-backup/ s3://my-bucket/`
```

### 6.2 使用 Skills

```bash
# 加载并执行 Skill
claude "使用 deploy skill 部署到生产环境"

# 列出可用 Skills
claude /skills

# 创建新 Skill
claude "创建一个 skill，用于自动化数据库备份"
```

---

## 七、客户端工具 vs 服务端工具

### 7.1 两种工具类型

```
客户端工具（Client-side Tools）：
├── 在用户的应用中执行
├── 需要用户应用返回结果给 Claude
├── 适合：访问本地资源、私有 API
└── 示例：读取本地文件、调用内部 API

服务端工具（Server-side Tools）：
├── 在 Anthropic 基础设施上执行
├── Claude 直接获取结果
├── 适合：网络搜索、公共 API
└── 示例：web_search、web_fetch
```

### 7.2 工具选择策略

```bash
# 优先使用服务端工具（无需用户干预）
claude "搜索最新的 React 18 文档"  # 使用 web_search（服务端）

# 必要时使用客户端工具
claude "读取我的本地配置文件"  # 使用 read（客户端）
```

---

## 八、工具使用最佳实践

### 8.1 工具组合策略

```bash
# 好的工具组合：先搜索，再精确读取
claude "找到所有处理支付的代码，然后分析安全漏洞"
# 内部：grep("payment") → read(matched_files) → analyze

# 好的工具组合：先规划，再执行
claude --plan "重构整个认证模块"
# 内部：read(auth_files) → plan → confirm → edit(files)
```

### 8.2 减少不必要的工具调用

```bash
# 不好：多次读取同一文件
claude "读取 config.ts，然后再读取 config.ts 检查一下"

# 好：一次读取，多次分析
claude "读取 config.ts，分析所有潜在问题"
```

### 8.3 工具超时处理

```bash
# 对于长时间运行的命令，设置合理的超时
claude "运行完整的集成测试套件（可能需要 5 分钟）"

# 对于可能超时的操作，使用后台执行
claude "在后台运行构建，完成后通知我"
```

---

## 九、常见问题

### Q: 为什么 Claude 有时候不使用工具，而是直接回答？

Claude 会根据任务复杂度决定是否使用工具。对于简单的问题（如解释概念），直接回答更高效。对于需要实际操作的任务，Claude 会使用工具。

### Q: MCP 工具和内置工具有什么区别？

内置工具是 Claude Code 自带的，无需配置。MCP 工具需要安装和配置 MCP 服务器，但可以连接几乎任何外部服务。

### Q: 如何知道 Claude 使用了哪些工具？

```bash
# 启用工具调用日志
claude --verbose

# 或者查看会话统计
claude /status
# 输出：Tools called: 47 (read: 12, bash: 8, grep: 15, write: 12)
```

---

*本文档基于 Claude Code 泄漏源码分析及官方文档整理，部分内部实现细节可能随版本更新而变化。*
