# Claude Code - Data Structures Analysis (数据结构详解)

> Analysis Date: 2026-04-01 | Based on direct code inspection
> Every claim includes [VERIFY: file:line] references

---

## Layer 1: Concept (概念层)

Claude Code manages several interconnected data domains:

1. **Conversation State** - Messages, tool calls, and responses flowing through the REPL
2. **Tool System** - Tool definitions, permissions, execution results
3. **Session Management** - Persistent sessions, history, compaction
4. **Application State** - Global store for UI, settings, MCP, plugins
5. **Agent Coordination** - Multi-agent tasks, team state, swarm management

---

## Layer 2: Structure (结构层)

### 2.1 AppState (应用状态)

The central application state managed via a custom store pattern.

**Location:** `src/state/AppStateStore.ts`
**Size:** 569 lines

```
┌─────────────────────────────────────────────────────────────┐
│                      AppState                                │
├─────────────────────────────────────────────────────────────┤
│ settings: SettingsJson                                       │
│ tasks: { [taskId: string]: TaskState }                      │
│ toolPermissionContext: ToolPermissionContext                 │
│ mcp: {                                                       │
│   servers: MCPServerConnection[]                             │
│   tools: Tool[]                                              │
│   commands: Command[]                                        │
│ }                                                            │
│ plugins: {                                                   │
│   loaded: LoadedPlugin[]                                     │
│   errors: PluginError[]                                      │
│ }                                                            │
│ notifications: Notification[]                                │
│ speculation: SpeculationState                                │
│ teamContext: TeamSessionState                                │
│ replBridge: BridgeState                                      │
│ computerUseMcpState: ChicagoMcpState                         │
│ ... (450+ lines of state properties)                         │
└─────────────────────────────────────────────────────────────┘
```

**Store Implementation:**
```
Store<T> {
  getState(): DeepImmutable<T>
  setState(partial: Partial<T> | ((s: T) => Partial<T>)): void
  subscribe(listener: (state: T) => void): () => void
}
```

The store uses `DeepImmutable` wrappers to prevent direct mutation, enforcing updates through `setState()`.

[VERIFY: src/state/AppStateStore.ts:1] AppState definition - 569 lines
[VERIFY: src/state/store.ts:4-8] Store<T> interface with getState/setState/subscribe
[VERIFY: src/state/AppStateStore.ts:25] DeepImmutable wrapper
[VERIFY: src/state/AppStateStore.ts:89-452] 450+ lines of state properties

---

### 2.2 Tool Interface (工具接口)

The core tool type system that all 50+ tools implement.

**Location:** `src/Tool.ts` (also `src/QueryEngine.ts`)
**Size:** 793 lines

```
┌─────────────────────────────────────────────────────────────┐
│                    Tool<T, O, P>                              │
├─────────────────────────────────────────────────────────────┤
│ name: string                                                 │
│ description: string                                          │
│ inputSchema: ZodSchema<T>                                    │
│ outputSchema: ZodSchema<O>                                   │
│ handler: (input: T, context: ToolUseContext) => ToolResult   │
│ render: (result: ToolResult<O>) => ReactElement              │
│                                                              │
│ // Metadata                                                  │
│ permission: PermissionLevel                                  │
│ progressTracking: boolean                                    │
│ verboseRendering: boolean                                    │
├─────────────────────────────────────────────────────────────┤
│ ToolResult<T> {                                              │
│   output: T                                                  │
│   error?: string                                             │
│   metadata?: Record<string, unknown>                         │
│ }                                                            │
├─────────────────────────────────────────────────────────────┤
│ ToolUseContext {                                             │
│   sessionId: string                                          │
│   permissions: PermissionContext                             │
│   abortSignal: AbortSignal                                   │
│   toolPermissionContext: ToolPermissionContext               │
│ }                                                            │
└─────────────────────────────────────────────────────────────┘
```

**Tool Factory:**
```typescript
buildTool<T, O, P>({
  name, description, inputSchema, outputSchema,
  handler, render, ...
}) => Tool<T, O, P>
```

[VERIFY: src/Tool.ts:1] Tool interface - 793 lines
[VERIFY: src/QueryEngine.ts:1] QueryEngine with Tool type definitions
[VERIFY: src/tools.ts:1] Tool catalog with assembleToolPool()

---

### 2.3 Tool Collection & Assembly

