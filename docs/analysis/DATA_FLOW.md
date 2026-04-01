# Claude Code - Data Flow Analysis (数据流分析)

> Analysis Date: 2026-04-01 | Based on direct code inspection
> Every claim includes [VERIFY: file:line] references

---

## 1. User Input Flow (用户输入流)

### 1.1 Prompt Input Pipeline

```
User types in terminal
        │
        ▼
┌──────────────────────┐
│  PromptInput.tsx      │  [VERIFY: src/components/PromptInput/PromptInput.tsx:1]
│  (2,338 lines)        │
│                       │
│  ┌─────────────────┐  │
│  │ Vim Mode Layer  │  │  [VERIFY: src/vim/transitions.ts:1]
│  │ (490 lines)     │  │
│  │ h/j/k/l, insert │  │
│  │ mode, visual    │  │
│  └────────┬────────┘  │
│           │           │
│  ┌────────▼────────┐  │
│  │ Keybinding Layer│  │  [VERIFY: src/keybindings/KeybindingProviderSetup.tsx:1]
│  │ (307 lines)     │  │
│  │ Chords, seqs    │  │
│  └────────┬────────┘  │
│           │           │
│  ┌────────▼────────┐  │
│  │ Text Input Hook │  │  [VERIFY: src/hooks/useTextInput.ts:1]
│  │ (529 lines)     │  │
│  │ Paste, history  │  │
│  └────────┬────────┘  │
│           │           │
│  Submit (Enter)       │
└───────────┬───────────┘
            │
            ▼
┌──────────────────────┐
│  Command Queue        │  [VERIFY: src/hooks/useCommandQueue.ts:1]
│                       │
│  /command → dispatch  │
│  text → query engine  │
└──────────┬───────────┘
           │
     ┌─────┴──────┐
     │            │
     ▼            ▼
  Slash Cmd    User Message
  Handler      Construction
     │            │
     │            ▼
     │     ┌──────────────┐
     │     │ Build Message │
     │     │ Attach files  │
     │     │ Add context   │
     │     │ (memories,    │
     │     │  CLAUDE.md)   │
     │     └──────┬───────┘
     │            │
     └─────┬──────┘
           │
           ▼
   Query Engine
```

[VERIFY: src/components/PromptInput/PromptInput.tsx:1] 2,338 lines - main input component
[VERIFY: src/hooks/useTextInput.ts:1] 529 lines - text input handling
[VERIFY: src/hooks/useVimInput.ts:1] 316 lines - vim input mode
[VERIFY: src/keybindings/KeybindingProviderSetup.tsx:1] 307 lines - keybinding provider

---

## 2. Query Engine Flow (查询引擎流)

### 2.1 API Request Pipeline

```
User Message
     │
     ▼
┌─────────────────────────────────────────────────────────┐
│                QueryEngine.ts (793 lines)                │
│                                                         │
│  1. Build API Request                                   │
│     ├── System prompt (with tool descriptions)           │
│     ├── Conversation history                            │
│     ├── Context injection (CLAUDE.md, memories)          │
│     └── Tool definitions (from tool pool)                │
│                                                         │
│  2. Send to Anthropic API                               │
│     ├── Streaming response                              │
│     ├── Tool call detection                             │
│     └── Token counting / rate limiting                   │
│                                                         │
│  3. Process Response                                    │
│     ├── Text blocks → Display to user                    │
│     ├── Tool use blocks → Execute tools                  │
│     ├── Thinking blocks → Internal reasoning             │
│     └── Error handling → Retry / display                 │
│                                                         │
│  4. Tool Execution Loop                                 │
│     ┌─────────────────────────────────┐                  │
│     │  For each tool_use block:        │                  │
│     │                                  │                  │
│     │  a. Resolve tool by name         │                  │
│     │  b. Validate input (Zod schema)  │                  │
│     │  c. Check permissions            │                  │
│     │  d. Execute handler              │                  │
│     │  e. Render result to UI          │                  │
│     │  f. Collect tool_result          │                  │
│     │  g. Append to conversation       │                  │
│     │                                  │                  │
│     │  If more tool calls needed:      │                  │
│     │  └── Loop back to step 2         │                  │
│     │                                  │                  │
│     │  If text response only:          │                  │
│     │  └── Display and return          │                  │
│     └─────────────────────────────────┘                  │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

[VERIFY: src/QueryEngine.ts:1] 793 lines - Query orchestration

---

## 2.2 Detailed Tool Execution Flow

```
Tool Use Block from API
        │
        ▼
