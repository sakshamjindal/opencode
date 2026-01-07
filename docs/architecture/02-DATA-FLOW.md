# Data Flow and State Management

## 1. Storage Architecture

### Overview
OpenCode uses a file-based JSON storage system with read-write locking for data persistence.

```
┌─────────────────────────────────────────────────────────────────┐
│                     Storage Architecture                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ~/.local/share/opencode/                                        │
│  └── storage/                                                    │
│      ├── session/{projectID}/{sessionID}.json                   │
│      ├── message/{sessionID}/{messageID}.json                   │
│      ├── part/{messageID}/{partID}.json                         │
│      ├── session_diff/{sessionID}.json                          │
│      └── migration (version tracking)                           │
│                                                                  │
│  ~/.cache/opencode/                                              │
│  ├── version (cache version: "14")                              │
│  └── [transient data]                                           │
│                                                                  │
│  ~/.config/opencode/                                             │
│  ├── opencode.jsonc (global config)                             │
│  ├── auth.json (authentication tokens)                          │
│  └── tool/*.ts (custom tools)                                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Storage API
```typescript
// Location: packages/opencode/src/storage/storage.ts

namespace Storage {
  // Read with shared lock
  async function read<T>(key: string[]): Promise<T>

  // Write with exclusive lock
  async function write<T>(key: string[], content: T): Promise<void>

  // Atomic read-modify-write
  async function update<T>(key: string[], fn: (draft: T) => void): Promise<T>

  // List entries by prefix
  async function list(prefix: string[]): Promise<string[][]>

  // Delete entry
  async function remove(key: string[]): Promise<void>
}
```

### Locking Mechanism
```
┌─────────────────────────────────────────────────────────────────┐
│                    Read-Write Lock System                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Multiple Readers (Concurrent)                                   │
│  ┌────────┐  ┌────────┐  ┌────────┐                             │
│  │ Read 1 │  │ Read 2 │  │ Read 3 │  ← All allowed              │
│  └────────┘  └────────┘  └────────┘                             │
│                                                                  │
│  Single Writer (Exclusive)                                       │
│  ┌─────────────────────────────────┐                            │
│  │           Write Lock            │  ← Blocks all reads        │
│  └─────────────────────────────────┘                            │
│                                                                  │
│  Writer Priority: Writers take precedence to prevent starvation │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Session Data Flow

### Message Processing Pipeline
```
┌───────────────────────────────────────────────────────────────────────────┐
│                        Message Processing Flow                             │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  User Input                                                                │
│      │                                                                     │
│      ▼                                                                     │
│  ┌─────────────────┐                                                       │
│  │  POST /message  │  HTTP API endpoint                                    │
│  └────────┬────────┘                                                       │
│           │                                                                │
│           ▼                                                                │
│  ┌─────────────────┐     ┌─────────────────┐                              │
│  │ Create Message  │────►│  Store in JSON  │                              │
│  │   (user role)   │     │    Storage      │                              │
│  └────────┬────────┘     └─────────────────┘                              │
│           │                       │                                        │
│           │                       ▼                                        │
│           │              Bus.publish(message.updated)                      │
│           │                                                                │
│           ▼                                                                │
│  ┌─────────────────┐                                                       │
│  │ Session.prompt()│  Start LLM interaction                               │
│  └────────┬────────┘                                                       │
│           │                                                                │
│           ▼                                                                │
│  ┌─────────────────────────────────────────────────────────┐              │
│  │                  Session Processor                       │              │
│  │  ┌─────────────────────────────────────────────────┐    │              │
│  │  │ for await (part of stream) {                     │    │              │
│  │  │   switch (part.type) {                           │    │              │
│  │  │     case "text": yield TextPart                  │    │              │
│  │  │     case "reasoning": yield ReasoningPart        │    │              │
│  │  │     case "tool-call": {                          │    │              │
│  │  │       await checkPermission()                    │    │              │
│  │  │       result = await tool.execute()              │    │              │
│  │  │       yield ToolResultPart                       │    │              │
│  │  │     }                                            │    │              │
│  │  │   }                                              │    │              │
│  │  │   Storage.write(part)                            │    │              │
│  │  │   Bus.publish(message.part.updated, {delta})     │    │              │
│  │  │ }                                                │    │              │
│  │  └─────────────────────────────────────────────────┘    │              │
│  └─────────────────────────────────────────────────────────┘              │
│                                                                            │
└───────────────────────────────────────────────────────────────────────────┘
```

### Part Types
| Part Type | Content | Use Case |
|-----------|---------|----------|
| `TextPart` | LLM text response | Normal conversation |
| `ReasoningPart` | Extended thinking | Chain-of-thought |
| `ToolInvocationPart` | Tool call + result | Tool execution |
| `FilePart` | File reference | Attachments |
| `SourcePart` | Source citation | Web search results |

---

## 3. Event-Driven Synchronization

