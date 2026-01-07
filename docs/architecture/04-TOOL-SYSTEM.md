# Tool System Architecture

## 1. Tool Overview

### Tool Categories
```
┌─────────────────────────────────────────────────────────────────┐
│                       Tool Registry                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  File Operations (6)                                             │
│  ├── read     - Read file contents (supports images, PDFs)      │
│  ├── write    - Create/overwrite files                          │
│  ├── edit     - String replacement in files                     │
│  ├── glob     - Find files by pattern                           │
│  ├── grep     - Search file contents (ripgrep)                  │
│  └── patch    - Apply unified diffs                             │
│                                                                  │
│  Code Analysis (2)                                               │
│  ├── lsp      - Language server operations                      │
│  └── codesearch - Search code examples (Exa API)               │
│                                                                  │
│  Web Access (2)                                                  │
│  ├── webfetch   - Fetch and convert web pages                  │
│  └── websearch  - Web search (Exa API)                         │
│                                                                  │
│  Execution (2)                                                   │
│  ├── bash     - Shell command execution                         │
│  └── task     - Subagent invocation                             │
│                                                                  │
│  Session Management (3)                                          │
│  ├── todowrite - Update todo list                               │
│  ├── todoread  - Read todo list                                 │
│  └── skill     - Load specialized skills                        │
│                                                                  │
│  Advanced (3)                                                    │
│  ├── multiedit - Edit multiple files atomically                 │
│  ├── batch     - Parallel tool execution (experimental)         │
│  └── invalid   - Error handler for invalid calls                │
│                                                                  │
│  External (Dynamic)                                              │
│  ├── Custom tools from ~/.opencode/tool/*.ts                   │
│  ├── Plugin-provided tools                                      │
│  └── MCP tools (mcp__{server}__{tool})                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Tool Definition Pattern

### Tool Interface
```typescript
// Location: packages/opencode/src/tool/tool.ts

interface ToolInfo<P extends z.ZodType, M extends Metadata> {
  id: string
  init: (ctx?: InitContext) => Promise<{
    // Tool metadata
    description: string
    parameters: P  // Zod schema

    // Execution function
    execute(args: z.infer<P>, ctx: Context): Promise<{
      title: string      // Display title
      metadata: M        // Tool-specific metadata
      output: string     // Result text
      attachments?: FilePart[]  // Optional file attachments
    }>

    // Optional error formatting
    formatValidationError?(error: z.ZodError): string
  }>
}

// Tool context provided during execution
interface Context<M extends Metadata> {
  sessionID: string
  messageID: string
  agent: string
  abort: AbortSignal
  callID?: string
  extra?: Record<string, any>

  // Update metadata during execution
  metadata(input: { title?: string; metadata?: M }): void

