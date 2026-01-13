# Jetson v0 - OpenCode MVP

## Vision

**Jetson v0** is the minimal viable foundation for OpenCode. It establishes the core architecture that will scale into the full OpenCode system.

```
Jetson v0 (MVP) ──▶ Jetson v1 ──▶ Jetson v2 ──▶ ... ──▶ OpenCode (Full)
     │                  │             │
     │                  │             └── Desktop, Web UI
     │                  └── MCP, LSP, Plugins
     └── Core + FastAPI + TUI
```

---

## Jetson v0 Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            JETSON v0                                         │
│                         OpenCode MVP                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                      INTERFACE LAYER                                 │   │
│   │                                                                      │   │
│   │   ┌─────────────┐                         ┌─────────────────────┐   │   │
│   │   │     CLI     │                         │    Textual TUI      │   │   │
│   │   │   (Typer)   │                         │                     │   │   │
│   │   │             │                         │  ┌───────────────┐  │   │   │
│   │   │ • run       │                         │  │ Standalone    │  │   │   │
│   │   │ • chat      │                         │  │ Mode (Direct) │  │   │   │
│   │   │ • serve     │                         │  ├───────────────┤  │   │   │
│   │   │             │                         │  │ Server Mode   │  │   │   │
│   │   │             │                         │  │ (via HTTP)    │  │   │   │
│   │   │             │                         │  └───────────────┘  │   │   │
│   │   └──────┬──────┘                         └──────────┬──────────┘   │   │
│   │          │                                           │              │   │
│   └──────────┼───────────────────────────────────────────┼──────────────┘   │
│              │                                           │                   │
│              │ Direct                                    │ HTTP/SSE          │
│              │                                           │                   │
│              ▼                                           ▼                   │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                       API LAYER                                      │   │
│   │                                                                      │   │
│   │   ┌─────────────────────────────────────────────────────────────┐   │   │
│   │   │                   FastAPI Server                             │   │   │
│   │   │                                                              │   │   │
│   │   │   Endpoints:                                                 │   │   │
│   │   │   ┌────────────────────────────────────────────────────┐    │   │   │
│   │   │   │  POST /api/session          Create session         │    │   │   │
│   │   │   │  GET  /api/session          List sessions          │    │   │   │
│   │   │   │  GET  /api/session/{id}     Get session details    │    │   │   │
│   │   │   │  POST /api/session/{id}/message   Send message     │    │   │   │
│   │   │   │       └──▶ Returns SSE stream                      │    │   │   │
│   │   │   │  GET  /api/events           Global event stream    │    │   │   │
│   │   │   │  GET  /api/tools            List available tools   │    │   │   │
│   │   │   └────────────────────────────────────────────────────┘    │   │   │
│   │   │                                                              │   │   │
│   │   │   SSE Events:                                                │   │   │
│   │   │   • text        - Streaming text chunks                      │   │   │
│   │   │   • tool_call   - Tool invocation                           │   │   │
│   │   │   • tool_result - Tool execution result                     │   │   │
│   │   │   • done        - Response complete                         │   │   │
│   │   │   • error       - Error occurred                            │   │   │
│   │   │                                                              │   │   │
│   │   └─────────────────────────────────────────────────────────────┘   │   │
│   │                                    │                                 │   │
│   └────────────────────────────────────┼─────────────────────────────────┘   │
│                                        │                                      │
│                                        ▼                                      │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                       CORE LAYER                                     │   │
│   │                                                                      │   │
│   │   ┌─────────────────────────────────────────────────────────────┐   │   │
│   │   │                     AGENT                                    │   │   │
│   │   │                                                              │   │   │
│   │   │    async def run(session, message) -> AsyncGenerator:        │   │   │
│   │   │                                                              │   │   │
│   │   │    ┌─────────┐     ┌─────────┐     ┌─────────────────┐     │   │   │
│   │   │    │ Add msg │────▶│  Call   │────▶│ Process Response│     │   │   │
│   │   │    │ to      │     │ LLM API │     │                 │     │   │   │
│   │   │    │ session │     │         │     │ ┌─────┐ ┌─────┐│     │   │   │
│   │   │    └─────────┘     └─────────┘     │ │Text │ │Tool ││     │   │   │
│   │   │                                     │ └──┬──┘ └──┬──┘│     │   │   │
│   │   │                                     └────┼───────┼───┘     │   │   │
│   │   │                                          │       │         │   │   │
│   │   │                         ┌────────────────┘       │         │   │   │
│   │   │                         ▼                        ▼         │   │   │
│   │   │                   ┌──────────┐           ┌──────────┐     │   │   │
│   │   │                   │  Yield   │           │ Execute  │     │   │   │
│   │   │                   │  Event   │           │  Tool    │     │   │   │
│   │   │                   └──────────┘           └────┬─────┘     │   │   │
│   │   │                                               │           │   │   │
│   │   │                                               ▼           │   │   │
│   │   │                                         ┌──────────┐     │   │   │
│   │   │                                         │  Loop    │     │   │   │
│   │   │                                         │  Back    │──┐  │   │   │
│   │   │                                         └──────────┘  │  │   │   │
│   │   │                                               ▲       │  │   │   │
│   │   │                                               └───────┘  │   │   │
│   │   │                                                          │   │   │
│   │   └──────────────────────────────────────────────────────────┘   │   │
│   │                                                                      │   │
│   │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │   │
│   │   │   PROVIDER   │  │   SESSION    │  │    TOOLS     │             │   │
│   │   │              │  │              │  │              │             │   │
│   │   │ • Anthropic  │  │ • id         │  │ • read_file  │             │   │
│   │   │ • stream()   │  │ • messages[] │  │ • write_file │             │   │
│   │   │ • complete() │  │ • created_at │  │ • edit_file  │             │   │
│   │   │              │  │ • title      │  │ • bash       │             │   │
│   │   │              │  │              │  │ • glob       │             │   │
│   │   │              │  │              │  │ • grep       │             │   │
│   │   └──────────────┘  └──────────────┘  └──────────────┘             │   │
│   │                                                                      │   │
│   │   ┌──────────────┐  ┌──────────────┐                                │   │
│   │   │   CONFIG     │  │  EVENT BUS   │                                │   │
│   │   │              │  │              │                                │   │
│   │   │ • model      │  │ • publish()  │                                │   │
│   │   │ • api_key    │  │ • subscribe()│                                │   │
│   │   │ • auto_read  │  │ • on()       │                                │   │
│   │   │              │  │              │                                │   │
│   │   └──────────────┘  └──────────────┘                                │   │
│   │                                                                      │   │
│   └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                      STORAGE LAYER                                   │   │
│   │                                                                      │   │
│   │   ~/.jetson/                                                        │   │
│   │   ├── config.json                                                   │   │
│   │   └── sessions/                                                     │   │
│   │       ├── {session_id}.json                                         │   │
│   │       └── ...                                                       │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Data Flow: Message Request

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                              MESSAGE FLOW                                     │
└──────────────────────────────────────────────────────────────────────────────┘

  ┌──────┐         ┌──────┐         ┌──────┐         ┌──────┐         ┌──────┐
  │ User │         │ TUI  │         │Server│         │Agent │         │Claude│
  └──┬───┘         └──┬───┘         └──┬───┘         └──┬───┘         └──┬───┘
     │                │                │                │                │
     │  "fix the bug" │                │                │                │
     │───────────────▶│                │                │                │
     │                │                │                │                │
     │                │  POST /message │                │                │
     │                │───────────────▶│                │                │
     │                │                │                │                │
     │                │                │  run(session,  │                │
     │                │                │      message)  │                │
     │                │                │───────────────▶│                │
     │                │                │                │                │
     │                │                │                │  stream()      │
     │                │                │                │───────────────▶│
     │                │                │                │                │
     │                │                │                │◀───────────────│
     │                │                │                │  text chunks   │
     │                │                │◀───────────────│                │
     │                │                │  yield Event   │                │
     │                │◀───────────────│                │                │
     │                │  SSE: text     │                │                │
     │◀───────────────│                │                │                │
     │  [streaming]   │                │                │                │
     │                │                │                │◀───────────────│
     │                │                │                │  tool_use      │
     │                │                │◀───────────────│                │
     │                │                │  yield Event   │                │
     │                │◀───────────────│                │                │
     │                │  SSE: tool_call│                │                │
     │◀───────────────│                │                │                │
     │  [tool: read]  │                │                │                │
     │                │                │                │                │
     │                │                │                │  execute_tool  │
     │                │                │                │───────────────▶│ FS
     │                │                │                │◀───────────────│
     │                │                │◀───────────────│                │
     │                │◀───────────────│  SSE: result   │                │
     │◀───────────────│                │                │                │
     │                │                │                │                │
     │                │                │                │  stream()      │
     │                │                │                │───────────────▶│
     │                │                │                │◀───────────────│
     │                │                │◀───────────────│  "Done!"       │
     │                │◀───────────────│                │                │
     │◀───────────────│  SSE: done     │                │                │
     │                │                │                │                │
  ┌──┴───┐         ┌──┴───┐         ┌──┴───┐         ┌──┴───┐         ┌──┴───┐
  │ User │         │ TUI  │         │Server│         │Agent │         │Claude│
  └──────┘         └──────┘         └──────┘         └──────┘         └──────┘
