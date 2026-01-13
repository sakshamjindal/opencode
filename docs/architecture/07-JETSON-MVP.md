# Jetson - Minimal AI Coding Agent

## What is Jetson?

**Jetson** is a minimal, from-scratch Python implementation inspired by OpenCode. It strips away all enterprise complexity to deliver a working AI coding assistant in ~1,000 lines of code.

---

## Jetson vs OpenCode

| Aspect | OpenCode (Full) | Jetson (MVP) |
|--------|-----------------|--------------|
| **Language** | TypeScript/Bun | Python |
| **LLM Providers** | 23+ (via Vercel AI SDK) | 1 (Anthropic only) |
| **Tools** | 15+ with MCP extensibility | 6 core tools |
| **UI Options** | CLI, TUI, Web, Desktop | CLI + Textual TUI |
| **Storage** | SQLite + file cache | JSON files only |
| **State Management** | Event-sourced, reactive | Simple in-memory + save |
| **Plugin System** | Full plugin architecture | None |
| **MCP/LSP** | Full integration | None |
| **OAuth/Auth** | Multi-provider OAuth | API key only |
| **Codebase Size** | ~50,000+ lines | ~1,000 lines |
| **Build Time** | Months | 2-3 weeks |

---

## System Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│                              JETSON                                         │
│                        AI Coding Agent MVP                                  │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        INTERFACE LAYER                               │   │
│  │                                                                      │   │
│  │    ┌──────────────────┐         ┌──────────────────┐               │   │
│  │    │       CLI        │         │   Textual TUI    │               │   │
│  │    │                  │         │                  │               │   │
│  │    │  • typer         │         │  • Async widgets │               │   │
│  │    │  • One-shot mode │         │  • Session list  │               │   │
│  │    │  • Interactive   │         │  • Chat screen   │               │   │
│  │    │    chat mode     │         │  • Streaming     │               │   │
│  │    │                  │         │  • Tool display  │               │   │
│  │    └────────┬─────────┘         └────────┬─────────┘               │   │
│  │             │                            │                          │   │
│  └─────────────┼────────────────────────────┼──────────────────────────┘   │
│                │                            │                               │
│                └──────────────┬─────────────┘                              │
│                               │                                             │
│                               ▼                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         CORE LAYER                                   │   │
│  │                                                                      │   │
│  │  ┌─────────────────────────────────────────────────────────────┐   │   │
│  │  │                      AGENT LOOP                              │   │   │
│  │  │                                                              │   │   │
│  │  │   ┌─────────┐    ┌─────────┐    ┌─────────┐               │   │   │
│  │  │   │ Receive │───▶│  Send   │───▶│ Process │               │   │   │
│  │  │   │ User    │    │ to LLM  │    │Response │               │   │   │
│  │  │   │ Input   │    │         │    │         │               │   │   │
│  │  │   └─────────┘    └─────────┘    └────┬────┘               │   │   │
│  │  │                                      │                     │   │   │
│  │  │                    ┌─────────────────┼─────────────────┐  │   │   │
│  │  │                    │                 │                 │  │   │   │
│  │  │                    ▼                 ▼                 │  │   │   │
│  │  │              ┌──────────┐     ┌──────────┐            │  │   │   │
│  │  │              │   Text   │     │Tool Call │            │  │   │   │
│  │  │              │ Response │     │ Request  │            │  │   │   │
│  │  │              └────┬─────┘     └────┬─────┘            │  │   │   │
│  │  │                   │                │                   │  │   │   │
│  │  │                   │                ▼                   │  │   │   │
│  │  │                   │         ┌──────────┐              │  │   │   │
│  │  │                   │         │ Execute  │──┐           │  │   │   │
│  │  │                   │         │   Tool   │  │           │  │   │   │
│  │  │                   │         └──────────┘  │           │  │   │   │
│  │  │                   │                │      │           │  │   │   │
│  │  │                   │                ▼      │ Loop      │  │   │   │
│  │  │                   │         ┌──────────┐  │ until     │  │   │   │
│  │  │                   │         │  Return  │  │ no more   │  │   │   │
│  │  │                   │         │  Result  │──┘ tools     │  │   │   │
│  │  │                   │         └──────────┘              │  │   │   │
│  │  │                   │                                    │  │   │   │
│  │  │                   └────────────────────────────────────┘  │   │   │
│  │  │                                                           │   │   │
│  │  └───────────────────────────────────────────────────────────┘   │   │
│  │                                                                      │   │
│  │  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐        │   │
│  │  │    PROVIDER    │  │    SESSION     │  │     TOOLS      │        │   │
│  │  │                │  │                │  │                │        │   │
│  │  │ • Anthropic    │  │ • Message list │  │ • read_file    │        │   │
│  │  │   SDK          │  │ • Timestamps   │  │ • write_file   │        │   │
│  │  │ • Streaming    │  │ • Session ID   │  │ • edit_file    │        │   │
│  │  │ • Tool calling │  │                │  │ • bash         │        │   │
│  │  │                │  │                │  │ • glob         │        │   │
│  │  │                │  │                │  │ • grep         │        │   │
│  │  └────────────────┘  └────────────────┘  └────────────────┘        │   │
│  │                                                                      │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                       STORAGE LAYER                                  │   │
│  │                                                                      │   │
│  │     ~/.jetson/                                                      │   │
│  │     ├── config.json          # User preferences                     │   │
│  │     └── sessions/                                                   │   │
│  │         ├── abc123.json      # Session with messages                │   │
│  │         └── def456.json                                             │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           USER INTERACTION                               │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  INPUT: "Create a Python script that reads a CSV file"                  │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                            CLI / TUI                                     │
│  1. Capture user input                                                   │
│  2. Add to session messages                                              │
│  3. Call agent.run(session, message)                                     │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           AGENT LOOP                                     │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │  STEP 1: Format messages for API                                   │ │
│  │                                                                    │ │
│  │  messages = [                                                      │ │
│  │    {"role": "user", "content": "Create a Python script..."}       │ │
│  │  ]                                                                 │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                    │                                     │
│                                    ▼                                     │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │  STEP 2: Call Anthropic API with tools                             │ │
│  │                                                                    │ │
│  │  response = client.messages.create(                                │ │
│  │      model="claude-sonnet-4-20250514",                             │ │
│  │      messages=messages,                                            │ │
│  │      tools=TOOL_DEFINITIONS                                        │ │
│  │  )                                                                 │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                    │                                     │
│                                    ▼                                     │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │  STEP 3: Process response blocks                                   │ │
│  │                                                                    │ │
│  │  for block in response.content:                                    │ │
│  │      if block.type == "text":                                      │ │
│  │          → Stream to UI                                            │ │
│  │      elif block.type == "tool_use":                                │ │
│  │          → Execute tool, collect result                            │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                    │                                     │
│                        ┌───────────┴───────────┐                        │
│                        ▼                       ▼                        │
│  ┌─────────────────────────────┐  ┌─────────────────────────────────┐  │
│  │  TEXT RESPONSE              │  │  TOOL CALL                      │  │
│  │                             │  │                                 │  │
│  │  "I'll create a script..."  │  │  name: "write_file"             │  │
│  │                             │  │  input: {                       │  │
│  │  → Display to user          │  │    path: "read_csv.py",         │  │
│  │  → Save to session          │  │    content: "import csv..."     │  │
│  │                             │  │  }                              │  │
│  └─────────────────────────────┘  └───────────────┬─────────────────┘  │
│                                                   │                     │
│                                                   ▼                     │
│                                   ┌─────────────────────────────────┐  │
│                                   │  TOOL EXECUTION                 │  │
│                                   │                                 │  │
│                                   │  1. Check permissions           │  │
│                                   │  2. Execute: write file         │  │
│                                   │  3. Return result               │  │
│                                   └───────────────┬─────────────────┘  │
│                                                   │                     │
│                                                   ▼                     │
│                                   ┌─────────────────────────────────┐  │
│                                   │  TOOL RESULT                    │  │
│                                   │                                 │  │
│                                   │  "Successfully wrote to         │  │
│                                   │   read_csv.py"                  │  │
│                                   │                                 │  │
│                                   │  → Add to messages              │  │
│                                   │  → Loop back to STEP 2          │  │
│                                   └─────────────────────────────────┘  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  LOOP CONTINUES until response has no tool_use blocks                   │
│  (stop_reason == "end_turn")                                            │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                            SAVE SESSION                                  │
│                                                                          │
│  ~/.jetson/sessions/abc123.json                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Component Details