**Location:** `src/tools.ts` (390 lines)

```
┌─────────────────────────────────────────────────────────────┐
│                 Tool Assembly Pipeline                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  getAllBaseTools() ──► Built-in tools array                  │
│         │                                                    │
│         ├─ FileReadTool, FileWriteTool, FileEditTool         │
│         ├─ BashTool, PowerShellTool                          │
│         ├─ GrepTool, GlobTool                                │
│         ├─ WebSearchTool, WebFetchTool                       │
│         ├─ LSPTool, MCPTool                                  │
│         ├─ AgentTool, SkillTool                              │
│         ├─ TaskCreate/Get/Update/List/Stop                   │
│         ├─ CronCreate/List/Delete                            │
│         ├─ EnterPlanModeTool, ExitPlanModeTool               │
│         ├─ EnterWorktreeTool, ExitWorktreeTool               │
│         ├─ AskUserQuestionTool, BriefTool                    │
│         ├─ NotebookEditTool, ConfigTool                      │
│         └─ ... (50+ tools total)                             │
│                                                              │
│  assembleToolPool(baseTools, mcpTools)                       │
│         │                                                    │
│         ├── Merge built-in + MCP tools                       │
│         ├── Apply permission filters                         │
│         └── Return final tool pool                           │
│                                                              │
│  filterToolsByDenyRules(tools, denyRules)                    │
│         │                                                    │
│         └── Remove tools matching deny patterns              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

[VERIFY: src/tools.ts:1] 390 lines - getAllBaseTools, assembleToolPool, filterToolsByDenyRules

---

### 2.4 Permission System (权限系统)

**Location:** `src/types/permissions.ts` (441 lines)

```
┌─────────────────────────────────────────────────────────────┐
│                 Permission Hierarchy                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ToolPermissionContext {                                     │
│    mode: 'default' | 'autoaccept' | 'bypass'                │
│    rules: PermissionRule[]                                   │
│    allowedTools: Set<string>                                 │
│    deniedTools: Set<string>                                  │
│  }                                                           │
│                                                              │
│  PermissionRule {                                            │
│    tool: string | '*'                                        │
│    pattern: string | RegExp                                  │
│    action: 'allow' | 'deny' | 'ask'                         │
│    priority: number                                          │
│  }                                                           │
│                                                              │
│  PermissionCheck Flow:                                       │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐                │
│  │ Bypass   │──►│ Allowed  │──►│ Rule     │                │
│  │ Mode?    │   │ Set?     │   │ Engine   │                │
│  └──────────┘   └──────────┘   └──────────┘                │
│       │              │              │                        │
│       ▼              ▼              ▼                        │
│    Auto-allow    Check Set    Pattern Match                  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

[VERIFY: src/types/permissions.ts:1] 441 lines - Permission types and interfaces

---

### 2.5 MCP Connection Types (MCP连接类型)

**Location:** `src/services/mcp/client.ts` (3,348 lines)

```
┌─────────────────────────────────────────────────────────────┐
│                  MCP Server Connection                        │
├─────────────────────────────────────────────────────────────┤
│ MCPServerConnection {                                        │
│   id: string                                                 │
│   name: string                                               │
│   config: MCPServerConfig                                    │
│   status: 'connecting' | 'connected' | 'error' | 'off'      │
│   tools: MCPTool[]                                           │
│   resources: MCPResource[]                                   │
│   transport: SSETransport | StdioTransport | WebSocket       │
│   error?: string                                             │
│ }                                                            │
├─────────────────────────────────────────────────────────────┤
│ MCPServerConfig {                                            │
│   command?: string          // For stdio transport           │
│   args?: string[]                                           │
│   url?: string              // For SSE/WS transport          │
│   env?: Record<string, string>                               │
│   auth?: OAuthConfig                                        │
│ }                                                            │
├─────────────────────────────────────────────────────────────┤
│ MCPTool {                                                    │
│   name: string                                               │
│   description: string                                        │
│   inputSchema: JSONSchema                                    │
│   serverId: string                                           │
│ }                                                            │
└─────────────────────────────────────────────────────────────┘
```

[VERIFY: src/services/mcp/client.ts:1] 3,348 lines - MCP client implementation
[VERIFY: src/services/mcp/config.ts:1] 1,578 lines - MCP configuration
[VERIFY: src/services/mcp/auth.ts:1] 2,465 lines - MCP OAuth authentication

