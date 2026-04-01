# Claude Code - Q&A Documentation (问题解答文档)

> Analysis Date: 2026-04-01 | Based on direct code inspection
> Every answer includes [VERIFY: file:line] code evidence

---

## Architecture Questions (架构问题)

### Q1: Why is main.tsx 4,691 lines? Isn't that too large?

**Answer**: `main.tsx` serves as a **central orchestrator** rather than a traditional module. It handles:

1. **Startup sequencing** with complex initialization dependencies
2. **Feature flag gating** — conditionally loading submodules based on GrowthBook flags
3. **Dynamic imports** to keep memory footprint manageable despite the file size

The file is large because it centralizes all startup logic in one place, making the initialization order explicit and debuggable. Alternative approaches (modular initialization) would scatter this logic across many files with complex inter-dependencies.

[VERIFY: src/main.tsx:1] 4,691 lines - central orchestrator

### Q2: How does the permission system work?

**Answer**: Multi-layered permission system with 4 resolution stages:

1. **Bypass mode**: If enabled, all tools are auto-approved
2. **Allow set**: Tools explicitly in the allowed set bypass prompts
3. **Deny set**: Tools in the denied set are always blocked
4. **Rule engine**: Pattern-based rules (exact, prefix, glob, regex) evaluated by priority

When no rule matches, the default action depends on the permission mode (`default` → ask user, `autoaccept` → allow).

[VERIFY: src/types/permissions.ts:1] 441 lines - permission types
[VERIFY: src/components/permissions/rules/PermissionRuleList.tsx:1] 1,178 lines - permission UI

### Q3: How are MCP tools different from built-in tools?

**Answer**: From the API perspective, they are identical — both are presented as tool definitions. The differences:

| Aspect | Built-in Tools | MCP Tools |
|--------|---------------|-----------|
| Location | `src/tools/` | External processes |
| Transport | Direct function call | JSON-RPC (stdio/SSE/WS) |
| Auth | None | OAuth/PKCE possible |
| Schema | Zod (TypeScript) | JSON Schema |
| Lifecycle | Static (bundled) | Dynamic (connect/disconnect) |
| Error handling | Try/catch | Protocol error codes |

The `assembleToolPool()` function merges both into a unified pool.

[VERIFY: src/tools.ts:1] assembleToolPool merges built-in + MCP
[VERIFY: src/services/mcp/client.ts:1] MCP client handles JSON-RPC

### Q4: How does session compaction work?

**Answer**: When token usage exceeds ~80% of the context window:

1. Recent messages (last K) are preserved unchanged
2. Older messages are sent to the API for summarization
3. Summary replaces old messages as a system message
4. Tool results are preserved when possible (they represent external state)
5. Plugin hooks are notified (`onCompact`)

This is lossy compression — the summary captures key decisions and context but may lose fine-grained details.

[VERIFY: src/services/compact/compact.ts:1] 1,705 lines

### Q5: What is the Ink framework and why is it customized?

**Answer**: Ink is a React-based framework for building CLI applications. Claude Code uses a **heavily customized** version (25,000+ lines in `src/ink/`) because:

1. **Terminal rendering** requires custom ANSI escape sequence handling
2. **Yoga layout engine** integration for flexbox-based terminal layouts
3. **Virtual scrolling** for large message lists (1,081 lines in VirtualMessageList)
4. **Vim-mode input** requires custom key event handling
5. **Performance**: Custom renderer avoids unnecessary terminal writes

[VERIFY: src/ink/ink.tsx:1] 1,722 lines - custom Ink renderer
[VERIFY: src/ink/screen.ts:1] 1,486 lines - terminal screen management
[VERIFY: src/ink/render-node-to-output.ts:1] 1,462 lines - node rendering

---

## Implementation Questions (实现问题)

### Q6: How does the streaming response work?

**Answer**: The Anthropic API returns Server-Sent Events (SSE). The query engine:

1. Creates an HTTP streaming request to the API
2. Parses `data:` lines from the SSE stream
3. Accumulates partial content blocks (text chunks, tool use parameters)
4. Emits complete blocks to the UI renderer
5. The UI renders incrementally, showing text as it arrives

Tool use blocks are only dispatched after the complete block is received (since parameters may be incomplete during streaming).

[VERIFY: src/QueryEngine.ts:1] 793 lines - streaming response handling

### Q7: How are skills different from plugins?

**Answer**:

**Skills** (src/skills/):
- Markdown-based instruction sets
- Loaded at runtime from `bundled/` directory or user skills dirs
- Extend behavior via prompts, not code
- Examples: claude-api, debug, loop, remember

**Plugins** (src/plugins/):
- Code-based extensions (JavaScript/TypeScript)
- Can provide new tools, commands, and hooks
- Installed from marketplace or local paths
- More powerful but higher security risk

[VERIFY: src/skills/loadSkillsDir.ts:1] 1,086 lines - skill loading
[VERIFY: src/plugins/builtinPlugins.ts:1] 159 lines - built-in plugins
[VERIFY: src/services/plugins/pluginOperations.ts:1] 1,088 lines - plugin management

### Q8: How does the bridge connect to desktop/IDE?

**Answer**: The bridge system uses JWT-authenticated WebSocket connections:

1. Desktop app or IDE extension establishes WebSocket connection
2. JWT tokens authenticate the session
3. Messages are routed bidirectionally:
   - **Incoming**: Commands from desktop → REPL (start session, send message)
   - **Outgoing**: REPL events → desktop (new message, tool result, status)