  // Request permission
  ask(request: PermissionRequest): Promise<void>
}
```

### Tool Definition Example
```typescript
// Example: Simple file read tool
export const ReadTool = Tool.define("read", async () => ({
  description: "Read file contents",

  parameters: z.object({
    path: z.string().describe("File path to read"),
    offset: z.number().optional().describe("Line offset"),
    limit: z.number().optional().describe("Max lines")
  }),

  async execute(args, ctx) {
    // Request permission
    await ctx.ask({
      permission: "read",
      patterns: [args.path],
      metadata: {}
    })

    // Read file
    const content = await Bun.file(args.path).text()

    // Update metadata for real-time feedback
    ctx.metadata({
      title: `Reading ${path.basename(args.path)}`,
      metadata: { lines: content.split('\n').length }
    })

    return {
      title: args.path,
      metadata: { path: args.path },
      output: content
    }
  }
}))
```

---

## 3. Tool Execution Pipeline

### Execution Flow
```
┌─────────────────────────────────────────────────────────────────┐
│                    Tool Execution Pipeline                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Tool Call from LLM                                          │
│     ┌─────────────────────────────────────────────────────┐     │
│     │  { name: "read", arguments: { path: "/file.ts" } }  │     │
│     └────────────────────────┬────────────────────────────┘     │
│                              │                                   │
│  2. Parameter Validation     ▼                                   │
│     ┌─────────────────────────────────────────────────────┐     │
│     │  Zod schema validation                               │     │
│     │  • Type checking                                     │     │
│     │  • Required field validation                         │     │
│     │  • Custom error formatting                           │     │
│     └────────────────────────┬────────────────────────────┘     │
│                              │                                   │
│  3. Permission Check         ▼                                   │
│     ┌─────────────────────────────────────────────────────┐     │
│     │  PermissionNext.ask({                                │     │
│     │    permission: "read",                               │     │
│     │    patterns: ["/file.ts"]                           │     │
│     │  })                                                  │     │
│     │                                                      │     │
│     │  Evaluate against ruleset:                           │     │
│     │  • Config permissions                                │     │
│     │  • Agent permissions                                 │     │
│     │  • Session approvals                                 │     │
│     │                                                      │     │
│     │  Actions: allow | deny | ask                        │     │
│     └────────────────────────┬────────────────────────────┘     │
│                              │                                   │
│  4. Plugin Hooks (Before)    ▼                                   │
│     ┌─────────────────────────────────────────────────────┐     │
│     │  await Plugin.trigger("tool.execute.before", {...}) │     │
│     └────────────────────────┬────────────────────────────┘     │
│                              │                                   │
│  5. Tool Execution           ▼                                   │
│     ┌─────────────────────────────────────────────────────┐     │
│     │  const result = await tool.execute(args, ctx)       │     │
│     │                                                      │     │
│     │  During execution:                                   │     │
│     │  • ctx.metadata() for real-time updates             │     │
│     │  • ctx.abort for cancellation                       │     │
│     │  • ctx.ask() for additional permissions             │     │
│     └────────────────────────┬────────────────────────────┘     │
│                              │                                   │
│  6. Output Truncation        ▼                                   │
│     ┌─────────────────────────────────────────────────────┐     │
│     │  Truncate.output(result)                             │     │
│     │  • Max 32,000 tokens (configurable)                 │     │
│     │  • Preserve start and end context                   │     │
│     └────────────────────────┬────────────────────────────┘     │
│                              │                                   │
│  7. Plugin Hooks (After)     ▼                                   │
│     ┌─────────────────────────────────────────────────────┐     │
│     │  await Plugin.trigger("tool.execute.after", {...})  │     │
│     └────────────────────────┬────────────────────────────┘     │
│                              │                                   │
│  8. Store Result             ▼                                   │
│     ┌─────────────────────────────────────────────────────┐     │
│     │  Session.updatePart(messageID, partID, result)      │     │
│     │  Bus.publish(message.part.updated, {...})           │     │
│     └─────────────────────────────────────────────────────┘     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4. Permission System

### Permission Architecture
```
┌─────────────────────────────────────────────────────────────────┐
│                    Permission System                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Permission Types                                                │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  File Operations:                                        │    │
│  │  • read - File reading                                  │    │
│  │  • edit - File modification (covers edit, write, patch) │    │
│  │  • glob - File pattern matching                         │    │
│  │  • grep - Content searching                             │    │
│  │                                                          │    │
│  │  Execution:                                              │    │
│  │  • bash - Shell command execution                       │    │
│  │  • task - Subagent invocation                           │    │
│  │                                                          │    │
│  │  External:                                               │    │
│  │  • webfetch - Web page fetching                         │    │
│  │  • websearch - Web searching                            │    │
│  │  • codesearch - Code example searching                  │    │
│  │  • external_directory - Access outside project          │    │
│  │                                                          │    │
│  │  Other:                                                  │    │
│  │  • lsp - Language server operations                     │    │
│  │  • skill - Skill loading                                │    │
│  │  • todowrite/todoread - Todo list operations           │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  Rule Evaluation                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Ruleset: Array of rules, last match wins               │    │
│  │                                                          │    │
│  │  Rule: {                                                 │    │
│  │    permission: string (wildcard supported)              │    │
│  │    pattern: string (wildcard supported)                 │    │
│  │    action: "allow" | "deny" | "ask"                    │    │
│  │  }                                                       │    │
│  │                                                          │    │
│  │  Example:                                                │    │
│  │  [                                                       │    │
│  │    { permission: "*", pattern: "*", action: "ask" },   │    │
│  │    { permission: "read", pattern: "*", action: "allow"},│    │
│  │    { permission: "bash", pattern: "rm *", action: "deny"}│   │
│  │  ]                                                       │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  Ruleset Sources (merged in order)                              │
│  1. Agent permissions (agent.permission)                        │
│  2. Config permissions (config.permission)                      │
│  3. Session approvals (accumulated during session)              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Bash Command Arity
```typescript
// Location: packages/opencode/src/permission/arity.ts