### 1. Interface Layer

**Why CLI + Textual TUI (not both Rich TUI and Textual)?**

In the original plan, we had:
- **Rich TUI** - Simple synchronous UI using Rich + prompt-toolkit
- **Textual TUI** - Full async framework with widgets, screens, CSS

For Jetson MVP, we choose **ONE**: Textual TUI, because:
- It can do everything Rich TUI does (they share the same rendering engine)
- Async-native, better for streaming
- Built-in widgets reduce code
- CSS styling for consistent look

```
Interface Layer (2 options, same core)
├── CLI Mode (cli.py)
│   ├── `jetson run "task"` - One-shot execution
│   └── `jetson chat` - Interactive REPL
│
└── TUI Mode (tui/)
    ├── Home Screen - List/select sessions
    └── Chat Screen - Message input + streaming display
```

### 2. Core Layer

The **Core Layer** is the brain - it contains all business logic independent of UI:

```python
# Core Layer Components

class Agent:
    """Orchestrates the conversation loop"""
    def run(session: Session, message: str) -> AsyncGenerator[Event]:
        # 1. Add user message to session
        # 2. Call provider with messages + tools
        # 3. Process response (text or tool calls)
        # 4. If tool calls, execute and loop
        # 5. Yield events for UI to display

class Provider:
    """Wraps Anthropic API"""
    def stream(messages, tools) -> AsyncGenerator[APIEvent]:
        # Call claude API with streaming

class Session:
    """Holds conversation state"""
    id: str
    messages: list[Message]
    created_at: datetime

class Tools:
    """Registry of available tools"""
    DEFINITIONS: list[dict]  # Tool schemas for API
    def execute(name, input) -> str  # Run tool, return result
```

