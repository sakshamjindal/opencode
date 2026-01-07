# External Integrations Architecture

## 1. LLM Provider Integration

### Provider Architecture
```
┌─────────────────────────────────────────────────────────────────┐
│                    LLM Provider System                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                  Vercel AI SDK                           │    │
│  │              (Unified Abstraction Layer)                 │    │
│  │                                                          │    │
│  │  • Common interface for all providers                    │    │
│  │  • Streaming support                                     │    │
│  │  • Tool calling normalization                            │    │
│  │  • Token counting                                        │    │
│  └────────────────────────┬────────────────────────────────┘    │
│                           │                                      │
│    ┌──────────┬───────────┼───────────┬──────────┐              │
│    ▼          ▼           ▼           ▼          ▼              │
│  ┌────┐    ┌────┐     ┌────┐     ┌────┐    ┌────────┐          │
│  │Anth│    │Open│     │Goog│     │Azur│    │OpenAI  │          │
│  │ropi│    │AI  │     │le  │     │e   │    │Compat. │          │
│  └────┘    └────┘     └────┘     └────┘    └────────┘          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Supported Providers (23+)

| Provider | Package | Features | Auth Method |
|----------|---------|----------|-------------|
| Anthropic | `@ai-sdk/anthropic` | Extended thinking, caching | API Key |
| OpenAI | `@ai-sdk/openai` | GPT-4o, o1, Responses API | API Key |
| Google | `@ai-sdk/google` | Gemini models | API Key |
| Google Vertex | `@ai-sdk/google-vertex` | Enterprise Gemini | Service Account |
| Azure | `@ai-sdk/azure` | OpenAI Service | API Key + Endpoint |
| Amazon Bedrock | `@ai-sdk/amazon-bedrock` | Claude, Titan | AWS Credentials |
| GitHub Copilot | Custom | VS Code integration | OAuth |
| OpenRouter | `@openrouter/ai-sdk-provider` | 100+ models | API Key |
| Mistral | `@ai-sdk/mistral` | Mistral/Mixtral | API Key |
| Groq | `@ai-sdk/groq` | Fast inference | API Key |
| Cohere | `@ai-sdk/cohere` | Command models | API Key |
| Together | `@ai-sdk/togetherai` | Open models | API Key |
| Perplexity | `@ai-sdk/perplexity` | Search-augmented | API Key |
| xAI | `@ai-sdk/xai` | Grok models | API Key |
| Cerebras | `@ai-sdk/cerebras` | Fast inference | API Key |
| DeepInfra | Custom | Hosted models | API Key |
| Cloudflare | Custom | AI Gateway | Account ID |
| SAP AI Core | Custom | Enterprise | OAuth |

### Provider Configuration
```typescript
// Location: packages/opencode/src/provider/provider.ts

interface ProviderConfig {
  // Provider identification
  id: string
  name: string

  // API configuration
  api?: {
    key?: string           // API key
    baseURL?: string       // Custom endpoint
    region?: string        // For AWS Bedrock
  }

  // Provider-specific options
  options?: {
    organization?: string  // OpenAI org
    project?: string       // Vertex project
    // ... provider-specific
  }
}

// Configuration sources (precedence):
// 1. Environment variables (ANTHROPIC_API_KEY, etc.)
// 2. Config file (opencode.jsonc)
// 3. Auth storage (~/.opencode/auth.json)
// 4. Interactive prompt
```

### Model Discovery
```typescript
// Location: packages/opencode/src/provider/models.ts

interface ModelInfo {
  id: string
  name: string
  provider: string

  // Capabilities
  features: {
    temperature: boolean
    reasoning: boolean
    toolCalling: boolean
  }

  // Modalities
  input: ("text" | "image" | "audio" | "video" | "pdf")[]
  output: ("text" | "image" | "audio")[]

  // Pricing (per million tokens)
  cost?: {
    input: number
    output: number
    cacheRead?: number
    cacheWrite?: number
  }
}
```

---

## 2. Model Context Protocol (MCP)

### MCP Architecture
```
┌─────────────────────────────────────────────────────────────────┐
│                        MCP System                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Configuration                                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  mcp: {                                                  │    │
│  │    "local-server": {                                     │    │
│  │      command: ["node", "server.js"],                    │    │
│  │      args: ["--port", "3000"],                          │    │
│  │      env: { API_KEY: "..." }                            │    │
│  │    },                                                    │    │
│  │    "remote-server": {                                    │    │
│  │      url: "https://mcp.example.com/sse",                │    │
│  │      type: "remote"                                      │    │
│  │    }                                                     │    │
│  │  }                                                       │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  Transport Types                                                 │
│  ┌──────────────────┐     ┌──────────────────┐                  │
│  │   Stdio (Local)  │     │  HTTP/SSE (Remote)│                  │
│  │                  │     │                  │                  │
│  │ spawn(command)   │     │ fetch(url)       │                  │
│  │ stdin/stdout     │     │ EventSource      │                  │
│  │ JSON-RPC         │     │ OAuth support    │                  │
│  └──────────────────┘     └──────────────────┘                  │
│                                                                  │
│  MCP Client Manager                                              │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  • Tool discovery and registration                       │    │
│  │  • Prompt listing and invocation                         │    │
│  │  • Resource reading                                      │    │
│  │  • Status tracking (connected/disabled/failed)           │    │
│  │  • OAuth flow handling                                   │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### MCP Tool Integration
```typescript
// Location: packages/opencode/src/mcp/index.ts

// MCP tools are converted to AI SDK format
async function convertMcpTool(mcpTool: MCPTool, client: MCPClient): Promise<Tool> {
  return {
    description: mcpTool.description,
    parameters: jsonSchema(mcpTool.inputSchema),
    execute: async (args) => {
      const result = await client.callTool({
        name: mcpTool.name,
        arguments: args
      })
      return formatResult(result)
    }
  }
}

// Tool naming: mcp__{serverName}__{toolName}
// Example: mcp__filesystem__readFile
```