// Commands are matched by prefix for permission patterns
// "git checkout main" → permission pattern "git checkout"

const ARITY: Record<string, number> = {
  // Single-word commands
  "cat": 1,      // cat → "cat"
  "ls": 1,       // ls -la → "ls"

  // Two-word commands
  "git": 2,      // git checkout → "git checkout"
  "npm": 2,      // npm install → "npm install"
  "docker": 2,   // docker run → "docker run"

  // Three-word commands
  "npm run": 3,  // npm run dev → "npm run dev"
  "docker compose": 3,

  // ... 160+ command patterns
}
```

---

## 5. Built-in Tools Detail

### File Operations

#### read
```typescript
parameters: {
  path: string           // Absolute or relative path
  offset?: number        // Start line (1-based)
  limit?: number         // Max lines to read
}

features:
  • Binary file detection
  • Image rendering (base64)
  • PDF text extraction
  • Line number formatting
  • Symlink following
```

#### write
```typescript
parameters: {
  path: string           // Target path
  content: string        // File content
}

features:
  • Directory creation
  • LSP diagnostics feedback
  • File time tracking
  • Atomic writes
```

#### edit
```typescript
parameters: {
  path: string           // Target file
  old: string            // Text to find
  new: string            // Replacement text
  all?: boolean          // Replace all occurrences
}

features:
  • Unique match validation
  • LSP diagnostics feedback
  • Undo support (via session revert)
```

#### glob
```typescript
parameters: {
  pattern: string        // Glob pattern
  path?: string          // Base directory
}

features:
  • Recursive matching
  • Modification time sorting
  • Result limiting (100 files)
```

#### grep
```typescript
parameters: {
  pattern: string        // Regex pattern
  path?: string          // Search directory
  include?: string       // File pattern filter
}

features:
  • ripgrep backend
  • Context lines
  • Result limiting (100 matches)
```

### Execution Tools

#### bash
```typescript
parameters: {
  command: string        // Shell command
  timeout?: number       // Max execution time (ms)
  workdir?: string       // Working directory
  background?: boolean   // Run in background
}

features:
  • Real-time output streaming
  • Timeout handling
  • Signal forwarding
  • Permission pattern extraction
  • Directory access checking
```

#### task
```typescript
parameters: {
  description: string    // Task description
  prompt: string         // Full prompt for subagent
  agent?: string         // Agent ID to use
}

features:
  • Subagent spawning
  • Context isolation
  • Result aggregation
```

### Web Tools

#### webfetch
```typescript
parameters: {
  url: string            // URL to fetch
  prompt: string         // Extraction prompt
}

features:
  • HTML to Markdown conversion
  • 5MB size limit
  • Redirect handling
  • Cache support (15 min)
```

#### websearch
```typescript
parameters: {
  query: string          // Search query
  mode?: "fast" | "deep" | "auto"
}

features:
  • Exa API integration
  • Result summarization
  • Source citations
```

---

## 6. Custom Tool Development

### Tool File Location
```
~/.opencode/tool/
├── my_tool.ts           # Single export
├── utils.ts             # Multiple exports
└── complex/
    └── index.ts         # Module with default export
```

### Custom Tool Example
```typescript
// ~/.opencode/tool/my_tool.ts

import { z } from "zod"