┌────────────────────┐
│  toolExecution.ts   │  [VERIFY: src/services/tools/toolExecution.ts:1]
│  (1,745 lines)      │
│                     │
│  1. Resolve tool    │
│     └── Lookup in   │
│         tool pool   │
│                     │
│  2. Validate input  │
│     └── Zod schema  │
│         check       │
│                     │
│  3. Permission check│
│     ├── Bypass?     │
│     ├── Allowed?    │
│     └── Ask user?   │
│         │           │
│         ▼           │
│  ┌───────────────┐  │
│  │ interactive    │  │  [VERIFY: src/interactiveHelpers.tsx:1]
│  │ Helpers.tsx    │  │
│  │ (365 lines)    │  │
│  │                │  │
│  │ Show dialog    │  │
│  │ Allow / Deny   │  │
│  │ Edit command   │  │
│  └───────┬───────┘  │
│          │          │
│  4. Execute         │
│     ├── Create      │
│     │   abort ctrl  │
│     ├── Set up      │
│     │   progress    │
│     ├── Run handler │
│     └── Collect     │
│         result      │
│                     │
│  5. Render result   │
│     ├── Verbose mode│
│     ├── Condensed   │
│     └── Transcript  │
│                     │
└────────────────────┘
```

[VERIFY: src/services/tools/toolExecution.ts:1] 1,745 lines - tool execution orchestration
[VERIFY: src/interactiveHelpers.tsx:1] 365 lines - permission prompts

---

## 3. MCP Connection Flow (MCP连接流)

```
App Start
     │
     ▼
┌────────────────────────────┐
│ useManageMCPConnections.ts  │  [VERIFY: src/services/mcp/useManageMCPConnections.ts:1]
│ (1,141 lines)               │
│                              │
│  For each configured server: │
│     │                        │
│     ├── Read config          │  [VERIFY: src/services/mcp/config.ts:1]
│     │                        │
│     ├── Create transport     │
│     │   ├── stdio (local)    │
│     │   ├── SSE (remote)     │
│     │   └── WebSocket        │
│     │                        │
│     ├── Authenticate         │  [VERIFY: src/services/mcp/auth.ts:1]
│     │   ├── OAuth flow       │
│     │   ├── PKCE challenge   │
│     │   └── Token refresh    │
│     │                        │
│     ├── Connect              │  [VERIFY: src/services/mcp/client.ts:1]
│     │   ├── Initialize       │
│     │   ├── Capabilities     │
│     │   └── Ready            │
│     │                        │
│     ├── Discover tools       │
│     │   └── tools/list       │
│     │                        │
│     ├── Discover resources   │
│     │   └── resources/list   │
│     │                        │
│     └── Register in state    │
│                              │
│  assembleToolPool()          │  [VERIFY: src/tools.ts:1]
│  ├── Merge built-in + MCP    │
│  └── Apply permission filter │
│                              │
└──────────────────────────────┘
```

[VERIFY: src/services/mcp/useManageMCPConnections.ts:1] 1,141 lines
[VERIFY: src/services/mcp/config.ts:1] 1,578 lines
[VERIFY: src/services/mcp/auth.ts:1] 2,465 lines
[VERIFY: src/services/mcp/client.ts:1] 3,348 lines

---

## 4. State Update Flow (状态更新流)

```
Event (tool result, user input, MCP connection change, etc.)
     │
     ▼