### 3. Storage Layer

Minimal JSON file storage:

```
~/.jetson/
├── config.json
│   {
│     "model": "claude-sonnet-4-20250514",
│     "auto_approve_reads": true
│   }
│
└── sessions/
    └── {session_id}.json
        {
          "id": "abc123",
          "created_at": "2024-01-15T10:30:00",
          "messages": [
            {"role": "user", "content": "...", "timestamp": "..."},
            {"role": "assistant", "content": [...], "timestamp": "..."}
          ]
        }
```

---

## File Structure

```
jetson/
├── pyproject.toml
├── src/
│   └── jetson/
│       ├── __init__.py          # Version: "0.1.0"
│       ├── __main__.py          # Entry: python -m jetson
│       │
│       ├── cli.py               # Typer commands (~80 lines)
│       │   ├── run(prompt)      # One-shot task
│       │   ├── chat()           # Interactive mode
│       │   └── tui()            # Launch TUI
│       │
│       ├── agent.py             # Agent loop (~150 lines)
│       │   └── run_loop(session, message, provider)
│       │
│       ├── provider.py          # Anthropic wrapper (~60 lines)
│       │   └── AsyncProvider.stream()
│       │
│       ├── tools.py             # Tool definitions + execution (~200 lines)
│       │   ├── TOOLS = [...]    # Schemas
│       │   └── execute(name, input)
│       │
│       ├── models.py            # Pydantic models (~50 lines)
│       │   ├── Message
│       │   └── Session
│       │
│       ├── storage.py           # JSON persistence (~60 lines)
│       │   ├── save_session()
│       │   ├── load_session()
│       │   └── list_sessions()
│       │
│       ├── config.py            # Configuration (~40 lines)
│       │   └── Config.load()
│       │
│       └── tui/                 # Textual TUI (~300 lines total)
│           ├── __init__.py
│           ├── app.py           # Main app
│           ├── screens/
│           │   ├── home.py      # Session list
│           │   └── chat.py      # Chat interface
│           └── widgets/
│               └── message.py   # Message display
│
└── tests/
    └── test_tools.py

Total: ~1,000 lines of Python
```

---

## Tool System

### 6 Core Tools

| Tool | Purpose | Requires Permission |
|------|---------|---------------------|
| `read_file` | Read file contents | No (auto-approve) |
| `write_file` | Create/overwrite file | **Yes** |
| `edit_file` | Find & replace in file | **Yes** |
| `bash` | Run shell command | **Yes** |
| `glob` | Find files by pattern | No |
| `grep` | Search file contents | No |