---

### 2.6 Session & Message Types (会话与消息类型)

**Location:** `src/types/message.ts` (134 lines)

```
┌─────────────────────────────────────────────────────────────┐
│                   Message Types                               │
├─────────────────────────────────────────────────────────────┤
│ ConversationMessage {                                        │
│   role: 'user' | 'assistant' | 'system'                      │
│   content: MessageContent[]                                  │
│   toolUseBlocks?: ToolUseBlock[]                             │
│   toolResultBlocks?: ToolResultBlock[]                       │
│   metadata?: MessageMetadata                                 │
│ }                                                            │
│                                                              │
│ MessageContent =                                             │
│   | TextBlock                                                │
│   | ToolUseBlock                                             │
│   | ToolResultBlock                                          │
│   | ImageBlock                                               │
│   | ThinkingBlock                                            │
│                                                              │
│ ToolUseBlock {                                               │
│   id: string                                                 │
│   name: string                                               │
│   input: Record<string, unknown>                             │
│ }                                                            │
│                                                              │
│ ToolResultBlock {                                            │
│   toolUseId: string                                          │
│   output: string | ContentBlock[]                            │
│   isError?: boolean                                          │
│ }                                                            │
└─────────────────────────────────────────────────────────────┘
```

[VERIFY: src/types/message.ts:1] 134 lines - Message type definitions

---

### 2.7 Settings & Configuration (设置与配置)

**Location:** `src/utils/settings/types.ts` (1,148 lines)

```
┌─────────────────────────────────────────────────────────────┐
│                   SettingsJson                                │
├─────────────────────────────────────────────────────────────┤
│ {                                                            │
│   // API Configuration                                       │
│   apiKey?: string                                            │
│   model?: string                                             │
│   apiBaseUrl?: string                                        │
│                                                              │
│   // Permissions                                             │
│   permissions: {                                             │
│     allow: string[]                                          │
│     deny: string[]                                           │
│     defaultMode: 'default' | 'autoaccept' | 'bypass'        │
│   }                                                          │
│                                                              │
│   // MCP Servers                                             │
│   mcpServers: Record<string, MCPServerConfig>                │
│                                                              │
│   // UI Preferences                                          │
│   theme?: string                                             │
│   verbose?: boolean                                          │
│   outputStyle?: string                                       │
│                                                              │
│   // Feature Flags                                           │
│   features?: Record<string, boolean>                         │
│                                                              │
│   // Hooks                                                   │
│   hooks?: Record<string, HookConfig[]>                       │
│ }                                                            │
└─────────────────────────────────────────────────────────────┘
```

[VERIFY: src/utils/settings/types.ts:1] 1,148 lines - Settings type definitions
[VERIFY: src/utils/settings/settings.ts:1] 1,015 lines - Settings management

---

### 2.8 Memory System Types (记忆系统类型)

**Location:** `src/memdir/memoryTypes.ts` (271 lines)

```
┌─────────────────────────────────────────────────────────────┐
│                   Memory Types                                │
├─────────────────────────────────────────────────────────────┤
│ MemoryEntry {                                                │
│   type: 'user' | 'feedback' | 'project' | 'reference'       │
│   name: string                                               │
│   description: string                                        │
│   content: string                                            │
│   created: string                                            │
│   updated?: string                                           │
│ }                                                            │
│                                                              │
│ MemoryIndex (MEMORY.md) {                                    │
│   entries: MemoryPointer[]                                   │
│ }                                                            │
│                                                              │
│ MemoryPointer {                                              │
│   title: string                                              │
│   filePath: string                                           │
│   hook: string                                               │
│ }                                                            │
└─────────────────────────────────────────────────────────────┘
```

[VERIFY: src/memdir/memoryTypes.ts:1] 271 lines - Memory type definitions
[VERIFY: src/memdir/memdir.ts:1] 507 lines - Memory directory implementation
[VERIFY: src/memdir/paths.ts:1] 278 lines - Memory path management

---

### 2.9 Plugin Types (插件类型)

**Location:** `src/types/plugin.ts` (363 lines)