export const myTool = {
  description: "My custom tool that does something",

  args: {
    input: z.string().describe("Input text"),
    count: z.number().optional().default(1)
  },

  async execute(args, ctx) {
    // Perform operation
    const result = args.input.repeat(args.count)

    // Return result
    return result
  }
}
```

### Plugin Tool Registration
```typescript
// Plugin that provides tools
export default (input) => ({
  tool: {
    "plugin_tool": {
      description: "Tool from plugin",
      args: {
        data: z.any()
      },
      async execute(args, ctx) {
        return JSON.stringify(args.data)
      }
    }
  }
})
```

---

## 7. MCP Tool Integration

### MCP Tool Flow
```
┌─────────────────────────────────────────────────────────────────┐
│                     MCP Tool Integration                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Server Connection                                            │
│     ┌─────────────────────────────────────────────────────┐     │
│     │  MCP.connect("server-name")                          │     │
│     │  • Spawn local process OR                           │     │
│     │  • Connect to remote HTTP endpoint                  │     │
│     └────────────────────────┬────────────────────────────┘     │
│                              │                                   │
│  2. Tool Discovery           ▼                                   │
│     ┌─────────────────────────────────────────────────────┐     │
│     │  client.listTools()                                  │     │
│     │  → [{ name, description, inputSchema }, ...]        │     │
│     └────────────────────────┬────────────────────────────┘     │
│                              │                                   │
│  3. Tool Conversion          ▼                                   │
│     ┌─────────────────────────────────────────────────────┐     │
│     │  // Convert MCP tool to AI SDK format               │     │
│     │  {                                                   │     │
│     │    id: "mcp__server__toolname",                     │     │
│     │    description: mcpTool.description,                 │     │
│     │    parameters: jsonSchema(mcpTool.inputSchema),     │     │
│     │    execute: (args) => client.callTool(...)          │     │
│     │  }                                                   │     │
│     └────────────────────────┬────────────────────────────┘     │
│                              │                                   │
│  4. Tool Execution           ▼                                   │
│     ┌─────────────────────────────────────────────────────┐     │
│     │  client.callTool({                                   │     │
│     │    name: "toolname",                                 │     │
│     │    arguments: { ... }                                │     │
│     │  })                                                  │     │
│     │  → { content: [...], isError: boolean }             │     │
│     └─────────────────────────────────────────────────────┘     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8. Tool Metadata & Output

### Real-Time Metadata Updates
```typescript
// During tool execution, update metadata for UI feedback
async execute(args, ctx) {
  let progress = 0

  const interval = setInterval(() => {
    progress += 10
    ctx.metadata({
      title: `Processing... ${progress}%`,
      metadata: { progress }
    })
  }, 100)

  try {
    const result = await longOperation()
    return { output: result, ... }
  } finally {
    clearInterval(interval)
  }
}
```

### Output Truncation
```typescript
// Location: packages/opencode/src/util/truncate.ts

function output(content: string): { content: string; truncated: boolean } {
  const maxTokens = Flag.OPENCODE_TOOL_OUTPUT_LIMIT ?? 32_000

  if (tokenCount(content) <= maxTokens) {
    return { content, truncated: false }
  }

  // Keep beginning and end for context
  const head = content.slice(0, headSize)
  const tail = content.slice(-tailSize)

  return {
    content: `${head}\n\n... [truncated] ...\n\n${tail}`,
    truncated: true
  }
}
```

---

## 9. Error Handling

### Tool Error Types
```typescript
// Validation errors (Zod)
class ValidationError extends Error {
  constructor(public zodError: z.ZodError) {}
}

// Permission denied
class PermissionDeniedError extends Error {
  constructor(
    public permission: string,
    public pattern: string
  ) {}
}

// Tool execution errors
class ToolExecutionError extends Error {
  constructor(
    public toolId: string,
    public cause: Error
  ) {}
}
```

### Error Flow
```
Tool Error
    │
    ▼
┌──────────────────────┐
│ Catch in processor   │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ Format error message │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ Store as tool result │
│ (error state)        │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ Return to LLM for    │
│ potential retry      │
└──────────────────────┘
```

---

## 10. Tool System Flags

| Flag | Default | Purpose |
|------|---------|---------|
| `OPENCODE_EXPERIMENTAL_LSP_TOOL` | false | Enable LSP tool |
| `OPENCODE_ENABLE_EXA` | false | Enable web/code search |
| `OPENCODE_TOOL_OUTPUT_LIMIT` | 32000 | Max output tokens |
| `OPENCODE_DISABLE_DEFAULT_PLUGINS` | false | Skip built-in plugins |
| `experimental.batch_tool` | false | Enable batch tool |