```

---

## File Structure

```
jetson/
├── pyproject.toml
├── README.md
├── src/
│   └── jetson/
│       │
│       ├── __init__.py              # __version__ = "0.1.0"
│       ├── __main__.py              # python -m jetson
│       │
│       │   ╔═══════════════════════════════════════════════════════════╗
│       │   ║                    INTERFACE LAYER                        ║
│       │   ╚═══════════════════════════════════════════════════════════╝
│       │
│       ├── cli.py                   # CLI commands (~100 lines)
│       │   │
│       │   ├── run(prompt)          # One-shot task
│       │   ├── chat()               # Interactive REPL
│       │   ├── serve(host, port)    # Start HTTP server
│       │   ├── tui()                # Launch Textual TUI (standalone)
│       │   └── ui(server)           # Launch TUI (server mode)
│       │
│       ├── tui/                     # Textual TUI (~350 lines)
│       │   ├── __init__.py
│       │   ├── app.py               # Main Textual application
│       │   ├── client.py            # HTTP/SSE client for server mode
│       │   ├── styles.tcss          # Textual CSS styles
│       │   ├── screens/
│       │   │   ├── __init__.py
│       │   │   ├── home.py          # Session list screen
│       │   │   └── chat.py          # Chat interface screen
│       │   └── widgets/
│       │       ├── __init__.py
│       │       ├── message.py       # Message display widget
│       │       └── tool_call.py     # Tool call display widget
│       │
│       │   ╔═══════════════════════════════════════════════════════════╗
│       │   ║                      API LAYER                            ║
│       │   ╚═══════════════════════════════════════════════════════════╝
│       │
│       ├── server/                  # FastAPI Server (~200 lines)
│       │   ├── __init__.py
│       │   ├── app.py               # FastAPI app + routes
│       │   ├── routes/
│       │   │   ├── __init__.py
│       │   │   ├── sessions.py      # Session CRUD endpoints
│       │   │   └── messages.py      # Message + SSE streaming
│       │   └── schemas.py           # Request/Response models
│       │
│       │   ╔═══════════════════════════════════════════════════════════╗
│       │   ║                     CORE LAYER                            ║
│       │   ╚═══════════════════════════════════════════════════════════╝
│       │
│       ├── agent.py                 # Agent loop (~180 lines)
│       │   │
│       │   ├── run_sync()           # Synchronous (for CLI)
│       │   └── run_async()          # Async generator (for server/TUI)
│       │
│       ├── provider.py              # LLM Provider (~80 lines)
│       │   │
│       │   ├── Provider             # Sync client
│       │   └── AsyncProvider        # Async client
│       │
│       ├── tools.py                 # Tool system (~250 lines)
│       │   │
│       │   ├── TOOLS[]              # Tool definitions
│       │   ├── execute()            # Sync execution
│       │   ├── execute_async()      # Async execution
│       │   └── ask_permission()     # Permission prompt
│       │
│       ├── models.py                # Data models (~60 lines)
│       │   │
│       │   ├── Message
│       │   ├── Session
│       │   ├── ToolCall
│       │   └── Event
│       │
│       ├── storage.py               # JSON storage (~80 lines)
│       │   │
│       │   ├── save_session()
│       │   ├── load_session()
│       │   └── list_sessions()
│       │
│       ├── config.py                # Configuration (~50 lines)
│       │   │
│       │   └── Config.load()
│       │
│       └── bus.py                   # Event bus (~60 lines)
│           │
│           ├── publish()
│           ├── subscribe()
│           └── subscribe_all()
│
└── tests/
    ├── test_agent.py
    ├── test_tools.py
    ├── test_server.py
    └── test_storage.py

