# Core Components Architecture

## 1. Agent System

### Overview
The agent system provides intelligent task execution with configurable behaviors, permissions, and model selections.

### Agent Types
```
┌─────────────────────────────────────────────────────────────────┐
│                        Agent System                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐    │
│  │   Primary    │     │   Subagent   │     │     All      │    │
│  │    Agent     │────►│    Agent     │     │    Agent     │    │
│  └──────────────┘     └──────────────┘     └──────────────┘    │
│        │                     │                    │             │
│        ▼                     ▼                    ▼             │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    Agent.Info Schema                     │   │
│  │  • id: string                                           │   │
│  │  • type: "primary" | "subagent" | "all"                 │   │
│  │  • model: { provider, model }                           │   │
│  │  • permission: Ruleset                                  │   │
│  │  • enabled: boolean                                     │   │
│  │  • toolChoice: "auto" | "required" | "none"            │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Built-in Agents
| Agent | Type | Purpose | Special Permissions |
|-------|------|---------|-------------------|
| `build` | primary | Code execution and generation | Full edit access |
| `plan` | primary | Strategic planning | Read-only (edit denied) |
| Custom | configurable | User-defined agents | Configurable |

### Agent Configuration
```typescript
// Location: packages/opencode/src/agent/agent.ts

interface AgentInfo {
  id: string
  type: "primary" | "subagent" | "all"
  model: { provider: string; model: string }
  permission: PermissionRuleset
  system?: string  // Custom system prompt
  enabled: boolean
  toolChoice: "auto" | "required" | "none"
}
```

### Permission System Integration
- Agents inherit base permissions from configuration
- Each agent can have additional permission overrides
- Tool availability filtered by agent permissions
- Doom loop detection (3 consecutive denials = stop)

---

## 2. Session System

### Overview
Sessions manage conversation state, message history, and tool execution context.

### Session Lifecycle
```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ Created  │───►│  Active  │───►│ Archived │───►│ Reverted │
└──────────┘    └──────────┘    └──────────┘    └──────────┘
     │               │               │               │
     │               │               │               │
     ▼               ▼               ▼               ▼
 Bus.publish    Bus.publish     Bus.publish     Bus.publish
 (Created)      (Updated)       (Archived)      (Reverted)
```

### Session Components
```
┌─────────────────────────────────────────────────────────────────┐
│                         Session                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Session.Info                                                    │
│  ├── id: Identifier<"session">                                  │
│  ├── projectID: string                                          │
│  ├── directory: string                                          │
│  ├── parentID?: Identifier<"session">  (forking)               │
│  ├── title: string                                              │
│  ├── time: { created, updated, compacting?, archived? }         │
│  ├── permission: "default" | "auto" | ...                       │
│  ├── summary: { additions, deletions, files, diffs }           │
│  ├── revert: { messageID, partID?, snapshot?, diff? }          │
│  └── share: { enabled, id? }                                    │
│                                                                  │
│  Messages (1:N)                                                  │
│  ├── MessageV2.Info                                             │
│  │   ├── id: Identifier<"message">                              │
│  │   ├── sessionID: Identifier<"session">                       │
│  │   ├── role: "user" | "assistant"                             │
│  │   ├── time: { created, updated }                             │
│  │   └── usage: { input, output, reasoning, cache... }         │
│  │                                                               │
│  └── Parts (1:N per message)                                    │
│      ├── TextPart                                               │
│      ├── ReasoningPart                                          │
│      ├── ToolInvocationPart                                     │
│      ├── FilePart                                               │
│      └── SourcePart                                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Session Processor
The session processor orchestrates the LLM interaction loop:

```typescript
// Location: packages/opencode/src/session/processor.ts

async function* process(input: ProcessorInput) {
  // 1. Build message context
  const messages = await buildContext(sessionID)

  // 2. Get available tools
  const tools = await ToolRegistry.tools(providerID, agent)

  // 3. Stream LLM response
  for await (const part of llm.stream(messages, tools)) {
    // Handle different part types
    switch (part.type) {
      case "text":
        yield { type: "text", content: part.text }
        break
      case "tool-call":
        // Execute tool with permission checking
        const result = await executeTool(part)
        yield { type: "tool-result", ...result }
        break
      case "reasoning":
        yield { type: "reasoning", content: part.reasoning }
        break
    }
  }
}
```

---

## 3. Tool System

