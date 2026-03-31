# CLAUDE.md — Claude Code Source Code Repository Guide

> **Context for AI Assistants**: This is a decompiled/reconstructed TypeScript source for **Claude Code v2.1.88**, extracted from the published npm package `@anthropic-ai/claude-code`. All code is Anthropic's intellectual property. This repo exists for research and educational purposes only.

---

## Repository Overview

| Property | Value |
|----------|-------|
| Version | 2.1.88 |
| Language | TypeScript + TSX (React) |
| Runtime | Node.js ≥ 18 |
| Build tool | esbuild (custom multi-phase pipeline) |
| UI framework | Ink (React for terminals) |
| Source files | ~1,884 files, ~205K LOC |
| Entry point | `src/entrypoints/cli.tsx` |

**Important caveat**: 108 feature-gated modules are absent (internal Anthropic infrastructure). `feature('FLAG')` calls in source transform to `false` in this build. See README.md for the full list.

---

## Directory Structure

```
.
├── src/
│   ├── entrypoints/          # CLI & SDK bootstraps
│   │   ├── cli.tsx           # Main CLI entry (fast-path flags, env setup)
│   │   ├── init.ts           # Async app initialization (telemetry, settings, OTEL)
│   │   ├── mcp.ts            # MCP server entry
│   │   └── sdk/              # SDK type exports
│   ├── commands/             # 88+ subcommands (one file per command)
│   ├── components/           # React/Ink UI components (~33 subdirectories)
│   ├── services/             # Core services (see Services section)
│   ├── tools/                # 40+ executable tools (see Tools section)
│   ├── state/                # App state (AppState, AppStateStore)
│   ├── context/              # React contexts (notifications, stats, voice, etc.)
│   ├── utils/                # 150+ utility modules
│   ├── ink/                  # Terminal rendering helpers
│   ├── skills/               # Bundled skills system
│   ├── plugins/              # Plugin management
│   ├── bootstrap/            # Bootstrap state & init helpers
│   ├── query.ts              # Main API query engine (68KB)
│   ├── QueryEngine.ts        # Query orchestration (46KB)
│   ├── main.tsx              # CLI command setup & REPL bootstrap (800KB)
│   ├── commands.ts           # Command registry with conditional imports
│   ├── Tool.ts               # Tool framework (ToolInputJSONSchema, execution)
│   ├── Task.ts               # Task abstraction
│   ├── App.tsx               # Top-level React component tree
│   ├── AppState.tsx          # App state React context provider
│   ├── AppStateStore.ts      # Zustand-like state store
│   ├── context.ts            # System/user context providers (git, CLAUDE.md, date)
│   └── costHook.ts           # Token & cost tracking
├── scripts/
│   ├── build.mjs             # esbuild-based 4-phase build script
│   ├── prepare-src.mjs       # Source preparation (copies src → build-src)
│   ├── transform.mjs         # Code transforms (feature flags, version injection)
│   └── stub-modules.mjs      # Auto-generates stubs for missing modules
├── stubs/
│   ├── bun-bundle.ts         # feature() stub → always returns false
│   └── global.d.ts           # Global types (MACRO.VERSION, MACRO.BUILD_TIME)
├── docs/
│   ├── en/                   # English deep-analysis reports
│   └── zh/                   # Chinese deep-analysis reports
├── vendor/                   # Native/external dependencies
├── types/                    # Legacy type stubs
├── tsconfig.json
├── package.json
├── README.md                 # Architecture overview + 108 missing modules list
└── QUICKSTART.md             # Build instructions
```

---

## Build System

### Commands

```bash
npm run prepare-src   # Copy src/ → build-src/ (preserves originals)
npm run build         # Full build: prepare-src + esbuild pipeline → dist/cli.js
npm run check         # TypeScript type-check only (no emit)
npm start             # Run built CLI: node dist/cli.js
```

### Build Pipeline (4 phases in `scripts/build.mjs`)

1. **Prepare** – copy `src/` → `build-src/` (originals untouched)
2. **Transform** – rewrite source in `build-src/`:
   - `feature('FLAG')` → `false` (compile-time Bun intrinsic → runtime constant)
   - `MACRO.VERSION` → `'2.1.88'`
   - `import from 'bun:bundle'` → stub
3. **Entry wrapper** – create `build-src/entry.js`
4. **Bundle** – iterative esbuild run; on missing-module errors, `stub-modules.mjs` auto-generates stubs and retries

### TypeScript Configuration (`tsconfig.json`)

- `target`: ES2022
- `lib`: ES2022, DOM (for compatibility)
- `jsx`: `react-jsx`
- `moduleResolution`: `bundler`
- `noEmit`: true (esbuild handles emit)
- Paths: `src/*` mapped for internal imports