Total: ~1,500 lines of Python
```

---

## API Specification

### Sessions

```
POST   /api/session                 Create new session
GET    /api/session                 List all sessions
GET    /api/session/{id}            Get session with messages
DELETE /api/session/{id}            Delete session
```

### Messages

```
POST   /api/session/{id}/message    Send message (returns SSE stream)

SSE Events:
  event: text
  data: {"content": "I'll help you..."}

  event: tool_call
  data: {"id": "tc_1", "name": "read_file", "input": {"path": "main.py"}}

  event: tool_result
  data: {"id": "tc_1", "output": "file contents..."}

  event: done
  data: {"session_id": "abc123"}

  event: error
  data: {"message": "Error description"}
```

### Tools & Events

```
GET    /api/tools                   List available tools
GET    /api/events                  Global SSE event stream
```

---

## Dependencies

```toml
[project]
name = "jetson"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    # Core
    "anthropic>=0.25.0",          # LLM API
    "pydantic>=2.0.0",            # Data models

    # CLI
    "typer>=0.9.0",               # CLI framework

    # Server
    "fastapi>=0.109.0",           # HTTP framework
    "uvicorn>=0.27.0",            # ASGI server
    "sse-starlette>=1.8.0",       # SSE support

    # TUI
    "textual>=0.47.0",            # Terminal UI
    "httpx>=0.26.0",              # Async HTTP client
]