### MCP OAuth Flow
```
┌─────────────────────────────────────────────────────────────────┐
│                      MCP OAuth Flow                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Discovery                                                    │
│     GET {server}/.well-known/oauth-authorization-server         │
│                                                                  │
│  2. Dynamic Client Registration (optional)                       │
│     POST {authorization_server}/register                        │
│     → client_id, client_secret (if confidential)                │
│                                                                  │
│  3. Authorization                                                │
│     Open browser → {authorization_endpoint}                     │
│     ?client_id=...                                               │
│     &redirect_uri=http://127.0.0.1:19876/mcp/oauth/callback     │
│     &code_challenge=... (PKCE)                                  │
│     &state=... (CSRF protection)                                │
│                                                                  │
│  4. Callback                                                     │
│     Local server receives code at port 19876                    │
│     Validates state parameter                                    │
│                                                                  │
│  5. Token Exchange                                               │
│     POST {token_endpoint}                                        │
│     → access_token, refresh_token, expires_in                   │
│                                                                  │
│  6. Storage                                                      │
│     Tokens stored in ~/.opencode/auth.json                      │
│     Keyed by server URL for security                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. Language Server Protocol (LSP)

### LSP Architecture
```
┌─────────────────────────────────────────────────────────────────┐
│                        LSP System                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Server Discovery                                                │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Built-in Servers (50+ languages)                        │    │
│  │  ├── TypeScript: typescript-language-server              │    │
│  │  ├── Python: pyright or ty                               │    │
│  │  ├── Go: gopls                                           │    │
│  │  ├── Rust: rust-analyzer                                 │    │
│  │  ├── Java: jdtls                                         │    │
│  │  └── ... 45+ more                                        │    │
│  │                                                          │    │
│  │  Custom Servers (config)                                  │    │
│  │  lsp: {                                                   │    │
│  │    "custom": {                                           │    │
│  │      command: ["my-lsp", "--stdio"],                    │    │
│  │      extensions: [".custom"]                             │    │
│  │    }                                                      │    │
│  │  }                                                        │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  Client Management                                               │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Per-Project Clients                                     │    │
│  │  (root, serverID) → LSPClient                           │    │
│  │                                                          │    │
│  │  Lazy Initialization                                     │    │
│  │  • Spawn on first file access                           │    │
│  │  • Cache for subsequent requests                        │    │
│  │  • Auto-cleanup on project close                        │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### LSP Features Used
| Feature | LSP Method | Integration Point |
|---------|-----------|-------------------|
| Diagnostics | `textDocument/publishDiagnostics` | Edit/Write tools |
| Go to Definition | `textDocument/definition` | LSP tool |
| Find References | `textDocument/references` | LSP tool |
| Hover | `textDocument/hover` | LSP tool |
| Document Symbols | `textDocument/documentSymbol` | LSP tool |
| Workspace Symbols | `workspace/symbol` | LSP tool |
| Call Hierarchy | `textDocument/prepareCallHierarchy` | LSP tool |

### Diagnostic Integration
```typescript
// After file edit/write:
async function getDiagnostics(file: string): Promise<Diagnostic[]> {
  // 1. Notify LSP of file change
  await LSP.touchFile(file)

  // 2. Wait for diagnostics (up to 3s, debounced)
  await LSP.waitForDiagnostics({ path: file })

  // 3. Fetch diagnostics
  const diagnostics = await LSP.diagnostics(file)

  // 4. Format for tool output
  return diagnostics.map(d =>
    `${severity(d.severity)} [${d.range.start.line}:${d.range.start.character}] ${d.message}`
  )
}
```

---

## 4. OAuth & Authentication

