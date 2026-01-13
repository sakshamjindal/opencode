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

## Self-Bootstrapping: Jetson Builds Itself

### The Key Insight

Once Jetson v0 exists, **it can build all future versions of itself**.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│   "Can an AI coding agent improve itself?"                                  │
│                                                                              │
│   YES. Jetson v0 has everything it needs:                                   │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                                                                      │   │
│   │   read_file   →  Understand existing code                           │   │
│   │   write_file  →  Create new modules                                 │   │
│   │   edit_file   →  Modify existing code                               │   │
│   │   bash        →  Run tests, git commit, pip install                 │   │
│   │   glob/grep   →  Navigate codebase                                  │   │
│   │                                                                      │   │
│   │   + Claude    →  Architectural understanding                        │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### The Bootstrapping Chain

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│   PHASE 0: Human + Claude Code                                              │
│   ─────────────────────────────                                             │
│                │                                                             │
│                │ builds from scratch                                        │
│                ▼                                                             │
│   ┌────────────────────────┐                                                │
│   │       JETSON v0        │  The Bootstrap                                 │
│   │                        │                                                │
│   │  • Core agent loop     │                                                │
│   │  • 6 tools             │                                                │
│   │  • FastAPI server      │                                                │
│   │  • Textual TUI         │                                                │
│   │  • ~1,500 lines        │                                                │
│   └───────────┬────────────┘                                                │
│               │                                                              │
│               │ User: "Add OpenAI provider support"                         │
│               │                                                              │
│               │ Jetson v0:                                                  │
│               │   1. read_file("src/jetson/provider.py")                    │
│               │   2. write_file("src/jetson/providers/openai.py")           │
│               │   3. edit_file("src/jetson/providers/__init__.py")          │
│               │   4. bash("pytest tests/")                                  │
│               │   5. bash("git add . && git commit -m 'Add OpenAI'")        │
│               │                                                              │
│               ▼                                                              │
│   ┌────────────────────────┐                                                │
│   │       JETSON v1        │  Self-improved                                 │
│   │                        │                                                │
│   │  • Multi-provider      │                                                │
│   │  • SQLite storage      │                                                │
│   │  • More tools          │                                                │
│   │  • ~3,000 lines        │                                                │
│   └───────────┬────────────┘                                                │
│               │                                                              │
│               │ User: "Add MCP protocol support"                            │
│               │                                                              │
│               │ Jetson v1:                                                  │
│               │   1. glob("**/*.py") - understand structure                 │
│               │   2. grep("tool") - find tool system                        │
│               │   3. write_file("src/jetson/mcp/client.py")                 │
│               │   4. write_file("src/jetson/mcp/server.py")                 │
│               │   5. edit_file("src/jetson/agent.py")                       │
│               │   6. bash("pytest && git commit")                           │
│               │                                                              │
│               ▼                                                              │
│   ┌────────────────────────┐                                                │
│   │       JETSON v2        │  Self-improved again                           │
│   │                        │                                                │
│   │  • MCP integration     │                                                │
│   │  • LSP integration     │                                                │
│   │  • Plugin system       │                                                │
│   │  • ~8,000 lines        │                                                │
│   └───────────┬────────────┘                                                │
│               │                                                              │
│               │ User: "Add web UI and desktop app"                          │
│               │                                                              │
│               ▼                                                              │
│   ┌────────────────────────┐                                                │
│   │       JETSON v3        │  ≈ OpenCode                                    │
│   │                        │                                                │
│   │  • Web UI (React)      │                                                │
│   │  • Desktop (Tauri)     │                                                │
│   │  • OAuth               │                                                │
│   │  • ~20,000 lines       │                                                │
│   └────────────────────────┘                                                │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### What Each Version Can Build

| Version | Can Build | Why |
|---------|-----------|-----|
| **v0** | v1, v2, v3... | Has all core tools (read/write/edit/bash) |
| **v1** | v2, v3... faster | More providers = faster responses |
| **v2** | v3... with MCP tools | External tools via MCP |
| **v3** | Everything | Full OpenCode equivalent |

### Example: Jetson v0 Adds SQLite Support