[project.optional-dependencies]
dev = [
    "pytest>=7.0.0",
    "pytest-asyncio>=0.23.0",
    "ruff>=0.1.0",
]

[project.scripts]
jetson = "jetson.cli:app"
```

**8 runtime dependencies.**

---

## Tool Definitions

| Tool | Description | Permission |
|------|-------------|------------|
| `read_file` | Read file contents with line numbers | Auto |
| `write_file` | Create or overwrite file | **Ask** |
| `edit_file` | Find and replace text | **Ask** |
| `bash` | Execute shell command | **Ask** |
| `glob` | Find files by pattern | Auto |
| `grep` | Search file contents | Auto |

---

## Build Phases: Jetson v0

| Phase | Days | Deliverable | Lines |
|-------|------|-------------|-------|
| **1** | 1-3 | **Core Chat** | ~300 |
| | | CLI with basic chat | |
| | | Provider (Anthropic) | |
| | | Models (Session, Message) | |
| **2** | 4-7 | **Tool System** | ~550 |
| | | 6 tools with permissions | |
| | | Agent loop with tool execution | |
| | | Session storage (JSON) | |
| **3** | 8-11 | **FastAPI Server** | ~750 |
| | | REST endpoints | |
| | | SSE streaming | |
| | | Event bus | |
| **4** | 12-16 | **Textual TUI** | ~1,100 |
| | | Standalone mode (direct) | |
| | | Server mode (HTTP client) | |
| | | Session list + chat screens | |
| **5** | 17-19 | **Polish** | ~1,500 |
| | | Config file support | |
| | | Error handling | |
| | | Testing | |

**~3 weeks to complete Jetson v0.**

---

## Quick Start

```bash
# Install
pip install jetson
# or from source
git clone ... && cd jetson && pip install -e .

# Set API key
export ANTHROPIC_API_KEY="sk-..."

# CLI: One-shot task
jetson run "Create a hello world script"

# CLI: Interactive chat
jetson chat

# CLI: Resume session
jetson chat --session abc123

# TUI: Standalone mode (no server needed)
jetson tui

# Server: Start HTTP backend
jetson serve --host 0.0.0.0 --port 8080