### Tool Architecture
```
┌─────────────────────────────────────────────────────────────────┐
│                        Tool Registry                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Built-in Tools (19+)                                           │
│  ├── File Operations: read, write, edit, glob, grep            │
│  ├── Code Analysis: lsp, codesearch                            │
│  ├── Web Access: webfetch, websearch                           │
│  ├── Execution: bash, task                                     │
│  ├── Session: todowrite, todoread, skill                       │
│  └── Special: patch, multiedit, batch                          │
│                                                                  │
│  Custom Tools                                                    │
│  ├── ~/.opencode/tool/*.{ts,js}                                │
│  └── Plugin-provided tools                                      │
│                                                                  │
│  MCP Tools                                                       │
│  └── Dynamically loaded from MCP servers                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Tool Definition Pattern
```typescript
// Location: packages/opencode/src/tool/tool.ts

interface ToolInfo<Params extends z.ZodType, Meta extends Metadata> {
  id: string
  init: (ctx?: InitContext) => Promise<{
    description: string
    parameters: Params
    execute(args: z.infer<Params>, ctx: Context): Promise<{
      title: string
      metadata: Meta
      output: string
      attachments?: FilePart[]
    }>
    formatValidationError?(error: z.ZodError): string
  }>
}
```

### Tool Execution Pipeline
```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ Tool Call    │────►│  Validate    │────►│  Permission  │
│ from LLM     │     │  Parameters  │     │    Check     │
└──────────────┘     └──────────────┘     └──────────────┘
                                                │
                                                ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Store      │◄────│   Execute    │◄────│   Plugin     │
│   Result     │     │    Tool      │     │   Hooks      │
└──────────────┘     └──────────────┘     └──────────────┘
```

---

## 4. Provider System

### Provider Architecture
```
┌─────────────────────────────────────────────────────────────────┐
│                      Provider System                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                   Vercel AI SDK                          │   │
│  │              (Common Abstraction Layer)                  │   │
│  └────────────────────────┬────────────────────────────────┘   │
│                           │                                      │
│    ┌──────────────────────┼──────────────────────┐              │
│    │                      │                      │              │
│    ▼                      ▼                      ▼              │
│  ┌──────────┐      ┌──────────┐          ┌──────────┐          │
│  │ Anthropic│      │  OpenAI  │   ...    │  Custom  │          │
│  │  Claude  │      │   GPT    │          │ Provider │          │
│  └──────────┘      └──────────┘          └──────────┘          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Supported Providers (23+)
| Provider | Models | Special Features |
|----------|--------|-----------------|
| Anthropic | Claude 3/3.5/Opus | Extended thinking, prompt caching |
| OpenAI | GPT-4/4o/o1 | Responses API, function calling |
| Google | Gemini Pro/Ultra | Vertex AI integration |
| Azure | OpenAI Service | Enterprise deployment |
| Amazon Bedrock | Various | AWS region support |
| GitHub Copilot | GPT-4 | GitHub integration |
| OpenRouter | 100+ models | Model aggregation |
| Mistral | Mistral/Mixtral | European provider |
| Groq | LLaMA/Mixtral | Fast inference |
| And 14+ more... | | |

### Provider Configuration
```typescript
// Location: packages/opencode/src/provider/provider.ts

interface ProviderConfig {
  id: string
  name: string
  api?: {
    key?: string
    baseURL?: string
    region?: string  // For AWS Bedrock
  }
  options?: Record<string, any>
}
```

---

## 5. HTTP Server

### Server Architecture
```
┌─────────────────────────────────────────────────────────────────┐
│                      Hono HTTP Server                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Middleware Stack                                                │
│  ├── CORS Handler (dynamic whitelist)                           │
│  ├── Error Handler (NamedError wrapper)                         │
│  ├── Request Logger (skip /log endpoint)                        │
│  └── Authentication (session-based)                             │
│                                                                  │
│  API Routes                                                       │
│  ├── /session/*      - Session CRUD                             │
│  ├── /message/*      - Message operations                        │
│  ├── /tool/*         - Tool execution                           │
│  ├── /provider/*     - Provider listing                         │
│  ├── /mcp/*          - MCP management                           │
│  ├── /lsp/*          - LSP status                               │
│  └── /event          - SSE stream                               │
│                                                                  │
│  Features                                                         │
│  ├── OpenAPI spec generation (Hono OpenAPI)                     │
│  ├── SSE for real-time updates                                  │
│  ├── WebSocket support                                          │
│  └── mDNS service discovery                                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Key Endpoints
```typescript
// Location: packages/opencode/src/server/server.ts

// Session endpoints
POST   /session           - Create session
GET    /session           - List sessions
GET    /session/:id       - Get session
PATCH  /session/:id       - Update session
DELETE /session/:id       - Archive session

// Message endpoints
POST   /session/:id/message    - Send message
GET    /session/:id/message    - List messages
DELETE /message/:id            - Delete message

// Streaming
GET    /event                  - SSE event stream
WS     /ws                     - WebSocket connection