```
User: "Replace JSON storage with SQLite for better performance"

Jetson v0 executes:

┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│  1. UNDERSTAND                                                               │
│     read_file("src/jetson/storage.py")                                      │
│     → Sees: save_session(), load_session(), list_sessions()                 │
│     → Sees: JSON file format, ~/.jetson/sessions/                           │
│                                                                              │
│  2. PLAN                                                                     │
│     "I need to:                                                             │
│      - Create SQLite schema for sessions/messages                           │
│      - Rewrite storage functions to use sqlite3                             │
│      - Add migration from JSON to SQLite                                    │
│      - Update dependencies"                                                 │
│                                                                              │
│  3. IMPLEMENT                                                                │
│     edit_file("pyproject.toml")                                             │
│       + "aiosqlite>=0.19.0"                                                 │
│                                                                              │
│     write_file("src/jetson/storage_sqlite.py")                              │
│       + SQLite implementation                                               │
│                                                                              │
│     write_file("src/jetson/migrations/001_json_to_sqlite.py")               │
│       + Migration script                                                    │
│                                                                              │
│     edit_file("src/jetson/storage.py")                                      │
│       + Import and use SQLite backend                                       │
│                                                                              │
│  4. TEST                                                                     │
│     bash("pip install -e .")                                                │
│     bash("pytest tests/test_storage.py -v")                                 │
│                                                                              │
│  5. COMMIT                                                                   │
│     bash("git add . && git commit -m 'feat: SQLite storage backend'")       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘

Result: Jetson v0 has upgraded itself to have SQLite storage.
```

### Implications

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│   1. MINIMAL BOOTSTRAP REQUIRED                                             │
│      Only need to build v0 manually. Everything after is self-service.     │
│                                                                              │
│   2. ACCELERATING RETURNS                                                   │
│      v0 builds v1 slowly (limited tools)                                    │
│      v1 builds v2 faster (more providers, better tools)                     │
│      v2 builds v3 fastest (MCP external tools, LSP validation)             │
│                                                                              │
│   3. HUMAN STAYS IN CONTROL                                                 │
│      User must approve each change (permission system)                      │
│      User provides direction ("add MCP support")                            │
│      User reviews code before commits                                       │
│                                                                              │
│   4. RECURSIVE IMPROVEMENT                                                  │
│      Jetson can fix its own bugs                                           │
│      Jetson can add its own features                                       │
│      Jetson can refactor its own code                                       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### The Minimal Viable Bootstrap

This is why **Jetson v0 must be built correctly**:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│   CRITICAL v0 COMPONENTS (Cannot self-bootstrap)                            │
│   ──────────────────────────────────────────────                            │
│                                                                              │
│   ✓ Agent loop        - The brain that orchestrates                        │
│   ✓ read_file         - Must exist to read its own code                    │
│   ✓ write_file        - Must exist to create new code                      │
│   ✓ edit_file         - Must exist to modify code                          │
│   ✓ bash              - Must exist to run tests/git                        │
│   ✓ Provider          - Must exist to call Claude                          │
│   ✓ Permission system - Must exist for safety                              │
│                                                                              │
│   These 7 components are the "genesis" - everything else can be            │
│   built by Jetson itself.                                                   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

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

**Built by:** Jetson v0 (with human guidance)

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

### File Changes (Built by Jetson v0)

```
src/jetson/
├── providers/                    # NEW DIRECTORY
│   ├── __init__.py              # Provider factory
│   ├── base.py                  # BaseProvider ABC
│   ├── anthropic.py             # Moved from provider.py
│   ├── openai.py                # NEW: OpenAI support
│   └── ollama.py                # NEW: Local models
│
├── storage/                      # REFACTORED
│   ├── __init__.py
│   ├── base.py                  # Storage interface
│   ├── json_storage.py          # Original JSON (kept for compat)
│   └── sqlite_storage.py        # NEW: SQLite backend
│
├── tools/                        # EXPANDED
│   ├── ...existing...
│   ├── web_fetch.py             # NEW
│   ├── tree.py                  # NEW
│   └── multi_edit.py            # NEW
│
└── provider.py                   # DEPRECATED → providers/
```

### How Jetson v0 Builds This