# TUI: Connect to server
jetson ui --server http://localhost:8080
```

---

## Jetson v0 vs Full OpenCode

| Feature | Jetson v0 | OpenCode |
|---------|-----------|----------|
| LLM Providers | 1 (Anthropic) | 23+ |
| Tools | 6 core | 15+ with MCP |
| UI | CLI + Textual | CLI, TUI, Web, Desktop |
| Storage | JSON files | SQLite + cache |
| Plugins | None | Full system |
| MCP | None | Full integration |
| LSP | None | Full integration |
| OAuth | None | Multi-provider |
| Lines | ~1,500 | ~50,000+ |

---

# Roadmap: Jetson v0 → OpenCode

## Phase Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│   Phase 1          Phase 2          Phase 3          Phase 4                │
│   ┌──────┐        ┌──────┐         ┌──────┐        ┌──────┐                │
│   │Jetson│───────▶│Jetson│────────▶│Jetson│───────▶│Jetson│                │
│   │  v0  │        │  v1  │         │  v2  │        │  v3  │                │
│   └──────┘        └──────┘         └──────┘        └──────┘                │
│      │               │                │               │                     │
│      │               │                │               │                     │
│   • Core         • Multi-         • MCP           • Web UI                 │
│   • FastAPI        provider       • LSP           • Desktop                │
│   • TUI          • More tools     • Plugins       • OAuth                  │
│   • 6 tools      • SQLite                         • Enterprise             │
│                                                                              │
│   3 weeks        +2 weeks         +3 weeks        +4 weeks                 │
│                                                                              │
│   ~1,500 LOC     ~3,000 LOC       ~8,000 LOC      ~20,000 LOC             │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Phase 1: Jetson v0 (MVP) - 3 weeks

**Goal:** Working AI coding assistant with FastAPI backend

### Deliverables
- [x] CLI (run, chat, serve, tui, ui)
- [x] FastAPI server with SSE
- [x] Textual TUI (standalone + server mode)
- [x] 6 core tools
- [x] JSON storage
- [x] Basic config

### Architecture
```
CLI ──────────┐
              ├──▶ Core Layer ──▶ Anthropic
TUI Standalone┘
                        ▲
                        │
TUI Server ───▶ FastAPI ┘
```

---

## Phase 2: Jetson v1 (Enhanced) - +2 weeks

**Goal:** Multi-provider support, better storage, more tools

### New Features
- [ ] OpenAI provider
- [ ] Local models (Ollama)
- [ ] SQLite storage (replace JSON)
- [ ] Additional tools:
  - [ ] `web_fetch` - Fetch web pages
  - [ ] `tree` - Directory structure
  - [ ] `multi_edit` - Multiple edits at once
- [ ] Session compaction
- [ ] Better error handling

### Architecture Changes
```python
# provider.py - Add provider factory
class ProviderFactory:
    @staticmethod
    def create(provider_type: str) -> BaseProvider:
        if provider_type == "anthropic":
            return AnthropicProvider()
        elif provider_type == "openai":
            return OpenAIProvider()
        elif provider_type == "ollama":
            return OllamaProvider()

# storage.py - Replace JSON with SQLite
class SQLiteStorage:
    def save_session(self, session: Session): ...
    def load_session(self, id: str) -> Session: ...
    def search_sessions(self, query: str) -> list[Session]: ...
```

---

## Phase 3: Jetson v2 (Professional) - +3 weeks

**Goal:** MCP, LSP, and plugin system

### New Features
- [ ] **MCP Integration**
  - [ ] MCP client for external tools
  - [ ] MCP server for exposing Jetson tools
  - [ ] Tool discovery
- [ ] **LSP Integration**
  - [ ] Diagnostics after edits
  - [ ] Go-to-definition
  - [ ] Auto-imports
- [ ] **Plugin System**
  - [ ] Plugin discovery
  - [ ] Custom tool plugins
  - [ ] Provider plugins

### Architecture Changes
```
┌─────────────────────────────────────────────────────────────────────┐
│                         JETSON v2                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐              │
│   │   CLI   │  │   TUI   │  │ FastAPI │  │   MCP   │              │
│   └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘              │
│        │            │            │            │                     │
│        └────────────┴────────────┴────────────┘                    │
│                          │                                          │
│                          ▼                                          │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │                    CORE LAYER                                │  │
│   │                                                              │  │
│   │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐       │  │
│   │  │  Agent  │  │Provider │  │  Tools  │  │ Plugin  │       │  │
│   │  │         │  │ Factory │  │Registry │  │ Manager │       │  │
│   │  └─────────┘  └─────────┘  └─────────┘  └─────────┘       │  │
│   │                                              │              │  │
│   │                                              ▼              │  │
│   │                                    ┌─────────────────┐     │  │
│   │                                    │  MCP Clients    │     │  │
│   │                                    │  ┌───┐ ┌───┐   │     │  │
│   │                                    │  │FS │ │Git│   │     │  │
│   │                                    │  └───┘ └───┘   │     │  │
│   │                                    └─────────────────┘     │  │
│   │                                                              │  │
│   └──────────────────────────────────────────────────────────────┘  │
│                                                                      │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │                      LSP LAYER                               │  │
│   │                                                              │  │
│   │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │  │
│   │  │  LSP Client │  │ Diagnostics │  │ Code Actions│         │  │
│   │  └─────────────┘  └─────────────┘  └─────────────┘         │  │
│   │                                                              │  │
│   └─────────────────────────────────────────────────────────────┘  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Phase 4: Jetson v3 (Enterprise) - +4 weeks

