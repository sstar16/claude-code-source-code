# CLAUDE.md — Claude Code 源码仓库指南

> **AI 助手上下文说明**：本仓库为 **Claude Code v2.1.88** 的反编译/重建 TypeScript 源码，提取自已发布的 npm 包 `@anthropic-ai/claude-code`。所有代码均为 Anthropic 的知识产权，本仓库仅供研究和教育目的使用。

---

## 仓库概览

| 属性 | 值 |
|------|-----|
| 版本 | 2.1.88 |
| 语言 | TypeScript + TSX (React) |
| 运行时 | Node.js ≥ 18 |
| 构建工具 | esbuild（自定义多阶段流水线） |
| UI 框架 | Ink（终端版 React） |
| 源文件数 | ~1,884 个文件，~205K 行代码 |
| 入口点 | `src/entrypoints/cli.tsx` |

**重要说明**：108 个功能门控模块缺失（Anthropic 内部基础设施）。源码中的 `feature('FLAG')` 调用在本构建中均转换为 `false`。完整列表请参见 README.md。

---

## 目录结构

```
.
├── src/
│   ├── entrypoints/          # CLI & SDK 启动入口
│   │   ├── cli.tsx           # 主 CLI 入口（快速路径标志、环境初始化）
│   │   ├── init.ts           # 异步应用初始化（遥测、设置、OTEL）
│   │   ├── mcp.ts            # MCP 服务器入口
│   │   └── sdk/              # SDK 类型导出
│   ├── commands/             # 88+ 个子命令（每个命令一个文件）
│   ├── components/           # React/Ink UI 组件（~33 个子目录）
│   ├── services/             # 核心服务（见服务章节）
│   ├── tools/                # 40+ 个可执行工具（见工具章节）
│   ├── state/                # 应用状态（AppState、AppStateStore）
│   ├── context/              # React 上下文（通知、统计、语音等）
│   ├── utils/                # 150+ 个工具模块
│   ├── ink/                  # 终端渲染辅助
│   ├── skills/               # 内置技能系统
│   ├── plugins/              # 插件管理
│   ├── bootstrap/            # 启动状态与初始化辅助
│   ├── query.ts              # 主 API 查询引擎（68KB）
│   ├── QueryEngine.ts        # 查询编排（46KB）
│   ├── main.tsx              # CLI 命令设置与 REPL 启动（800KB）
│   ├── commands.ts           # 命令注册表（条件导入）
│   ├── Tool.ts               # 工具框架（ToolInputJSONSchema、执行）
│   ├── Task.ts               # 任务抽象
│   ├── App.tsx               # 顶层 React 组件树
│   ├── AppState.tsx          # 应用状态 React 上下文提供者
│   ├── AppStateStore.ts      # 类 Zustand 状态存储
│   ├── context.ts            # 系统/用户上下文提供者（git、CLAUDE.md、日期）
│   └── costHook.ts           # Token 与费用追踪
├── scripts/
│   ├── build.mjs             # 基于 esbuild 的 4 阶段构建脚本
│   ├── prepare-src.mjs       # 源码准备（将 src/ 复制到 build-src/）
│   ├── transform.mjs         # 代码转换（功能标志、版本注入）
│   └── stub-modules.mjs      # 自动生成缺失模块的存根
├── stubs/
│   ├── bun-bundle.ts         # feature() 存根 → 始终返回 false
│   └── global.d.ts           # 全局类型（MACRO.VERSION、MACRO.BUILD_TIME）
├── docs/
│   ├── en/                   # 英文深度分析报告
│   └── zh/                   # 中文深度分析报告
├── vendor/                   # 原生/外部依赖
├── types/                    # 旧版类型存根
├── tsconfig.json
├── package.json
├── README.md                 # 架构概览 + 108 个缺失模块列表
└── QUICKSTART.md             # 构建说明
```

---

## 构建系统

### 命令

```bash
npm run prepare-src   # 将 src/ 复制到 build-src/（保留原始文件）
npm run build         # 完整构建：prepare-src + esbuild 流水线 → dist/cli.js
npm run check         # 仅 TypeScript 类型检查（不输出文件）
npm start             # 运行构建后的 CLI：node dist/cli.js
```

### 构建流水线（`scripts/build.mjs` 中的 4 个阶段）

1. **准备阶段** — 将 `src/` 复制到 `build-src/`（原始文件不变）
2. **转换阶段** — 重写 `build-src/` 中的源码：
   - `feature('FLAG')` → `false`（编译时 Bun 内置 → 运行时常量）
   - `MACRO.VERSION` → `'2.1.88'`
   - `import from 'bun:bundle'` → 存根
3. **入口包装阶段** — 创建 `build-src/entry.js`
4. **打包阶段** — 迭代执行 esbuild；遇到模块缺失错误时，`stub-modules.mjs` 自动生成存根并重试

### TypeScript 配置（`tsconfig.json`）

- `target`：ES2022
- `lib`：ES2022、DOM（兼容性用）
- `jsx`：`react-jsx`
- `moduleResolution`：`bundler`
- `noEmit`：true（由 esbuild 负责输出）
- 路径：`src/*` 已映射用于内部导入