### Event Flow Architecture
```
┌─────────────────────────────────────────────────────────────────┐
│                    Event Propagation                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Backend (Node.js)                                               │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  Session.update()                                        │    │
│  │       │                                                  │    │
│  │       ▼                                                  │    │
│  │  Storage.write()                                         │    │
│  │       │                                                  │    │
│  │       ▼                                                  │    │
│  │  Bus.publish(session.updated, {...})                     │    │
│  │       │                                                  │    │
│  │       ├──────────────────────────────────────────┐      │    │
│  │       ▼                                          ▼      │    │
│  │  Instance Bus                              Global Bus   │    │
│  │  (local handlers)                    (cross-instance)   │    │
│  │                                                          │    │
│  └──────────────────────────┬──────────────────────────────┘    │
│                             │                                    │
│                             ▼                                    │
│  SSE Endpoint: GET /event                                        │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Bus.subscribeAll((event) => {                          │    │
│  │    sse.send(JSON.stringify(event))                      │    │
│  │  })                                                      │    │
│  └─────────────────────────────────────────────────────────┘    │
│                             │                                    │
└─────────────────────────────┼────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  Frontend (SolidJS)                                              │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  GlobalSync Context                                      │    │
│  │                                                          │    │
│  │  eventSource.onmessage = (event) => {                   │    │
│  │    const parsed = JSON.parse(event.data)                │    │
│  │    switch (parsed.type) {                               │    │
│  │      case "session.updated":                            │    │
│  │        setStore("session", idx, reconcile(data))        │    │
│  │      case "message.part.updated":                       │    │
│  │        // Apply delta for streaming                     │    │
│  │        setStore("part", msgId, applyDelta(delta))       │    │
│  │    }                                                     │    │
│  │  }                                                       │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Key Events
```typescript
// Session Events
Session.Event.Created   // New session created
Session.Event.Updated   // Session metadata changed
Session.Event.Archived  // Session archived
Session.Event.Reverted  // Session rolled back

// Message Events
MessageV2.Event.Updated     // Message metadata changed
MessageV2.Event.Removed     // Message deleted
MessageV2.Event.PartUpdated // Part added/modified (includes delta)
MessageV2.Event.PartRemoved // Part deleted

// Status Events
SessionStatus.Event.Status  // idle | busy | retry

// Permission Events
PermissionNext.Event.Asked   // Permission request
PermissionNext.Event.Replied // Permission response

// Integration Events
MCP.Event.ToolsChanged  // MCP tools updated
LSP.Event.Updated       // LSP client connected
LSP.Event.Diagnostics   // LSP diagnostics received
```

---

## 4. Instance Context Management

### Per-Directory Isolation
```
┌─────────────────────────────────────────────────────────────────┐
│                   Instance Context Pattern                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Instance.provide({ directory: "/project/a", fn: () => ... })   │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Instance Cache (Map<directory, Context>)                │    │
│  │                                                          │    │
│  │  /project/a ──► Context A                               │    │
│  │                  ├── Project metadata                    │    │
│  │                  ├── VCS info (git)                      │    │
│  │                  ├── State registry                      │    │
│  │                  └── Dispose handlers                    │    │
│  │                                                          │    │
│  │  /project/b ──► Context B                               │    │
│  │                  └── (independent state)                 │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  Instance.state<T>(init, dispose?)                               │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  State Registry (per instance)                           │    │
│  │                                                          │    │
│  │  (init_fn_ref, directory) ──► Cached State              │    │
│  │                                                          │    │
│  │  • Lazy initialization on first access                   │    │
│  │  • Automatic cleanup on Instance.dispose()               │    │
│  │  • Function reference as cache key                       │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### State Lifecycle
```typescript
// Location: packages/opencode/src/project/instance.ts

// 1. Define state with initializer
const sessionCache = Instance.state(
  async () => new Map<string, SessionInfo>(),  // init
  async (map) => map.clear()                    // dispose
)

// 2. Access state (auto-initializes)
const cache = sessionCache()  // Returns Map instance

// 3. Cleanup on instance dispose
await Instance.dispose()  // Calls all dispose handlers
```

---

## 5. Frontend State Management

### SolidJS Store Architecture
```
┌─────────────────────────────────────────────────────────────────┐
│                   Frontend State Structure                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Global Store (Singleton)                                        │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  {                                                       │    │
│  │    agent: Agent[]                                        │    │
│  │    command: Command[]                                    │    │
│  │    provider: ProviderListResponse                        │    │
│  │    config: Config                                        │    │
│  │    session: Session[]                                    │    │
│  │    session_status: Record<sessionID, SessionStatus>      │    │
│  │    session_diff: Record<sessionID, FileDiff[]>          │    │
│  │    message: Record<sessionID, Message[]>                 │    │
│  │    part: Record<messageID, Part[]>                       │    │
│  │    todo: Record<sessionID, Todo[]>                       │    │
│  │    permission: Record<sessionID, PermissionRequest[]>    │    │
│  │    mcp: Record<name, McpStatus>                          │    │
│  │    lsp: LspStatus[]                                      │    │
│  │    vcs: VcsInfo                                          │    │
│  │  }                                                       │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  Update Patterns                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  // Efficient reconciliation with key matching           │    │
│  │  setStore("message", sessionID,                          │    │
│  │    reconcile(newMessages, { key: "id" }))               │    │
│  │                                                          │    │
│  │  // Binary search for sorted insertions                  │    │
│  │  const idx = Binary.search(sessions, id, s => s.id)     │    │
│  │  setStore("session", idx.index, newSession)              │    │
│  │                                                          │    │
│  │  // Produce for immutable updates                        │    │
│  │  setStore("part", msgId, produce(draft => {             │    │
│  │    draft.push(newPart)                                   │    │
│  │  }))                                                     │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Context Provider Hierarchy
```
<App>
  <RouteProvider>           // Navigation state
    <SDKProvider>           // API client
      <SyncProvider>        // Real-time sync
        <LocalProvider>     // UI state (selection, filters)
          <ThemeProvider>   // Color schemes
            <KeybindProvider>    // Keyboard shortcuts
              <CommandProvider>  // Command palette
                <DialogProvider> // Modal dialogs
                  <Content />