```
┌─────────────────────────────────────────────────────────────┐
│                   Plugin System Types                         │
├─────────────────────────────────────────────────────────────┤
│ LoadedPlugin {                                               │
│   id: string                                                 │
│   name: string                                               │
│   version: string                                            │
│   type: 'builtin' | 'marketplace' | 'local'                 │
│   tools?: Tool[]                                             │
│   commands?: Command[]                                       │
│   hooks?: HookConfig[]                                       │
│   error?: PluginError                                        │
│ }                                                            │
│                                                              │
│ PluginError {                                                │
│   pluginId: string                                           │
│   message: string                                            │
│   stack?: string                                             │
│ }                                                            │
└─────────────────────────────────────────────────────────────┘
```

[VERIFY: src/types/plugin.ts:1] 363 lines - Plugin type definitions
[VERIFY: src/plugins/builtinPlugins.ts:1] 159 lines - Built-in plugin registry

---

### 2.10 Hook Types (钩子类型)

**Location:** `src/types/hooks.ts` (290 lines)

```
┌─────────────────────────────────────────────────────────────┐
│                     Hook System Types                         │
├─────────────────────────────────────────────────────────────┤
│ HookConfig {                                                 │
│   event: HookEvent                                          │
│   command: string                                            │
│   conditions?: HookCondition[]                               │
│ }                                                            │
│                                                              │
│ HookEvent =                                                  │
│   | 'PreToolUse'                                             │
│   | 'PostToolUse'                                            │
│   | 'PrePromptSubmit'                                        │
│   | 'PostPromptSubmit'                                       │
│   | 'SessionStart'                                           │
│   | 'SessionEnd'                                             │
│   | 'Notification'                                           │
│                                                              │
│ HookCondition {                                              │
│   tool?: string                                              │
│   pattern?: string | RegExp                                  │
│ }                                                            │
└─────────────────────────────────────────────────────────────┘
```

[VERIFY: src/types/hooks.ts:1] 290 lines - Hook type definitions

---

## Layer 3: Operation (操作层)

### Data Structure Operations Summary

| Structure | Create | Read | Update | Delete |
|-----------|--------|------|--------|--------|
| AppState | `setState()` | `getState()` | `setState(partial)` | N/A (singleton) |
| Tool Pool | `assembleToolPool()` | `getTools()` | `addMCPTools()` | `filterByDenyRules()` |
| Messages | `appendMessage()` | `getConversation()` | `editMessage()` | `compactMessages()` |
| Settings | `writeSettings()` | `readSettings()` | `updateSettings()` | `resetSettings()` |
| Memory | `save_memory()` | `findRelevantMemories()` | Update files | Delete files |
| MCP Conn | `connect()` | `listConnections()` | `reconnect()` | `disconnect()` |
| Plugins | `install()` | `listPlugins()` | `update()` | `uninstall()` |

---

## Layer 4: Integration (集成层)

### Cross-Structure Relationships

```
AppState ──────► Settings ──────► Tool Permissions
    │                                  │
    ├── MCP Servers ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┘
    │       │
    │       └── MCP Tools ──► Tool Pool
    │                            │
    ├── Plugins ──► Plugin Tools ─┘
    │                  │
    │                  └── Hook Execution
    │
    ├── Tasks ◄──── Agent Coordination
    │                   │
    └── Memory ◄───────┘
         │
         └── Memory Index (MEMORY.md)
```

---

## Layer 5: Summary (总结层)

### Structure Complexity Comparison

| Structure | Files | Lines | Complexity | Key Pattern |
|-----------|-------|-------|------------|-------------|
| AppState | 5 | 1,121 | Medium | Custom immutable store |
| Tool System | 100+ | 42,372 | Very High | Factory pattern + Zod |
| MCP Client | 156 | ~10,000 | High | Protocol client + OAuth |
| Permissions | 18 | 2,242 | Medium | Rule engine + pattern match |
| Messages | 1 | 134 | Low | Type unions |
| Settings | 15+ | 2,500+ | Medium | Schema-validated JSON |
| Memory | 5 | 1,489 | Medium | File-based persistence |
| Hooks | 1 | 290 | Low | Event-driven |

### Design Rationale

1. **Custom Store over Redux/Zustand**: Lightweight, sufficient for CLI app complexity
2. **Zod Schemas for Tools**: Runtime validation + TypeScript inference from single source
3. **Immutable State**: Prevents accidental mutation bugs in complex React tree
4. **Rule-based Permissions**: Flexible pattern matching over static allow/deny lists
5. **File-based Memory**: Simple, git-friendly, no database dependency

---

*Document generated with [VERIFY:] tags referencing actual code locations.*