```
User: "Add OpenAI provider support"

Jetson v0:
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  Step 1: Understand current provider                                       │
│  ─────────────────────────────────────                                     │
│  > read_file("src/jetson/provider.py")                                     │
│                                                                             │
│  Step 2: Create provider abstraction                                       │
│  ──────────────────────────────────────                                    │
│  > write_file("src/jetson/providers/base.py")                              │
│    ```python                                                               │
│    from abc import ABC, abstractmethod                                     │
│                                                                             │
│    class BaseProvider(ABC):                                                │
│        @abstractmethod                                                     │
│        async def stream(self, messages, tools): ...                        │
│    ```                                                                     │
│                                                                             │
│  Step 3: Move Anthropic to new structure                                   │
│  ─────────────────────────────────────────                                 │
│  > write_file("src/jetson/providers/anthropic.py")                         │
│    (refactored from provider.py)                                           │
│                                                                             │
│  Step 4: Add OpenAI provider                                               │
│  ─────────────────────────────                                             │
│  > write_file("src/jetson/providers/openai.py")                            │
│    ```python                                                               │
│    from openai import AsyncOpenAI                                          │
│    from .base import BaseProvider                                          │
│                                                                             │
│    class OpenAIProvider(BaseProvider):                                     │
│        async def stream(self, messages, tools):                            │
│            # OpenAI-specific implementation                                │
│    ```                                                                     │
│                                                                             │
│  Step 5: Create factory                                                    │
│  ─────────────────────────                                                 │
│  > write_file("src/jetson/providers/__init__.py")                          │
│    ```python                                                               │
│    def create_provider(name: str) -> BaseProvider:                         │
│        if name == "anthropic": return AnthropicProvider()                  │
│        if name == "openai": return OpenAIProvider()                        │
│    ```                                                                     │
│                                                                             │
│  Step 6: Update dependencies                                               │
│  ───────────────────────────                                               │
│  > edit_file("pyproject.toml")                                             │
│    + "openai>=1.0.0"                                                       │
│                                                                             │
│  Step 7: Update agent to use factory                                       │
│  ──────────────────────────────────                                        │
│  > edit_file("src/jetson/agent.py")                                        │
│    - from .provider import Provider                                        │
│    + from .providers import create_provider                                │
│                                                                             │
│  Step 8: Test                                                              │
│  ───────────                                                               │
│  > bash("pip install -e . && pytest")                                      │
│                                                                             │
│  Step 9: Commit                                                            │
│  ─────────────                                                             │
│  > bash("git add . && git commit -m 'feat: multi-provider support'")       │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### New Dependencies
```diff
  dependencies = [
      "anthropic>=0.25.0",
+     "openai>=1.0.0",
+     "ollama>=0.1.0",
+     "aiosqlite>=0.19.0",
+     "httpx>=0.26.0",        # For web_fetch
+     "beautifulsoup4>=4.12.0",
      ...
  ]
```

---

## Phase 3: Jetson v2 (Professional) - +3 weeks

**Goal:** MCP, LSP, and plugin system

**Built by:** Jetson v1 (with human guidance)

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

### File Changes (Built by Jetson v1)

```
src/jetson/
├── mcp/                          # NEW: Model Context Protocol
│   ├── __init__.py
│   ├── client.py                # Connect to MCP servers
│   ├── server.py                # Expose Jetson as MCP server
│   ├── transport.py             # stdio/SSE transport
│   └── discovery.py             # Find available MCP servers
│
├── lsp/                          # NEW: Language Server Protocol
│   ├── __init__.py
│   ├── client.py                # LSP client manager
│   ├── diagnostics.py           # Get errors/warnings after edits
│   └── actions.py               # Code actions (auto-import, etc.)
│
├── plugins/                      # NEW: Plugin System
│   ├── __init__.py
│   ├── manager.py               # Load/unload plugins
│   ├── base.py                  # Plugin base class
│   └── discovery.py             # Find plugins in ~/.jetson/plugins/
│
├── tools/
│   ├── ...existing...
│   └── registry.py              # REFACTORED: Dynamic tool registry
│
└── agent.py                      # UPDATED: Use tool registry + LSP
```

### How Jetson v1 Builds MCP Support

```
User: "Add MCP client support so I can use external tool servers"