---

## 应用架构

### 启动流程

```
cli.tsx
  └─ 快速路径检查（--version、--dump-system-prompt、--daemon-worker）
  └─ 动态导入 main.tsx（延迟加载以提升启动性能）
       └─ Commander.js CLI 设置
       └─ init.ts（遥测、OTEL、设置、策略）
       └─ Ink REPL 渲染（App.tsx）
            └─ AppStateProvider → FpsMetricsProvider → StatsProvider
            └─ 通过 QueryEngine.ts → query.ts 执行工具循环
```

### 查询引擎（`query.ts`、`QueryEngine.ts`）

核心 Agent 循环：
1. 从 `context.ts` 构建系统提示词（git 状态、CLAUDE.md、日期、用户上下文）
2. 通过 `@anthropic-ai/sdk` 调用 Claude API
3. 处理流式响应（文本块 + tool_use 块）
4. 执行工具调用，收集结果
5. 按需压缩上下文（microcompact / reactive / snip 策略）
6. 循环直到 `stop_reason === 'end_turn'` 或用户中断

### 状态管理

- **AppStateStore**（`AppStateStore.ts`）：类 Zustand 存储，提供 `getState()` / `setState()`
- **AppState 上下文**（`AppState.tsx`）：包装存储的 React 上下文
- 其他上下文：`NotificationContext`、`StatsContext`、`VoiceContext`、`ModalContext`、`OverlayContext`
- 记忆化：`memoize()` 在工具模块中大量使用（如 `getSystemContext`、`getUserContext`）

---

## 工具系统

工具位于 `src/tools/` 中，每个工具导出一个符合 `Tool.ts` 框架的工具对象。

### 可用工具（40+）

| 工具 | 文件 | 用途 |
|------|------|------|
| `AgentTool` | `AgentTool/` | 创建子 Agent |
| `AskUserQuestionTool` | `AskUserQuestionTool/` | 向用户提问 |
| `BashTool` | `BashTool/` | 执行 Shell 命令 |
| `BriefTool` | `BriefTool/` | 生成任务摘要 |
| `ConfigTool` | `ConfigTool/` | 管理配置 |
| `EnterPlanModeTool` | `EnterPlanModeTool/` | 进入计划模式 |
| `ExitPlanModeTool` | `ExitPlanModeTool/` | 退出计划模式 |
| `EnterWorktreeTool` | `EnterWorktreeTool/` | 进入 git worktree |
| `ExitWorktreeTool` | `ExitWorktreeTool/` | 退出 git worktree |
| `FileEditTool` | `FileEditTool/` | 编辑文件 |
| `FileReadTool` | `FileReadTool/` | 读取文件 |
| `FileWriteTool` | `FileWriteTool/` | 写入文件 |
| `GlobTool` | `GlobTool/` | 文件模式匹配 |
| `GrepTool` | `GrepTool/` | 内容搜索（ripgrep） |
| `LSPTool` | `LSPTool/` | 语言服务器协议 |
| `MCPTool` | `MCPTool/` | MCP 服务器交互 |
| `NotebookEditTool` | `NotebookEditTool/` | 编辑 Jupyter 笔记本 |
| `RemoteTriggerTool` | `RemoteTriggerTool/` | 触发远程 Agent |
| `REPLTool` | `REPLTool/` | 交互式 REPL |
| `SkillTool` | `SkillTool/` | 执行技能 |
| `TaskCreateTool` | `TaskCreateTool/` | 创建异步任务 |
| `TodoWriteTool` | `TodoWriteTool/` | 管理待办列表 |
| `WebFetchTool` | `WebFetchTool/` | 抓取网页内容 |
| `WebSearchTool` | `WebSearchTool/` | 网页搜索 |

### 工具框架（`Tool.ts`）

```typescript
interface Tool {
  name: string;
  description: string;
  inputSchema: ToolInputJSONSchema;   // 参数的 JSON Schema
  execute(input, context: ToolUseContext): Promise<ToolResult>;
}
```

权限流程：用户设置 → 自动模式委托 → 单工具 `needsPermission()` → 提示用户。

---

## 核心服务（`src/services/`）

| 服务 | 用途 |
|------|------|
| `analytics/` | 遥测：两个数据接收端（Anthropic 第一方 + Datadog 第三方），GrowthBook 功能标志 |
| `api/` | Claude API 客户端、启动数据、Files API、请求日志 |
| `mcp/` | Model Context Protocol 服务器管理、资源处理 |
| `compact/` | 上下文压缩策略（microcompact、reactive、snip） |
| `remoteManagedSettings/` | 每小时轮询 `/api/claude_code/settings` 获取远程配置 |
| `policyLimits/` | Token/配额追踪与执行 |
| `plugins/` | 插件发现、加载与管理 |
| `lsp/` | 语言服务器协议集成 |
| `oauth/` | OAuth 认证流程 |
| `toolUseSummary/` | 工具调用结果摘要 |

---

## 命令（`src/commands/`）

88+ 个子命令通过 `main.tsx` 中的 Commander.js 注册。注册表位于 `commands.ts`，使用条件导入。主要命令包括：