---

## Application Architecture

### Startup Flow

```
cli.tsx
  └─ fast-path checks (--version, --dump-system-prompt, --daemon-worker)
  └─ dynamic import main.tsx  (deferred for startup performance)
       └─ Commander.js CLI setup
       └─ init.ts (telemetry, OTEL, settings, policies)
       └─ Ink REPL render (App.tsx)
            └─ AppStateProvider → FpsMetricsProvider → StatsProvider
            └─ Tool loop via QueryEngine.ts → query.ts
```

### Query Engine (`query.ts`, `QueryEngine.ts`)

The core agent loop:
1. Build system prompt from `context.ts` (git status, CLAUDE.md, date, user context)
2. Call Claude API via `@anthropic-ai/sdk`
3. Process streaming response (text + tool_use blocks)
4. Execute tool calls, collect results
5. Compact context when needed (microcompact / reactive / snip strategies)
6. Loop until `stop_reason === 'end_turn'` or user interrupts

### State Management

- **AppStateStore** (`AppStateStore.ts`): Zustand-like store with `getState()` / `setState()`
- **AppState context** (`AppState.tsx`): React context wrapping the store
- Additional contexts: `NotificationContext`, `StatsContext`, `VoiceContext`, `ModalContext`, `OverlayContext`
- Memoization: `memoize()` used heavily in utils (e.g., `getSystemContext`, `getUserContext`)

---

## Tool System

Tools live in `src/tools/` — each exports a tool object conforming to the `Tool.ts` framework.

### Available Tools (40+)

| Tool | File | Purpose |
|------|------|---------|
| `AgentTool` | `AgentTool/` | Spawn sub-agents |
| `AskUserQuestionTool` | `AskUserQuestionTool/` | Ask user clarifying questions |
| `BashTool` | `BashTool/` | Execute shell commands |
| `BriefTool` | `BriefTool/` | Generate task briefs |
| `ConfigTool` | `ConfigTool/` | Manage configuration |
| `EnterPlanModeTool` | `EnterPlanModeTool/` | Switch to plan mode |
| `ExitPlanModeTool` | `ExitPlanModeTool/` | Exit plan mode |
| `EnterWorktreeTool` | `EnterWorktreeTool/` | Enter git worktree |
| `ExitWorktreeTool` | `ExitWorktreeTool/` | Exit git worktree |
| `FileEditTool` | `FileEditTool/` | Edit files |
| `FileReadTool` | `FileReadTool/` | Read files |
| `FileWriteTool` | `FileWriteTool/` | Write files |
| `GlobTool` | `GlobTool/` | File pattern matching |
| `GrepTool` | `GrepTool/` | Content search (ripgrep) |
| `LSPTool` | `LSPTool/` | Language Server Protocol |
| `MCPTool` | `MCPTool/` | MCP server interaction |
| `NotebookEditTool` | `NotebookEditTool/` | Edit Jupyter notebooks |
| `RemoteTriggerTool` | `RemoteTriggerTool/` | Trigger remote agents |
| `REPLTool` | `REPLTool/` | Interactive REPL |
| `SkillTool` | `SkillTool/` | Execute skills |
| `TaskCreateTool` | `TaskCreateTool/` | Create async tasks |
| `TodoWriteTool` | `TodoWriteTool/` | Manage todo lists |
| `WebFetchTool` | `WebFetchTool/` | Fetch web content |
| `WebSearchTool` | `WebSearchTool/` | Web search |

### Tool Framework (`Tool.ts`)

```typescript
interface Tool {
  name: string;
  description: string;
  inputSchema: ToolInputJSONSchema;   // JSON Schema for parameters
  execute(input, context: ToolUseContext): Promise<ToolResult>;
}
```

Permission flow: user settings → auto-mode delegation → per-tool `needsPermission()` → prompt user.

---

## Key Services (`src/services/`)

| Service | Purpose |
|---------|---------|
| `analytics/` | Telemetry: two sinks (Anthropic 1P + Datadog 3P), GrowthBook feature flags |
| `api/` | Claude API client, bootstrap data, Files API, request logging |
| `mcp/` | Model Context Protocol server management, resource handling |
| `compact/` | Context compaction strategies (microcompact, reactive, snip) |
| `remoteManagedSettings/` | Hourly polling of `/api/claude_code/settings` for remote config |
| `policyLimits/` | Token/quota tracking and enforcement |
| `plugins/` | Plugin discovery, loading, and management |
| `lsp/` | Language Server Protocol integration |
| `oauth/` | OAuth authentication flow |
| `toolUseSummary/` | Summarization for tool call results |

---

## Commands (`src/commands/`)