Jetson v1:
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  Step 1: Research MCP protocol                                             │
│  ─────────────────────────────────                                         │
│  > web_fetch("https://modelcontextprotocol.io/docs")                       │
│  > read_file("docs/architecture/03-INTEGRATIONS.md")                       │
│                                                                             │
│  Step 2: Create MCP client                                                 │
│  ───────────────────────────                                               │
│  > write_file("src/jetson/mcp/__init__.py")                                │
│  > write_file("src/jetson/mcp/client.py")                                  │
│    ```python                                                               │
│    import asyncio                                                          │
│    import json                                                             │
│                                                                             │
│    class MCPClient:                                                        │
│        def __init__(self, command: list[str]):                             │
│            self.process = None                                             │
│                                                                             │
│        async def connect(self):                                            │
│            self.process = await asyncio.create_subprocess_exec(            │
│                *self.command,                                              │
│                stdin=asyncio.subprocess.PIPE,                              │
│                stdout=asyncio.subprocess.PIPE                              │
│            )                                                               │
│            await self._initialize()                                        │
│                                                                             │
│        async def list_tools(self) -> list[dict]:                           │
│            response = await self._request("tools/list", {})                │
│            return response["tools"]                                        │
│                                                                             │
│        async def call_tool(self, name: str, args: dict):                   │
│            return await self._request("tools/call", {                      │
│                "name": name,                                               │
│                "arguments": args                                           │
│            })                                                              │
│    ```                                                                     │
│                                                                             │
│  Step 3: Create tool registry that includes MCP tools                     │
│  ─────────────────────────────────────────────────────                     │
│  > write_file("src/jetson/tools/registry.py")                              │
│    ```python                                                               │
│    class ToolRegistry:                                                     │
│        def __init__(self):                                                 │
│            self.builtin_tools = [...]                                      │
│            self.mcp_tools = []                                             │
│                                                                             │
│        async def discover_mcp_tools(self):                                 │
│            for client in self.mcp_clients:                                 │
│                tools = await client.list_tools()                           │
│                self.mcp_tools.extend(tools)                                │
│                                                                             │
│        def get_all_tools(self):                                            │
│            return self.builtin_tools + self.mcp_tools                      │
│    ```                                                                     │
│                                                                             │
│  Step 4: Update agent to use registry                                     │
│  ──────────────────────────────────────                                    │
│  > edit_file("src/jetson/agent.py")                                        │
│    - tools=TOOLS                                                           │
│    + tools=registry.get_all_tools()                                        │
│                                                                             │
│  Step 5: Add MCP config                                                    │
│  ────────────────────────                                                  │
│  > edit_file("src/jetson/config.py")                                       │
│    + mcp_servers: list[MCPServerConfig] = []                               │
│                                                                             │
│  Step 6: Test with filesystem MCP server                                   │
│  ─────────────────────────────────────────                                 │
│  > bash("pip install mcp-server-filesystem")                               │
│  > bash("pytest tests/test_mcp.py")                                        │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Architecture Diagram

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
│   │  └─────────┘  └─────────┘  └────┬────┘  └─────────┘       │  │
│   │                                  │                          │  │
│   │                    ┌─────────────┼─────────────┐           │  │
│   │                    ▼             ▼             ▼           │  │
│   │              ┌──────────┐ ┌──────────┐ ┌──────────┐       │  │
│   │              │ Built-in │ │   MCP    │ │  Plugin  │       │  │
│   │              │  Tools   │ │  Tools   │ │  Tools   │       │  │
│   │              └──────────┘ └──────────┘ └──────────┘       │  │
│   │                                                              │  │
│   └──────────────────────────────────────────────────────────────┘  │
│                                                                      │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │                      LSP LAYER                               │  │
│   │                                                              │  │
│   │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │  │
│   │  │  LSP Client │  │ Diagnostics │  │ Code Actions│         │  │
│   │  │  (pyright)  │  │ (on edit)   │  │(auto-import)│         │  │
│   │  └─────────────┘  └─────────────┘  └─────────────┘         │  │
│   │                                                              │  │
│   └─────────────────────────────────────────────────────────────┘  │
│                                                                      │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │                     MCP CONNECTIONS                          │  │
│   │                                                              │  │
│   │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐       │  │
│   │  │Filesys- │  │  Git    │  │ Docker  │  │ Custom  │       │  │
│   │  │  tem    │  │ Server  │  │ Server  │  │ Server  │       │  │
│   │  └─────────┘  └─────────┘  └─────────┘  └─────────┘       │  │
│   │                                                              │  │
│   └─────────────────────────────────────────────────────────────┘  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### New Dependencies
```diff
  dependencies = [
      ...existing v1 deps...
+     "pygls>=1.2.0",           # LSP client
+     "mcp>=0.1.0",             # MCP protocol
  ]
```

---

## Phase 4: Jetson v3 (Enterprise) - +4 weeks

**Goal:** Web UI, desktop app, enterprise features

**Built by:** Jetson v2 (with human guidance)

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

### File Changes (Built by Jetson v2)