### Tool Schema Example

```python
TOOLS = [
    {
        "name": "read_file",
        "description": "Read contents of a file",
        "input_schema": {
            "type": "object",
            "properties": {
                "path": {"type": "string", "description": "File path to read"}
            },
            "required": ["path"]
        }
    },
    # ... other tools
]
```

### Permission Flow

```
┌─────────────────────────────────────────┐
│  Tool Call: write_file                  │
│  Path: src/main.py                      │
│  Content: [50 lines of code]            │
├─────────────────────────────────────────┤
│                                         │
│  Allow this action? (y/n): █            │
│                                         │
└─────────────────────────────────────────┘
```

---

## Dependencies

```toml
[project]
name = "jetson"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "anthropic>=0.25.0",      # LLM API
    "typer>=0.9.0",           # CLI framework
    "pydantic>=2.0.0",        # Data models
    "textual>=0.47.0",        # TUI framework
]

[project.scripts]
jetson = "jetson.cli:app"
```

**4 dependencies. That's it.**

---

## Build Phases

| Phase | Days | Deliverable |
|-------|------|-------------|
| **1** | 1-2 | CLI chat with Claude (no tools) |
| **2** | 3-5 | Add 6 tools, permission prompts |
| **3** | 6-8 | Session persistence |
| **4** | 9-11 | Textual TUI |
| **5** | 12-14 | Polish, config, edge cases |

**2 weeks to complete MVP.**

---

## Sequence Diagram: User Request

```
User          CLI/TUI         Agent          Provider        Tools
  │              │               │               │              │
  │──"fix bug"──▶│               │               │              │
  │              │               │               │              │
  │              │──run(msg)────▶│               │              │
  │              │               │               │              │
  │              │               │──stream()────▶│              │
  │              │               │               │──API call───▶│ Claude
  │              │               │               │◀─response────│
  │              │               │◀─events───────│              │
  │              │               │               │              │
  │              │               │   [tool_use: read_file]      │
  │              │               │               │              │
  │              │               │──execute()───────────────────▶│
  │              │               │◀─result──────────────────────│
  │              │               │               │              │
  │              │               │──stream()────▶│              │
  │              │               │               │──API call───▶│ Claude
  │              │               │◀─events───────│              │
  │              │               │               │              │
  │              │               │   [tool_use: edit_file]      │
  │              │◀─"Allow?"─────│               │              │
  │◀─"Allow?"────│               │               │              │
  │──"y"────────▶│               │               │              │
  │              │──"y"─────────▶│               │              │
  │              │               │──execute()───────────────────▶│
  │              │               │◀─result──────────────────────│
  │              │               │               │              │
  │              │               │──stream()────▶│              │
  │              │               │◀─"Done!"──────│              │
  │              │◀─"Done!"──────│               │              │
  │◀─"Done!"─────│               │               │              │
  │              │               │               │              │
```

---

## Quick Start

```bash
# Install
pip install jetson
# or from source
git clone ... && pip install -e .

# Set API key
export ANTHROPIC_API_KEY="sk-..."

# Run one-shot task
jetson run "Create a hello world Python script"

# Interactive chat
jetson chat

# Launch TUI
jetson tui

# Resume session
jetson chat --session abc123
```

---

## What Jetson Intentionally Skips

| Feature | Why Skipped |
|---------|-------------|
| Multi-provider | Anthropic is best for coding; add others later |
| MCP/LSP | Complex; not needed for basic tasks |
| Web UI | TUI is sufficient for developers |
| Plugins | Keep it simple; fork to extend |
| OAuth | API key works; enterprise can add later |
| SQLite | JSON files are simpler, human-readable |
| Event sourcing | In-memory state is fine for MVP |
| Compaction | Sessions are small enough |

---

## Summary

**Jetson** is what OpenCode would look like if you stripped away everything except:
- Talk to Claude ✓
- Read/write files ✓
- Run commands ✓
- Remember conversations ✓
- Nice terminal UI ✓

**~1,000 lines. 4 dependencies. 2 weeks.**

That's the MVP.