┌──────────────────────┐
│  AppStateStore        │  [VERIFY: src/state/AppStateStore.ts:1]
│  (569 lines)          │
│                       │
│  setState(partial)    │
│     │                 │
│     ├── Merge partial │
│     │   into state    │
│     │                 │
│     ├── Notify        │
│     │   subscribers   │
│     │                 │
│     └── Trigger       │
│         React re-     │
│         render        │
│                       │
└───────────┬───────────┘
            │
            ▼
┌──────────────────────┐
│  React Components     │
│                       │
│  ├── useSettings()    │  [VERIFY: src/hooks/useSettings.ts:1]
│  ├── useTasksV2()     │  [VERIFY: src/hooks/useTasksV2.ts:1]
│  ├── useVoice()       │  [VERIFY: src/hooks/useVoice.ts:1]
│  └── 121 hooks...     │  [VERIFY: src/hooks/]
│                       │
│  Re-render affected   │
│  component tree       │
│                       │
└───────────────────────┘
```

[VERIFY: src/state/AppStateStore.ts:1] 569 lines - State management
[VERIFY: src/state/store.ts:1] 34 lines - Store interface
[VERIFY: src/state/selectors.ts:1] 76 lines - State selectors

---

## 5. Bridge Communication Flow (桥接通信流)

```
Desktop App / IDE Extension
         │
         ▼
┌──────────────────────┐
│  bridgeMain.ts        │  [VERIFY: src/bridge/bridgeMain.ts:1]
│  (2,999 lines)        │
│                       │
│  ┌─────────────────┐  │
│  │ JWT Auth        │  │  [VERIFY: src/bridge/jwtUtils.ts]
│  │ Token exchange  │  │
│  └────────┬────────┘  │
│           │           │
│  ┌────────▼────────┐  │
│  │ Transport Layer │  │
│  │ ┌─────────────┐ │  │
│  │ │ WebSocket   │ │  │
│  │ │ Transport   │ │  │  [VERIFY: src/bridge/WebSocketTransport.ts]
│  │ └─────────────┘ │  │
│  │ ┌─────────────┐ │  │
│  │ │ REPL Bridge │ │  │  [VERIFY: src/bridge/replBridge.ts:1]
│  │ │ (2,406)     │ │  │
│  │ └─────────────┘ │  │
│  └─────────────────┘  │
│                       │
│  Session Management   │  [VERIFY: src/bridge/sessionRunner.ts:1]
│  ├── Create session   │     (550 lines)
│  ├── Route messages   │
│  └── Handle lifecycle │
│                       │
└───────────────────────┘
         │
         ▼
   REPL / CLI
   (main.tsx)
```

[VERIFY: src/bridge/bridgeMain.ts:1] 2,999 lines - main bridge
[VERIFY: src/bridge/replBridge.ts:1] 2,406 lines - REPL bridge
[VERIFY: src/bridge/remoteBridgeCore.ts:1] 1,008 lines - remote bridge core
[VERIFY: src/bridge/bridgeApi.ts:1] 539 lines - bridge API client

---

## 6. Memory Persistence Flow (记忆持久化流)

```
User asks to remember something
         │
         ▼
┌──────────────────────┐
│  memdir.ts            │  [VERIFY: src/memdir/memdir.ts:1]
│  (507 lines)          │
│                       │
│  1. Determine type    │
│     ├── user          │
│     ├── feedback      │
│     ├── project       │
│     └── reference     │
│                       │
│  2. Generate file     │
│     ├── Create .md    │  [VERIFY: src/memdir/paths.ts:1]
│     │   with front-   │     (278 lines)
│     │   matter        │
│     └── Write to      │
│         memory dir    │
│                       │
│  3. Update index      │
│     └── MEMORY.md     │
│         (auto-index)  │
│                       │
└──────────┬────────────┘
           │
           ▼