```

---

## 6. Caching Strategies

### Multi-Layer Cache
```
┌─────────────────────────────────────────────────────────────────┐
│                      Caching Layers                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Layer 1: Lazy Evaluation (In-Memory)                           │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  const config = lazy(async () => loadConfig())          │    │
│  │  • Single initialization, cached forever                │    │
│  │  • Can be reset with config.reset()                     │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  Layer 2: Instance State (Per-Directory)                        │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  const state = Instance.state(() => computeState())     │    │
│  │  • Cached per directory                                 │    │
│  │  • Cleared on Instance.dispose()                        │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  Layer 3: File Cache (Persistent)                               │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  ~/.cache/opencode/                                      │    │
│  │  • Version-controlled (CACHE_VERSION = "14")            │    │
│  │  • Auto-invalidated on version mismatch                  │    │
│  │  • Used for compiled binaries, downloads                 │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  Layer 4: Frontend Store (Reactive)                             │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  SolidJS createStore()                                   │    │
│  │  • Reactive updates via reconcile()                      │    │
│  │  • Key-based diffing for minimal updates                 │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  Layer 5: Persistent Preferences                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  persisted(key, createStore(defaults))                   │    │
│  │  • LocalStorage backed                                   │    │
│  │  • Survives page refreshes                               │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 7. Streaming Data Flow

### Real-Time Message Streaming
```
┌─────────────────────────────────────────────────────────────────┐
│               Streaming Pipeline                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  LLM Provider                                                    │
│      │                                                           │
│      │ SSE Stream (text-stream, tool-call, etc.)                │
│      ▼                                                           │
│  ┌─────────────────┐                                            │
│  │ AI SDK Adapter  │  Normalizes provider-specific streams      │
│  └────────┬────────┘                                            │
│           │                                                      │
│           │ AsyncGenerator<StreamPart>                          │
│           ▼                                                      │
│  ┌─────────────────┐                                            │
│  │ Session.prompt()│  Orchestrates streaming                    │
│  └────────┬────────┘                                            │
│           │                                                      │
│           │ for await (part of stream)                          │
│           ▼                                                      │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Part Processing                                          │    │
│  │                                                          │    │
│  │ 1. Store part: Storage.write(["part", msgId, partId])   │    │
│  │                                                          │    │
│  │ 2. Calculate delta for incremental updates              │    │
│  │    delta = { index: startPos, content: newChars }       │    │
│  │                                                          │    │
│  │ 3. Publish event with delta                              │    │
│  │    Bus.publish(MessageV2.Event.PartUpdated, {           │    │
│  │      info: partInfo,                                     │    │
│  │      delta: { index, content }                          │    │
│  │    })                                                    │    │
│  └─────────────────────────────────────────────────────────┘    │
│           │                                                      │
│           ▼                                                      │
│  SSE to Frontend                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  event: message.part.updated                             │    │
│  │  data: {"info":{...},"delta":{"index":50,"content":"x"}}│    │
│  └─────────────────────────────────────────────────────────┘    │
│           │                                                      │
│           ▼                                                      │
│  Frontend Handler                                                │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  // Apply delta efficiently                              │    │
│  │  setStore("part", msgId, partIdx, "content",            │    │
│  │    prev => prev.slice(0, delta.index) + delta.content)  │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8. Migration System

### Storage Migrations
```typescript
// Location: packages/opencode/src/storage/storage.ts

const MIGRATIONS = [
  // Migration 0: Restructure project-based storage
  async (dir: string) => {
    // Move from old structure to new
  },

  // Migration 1: Separate session diffs
  async (dir: string) => {
    // Extract diffs to dedicated storage
  },
]

// Migration tracking
// Stored in: storage/migration (single number)
```

### Cache Versioning
```typescript
// Location: packages/opencode/src/global/index.ts

const CACHE_VERSION = "14"

// On startup:
const version = await readVersion()
if (version !== CACHE_VERSION) {
  await rm(cacheDir, { recursive: true })
  await writeVersion(CACHE_VERSION)
}
```