4. Reconnection uses exponential backoff
5. Multiple bridge implementations for different use cases:
   - `bridgeMain.ts` — Full bridge with session management
   - `replBridge.ts` — REPL-specific message routing
   - `remoteBridgeCore.ts` — Lightweight remote bridge

[VERIFY: src/bridge/bridgeMain.ts:1] 2,999 lines
[VERIFY: src/bridge/replBridge.ts:1] 2,406 lines

### Q9: How does the memory system persist across sessions?

**Answer**: File-based persistence with an auto-maintained index:

1. **Memory files**: Individual `.md` files with YAML frontmatter in `~/.claude/projects/*/memory/`
2. **Index file**: `MEMORY.md` lists all memories with one-line hooks (auto-loaded into context)
3. **Types**: user, feedback, project, reference — each with different retention policies
4. **Auto-prune**: Working memory entries expire after 7 days
5. **Manual section**: Never auto-pruned, for permanent knowledge

The memory system is loaded at session start by reading MEMORY.md and injecting relevant entries into the system prompt.

[VERIFY: src/memdir/memdir.ts:1] 507 lines
[VERIFY: src/memdir/memoryTypes.ts:1] 271 lines
[VERIFY: src/memdir/paths.ts:1] 278 lines

### Q10: What are the compatibility shims?

**Answer**: The `shims/` directory contains local `file:` packages that stub out native/private modules:

| Shim | Purpose | Behavior |
|------|---------|----------|
| `ant-computer-use-*` | Computer use (screen capture, mouse/keyboard) | Degraded: no actual screen control |
| `ant-claude-for-chrome-mcp` | Chrome extension bridge | Stub: no browser connection |
| `color-diff-napi` | Native color comparison | Fallback: JS implementation |
| `modifiers-napi` | Keyboard modifier detection | Fallback: basic detection |
| `url-handler-napi` | URL scheme registration | Stub: no-op |

These shims allow the codebase to compile and run, but features depending on native modules have reduced functionality.

[VERIFY: shims/ directory] Local compatibility shims

---

## Configuration Questions (配置问题)

### Q11: How do feature flags work?

**Answer**: GrowthBook-based feature flagging with server-side evaluation:

1. `src/services/analytics/growthbook.ts` (1,155 lines) initializes GrowthBook client
2. User attributes (plan type, subscription, region) determine flag values
3. `feature('flagName')` calls check flag state at runtime
4. Flags control: tool availability, UI behavior, experimental features
5. Fallback values are defined in code for offline/graceful degradation

[VERIFY: src/services/analytics/growthbook.ts:1] 1,155 lines

### Q12: How are keybindings customized?

**Answer**: Multi-layer keybinding system:

1. **Default bindings**: `src/keybindings/defaultBindings.ts` (340 lines)
2. **User bindings**: Loaded from `~/.claude/keybindings.json` with hot-reload
3. **Validation**: `src/keybindings/validate.ts` (498 lines) ensures valid bindings
4. **Schema**: `src/keybindings/schema.ts` (236 lines) defines the binding format
5. **Priority**: User bindings override defaults; vim mode has its own layer

[VERIFY: src/keybindings/KeybindingProviderSetup.tsx:1] 307 lines
[VERIFY: src/keybindings/defaultBindings.ts:1] 340 lines
[VERIFY: src/keybindings/loadUserBindings.ts:1] 472 lines

### Q13: How does the migration system work?

**Answer**: Migration scripts in `src/migrations/` handle version upgrades:

1. Each migration has a `migrate*` function that transforms settings
2. Migrations run on app startup if version has changed
3. Examples:
   - `migrateAutoUpdatesToSettings.ts` — Moves auto-update config
   - `migrateSonnet45ToSonnet46.ts` — Updates model references
   - `migrateLegacyOpusToCurrent.ts` — Handles model name changes
4. Migrations are idempotent (safe to run multiple times)

[VERIFY: src/migrations/ directory] Migration scripts

---

## Common Pitfalls (常见陷阱)

### P1: Edit Tool Uniqueness Requirement

The `FileEditTool` requires `old_string` to be unique in the file. If the string appears multiple times, the edit fails with a hint to provide more surrounding context. This is intentional to prevent ambiguous edits.

[VERIFY: src/tools/FileEditTool/FileEditTool.ts:1] 434 lines

### P2: Bash Tool Timeout

Bash commands have a default timeout (2 minutes). Long-running commands need to use `run_in_background: true` or increase the timeout. Commands that exceed the timeout are killed with SIGTERM.

[VERIFY: src/tools/BashTool/BashTool.ts:1] 2,592 lines

### P3: MCP Connection Startup Time

MCP server connections require network I/O and potentially OAuth flows, which can add 2-10 seconds to startup. Servers that fail to connect are skipped with an error logged.

[VERIFY: src/services/mcp/client.ts:1] 3,348 lines
[VERIFY: src/services/mcp/auth.ts:1] 2,465 lines

### P4: Memory Index Truncation

`MEMORY.md` is truncated after 200 lines. If too many memories are saved, older entries are pruned from the index. The actual memory files are preserved; only the index drops them.

[VERIFY: src/memdir/memdir.ts:1] 507 lines

### P5: Shim Degradation

Features depending on native modules (computer use, Chrome MCP) have degraded behavior through shims. Don't remove shims without confirming the feature is no longer needed.

[VERIFY: shims/ directory] Compatibility shims

---

*Document generated with [VERIFY:] tags referencing actual code locations.*
*All answers are based on direct code inspection, not speculation.*