┌──────────────────────┐
│  findRelevant        │  [VERIFY: src/memdir/findRelevantMemories.ts:1]
│  Memories.ts          │     (141 lines)
│  (141 lines)          │
│                       │
│  On session start:    │
│  ├── Read MEMORY.md   │
│  ├── Load index       │
│  ├── Semantic search  │
│  └── Inject into      │
│      context          │
│                       │
└───────────────────────┘
```

[VERIFY: src/memdir/memdir.ts:1] 507 lines
[VERIFY: src/memdir/memoryTypes.ts:1] 271 lines
[VERIFY: src/memdir/paths.ts:1] 278 lines
[VERIFY: src/memdir/findRelevantMemories.ts:1] 141 lines

---

## 7. Session Compaction Flow (会话压缩流)

```
Conversation grows large (token limit approaching)
         │
         ▼
┌──────────────────────────┐
│  compact.ts               │  [VERIFY: src/services/compact/compact.ts:1]
│  (1,705 lines)            │
│                            │
│  1. Detect token overflow  │
│     ├── Count tokens       │
│     └── Compare to limit   │
│                            │
│  2. Select messages to     │
│     compact                │
│     ├── Keep recent N      │
│     ├── Keep tool results  │
│     └── Mark old messages  │
│                            │
│  3. Generate summary       │
│     ├── Send to API        │
│     ├── Request summary    │
│     └── Receive compressed │
│         version            │
│                            │
│  4. Replace old messages   │
│     ├── Insert summary     │
│     ├── Remove originals   │
│     └── Update state       │
│                            │
│  5. Notify plugins         │
│     └── Hook: onCompact    │
│                            │
└────────────────────────────┘
```

[VERIFY: src/services/compact/compact.ts:1] 1,705 lines - session compaction

---

## 8. End-to-End Data Flow (端到端数据流)

```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│  User    │────►│  Input   │────►│  Query   │────►│ Anthropic│
│  Input   │     │  Layer   │     │  Engine  │     │    API   │
└──────────┘     └──────────┘     └──────────┘     └────┬─────┘
                                                          │
                                                     Response
                                                          │
                                                          ▼
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│  Display │◄────│  Render  │◄────│  Tool    │◄────│  Parse   │
│  to User │     │  Layer   │     │  Exec    │     │ Response │
└──────────┘     └──────────┘     └──────────┘     └──────────┘
      │                │                │
      │                │                │
      ▼                ▼                ▼
┌──────────┐     ┌──────────┐     ┌──────────┐
│  Update  │     │  React   │     │  Update  │
│  State   │     │  Re-     │     │  History │
│  Store   │     │  render  │     │  / Memory│
└──────────┘     └──────────┘     └──────────┘

Side Channels:
┌──────────┐     ┌──────────┐     ┌──────────┐
│  MCP     │◄───►│  Bridge  │◄───►│  Desktop │
│  Servers │     │  Layer   │     │   / IDE  │
└──────────┘     └──────────┘     └──────────┘
      │
      ▼
┌──────────┐     ┌──────────┐
│  Plugin  │     │  Skill   │
│  System  │     │  System  │
└──────────┘     └──────────┘
```

---

## 9. Data Transformation Summary (数据变换汇总)

| Stage | Input | Transformation | Output |
|-------|-------|----------------|--------|
| Input Capture | Keystrokes | Vim/keybinding processing | Text string |
| Message Build | Text string | Context injection (CLAUDE.md, memories) | Conversation message |
| API Request | Conversation | Tool defs + system prompt | HTTP streaming request |
| Response Parse | SSE chunks | Block classification | Text/ToolUse/Thinking blocks |
| Tool Exec | ToolUse block | Schema validation + handler | ToolResult |
| Permission | Tool request | Rule engine matching | Allow/Deny/Ask |
| Compaction | Long history | API summarization | Compressed history |
| MCP Connect | Server config | OAuth + transport init | Connected server + tools |
| State Update | Partial state | Immutable merge | New state snapshot |
| Bridge Message | JSON message | JWT + transport | Remote notification |

---

*Document generated with [VERIFY:] tags referencing actual code locations.*