- `add-dir`、`remove-dir` — 管理监听目录
- `commit` — AI 辅助 git 提交
- `review` — 代码审查
- `mcp` — MCP 服务器管理
- `config` — 配置管理
- `migrate` — 设置迁移

仅限 Anthropic 员工的内部命令通过 `USER_TYPE === 'ant'` 检查进行门控。

---

## 关键约定

### 功能门控

```typescript
import { feature } from 'bun:bundle';

if (feature('VOICE_MODE')) {
  // 在 npm 构建中此代码为死代码 — feature() 在此返回 false
}
```

不要在 `feature()` 块内添加期望在本构建中运行的代码。所有 108 个门控功能均不存在。

### 懒加载导入（性能模式）

```typescript
// 用于打破循环依赖并延迟模块加载
const getComponent = (): typeof import('./Component.js') =>
  require('./Component.js');
```

### 记忆化

```typescript
import { memoize } from './utils/memoize.js';
export const getSystemContext = memoize(async () => { ... });
// 使用 getSystemContext.cache.clear() 清除缓存
```

### 上下文构建（`context.ts`）

系统提示词由以下内容组装：
1. 静态系统上下文（操作系统、Shell、日期、Node 版本）
2. 动态 git 上下文（分支、状态、最近提交）— 在 `NODE_ENV === 'test'` 时跳过
3. CLAUDE.md 文件（本文件 + 父目录中的任何文件）
4. 来自 `getUserContext()` 的用户特定上下文

### 遥测

大多数用户操作会触发两个分析数据接收端：
- **第一方** → Anthropic 服务器（无 UI 退出选项）
- **第三方** → Datadog

设置 `OTEL_LOG_TOOL_DETAILS=1` 可在日志中捕获完整的工具输入。

---

## 开发工作流

### 使用本代码库

由于这是反编译源码，**不要期望开箱即可编译**。请按照 QUICKSTART.md 操作：

1. `npm install` — 安装构建依赖
2. `npm run build` — 执行 4 阶段流水线；自动生成缺失模块存根
3. `npm start` — 运行构建后的 CLI

### 类型检查

```bash
npm run check   # tsc --noEmit（报告类型错误但不构建）
```

第三方或存根文件中的 TypeScript 错误是预期的，可以忽略。

### 添加新存根

如果构建因模块缺失错误而失败，`scripts/stub-modules.mjs` 会在下次构建时自动生成存根。如需手动添加存根，在 `stubs/` 中添加空导出文件，并在 `scripts/build.mjs` 的别名映射中引用它。

### 修改工具

`src/tools/<ToolName>/` 中的每个工具通常包含：
- `index.ts` — 主工具导出
- `utils.ts` — 辅助函数
- `testing/` — 测试夹具（部分工具包含）

工具必须实现 `src/Tool.ts` 中的 `Tool` 接口。在 `src/commands.ts` 或 `main.tsx` 中的工具注册表注册新工具。

---

## 重要导航文件

| 文件 | 重要原因 |
|------|---------|
| `src/query.ts` | 核心 Agent 循环 — 理解 Claude 处理请求方式的起点 |
| `src/main.tsx` | 所有 CLI 命令和初始化 — 800KB，建议搜索而非线性阅读 |
| `src/Tool.ts` | 工具类型系统 — 修改任何工具前必读 |
| `src/context.ts` | 系统提示词的组装方式 |
| `src/utils/config.ts` | 用户配置处理 |
| `src/utils/permissions/` | 权限系统 — 安全敏感性修改时必读 |
| `src/services/analytics/` | 遥测 — 记录新事件前需了解 |
| `stubs/bun-bundle.ts` | 展示 `feature()` 的存根方式 |
| `scripts/build.mjs` | 完整构建流水线 — 修改构建配置前必读 |

---

## 本构建的已知限制

1. **108 个缺失模块** — 所有 `feature()` 门控的内部模块均不存在；存根自动生成
2. **Bun 内置 API** — `bun:bundle`、`bun:ffi` 及其他 Bun 特有 API 均已存根化
3. **React Compiler 输出** — 部分文件包含来自 React Compiler 的 `_c()` / 编译后记忆化模式，非手写代码
4. **反编译标识符** — 部分变量名可能经过压缩或描述性不强
5. **不含测试** — npm 包不包含测试文件；`NODE_ENV === 'test'` 分支存在但测试基础设施缺失

---

## 研究笔记

本仓库在 `docs/zh/` 中包含深度分析文档：

| 文档 | 主题 |
|------|------|
| `01-遥测与隐私分析.md` | 收集了哪些遥测数据、接收端、退出限制 |
| `02-隐藏功能与模型代号.md` | 模型代号（Capybara、Tengu、Numbat）、功能标志 |
| `03-卧底模式分析.md` | Anthropic 员工如何在公开仓库中隐藏 AI 身份 |
| `04-远程控制与紧急开关.md` | 远程设置轮询、紧急开关、GrowthBook 标志 |
| `05-未来路线图.md` | KAIROS 自主 Agent 模式、语音模式、17 个未发布工具 |