88+ subcommands registered via Commander.js in `main.tsx`. The registry is in `commands.ts` with conditional imports. Key commands include:

- `add-dir`, `remove-dir` — manage watched directories
- `commit` — AI-assisted git commits
- `review` — code review
- `mcp` — MCP server management
- `config` — configuration management
- `migrate` — settings migration

Internal (Anthropic-employee-only) commands are gated by `USER_TYPE === 'ant'` checks.

---

## Key Conventions

### Feature Gating

```typescript
import { feature } from 'bun:bundle';

if (feature('VOICE_MODE')) {
  // This code is dead in npm builds — feature() → false here
}
```

Never add code inside `feature()` blocks expecting it to run in this build. All 108 gated features are absent.

### Lazy Imports (Performance Pattern)

```typescript
// Used to break circular dependencies and defer module loading
const getComponent = (): typeof import('./Component.js') =>
  require('./Component.js');
```

### Memoization

```typescript
import { memoize } from './utils/memoize.js';
export const getSystemContext = memoize(async () => { ... });
// Invalidate with: getSystemContext.cache.clear()
```

### Context Building (`context.ts`)

The system prompt is assembled from:
1. Static system context (OS, shell, date, Node version)
2. Dynamic git context (branch, status, recent commits) — skipped in `NODE_ENV === 'test'`
3. CLAUDE.md files (this file + any in parent directories)
4. User-specific context from `getUserContext()`

### Telemetry

Two analytics sinks fire on most user actions:
- **1st-party** → Anthropic servers (no UI opt-out)
- **3rd-party** → Datadog

Set `OTEL_LOG_TOOL_DETAILS=1` to capture full tool inputs in logs.

---

## Development Workflow

### Working with This Codebase

Since this is decompiled source, **do not expect a clean compile out of the box**. Follow QUICKSTART.md:

1. `npm install` — install build dependencies
2. `npm run build` — runs 4-phase pipeline; auto-stubs missing modules
3. `npm start` — run the built CLI

### Type Checking

```bash
npm run check   # tsc --noEmit (reports type errors without building)
```

TypeScript errors in 3rd-party or stub files are expected and can be ignored.

### Adding New Stubs

If the build fails with a missing module error, `scripts/stub-modules.mjs` auto-generates a stub on the next build attempt. For manual stubs, add an empty export file to `stubs/` and reference it in `scripts/build.mjs` alias map.

### Modifying Tools

Each tool in `src/tools/<ToolName>/` typically has:
- `index.ts` — main tool export
- `utils.ts` — helper functions
- `testing/` — test fixtures (present in some tools)

Tools must implement the `Tool` interface from `src/Tool.ts`. Register new tools in `src/commands.ts` or the tool registry in `main.tsx`.

---

## Important Files for Navigation

| File | Why it matters |
|------|----------------|
| `src/query.ts` | Core agent loop — start here to understand how Claude processes requests |
| `src/main.tsx` | All CLI commands and initialization — 800KB, search rather than read linearly |
| `src/Tool.ts` | Tool type system — required reading before modifying any tool |
| `src/context.ts` | How the system prompt is assembled |
| `src/utils/config.ts` | User config handling |
| `src/utils/permissions/` | Permission system — important for security-sensitive changes |
| `src/services/analytics/` | Telemetry — understand before logging new events |
| `stubs/bun-bundle.ts` | Shows how `feature()` is stubbed |
| `scripts/build.mjs` | Full build pipeline — read before changing build config |

---

## Known Limitations of This Build

1. **108 missing modules** — all `feature()`-gated internal modules are absent; stubs are generated automatically
2. **Bun intrinsics** — `bun:bundle`, `bun:ffi`, and other Bun-specific APIs are stubbed
3. **React Compiler output** — some files contain `_c()` / compiled memoization patterns from React Compiler, not hand-written code
4. **Decompiled identifiers** — some variable names may be minified or non-descriptive
5. **No tests included** — the npm package does not ship test files; `NODE_ENV === 'test'` branches exist but test infrastructure is absent

---

## Research Notes

This repository includes deep-analysis documents in `docs/en/`:

| Doc | Topic |
|-----|-------|
| `01-telemetry-and-privacy.md` | What telemetry is collected, sinks, opt-out limitations |
| `02-hidden-features-and-codenames.md` | Model codenames (Capybara, Tengu, Numbat), feature flags |
| `03-undercover-mode.md` | How Anthropic employees hide AI authorship in public repos |
| `04-remote-control-and-killswitches.md` | Remote settings polling, killswitches, GrowthBook flags |
| `05-future-roadmap.md` | KAIROS autonomous agent mode, voice mode, 17 unreleased tools |