```
jetson/
├── src/jetson/                   # Python backend (existing)
│   ├── ...all v2 code...
│   ├── server/
│   │   ├── ...existing...
│   │   ├── auth/                 # NEW: Authentication
│   │   │   ├── __init__.py
│   │   │   ├── oauth.py         # GitHub, Google OAuth
│   │   │   ├── jwt.py           # JWT token handling
│   │   │   └── middleware.py    # Auth middleware
│   │   └── audit.py             # NEW: Audit logging
│   │
│   └── enterprise/               # NEW: Enterprise features
│       ├── __init__.py
│       ├── rate_limit.py        # Rate limiting
│       ├── usage.py             # Usage tracking
│       └── billing.py           # Billing integration
│
├── web/                          # NEW: Web Frontend
│   ├── package.json
│   ├── src/
│   │   ├── App.tsx
│   │   ├── components/
│   │   │   ├── Chat.tsx
│   │   │   ├── SessionList.tsx
│   │   │   ├── MessageBubble.tsx
│   │   │   ├── ToolCallView.tsx
│   │   │   └── FileBrowser.tsx
│   │   ├── hooks/
│   │   │   ├── useSSE.ts        # SSE subscription
│   │   │   └── useSession.ts
│   │   └── api/
│   │       └── client.ts        # API client
│   └── public/
│
└── desktop/                      # NEW: Desktop App
    ├── src-tauri/
    │   ├── Cargo.toml
    │   ├── src/
    │   │   ├── main.rs
    │   │   ├── tray.rs          # System tray
    │   │   └── hotkey.rs        # Global hotkey
    │   └── tauri.conf.json
    └── src/                      # Shared with web/
```

### How Jetson v2 Builds Web UI

```
User: "Add a web UI so I can use Jetson from a browser"

Jetson v2:
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  Step 1: Create React project                                              │
│  ─────────────────────────────                                             │
│  > bash("cd web && npm create vite@latest . -- --template react-ts")       │
│  > bash("npm install")                                                     │
│                                                                             │
│  Step 2: Create SSE hook for real-time updates                            │
│  ───────────────────────────────────────────────                           │
│  > write_file("web/src/hooks/useSSE.ts")                                   │
│    ```typescript                                                           │
│    export function useSSE(url: string) {                                   │
│      const [events, setEvents] = useState<Event[]>([]);                    │
│                                                                             │
│      useEffect(() => {                                                     │
│        const source = new EventSource(url);                                │
│                                                                             │
│        source.addEventListener('text', (e) => {                            │
│          setEvents(prev => [...prev, JSON.parse(e.data)]);                 │
│        });                                                                 │
│                                                                             │
│        source.addEventListener('tool_call', (e) => { ... });               │
│                                                                             │
│        return () => source.close();                                        │
│      }, [url]);                                                            │
│                                                                             │
│      return events;                                                        │
│    }                                                                       │
│    ```                                                                     │
│                                                                             │
│  Step 3: Create Chat component                                             │
│  ─────────────────────────────                                             │
│  > write_file("web/src/components/Chat.tsx")                               │
│    ```typescript                                                           │
│    export function Chat({ sessionId }: { sessionId: string }) {            │
│      const [input, setInput] = useState('');                               │
│      const messages = useSSE(`/api/session/${sessionId}/events`);          │
│                                                                             │
│      const sendMessage = async () => {                                     │
│        await fetch(`/api/session/${sessionId}/message`, {                  │
│          method: 'POST',                                                   │
│          body: JSON.stringify({ content: input })                          │
│        });                                                                 │
│        setInput('');                                                       │
│      };                                                                    │
│                                                                             │
│      return (                                                              │
│        <div className="chat">                                              │
│          <MessageList messages={messages} />                               │
│          <input value={input} onChange={e => setInput(e.target.value)} /> │
│          <button onClick={sendMessage}>Send</button>                       │
│        </div>                                                              │
│      );                                                                    │
│    }                                                                       │
│    ```                                                                     │
│                                                                             │
│  Step 4: Update FastAPI to serve static files                             │
│  ──────────────────────────────────────────────                            │
│  > edit_file("src/jetson/server/app.py")                                   │
│    + from fastapi.staticfiles import StaticFiles                           │
│    + app.mount("/", StaticFiles(directory="web/dist"), name="static")     │
│                                                                             │
│  Step 5: Build and test                                                    │
│  ─────────────────────                                                     │
│  > bash("cd web && npm run build")                                         │
│  > bash("jetson serve")                                                    │
│  # Open http://localhost:8080                                              │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Architecture Diagram

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
│   │   │   ┌────────────┐  ┌───────────────┐  ┌────────────┐       │  │   │
│   │   │   │ Rate Limit │  │    Audit      │  │   Usage    │       │  │   │
│   │   │   │            │  │    Logging    │  │  Tracking  │       │  │   │
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

### New Technologies
```
Web UI:
  - React 18 / SolidJS
  - TypeScript
  - Tailwind CSS
  - Vite

Desktop:
  - Tauri 2.0
  - Rust (for native features)

Enterprise:
  - authlib (OAuth)
  - PyJWT
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