// Tools & Providers
GET    /tool                   - List tools
GET    /provider               - List providers
GET    /model                  - List models
```

---

## 6. Configuration System

### Configuration Hierarchy
```
┌─────────────────────────────────────────────────────────────────┐
│                   Configuration Precedence                       │
│                    (Highest to Lowest)                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Inline Config      OPENCODE_CONFIG_CONTENT env var          │
│           │                                                      │
│           ▼                                                      │
│  2. Project Config     ./opencode.jsonc or ./opencode.json      │
│           │                                                      │
│           ▼                                                      │
│  3. Custom Path        OPENCODE_CONFIG flag                     │
│           │                                                      │
│           ▼                                                      │
│  4. Global Config      ~/.opencode/opencode.jsonc               │
│           │                                                      │
│           ▼                                                      │
│  5. Remote Config      .well-known/opencode endpoint            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Configuration Schema
```typescript
// Location: packages/opencode/src/config/config.ts

interface Config {
  // Provider settings
  provider?: Record<string, ProviderConfig>

  // Agent definitions
  agent?: Record<string, AgentConfig>

  // Permission rules
  permission?: PermissionRuleset

  // MCP servers
  mcp?: {
    [name: string]: MCPServerConfig
  }

  // LSP settings
  lsp?: false | Record<string, LSPConfig>

  // Plugins
  plugin?: string[]

  // Feature flags
  experimental?: {
    batch_tool?: boolean
    // ... other flags
  }

  // User instructions
  instructions?: string
}
```

---

## 7. Plugin System

### Plugin Architecture
```
┌─────────────────────────────────────────────────────────────────┐
│                       Plugin System                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Plugin Sources                                                  │
│  ├── Built-in: opencode-copilot-auth, opencode-anthropic-auth  │
│  ├── npm: package@version                                       │
│  └── Local: file:///path/to/plugin.ts                          │
│                                                                  │
│  Plugin Hooks                                                    │
│  ├── auth.*           - Authentication hooks                    │
│  ├── event.*          - Event handling                          │
│  ├── tool.*           - Tool execution hooks                    │
│  │   ├── execute.before                                         │
│  │   └── execute.after                                          │
│  └── session.*        - Session lifecycle                       │
│                                                                  │
│  Plugin Loading                                                  │
│  1. Parse plugin string (package@version)                       │
│  2. Install via bun if needed                                   │
│  3. Import and initialize                                       │
│  4. Register hooks                                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Plugin Definition
```typescript
// Location: packages/opencode/src/plugin/index.ts

interface PluginHooks {
  auth?: {
    [provider: string]: () => Promise<AuthResult>
  }
  tool?: {
    [toolId: string]: ToolDefinition
  }
  event?: {
    [eventName: string]: (event: any) => void
  }
  // Additional hook types...
}

type PluginInstance = (input: PluginInput) => Promise<PluginHooks>
```

---

## 8. Event Bus

### Bus Architecture
```
┌─────────────────────────────────────────────────────────────────┐
│                        Event Bus                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Instance Bus (Per-Directory)                                    │
│  ├── Type-safe events via BusEvent.define()                     │
│  ├── Zod schema validation                                      │
│  └── Synchronous delivery within instance                       │
│                                                                  │
│  Global Bus (Cross-Instance)                                     │
│  ├── Node.js EventEmitter                                       │
│  ├── Directory-scoped events                                    │
│  └── Cross-process communication                                │
│                                                                  │
│  Key Events                                                      │
│  ├── session.created/updated/archived/reverted                  │
│  ├── message.updated/removed                                    │
│  ├── message.part.updated/removed                               │
│  ├── session.status (idle/busy/retry)                          │
│  ├── permission.asked/replied                                   │
│  ├── mcp.tools.changed                                          │
│  └── lsp.updated                                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Event Definition Pattern
```typescript
// Location: packages/opencode/src/bus/

// Define event with schema
const MyEvent = BusEvent.define("my.event", z.object({
  id: z.string(),
  data: z.any()
}))

// Publish event
Bus.publish(MyEvent, { id: "123", data: {...} })

// Subscribe to event
Bus.subscribe(MyEvent, (event) => {
  console.log(event.properties.id)
})
```

---

## Component Interaction Matrix

| Component | Depends On | Events Published | Events Consumed |
|-----------|-----------|------------------|-----------------|
| Agent | Config, Provider | - | session.status |
| Session | Storage, Agent, Tool | session.*, message.* | - |
| Tool | Permission, LSP, MCP | - | permission.replied |
| Provider | Config, Auth | - | - |
| Server | Session, Tool, Provider | - | All (for SSE) |
| Permission | Storage | permission.* | - |
| LSP | Config | lsp.* | - |
| MCP | Config, OAuth | mcp.* | - |
