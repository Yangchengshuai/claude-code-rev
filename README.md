<div align="center">

# Claude Code — Restored Source Tree

**Anthropic 官方 CLI 工具 Claude Code 的源码逆向还原与深度分析项目**

[![TypeScript](https://img.shields.io/badge/TypeScript-ESNext-3178C6?logo=typescript)](https://www.typescriptlang.org/)
[![React + Ink](https://img.shields.io/badge/UI-React%20%2B%20Ink-61DAFB?logo=react)](https://github.com/vadimdemedes/ink)
[![Bun](https://img.shields.io/badge/Runtime-Bun%20%E2%89%A5%201.3.5-FBF0DB?logo=bun)](https://bun.sh/)
[![License](https://img.shields.io/badge/License-SEE_LICENSE_IN_LICENSE.md-blue)]()

</div>

---

English | [中文](#中文说明)

---

## Background

On **March 31, 2026**, Claude Code source material was discovered to be publicly accessible through `.map` files shipped in the npm distribution. The published source maps referenced unobfuscated TypeScript sources hosted in Anthropic's R2 storage bucket, making the entire `src/` snapshot publicly downloadable.

This repository is a **restored Claude Code source tree** based on that snapshot, with additional backfilling of missing modules, compatibility shims for unrecoverable native bindings, and **comprehensive architectural analysis documentation**.

> This repository is for **educational and research purposes only**. It does not claim ownership of the original code, and is not affiliated with, endorsed by, or maintained by Anthropic.

---

## Scale

| Metric | Count |
|--------|-------|
| TypeScript / TSX files | 1,989 |
| Total lines of code | ~514,000 |
| Source directories | 37 |
| Built-in tools | 50+ |
| CLI subcommands | 195 |
| React / Ink components | 357 |
| React hooks | 121 |
| Utility modules | 555 |

---

## Architecture

### Bootstrap Flow

```
bun run dev
  │
  ├─► src/bootstrap-entry.ts        ─── Initialize macro, route to CLI
  │     └─► src/entrypoints/cli.tsx  ─── Fast-path routing, subcommand dispatch
  │           └─► src/main.tsx       ─── Main REPL (4,691 lines)
  │                 │
  │                 ├── Parse CLI args (Commander.js)
  │                 ├── Initialize auth & permissions
  │                 ├── Assemble tool pool (50+ tools + MCP + plugins)
  │                 ├── Configure model & API
  │                 ├── Load session & memories
  │                 └── Launch Ink TUI (React terminal UI)
  │
  └─► src/dev-entry.ts              ─── Dev mode: scan missing imports
```

### System Layers

```
┌────────────────────────────────────────────────────────────┐
│                      UI Layer                               │
│   Ink TUI (25K LOC) · Vim Input · Keybindings · Components │
├────────────────────────────────────────────────────────────┤
│                   Application Layer                         │
│   Main REPL (4.7K) · Query Engine · 195 Commands · Cron    │
├────────────────────────────────────────────────────────────┤
│                     Tool Layer (42K LOC)                    │
│   Bash · File R/W/Edit · Grep · Glob · Web · LSP · MCP    │
│   Agent · Skill · Task · Plan · Worktree · Notebook · Cron │
├────────────────────────────────────────────────────────────┤
│                  Services Layer (80K+ LOC)                  │
│   API Client · MCP Client · OAuth · Plugins · Voice        │
│   LSP · Compaction · Analytics · Rate Limiting              │
├────────────────────────────────────────────────────────────┤
│              Bridge & Communication Layer                   │
│   Desktop/IDE Bridge · WebSocket · JWT · Session Mgmt      │
├────────────────────────────────────────────────────────────┤
│                  Infrastructure Layer                       │
│   State Store · 121 Hooks · 555 Utils · 18 Type Defs      │
└────────────────────────────────────────────────────────────┘
```

### Core Modules

| Module | Lines | Purpose |
|--------|-------|---------|
| `src/main.tsx` | 4,691 | Main REPL — command dispatch, tool loop, TUI |
| `src/services/api/claude.ts` | 3,419 | Anthropic API client |
| `src/services/mcp/client.ts` | 3,348 | MCP protocol client |
| `src/bridge/bridgeMain.ts` | 2,999 | Desktop/IDE remote bridge |
| `src/bridge/replBridge.ts` | 2,406 | REPL-specific bridge |
| `src/services/mcp/auth.ts` | 2,465 | MCP OAuth authentication |
| `src/ink/ink.tsx` | 1,722 | Custom Ink terminal renderer |
| `src/services/compact/compact.ts` | 1,705 | Session memory compaction |
| `src/QueryEngine.ts` | 793 | Query orchestration & tool-call loop |
| `src/Tool.ts` | 793 | Tool base class, `buildTool()` factory |

### Directory Map

```
src/
├── tools/          # 50+ agent tool implementations (42K LOC)
├── components/     # 357 React/Ink TUI components
├── utils/          # 555 utility modules (auth, shell, streaming, etc.)
├── services/       # API, MCP, OAuth, LSP, voice, plugins, analytics
├── hooks/          # 121 React hooks (input, IDE, permissions, tasks)
├── commands/       # 195 CLI subcommands (git, config, skills, etc.)
├── bridge/         # Desktop/IDE remote bridge (WebSocket, JWT)
├── ink/            # Ink framework customization & rendering
├── state/          # App state store (immutable, custom)
├── types/          # TypeScript type definitions
├── skills/         # Bundled skill system
├── plugins/        # Plugin architecture
├── vim/            # Vim-mode input handling
├── keybindings/    # Keybinding definitions & dispatch
├── memdir/         # Persistent memory system
├── buddy/          # Animated companion sprites
├── voice/          # Voice command support
├── coordinator/    # Multi-agent coordination
├── migrations/     # Version migration scripts
├── context/        # React context providers
├── schemas/        # JSON schema definitions
├── screens/        # Full-screen UI screens
├── bootstrap/      # Bootstrap utilities
├── entrypoints/    # CLI & assistant entry points
├── ssh/            # SSH session support
├── proactive/      # Proactive assistance
├── jobs/           # Background job processing
├── native-ts/      # Native module TypeScript implementations
├── outputStyles/   # Output formatting styles
└── moreright/      # Additional UI/UX features

shims/              # Compatibility shims for unrecoverable native modules
vendor/             # Third-party native module sources
docs/analysis/      # Comprehensive codebase analysis documents
```

---

## Tool System

Every tool Claude Code can invoke is a self-contained module with its own input schema, permission model, and execution logic.

### Tool Catalog

| Tool | Description |
|------|-------------|
| `BashTool` | Shell command execution with sandbox & permission model |
| `PowerShellTool` | Windows PowerShell equivalent |
| `FileReadTool` | File reading with PDF, image, notebook support |
| `FileWriteTool` | File creation / overwrite |
| `FileEditTool` | Partial file modification (string replacement) |
| `GlobTool` | Fast file pattern matching search |
| `GrepTool` | ripgrep-based content search |
| `WebSearchTool` | Web search with filtering |
| `WebFetchTool` | Fetch and convert web content |
| `LSPTool` | Language Server Protocol (definition, references, hover, diagnostics) |
| `MCPTool` | MCP server tool invocation |
| `AgentTool` | Sub-agent spawning (explore, plan, verify, etc.) |
| `SkillTool` | Skill execution |
| `TaskCreate/Get/Update/List/Stop` | Task lifecycle management |
| `CronCreate/List/Delete` | Scheduled trigger management |
| `EnterPlanModeTool` / `ExitPlanModeTool` | Plan mode toggle |
| `EnterWorktreeTool` / `ExitWorktreeTool` | Git worktree isolation |
| `NotebookEditTool` | Jupyter notebook editing |
| `AskUserQuestionTool` | Interactive user questions |
| `ConfigTool` | Configuration management |
| `BriefTool` | User notification messages |
| `SleepTool` | Wait for specified duration |
| `RemoteTriggerTool` | Remote trigger actions |

---

## Command System

User-facing slash commands invoked with `/` prefix.

| Command | Description |
|---------|-------------|
| `/commit` | Create a git commit |
| `/review` | Code review |
| `/compact` | Context compression |
| `/mcp` | MCP server management |
| `/config` | Settings management |
| `/doctor` | Environment diagnostics |
| `/login` / `/logout` | Authentication |
| `/memory` | Persistent memory management |
| `/skills` | Skill management |
| `/tasks` | Task management |
| `/vim` | Vim mode toggle |
| `/diff` | View changes |
| `/cost` | Check usage cost |
| `/theme` | Change theme |
| `/context` | Context visualization |
| `/pr_comments` | View PR comments |
| `/resume` | Restore previous session |
| `/share` | Share session |
| `/desktop` | Desktop app handoff |
| `/mobile` | Mobile app handoff |
| `/permissions` | Permission management |
| `/keybindings` | Keybinding configuration |
| `/init` | Initialize project |
| `/model` | Switch model |
| `/fast` | Toggle fast mode |
| `/upgrade` | Check for updates |

---

## Design Patterns

### Parallel Prefetch
Startup time is optimized by prefetching MDM settings, keychain reads, and API preconnect in parallel before heavy module evaluation:
```ts
// main.tsx — fired as side-effects before other imports
startMdmRawRead()
startKeychainPrefetch()
```

### Feature Flags (Dead Code Elimination)
Bun's `bun:bundle` feature flags enable build-time code stripping:
```ts
import { feature } from 'bun:bundle'

// Inactive code is completely stripped at build time
const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default
  : null
```

Notable flags: `PROACTIVE`, `KAIROS`, `BRIDGE_MODE`, `DAEMON`, `VOICE_MODE`, `AGENT_TRIGGERS`, `MONITOR_TOOL`

### Lazy Loading
Heavy modules (OpenTelemetry, gRPC, analytics, feature-gated subsystems) are deferred via dynamic `import()` until actually needed.

### Agent Swarms
Sub-agents are spawned via `AgentTool`, with `coordinator/` handling multi-agent orchestration. `TeamCreateTool` enables team-level parallel work.

---

## Service Layer

| Service | Description |
|---------|-------------|
| `api/` | Anthropic API client, file API, bootstrap |
| `mcp/` | Model Context Protocol server connection and management |
| `oauth/` | OAuth 2.0 authentication flow (PKCE) |
| `lsp/` | Language Server Protocol manager |
| `analytics/` | GrowthBook-based feature flags and analytics |
| `plugins/` | Plugin loader and marketplace |
| `compact/` | Conversation context compression |
| `voice/` | Voice streaming and speech-to-text |
| `teamMemorySync/` | Team memory synchronization |
| `extractMemories/` | Automatic memory extraction |

---

## Bridge System

Bidirectional communication layer connecting IDE extensions (VS Code, JetBrains) with the CLI:

- `bridgeMain.ts` — Bridge main loop with reconnection & backoff
- `replBridge.ts` — REPL session message routing
- `remoteBridgeCore.ts` — Lightweight remote bridge core
- `bridgeApi.ts` — Bridge API client
- `jwtUtils.ts` — JWT-based authentication
- `sessionRunner.ts` — Session execution management

---

## Quick Start

### Requirements

- **Bun** >= 1.3.5
- **Node.js** >= 24

### Install & Run

```bash
git clone https://github.com/Yangchengshuai/claude-code-rev.git
cd claude-code-rev
bun install           # Install dependencies (includes local shim packages)
bun run dev           # Start the restored CLI interactively
bun run version       # Print version + missing import count
```

### Available Scripts

| Command | Description |
|---------|-------------|
| `bun run dev` | Start the restored CLI interactively |
| `bun run start` | Alias for `dev` |
| `bun run version` | Print version + missing import count |
| `bun run dev:restore-check` | Run via dev-entry with restoration diagnostics |

---

## Compatibility Shims

Local `file:` packages in `shims/` that stub unrecoverable native modules:

| Shim | Original Module | Status |
|------|----------------|--------|
| `ant-computer-use-input` | Screen capture / mouse input | Stub — no actual capture |
| `ant-computer-use-mcp` | Computer use MCP server | Degraded — empty tool catalog |
| `ant-computer-use-swift` | macOS Swift integration | Stub — no native bridge |
| `ant-claude-for-chrome-mcp` | Chrome extension bridge | Stub — no browser connection |
| `color-diff-napi` | Native color diff (N-API) | Fallback — JS implementation |
| `modifiers-napi` | Keyboard modifier detection | Fallback — basic detection |
| `url-handler-napi` | URL scheme registration | Stub — no-op |

---

## Tech Stack

| Technology | Purpose |
|-----------|---------|
| TypeScript (ESNext) | Primary language |
| React + Ink | Terminal UI framework |
| Bun | Runtime & package manager |
| Zod | Runtime schema validation |
| Anthropic SDK | Claude API client |
| MCP SDK | Model Context Protocol |
| Commander.js | CLI argument parsing |
| GrowthBook | Feature flags / A/B testing |
| Yoga | Terminal flexbox layout |
| Statsig / Datadog | Analytics & monitoring |
| OpenTelemetry + gRPC | Telemetry |
| OAuth 2.0 / JWT / PKCE | Authentication |

---

## Documentation

Comprehensive codebase analysis documents in `docs/analysis/`:

| Document | Content |
|----------|---------|
| [System Overview](docs/analysis/SYSTEM_OVERVIEW.md) | Architecture, module inventory, bootstrap flow, diagrams |
| [Data Structures](docs/analysis/DATA_STRUCTURES.md) | AppState, Tool interface, Permissions, MCP, Settings, Memory |
| [Data Flow](docs/analysis/DATA_FLOW.md) | Input pipeline, query engine, MCP connections, state updates |
| [REPL Loop Algorithm](docs/analysis/ALGORITHM_REPL_LOOP.md) | Core loop state machine, permission resolution, compaction |
| [Tool System Algorithm](docs/analysis/ALGORITHM_TOOL_SYSTEM.md) | Registration, BashTool security, file ops, agent orchestration |
| [Key Functions](docs/analysis/KEY_FUNCTIONS.md) | main(), assembleToolPool(), buildTool(), query engine, MCP client |
| [Q&A](docs/analysis/KEY_QUESTIONS.md) | 13 architecture questions + 5 common pitfalls with code evidence |

---

## Restoration Notes

Known limitations of the reconstructed source tree:

- **Missing type-only files** — Not present in source maps
- **Build artifacts absent** — Generated files not recoverable
- **Native bindings stubbed** — N-API modules replaced with shims
- **Private packages shimmed** — Internal Anthropic packages degraded
- **Dynamic imports incomplete** — Some lazy-loaded modules missing

`src/dev-entry.ts` scans for missing relative imports on startup. The count shown by `bun run version` is a useful restoration health metric — when it reaches 0, the dev-entry forwards directly to the real CLI.

---

## Related Projects

| Project | Description |
|---------|-------------|
| [oboard/claude-code-rev](https://github.com/oboard/claude-code-rev) | Original restored source tree (this repo's upstream) |
| [nfeyre/claudecode-src](https://cnb.cool/nfeyre/claudecode-src) | Research-oriented mirror with detailed architecture docs |
| [instructkr/claw-code](https://github.com/instructkr/claw-code) | Clean-room Python/Rust rewrite of Claude Code's harness |

---

## License

See [LICENSE.md](LICENSE.md) for details.

---

<a id="中文说明"></a>

## 中文说明

2026 年 3 月 31 日，Claude Code 的源码通过 npm 包中暴露的 `.map` 文件被公开发现。该 source map 引用了 Anthropic R2 存储桶中未混淆的 TypeScript 源码，使整个 `src/` 快照可被公开下载。

本仓库基于该快照进行**逆向还原**，补齐缺失模块、添加兼容性 shim，并附带**完整的架构分析文档**。

> 本仓库仅供**教育和研究目的**。不声称对原始代码的所有权，与 Anthropic 无关联。

### 项目规模

- **1,989** 个 TypeScript/TSX 文件 | **~514,000** 行源代码
- **50+** 内置工具 | **195** 个 CLI 子命令 | **357** 个 React 组件

### 快速开始

```bash
git clone https://github.com/Yangchengshuai/claude-code-rev.git
cd claude-code-rev
bun install
bun run dev        # 启动还原后的 CLI
bun run version    # 查看版本及缺失导入数
```

### 核心架构

```
用户输入 → PromptInput (Vim/键绑定层) → 查询引擎 → Anthropic API
                ↑                                    │
                │                            流式响应解析
                │                                    │
                └───── 工具执行结果 ←─── 工具执行管道 ←─┘
                         (权限检查 → 执行 → 渲染)
```

六层架构：**UI 层** → **应用层** → **工具层** → **服务层** → **桥接层** → **基础设施层**

### 核心设计模式

- **并行预取**：启动时并行加载 MDM 配置、Keychain 和 API 预连接
- **Feature Flags**：通过 `bun:bundle` 实现编译期死代码消除
- **懒加载**：重型模块（OpenTelemetry、gRPC 等）延迟至需要时加载
- **Agent Swarms**：通过 AgentTool 生成子代理，coordinator 管理多代理编排

### 详细文档

[`docs/analysis/`](docs/analysis/) 目录包含 7 份完整源码分析文档，涵盖系统概述、数据结构、数据流、算法分析、关键函数和常见问题。

### 相关项目

- [oboard/claude-code-rev](https://github.com/oboard/claude-code-rev) — 原始还原源码树（本仓库上游）
- [nfeyre/claudecode-src](https://cnb.cool/nfeyre/claudecode-src) — 研究导向的镜像，含详细架构文档
- [instructkr/claw-code](https://github.com/instructkr/claw-code) — Claude Code harness 的 Python/Rust 重写

---

<div align="center">
<i>Reconstructed & Analyzed with Claude Code · 2026</i>
</div>