### Auth Architecture
```
┌─────────────────────────────────────────────────────────────────┐
│                   Authentication System                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Auth Sources (Priority Order)                                   │
│  1. Environment Variables (ANTHROPIC_API_KEY, etc.)             │
│  2. Config File (provider.api.key)                              │
│  3. Auth Storage (~/.opencode/auth.json)                        │
│  4. Plugin Auth Hooks                                            │
│  5. Interactive Prompt                                           │
│                                                                  │
│  Auth Storage                                                    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  ~/.opencode/auth.json (mode 0600)                       │    │
│  │  {                                                       │    │
│  │    "anthropic": { "key": "sk-..." },                    │    │
│  │    "openai": { "key": "sk-..." },                       │    │
│  │    "mcp:https://server.com": {                          │    │
│  │      "accessToken": "...",                              │    │
│  │      "refreshToken": "...",                             │    │
│  │      "expiresAt": 1234567890                            │    │
│  │    }                                                     │    │
│  │  }                                                       │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  Plugin Auth Hooks                                               │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  // Built-in plugins                                     │    │
│  │  opencode-copilot-auth   // GitHub Copilot OAuth        │    │
│  │  opencode-anthropic-auth // Anthropic Console OAuth     │    │
│  │                                                          │    │
│  │  // Custom plugin auth                                   │    │
│  │  export default (input) => ({                           │    │
│  │    auth: {                                              │    │
│  │      "my-provider": async () => {                       │    │
│  │        return { apiKey: await fetchKey() }              │    │
│  │      }                                                   │    │
│  │    }                                                     │    │
│  │  })                                                      │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5. Billing & Console (Enterprise)

### Billing Architecture
```
┌─────────────────────────────────────────────────────────────────┐
│                     Console Billing                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Database: PlanetScale (MySQL)                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Tables:                                                 │    │
│  │  • BillingTable: subscription, payment methods          │    │
│  │  • PaymentTable: transaction history                     │    │
│  │  • UsageTable: API usage tracking                        │    │
│  │  • WorkspaceTable: multi-tenant workspaces              │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  Stripe Integration                                              │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  • Payment method management                             │    │
│  │  • Automatic reload on low balance                       │    │
│  │  • Monthly usage tracking                                │    │
│  │  • Invoice generation                                    │    │
│  │  • Webhook handling                                      │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  Zen API (Proxy)                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  /zen/v1/chat/completions                               │    │
│  │  /zen/v1/messages                                        │    │
│  │  /zen/v1/responses                                       │    │
│  │                                                          │    │
│  │  Features:                                               │    │
│  │  • Rate limiting per workspace                           │    │
│  │  • Usage tracking and cost calculation                   │    │
│  │  • Provider format conversion                            │    │
│  │  • Monthly spending limits                               │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6. Infrastructure

### Deployment Architecture
```
┌─────────────────────────────────────────────────────────────────┐
│                    Infrastructure (SST)                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Cloudflare                                                      │
│  ├── Workers: Serverless API endpoints                          │
│  ├── KV: Key-value storage                                      │
│  ├── D1: SQLite at the edge                                     │
│  └── R2: Object storage                                         │
│                                                                  │
│  AWS                                                             │
│  ├── Lambda: Backend functions                                  │
│  ├── S3: File storage                                           │
│  └── CloudFront: CDN                                            │
│                                                                  │
│  External Services                                               │
│  ├── PlanetScale: MySQL database                                │
│  ├── Stripe: Payments                                           │
│  └── GitHub: OAuth, releases                                    │
│                                                                  │
│  SST Config (sst.config.ts)                                     │
│  ├── Stage management (dev/prod)                                │
│  ├── Secret handling                                            │
│  └── Resource linking                                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Distribution Channels
| Channel | Platform | Package |
|---------|----------|---------|
| npm | All | `opencode` |
| Homebrew | macOS/Linux | `opencode` |
| Scoop | Windows | `opencode` |
| Chocolatey | Windows | `opencode` |
| AUR | Arch Linux | `opencode-bin` |
| Nix | NixOS | `opencode` |
| GitHub Releases | All | Binary downloads |

---

## 7. Integration Summary

### External Dependencies
```
┌─────────────────────────────────────────────────────────────────┐
│                   External Dependencies                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  AI/LLM                                                          │
│  • 23+ LLM providers via Vercel AI SDK                          │
│  • MCP servers (local and remote)                               │
│                                                                  │
│  Development Tools                                               │
│  • 50+ LSP servers for code intelligence                        │
│  • Git for version control                                      │
│  • ripgrep for fast file search                                 │
│                                                                  │
│  Web Services                                                    │
│  • Exa API (web search, code search)                            │
│  • OAuth providers (GitHub, Google, etc.)                       │
│                                                                  │
│  Infrastructure                                                  │
│  • Cloudflare (Workers, KV, D1, R2)                             │
│  • AWS (Lambda, S3, CloudFront)                                 │
│  • PlanetScale (MySQL)                                          │
│  • Stripe (payments)                                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### API Surface
| API | Protocol | Auth | Purpose |
|-----|----------|------|---------|
| OpenCode Server | HTTP/SSE | Session | Local CLI ↔ TUI |
| Console API | HTTP | OAuth | Web console |
| Zen API | HTTP | API Key | LLM proxy |
| MCP | JSON-RPC | OAuth/None | Tool integration |
| LSP | JSON-RPC | None | Code intelligence |
