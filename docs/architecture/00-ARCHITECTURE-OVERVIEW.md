# OpenCode System Architecture Overview

## Executive Summary

OpenCode is a comprehensive, open-source AI-powered development agent that provides cross-platform support for CLI, Desktop, and Web interfaces. It features a provider-agnostic LLM integration supporting 23+ providers, a sophisticated tool system, and enterprise-grade features including billing, authentication, and multi-tenant support.

## System Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            OPENCODE ARCHITECTURE                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
│  │     CLI      │  │     TUI      │  │   Desktop    │  │     Web      │    │
│  │   (Yargs)    │  │ (OpenTUI+    │  │   (Tauri)    │  │  (SolidJS+   │    │
│  │              │  │  SolidJS)    │  │              │  │   Astro)     │    │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘    │
│         │                 │                 │                 │             │
│         └─────────────────┴─────────────────┴─────────────────┘             │
│                                   │                                          │
│                                   ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         HTTP Server (Hono)                          │   │
│  │  • RESTful API with OpenAPI spec  • SSE/WebSocket streaming         │   │
│  │  • Session management             • Permission handling              │   │
│  └────────────────────────────────┬────────────────────────────────────┘   │
│                                   │                                          │
│         ┌─────────────────────────┼─────────────────────────┐               │
│         ▼                         ▼                         ▼               │
│  ┌────────────┐          ┌────────────────┐         ┌────────────┐         │
│  │   Agent    │◄────────►│    Session     │◄───────►│    Tool    │         │
│  │   System   │          │   Processor    │         │  Registry  │         │
│  └────────────┘          └────────────────┘         └────────────┘         │
│         │                        │                         │                │
│         ▼                        ▼                         ▼                │
│  ┌────────────┐          ┌────────────────┐         ┌────────────┐         │
│  │  Provider  │          │    Storage     │         │    LSP     │         │
│  │   Layer    │          │    Layer       │         │  Manager   │         │
│  └────────────┘          └────────────────┘         └────────────┘         │
│         │                        │                         │                │
│         ▼                        ▼                         ▼                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                           Event Bus                                  │   │
│  │           (Pub/Sub for component communication)                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘

                    ┌────────────────────────────────────┐
                    │        External Services           │
                    ├────────────────────────────────────┤
                    │ • 23+ LLM Providers               │
                    │ • MCP Servers (stdio/HTTP)        │
                    │ • LSP Servers (50+ languages)     │
                    │ • OAuth Providers                 │
                    │ • Stripe (Billing)                │
                    │ • PlanetScale (Database)          │
                    └────────────────────────────────────┘
```

## Technology Stack

### Core Runtime
| Component | Technology | Purpose |
|-----------|------------|---------|
| Runtime | Bun 1.3.5+ | Fast JavaScript runtime |
| Language | TypeScript (ESM) | Type-safe development |
| Build System | Turbo | Monorepo orchestration |
| Package Manager | Bun Workspaces | Dependency management |

### Frontend
| Component | Technology | Purpose |
|-----------|------------|---------|
| UI Framework | SolidJS 1.9.10 | Reactive UI components |
| TUI Framework | OpenTUI | Terminal UI rendering |
| Desktop | Tauri 2.x | Native cross-platform app |
| Web Framework | Astro 5.7.13 | Static site generation |
| Styling | Tailwind CSS | Utility-first CSS |

### Backend
| Component | Technology | Purpose |
|-----------|------------|---------|
| HTTP Server | Hono 4.10.7 | Lightweight web framework |
| AI Integration | Vercel AI SDK 5.0.97 | LLM abstraction layer |
| Database | PlanetScale | MySQL-compatible DB |
| Payments | Stripe | Billing and subscriptions |
| Infrastructure | SST (Cloudflare) | Serverless deployment |

### Development Tools
| Component | Technology | Purpose |
|-----------|------------|---------|
| Testing | Bun Test | Unit and integration tests |
| Formatting | Prettier | Code formatting |
| Environment | Nix | Reproducible builds |
| CI/CD | GitHub Actions | Automated workflows |

## Monorepo Structure

```
opencode/
├── packages/
│   ├── opencode/          # Core CLI & Engine (26K+ LOC)
│   │   ├── src/
│   │   │   ├── agent/     # Agent implementations
│   │   │   ├── acp/       # Agent Client Protocol
│   │   │   ├── auth/      # Authentication
│   │   │   ├── bus/       # Event bus system
│   │   │   ├── cli/       # CLI commands
│   │   │   ├── command/   # Command system
│   │   │   ├── config/    # Configuration management
│   │   │   ├── lsp/       # Language Server Protocol
│   │   │   ├── mcp/       # Model Context Protocol
│   │   │   ├── permission/# Permission system
│   │   │   ├── plugin/    # Plugin architecture
│   │   │   ├── provider/  # LLM providers
│   │   │   ├── server/    # HTTP server
│   │   │   ├── session/   # Session management
│   │   │   ├── storage/   # Persistence layer
│   │   │   ├── tool/      # Tool system
│   │   │   └── ...
│   │   └── test/          # Test suite
│   │
│   ├── app/               # TUI Application (SolidJS)
│   ├── console/           # Admin Console (Web)
│   │   ├── app/           # SolidJS frontend
│   │   ├── core/          # Backend logic
│   │   └── ...
│   ├── desktop/           # Desktop App (Tauri)
│   ├── web/               # Marketing Site (Astro)
│   ├── ui/                # Shared UI Components
│   ├── sdk/js/            # JavaScript SDK
│   ├── plugin/            # Plugin SDK
│   └── util/              # Shared utilities
│
├── infra/                 # SST Infrastructure
├── nix/                   # Nix packaging
├── script/                # Build scripts
└── .github/workflows/     # CI/CD (21 workflows)
```

## Key Architectural Patterns

### 1. Event-Driven Architecture
- Pub/Sub event bus for component decoupling
- Global event emitter for cross-instance communication
- Type-safe events with Zod schema validation

### 2. Provider Pattern
- Pluggable LLM provider system (23+ providers)
- Common interface abstraction via Vercel AI SDK
- Dynamic provider loading and configuration

### 3. Instance Context Pattern
- Per-directory instance isolation
- Lazy initialization with caching
- Proper resource cleanup via disposables

### 4. Tool Registry Pattern
- Declarative tool definitions with Zod schemas
- Permission-aware execution pipeline
- Plugin and MCP integration

### 5. Session-Based State Management
- File-based JSON storage with locking
- Event-driven state synchronization
- Real-time streaming via SSE/WebSocket

## Version Information

- **Current Version**: 1.1.6
- **License**: MIT
- **Repository**: github.com/anomalyco/opencode

## Document Index

1. [Architecture Overview](./00-ARCHITECTURE-OVERVIEW.md) - This document
2. [Core Components](./01-CORE-COMPONENTS.md) - Detailed component breakdown
3. [Data Flow](./02-DATA-FLOW.md) - State management and data flow
4. [Integration Architecture](./03-INTEGRATIONS.md) - External service integrations
5. [Tool System](./04-TOOL-SYSTEM.md) - Tool architecture and lifecycle
6. [Python Reimplementation Plan](./05-PYTHON-REIMPLEMENTATION.md) - Migration guide
