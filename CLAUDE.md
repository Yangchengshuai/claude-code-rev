# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **restored Claude Code source tree** reconstructed from source maps and backfilled missing modules. It is not the original upstream repository — some modules are compatibility shims with degraded behavior. The project is runnable but has gaps where native bindings, type-only files, and build-time artifacts were unrecoverable.

## Commands

```bash
bun install            # Install dependencies (includes local shim packages)
bun run dev            # Start the restored CLI interactively (routes to src/entrypoints/cli.tsx)
bun run start          # Alias for dev
bun run version        # Print version and count of missing relative imports
bun run dev:restore-check  # Run through the dev-entry shim
```

There is no `lint`, `test`, or `build` script. Changes are validated by running the CLI and exercising the affected path manually.

## Requirements

- Bun >= 1.3.5 (package manager and runtime)
- Node.js >= 24

## Architecture

### Bootstrap Flow

```
package.json "dev" script
  → src/bootstrap-entry.ts          # Scans for missing imports, falls back gracefully
    → src/entrypoints/cli.tsx       # CLI entry point
      → src/main.tsx (4690 lines)   # Main REPL / interactive app (React/Ink TUI)
```

`src/bootstrapMacro.ts` injects build-time constants (version, package URL, etc.) into `globalThis.MACRO` so downstream modules can reference them without a build step.

### Core Modules (src/)

- **`main.tsx`** — The monolithic REPL: Ink-based TUI, command dispatch, tool execution loop, conversation history.
- **`QueryEngine.ts`` / `query.ts`** — Query orchestration: sends prompts to the API, manages streaming, handles tool-call loops.
- **`Tool.ts` / `tools.ts`** — Tool base class and registration. Each tool in `src/tools/` (e.g. `BashTool`, `FileEditTool`, `GrepTool`, `MCPTool`) extends a common interface.
- **`commands.ts`** — CLI subcommand registration. Individual commands live in `src/commands/<name>/` folders (100+ commands).
- **`interactiveHelpers.tsx`** — Interactive permission prompts and user-facing dialogs.

### Key Directories

| Directory | Purpose |
|-----------|---------|
| `src/tools/` | Agent tool implementations (Bash, file ops, glob, grep, MCP, web, LSP, etc.) |
| `src/commands/` | CLI subcommands — each folder is a command with its own handler |
| `src/components/` | React/Ink TUI components (140+ files) |
| `src/services/` | Service modules: API client, LSP, MCP, OAuth, plugins, voice, rate limiting |
| `src/bridge/` | Remote bridge for desktop/IDE connections, session management, JWT auth |
| `src/hooks/` | React hooks (50+): keybindings, IDE integration, command queue, permissions |
| `src/utils/` | Utility modules (330+ files): auth, shell, API, file ops, streaming, etc. |
| `src/state/` | App state management (AppState store, selectors) |
| `src/context/` | React context providers (mailbox, notifications, voice, modals) |
| `src/skills/` | Bundled skill system — `bundled/` has skill content, loadable via `loadSkillsDir.ts` |
| `src/ink/` | Ink framework customization and rendering |
| `src/vim/` | Vim-mode input handling |
| `src/keybindings/` | Keybinding definitions and dispatch |

### Compatibility Shims (shims/)

Local `file:` packages that stub out native/private modules not recoverable from source maps:

- `ant-computer-use-*` — Computer use input, MCP, and Swift integrations
- `ant-claude-for-chrome-mcp` — Chrome extension MCP bridge
- `color-diff-napi`, `modifiers-napi`, `url-handler-napi` — Native N-API bindings

### Vendor Sources (vendor/)

Third-party native module sources: `audio-capture-src`, `image-processor-src`, `modifiers-napi-src`, `url-handler-src`.

## Coding Conventions

- TypeScript with ESM imports (`"type": "module"`). No semicolons in most files. Single quotes.
- `camelCase` for variables/functions, `PascalCase` for React components and manager classes, `kebab-case` for command folders.
- `react-jsx` JSX transform (no explicit React imports needed in components).
- TypeScript `strict` mode is **off**. Path alias `src/*` maps to `./src/*`.
- Some files have comments warning against reordering imports — respect those.
- Small focused modules are preferred over broad utility dumps.

## Restoration-Specific Guidelines

- When modifying restored code, prefer minimal, auditable changes. Document workarounds added because a module uses a degraded shim or fallback.
- `src/bootstrap-entry.ts` scans all source files for missing relative imports on startup. If missing imports reach 0, it forwards directly to the real CLI entry. The count shown by `bun run version` is a useful restoration health metric.
- Shim packages in `shims/` intentionally provide reduced functionality. Do not remove them without confirming the original dependency is no longer needed.
- `src/main.tsx` is the restored monolith (4690 lines). It was reconstructed from a bundled source map and may contain artifacts of that process.