**Goal:** Web UI, desktop app, enterprise features

### New Features
- [ ] **Web UI**
  - [ ] React/SolidJS frontend
  - [ ] Real-time updates via SSE
  - [ ] File browser
  - [ ] Diff viewer
- [ ] **Desktop App**
  - [ ] Tauri wrapper
  - [ ] System tray
  - [ ] Global hotkey
- [ ] **Enterprise**
  - [ ] OAuth (GitHub, Google)
  - [ ] Multi-user sessions
  - [ ] Audit logging
  - [ ] Rate limiting
  - [ ] Usage tracking

### Architecture Changes
```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              JETSON v3 / OPENCODE                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   CLIENTS                                                                    │
│   ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐         │
│   │   CLI   │  │   TUI   │  │  Web UI │  │ Desktop │  │   MCP   │         │
│   │ (Typer) │  │(Textual)│  │ (React) │  │ (Tauri) │  │ Clients │         │
│   └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘         │
│        │            │            │            │            │               │
│        └────────────┴────────────┴────────────┴────────────┘               │
│                                    │                                        │
│   ┌────────────────────────────────┼───────────────────────────────────┐   │
│   │                          API LAYER                                  │   │
│   │                                │                                    │   │
│   │   ┌────────────────────────────┼────────────────────────────────┐  │   │
│   │   │               FastAPI Server                                 │  │   │
│   │   │                            │                                 │  │   │
│   │   │   ┌────────────┐  ┌───────┴───────┐  ┌────────────┐       │  │   │
│   │   │   │   Auth     │  │   Sessions    │  │   Events   │       │  │   │
│   │   │   │  (OAuth)   │  │   Messages    │  │    (SSE)   │       │  │   │
│   │   │   └────────────┘  └───────────────┘  └────────────┘       │  │   │
│   │   │                                                             │  │   │
│   │   └─────────────────────────────────────────────────────────────┘  │   │
│   │                                                                      │   │
│   └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                          CORE LAYER                                  │   │
│   │                                                                      │   │
│   │   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐          │   │
│   │   │  Agent   │  │ Provider │  │   Tool   │  │  Plugin  │          │   │
│   │   │  Engine  │  │  Factory │  │ Registry │  │  Manager │          │   │
│   │   └──────────┘  └──────────┘  └──────────┘  └──────────┘          │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                     │
│   │  MCP Layer   │  │  LSP Layer   │  │   Storage    │                     │
│   │              │  │              │  │   (SQLite)   │                     │
│   └──────────────┘  └──────────────┘  └──────────────┘                     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Timeline Summary

| Phase | Version | Duration | Cumulative | Key Features |
|-------|---------|----------|------------|--------------|
| 1 | v0 | 3 weeks | 3 weeks | Core + FastAPI + TUI |
| 2 | v1 | 2 weeks | 5 weeks | Multi-provider + SQLite |
| 3 | v2 | 3 weeks | 8 weeks | MCP + LSP + Plugins |
| 4 | v3 | 4 weeks | 12 weeks | Web + Desktop + Enterprise |

**3 months from Jetson v0 to full OpenCode equivalent.**

---

## Summary

### Jetson v0 (This Document)

```
┌─────────────────────────────────────────┐
│              JETSON v0                   │
├─────────────────────────────────────────┤
│                                          │
│  CLI ─────┐                             │
│           ├──▶ Core ──▶ Anthropic       │
│  TUI ─────┤      ▲                      │
│  (direct) │      │                      │
│           │      │                      │
│  TUI ─────┼──▶ FastAPI                  │
│  (server) │                             │
│                                          │
│  • 1,500 lines                          │
│  • 8 dependencies                        │
│  • 3 weeks                               │
│                                          │
└─────────────────────────────────────────┘
```

### Commands

```bash
jetson run "task"              # One-shot
jetson chat                    # Interactive CLI
jetson tui                     # Standalone TUI
jetson serve                   # Start server
jetson ui --server URL         # TUI via server
```

This is the foundation. Everything else builds on top.
