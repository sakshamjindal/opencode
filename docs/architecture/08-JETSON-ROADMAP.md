# Jetson Roadmap: Self-Bootstrapping to Full OpenCode

## The Vision

Jetson is built once by humans (with Claude Code), then **builds itself forward**.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│   THE AUTONOMY PROGRESSION                                                   │
│   ────────────────────────                                                   │
│                                                                              │
│   Phase 1 (v0)        Phase 2 (v1)        Phase 3-4 (v2-v3)    Phase 5+ (v4+)│
│   ┌──────────┐        ┌──────────┐        ┌──────────┐        ┌──────────┐  │
│   │  Claude  │        │  Jetson  │        │  Jetson  │        │  Jetson  │  │
│   │   Code   │───────▶│    +     │───────▶│  mostly  │───────▶│  fully   │  │
│   │  builds  │        │  Claude  │        │  alone   │        │autonomous│  │
│   │   v0     │        │   Code   │        │          │        │          │  │
│   └──────────┘        └──────────┘        └──────────┘        └──────────┘  │
│                                                                              │
│   Human: 100%         Human: 30%          Human: 10%          Human: 5%     │
│   Jetson: 0%          Jetson: 70%         Jetson: 90%         Jetson: 95%   │
│                                                                              │
│   "Build the          "Add features       "Jetson handles     "Jetson can   │
│    bootstrap"          together"           complex tasks"      build anything"│
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Table of Contents

1. [Self-Bootstrapping Concept](#self-bootstrapping-concept)
2. [Feature Priority Matrix](#feature-priority-matrix)
3. [Phase 1: v0 Bootstrap](#phase-1-v0-bootstrap---built-by-claude-code)
4. [Phase 2: v1 Foundation](#phase-2-v1-foundation---jetson--claude-code)
5. [Phase 3: v2 Professional](#phase-3-v2-professional---jetson-with-guidance)
6. [Phase 4: v3 Advanced](#phase-4-v3-advanced---jetson-mostly-autonomous)
7. [Phase 5: v4 Enterprise](#phase-5-v4-enterprise---jetson-fully-autonomous)
8. [Phase 6: Jetson Final](#phase-6-jetson-final---feature-complete)
9. [Timeline Summary](#timeline-summary)

---

## Self-Bootstrapping Concept

### Why It Works

Once Jetson v0 exists, it has **all the primitive tools** needed to build anything:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│   THE 7 GENESIS COMPONENTS (Built by Claude Code, cannot self-bootstrap)    │
│   ──────────────────────────────────────────────────────────────────────    │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                                                                      │   │
│   │   1. Agent Loop     - The orchestration brain                       │   │
│   │   2. read_file      - To understand existing code                   │   │
│   │   3. write_file     - To create new code                            │   │
│   │   4. edit_file      - To modify existing code                       │   │
│   │   5. bash           - To run tests, git, pip install                │   │
│   │   6. Provider       - To call the LLM                               │   │
│   │   7. Permission     - For safety (human approval)                   │   │
│   │                                                                      │   │
│   │   + glob/grep       - Navigation helpers (nice to have)             │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   With these 7 components, Jetson can:                                      │
│                                                                              │
│   • Read its own source code                                                │
│   • Create new modules and files                                            │
│   • Modify existing implementations                                         │
│   • Run tests to verify changes                                             │
│   • Commit changes to git                                                   │
│   • Install new dependencies                                                │
│   • Refactor and improve itself                                             │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### The Acceleration Effect

Each version makes the next version **faster to build**:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│   Version    New Capabilities              Build Speed for Next Version     │
│   ───────    ────────────────              ─────────────────────────────    │
│                                                                              │
│   v0         6 tools, 1 provider           Slow (limited tools)             │
│              │                                                               │
│              ▼                                                               │
│   v1         + multi-provider              Faster (better models available) │
│              + web_fetch                   Can research documentation       │
│              + SQLite                      Better context management        │
│              │                                                               │
│              ▼                                                               │
│   v2         + MCP tools                   Much faster (external tools)     │
│              + LSP diagnostics             Catches errors automatically     │
│              + task tool                   Can delegate subtasks            │
│              │                                                               │
│              ▼                                                               │
│   v3         + session revert              Can undo mistakes quickly        │
│              + reasoning                   Better planning                  │
│              + more providers              Cost optimization                │
│              │                                                               │
│              ▼                                                               │
│   v4+        + full OpenCode parity        Can build anything               │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Feature Priority Matrix

Features ordered by **when they're needed** for self-improvement:

### Priority 1: Critical for Self-Building (v1)
| Feature | Why Critical |
|---------|--------------|
| **task tool** (subagent) | Delegate complex subtasks |
| **multi-provider** | Use faster/cheaper models |
| **web_fetch** | Research documentation |
| **SQLite storage** | Handle larger contexts |
| **session compaction** | Fit more history in context |

### Priority 2: Safety & Reliability (v2)
| Feature | Why Critical |
|---------|--------------|
| **doom loop detection** | Stop infinite retry loops |
| **session revert** | Undo dangerous changes |
| **permission ruleset** | Configurable safety |
| **LSP diagnostics** | Catch errors before commit |
| **MCP client** | Use external tools |

### Priority 3: Developer Experience (v3)
| Feature | Why Critical |
|---------|--------------|
| **todowrite/todoread** | Track complex tasks |
| **patch tool** | Apply unified diffs |
| **reasoning support** | Extended thinking |
| **multi_edit** | Atomic file changes |
| **delta streaming** | Efficient updates |

### Priority 4: Professional Features (v4)
| Feature | Why Critical |
|---------|--------------|
| **agent types** | Build vs plan modes |
| **session forking** | Branch conversations |
| **plugin system** | Extensibility |
| **more LSP features** | Better code intelligence |
| **MCP OAuth** | Secure external tools |

### Priority 5: Enterprise & Scale (v5+)
| Feature | Why Critical |
|---------|--------------|
| **Web UI** | Browser access |
| **Desktop app** | Native experience |
| **OAuth** | User authentication |
| **billing** | Usage tracking |
| **20+ providers** | Full flexibility |

---

## Phase 1: v0 Bootstrap - Built by Claude Code

**Duration:** 3 weeks
**Builder:** Human + Claude Code (100%)
**Lines:** ~1,500

### Goal
Create the minimal viable AI coding agent that can improve itself.

### What Gets Built

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              JETSON v0                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Interface Layer                                                            │
│   ├── CLI (Typer)           run, chat, serve, tui, ui                       │
│   └── TUI (Textual)         standalone + server mode                        │
│                                                                              │
│   API Layer                                                                  │
│   └── FastAPI Server        REST + SSE streaming                            │
│                                                                              │
│   Core Layer                                                                 │
│   ├── Agent                 message loop, tool execution                    │
│   ├── Provider              Anthropic only                                  │
│   ├── Tools                 read, write, edit, bash, glob, grep            │
│   ├── Storage               JSON files                                      │
│   ├── Config                Basic settings                                  │
│   └── Event Bus             pub/sub for components                          │
│                                                                              │
│   8 dependencies, ~1,500 lines                                              │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Files Created

```
src/jetson/
├── __init__.py              # Version
├── __main__.py              # Entry point
├── cli.py                   # CLI commands (~100 lines)
├── agent.py                 # Agent loop (~180 lines)
├── provider.py              # Anthropic client (~80 lines)
├── tools.py                 # 6 tools (~250 lines)
├── models.py                # Data models (~60 lines)
├── storage.py               # JSON persistence (~80 lines)
├── config.py                # Configuration (~50 lines)
├── bus.py                   # Event bus (~60 lines)
├── server/
│   ├── app.py               # FastAPI (~150 lines)
│   └── schemas.py           # Request/Response
└── tui/
    ├── app.py               # Textual app (~200 lines)
    └── widgets/             # UI components (~150 lines)
```

### Success Criteria
- [ ] `jetson run "create hello.py"` works
- [ ] `jetson chat` interactive mode works
- [ ] `jetson serve` + `jetson ui` works
- [ ] Can read/write/edit files with permission prompts
- [ ] Can run bash commands with permission prompts

---

## Phase 2: v1 Foundation - Jetson + Claude Code

**Duration:** +2 weeks (cumulative: 5 weeks)
**Builder:** Jetson v0 (70%) + Claude Code assistance (30%)
**Lines:** ~3,500

### Goal
Add capabilities that make Jetson faster at building itself.

### What Gets Built

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              JETSON v1                                       │
│                         (Built by Jetson v0)                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   NEW: Multi-Provider System                                                 │
│   ├── Anthropic (Claude 3.5/4)                                              │
│   ├── OpenAI (GPT-4o, o1)                                                   │
│   └── Ollama (local models)                                                 │
│                                                                              │
│   NEW: Enhanced Tools                                                        │
│   ├── task         - Spawn subagent for complex work                        │
│   ├── web_fetch    - Fetch and parse web pages                              │
│   ├── tree         - Directory structure view                               │
│   └── multi_edit   - Multiple edits atomically                              │
│                                                                              │
│   NEW: Better Storage                                                        │
│   ├── SQLite backend                                                        │
│   ├── Session compaction                                                    │
│   └── Migration system                                                      │
│                                                                              │
│   NEW: Safety Features                                                       │
│   ├── Doom loop detection (3 consecutive denials = stop)                    │
│   └── Better error messages                                                 │
│                                                                              │
│   12 dependencies, ~3,500 lines (+2,000)                                    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### How Jetson v0 Builds This

```
User: "Add the task tool so you can delegate subtasks to yourself"

Jetson v0:
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  1. RESEARCH                                                                │
│     > read_file("src/jetson/tools.py")                                     │
│     > read_file("src/jetson/agent.py")                                     │
│     → Understand how tools work and how agent runs                         │
│                                                                             │
│  2. DESIGN                                                                  │
│     "task tool needs to:                                                   │
│      - Accept a prompt and optional agent config                           │
│      - Create a new agent instance                                         │
│      - Run it with isolated context                                        │
│      - Return the result"                                                  │
│                                                                             │
│  3. IMPLEMENT                                                               │
│     > edit_file("src/jetson/tools.py")                                     │
│       + Add TaskTool class                                                 │
│       + Add to TOOLS list                                                  │
│                                                                             │
│  4. TEST                                                                    │
│     > bash("pytest tests/test_tools.py -k task -v")                        │
│                                                                             │
│  5. COMMIT                                                                  │
│     > bash("git add . && git commit -m 'feat: add task tool'")             │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Files Changed/Added

```diff
src/jetson/
├── providers/                    # NEW DIRECTORY
│   ├── __init__.py              # Provider factory
│   ├── base.py                  # BaseProvider ABC
│   ├── anthropic.py             # Refactored
│   ├── openai.py                # NEW
│   └── ollama.py                # NEW
│
├── storage/                      # REFACTORED
│   ├── __init__.py
│   ├── base.py                  # Storage interface
│   ├── json.py                  # Original (kept)
│   ├── sqlite.py                # NEW
│   └── migrations.py            # NEW
│
├── tools/                        # REFACTORED TO DIRECTORY
│   ├── __init__.py              # Tool registry
│   ├── base.py                  # Tool base class
│   ├── file_ops.py              # read, write, edit
│   ├── search.py                # glob, grep
│   ├── bash.py                  # bash command
│   ├── task.py                  # NEW: subagent
│   ├── web.py                   # NEW: web_fetch
│   └── tree.py                  # NEW: directory tree
│
├── safety.py                     # NEW: doom loop detection
└── provider.py                   # DEPRECATED
```

### New Dependencies

```diff
dependencies = [
    "anthropic>=0.25.0",
+   "openai>=1.0.0",
+   "ollama>=0.1.0",
+   "aiosqlite>=0.19.0",
+   "httpx>=0.26.0",
+   "beautifulsoup4>=4.12.0",
    "pydantic>=2.0.0",
    "typer>=0.9.0",
    "fastapi>=0.109.0",
    "uvicorn>=0.27.0",
    "sse-starlette>=1.8.0",
    "textual>=0.47.0",
]
```

### Why Claude Code Still Helps (30%)

- **Architecture decisions**: "Should providers be a directory or single file?"
- **Edge cases**: "What if OpenAI returns a different error format?"
- **Testing strategy**: "How to mock LLM responses in tests?"
- **Performance**: "Is SQLite fast enough for session storage?"

---

## Phase 3: v2 Professional - Jetson with Guidance

**Duration:** +3 weeks (cumulative: 8 weeks)
**Builder:** Jetson v1 (85%) + Human guidance (15%)
**Lines:** ~8,000

### Goal
Add MCP, LSP, and safety features that make Jetson professional-grade.

### What Gets Built

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              JETSON v2                                       │
│                         (Built by Jetson v1)                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   NEW: MCP Integration                                                       │
│   ├── MCP Client              Connect to external tool servers              │
│   ├── MCP Server              Expose Jetson tools via MCP                   │
│   ├── Tool Registry           Dynamic tool discovery                        │
│   └── stdio/SSE transport     Both transport types                          │
│                                                                              │
│   NEW: LSP Integration                                                       │
│   ├── LSP Client Manager      Per-language server management                │
│   ├── Diagnostics             Errors/warnings after edits                   │
│   ├── Go-to-definition        Navigate to symbol definitions                │
│   └── Built-in servers        Python (pyright), TypeScript, Go, Rust       │
│                                                                              │
│   NEW: Session Features                                                      │
│   ├── Session revert          Undo to any message                           │
│   ├── Session archive         Mark sessions as archived                     │
│   ├── Diff tracking           Track file changes per session                │
│   └── Summary generation      Auto-generate session titles                  │
│                                                                              │
│   NEW: Permission System                                                     │
│   ├── Permission ruleset      Configurable allow/deny/ask rules            │
│   ├── Wildcard patterns       "*.py" matches, "rm -rf" blocks              │
│   ├── Bash command arity      Smart command parsing (git checkout = 2)     │
│   └── Session approvals       Remember approvals during session            │
│                                                                              │
│   NEW: Tools                                                                 │
│   ├── todowrite/todoread      Task tracking during sessions                │
│   ├── patch                   Apply unified diffs                          │
│   ├── lsp                     Query language servers                       │
│   └── websearch               Web search (Exa API)                         │
│                                                                              │
│   18 dependencies, ~8,000 lines (+4,500)                                    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### How Jetson v1 Builds This

```
User: "Add MCP client support"

Jetson v1:
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  1. RESEARCH (using new web_fetch tool!)                                   │
│     > web_fetch("https://modelcontextprotocol.io/docs/concepts/clients")   │
│     > read_file("docs/architecture/03-INTEGRATIONS.md")                    │
│     → Understand MCP protocol, JSON-RPC, transports                        │
│                                                                             │
│  2. DELEGATE SUBTASK (using new task tool!)                                │
│     > task(prompt="Research JSON-RPC 2.0 message format")                  │
│     → Subagent returns detailed spec                                       │
│                                                                             │
│  3. IMPLEMENT                                                               │
│     > write_file("src/jetson/mcp/__init__.py")                             │
│     > write_file("src/jetson/mcp/client.py")                               │
│     > write_file("src/jetson/mcp/transport.py")                            │
│     > edit_file("src/jetson/tools/__init__.py")                            │
│       + Register MCP tools dynamically                                     │
│                                                                             │
│  4. TEST                                                                    │
│     > bash("pip install mcp-server-filesystem")                            │
│     > bash("pytest tests/test_mcp.py -v")                                  │
│                                                                             │
│  5. COMMIT                                                                  │
│     > bash("git add . && git commit -m 'feat: MCP client support'")        │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Files Changed/Added

```diff
src/jetson/
├── mcp/                          # NEW
│   ├── __init__.py
│   ├── client.py                # MCP client implementation
│   ├── server.py                # Expose Jetson as MCP server
│   ├── transport.py             # stdio, HTTP/SSE transports
│   └── discovery.py             # Find MCP servers in config
│
├── lsp/                          # NEW
│   ├── __init__.py
│   ├── client.py                # LSP client manager
│   ├── diagnostics.py           # Error/warning collection
│   ├── servers.py               # Built-in server definitions
│   └── actions.py               # Code actions (future)
│
├── permission/                   # NEW
│   ├── __init__.py
│   ├── ruleset.py               # Rule evaluation
│   ├── patterns.py              # Wildcard matching
│   └── arity.py                 # Bash command parsing
│
├── session/                      # REFACTORED
│   ├── __init__.py
│   ├── manager.py               # Session CRUD
│   ├── revert.py                # NEW: Rollback support
│   ├── diff.py                  # NEW: Change tracking
│   └── summary.py               # NEW: Title generation
│
├── tools/
│   ├── ...existing...
│   ├── todo.py                  # NEW: todowrite/todoread
│   ├── patch.py                 # NEW: unified diff
│   ├── lsp_tool.py              # NEW: LSP queries
│   └── websearch.py             # NEW: web search
│
└── config.py                     # UPDATED: MCP, LSP config
```

### Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              JETSON v2                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐                       │
│   │   CLI   │  │   TUI   │  │ FastAPI │  │   MCP   │                       │
│   └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘                       │
│        └────────────┴────────────┴────────────┘                             │
│                          │                                                   │
│                          ▼                                                   │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                         CORE LAYER                                   │   │
│   │  ┌─────────┐  ┌─────────┐  ┌─────────────┐  ┌─────────────┐        │   │
│   │  │  Agent  │  │Provider │  │    Tool     │  │  Permission │        │   │
│   │  │         │  │ Factory │  │  Registry   │  │   Ruleset   │        │   │
│   │  └─────────┘  └─────────┘  └──────┬──────┘  └─────────────┘        │   │
│   │                                   │                                  │   │
│   │                    ┌──────────────┼──────────────┐                  │   │
│   │                    ▼              ▼              ▼                  │   │
│   │              ┌──────────┐  ┌──────────┐  ┌──────────┐              │   │
│   │              │ Built-in │  │   MCP    │  │  Plugin  │              │   │
│   │              │  Tools   │  │  Tools   │  │  Tools   │              │   │
│   │              │  (12)    │  │(dynamic) │  │ (future) │              │   │
│   │              └──────────┘  └──────────┘  └──────────┘              │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                          LSP LAYER                                   │   │
│   │  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐        │   │
│   │  │  pyright  │  │    tsserver   │  │   gopls   │  │rust-analyzer│   │   │
│   │  │ (Python)  │  │(TypeScript)│  │   (Go)    │  │  (Rust)   │        │   │
│   │  └───────────┘  └───────────┘  └───────────┘  └───────────┘        │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                        MCP CONNECTIONS                               │   │
│   │  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐        │   │
│   │  │Filesystem │  │    Git    │  │  Docker   │  │  Custom   │        │   │
│   │  │  Server   │  │  Server   │  │  Server   │  │  Server   │        │   │
│   │  └───────────┘  └───────────┘  └───────────┘  └───────────┘        │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Why Human Still Guides (15%)

- **Protocol edge cases**: "What if MCP server doesn't support tool listing?"
- **Security review**: "Is the permission ruleset safe?"
- **Architecture validation**: "Is this LSP integration approach correct?"

---

## Phase 4: v3 Advanced - Jetson Mostly Autonomous

**Duration:** +3 weeks (cumulative: 11 weeks)
**Builder:** Jetson v2 (95%) + Human review (5%)
**Lines:** ~15,000

### Goal
Add reasoning, agent types, and advanced session features.

### What Gets Built

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              JETSON v3                                       │
│                         (Built by Jetson v2)                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   NEW: Agent System                                                          │
│   ├── Agent types             primary, subagent, all                        │
│   ├── Built-in agents         build (full), plan (read-only)               │
│   ├── Custom agents           User-defined via config                       │
│   └── Agent permissions       Per-agent permission overrides                │
│                                                                              │
│   NEW: Reasoning Support                                                     │
│   ├── Extended thinking       Claude's <thinking> blocks                    │
│   ├── ReasoningPart           Store reasoning in messages                   │
│   └── Reasoning display       Show thinking in TUI                          │
│                                                                              │
│   NEW: Advanced Sessions                                                     │
│   ├── Session forking         Branch conversations (parentID)               │
│   ├── Session sharing         Share sessions via URL                        │
│   └── Better compaction       Smarter context management                    │
│                                                                              │
│   NEW: Message Parts                                                         │
│   ├── FilePart                File attachments                              │
│   ├── SourcePart              Citations from web search                     │
│   └── Delta streaming         Efficient incremental updates                 │
│                                                                              │
│   NEW: More Providers                                                        │
│   ├── Google (Gemini)                                                       │
│   ├── Groq (fast inference)                                                 │
│   ├── Mistral                                                               │
│   └── Together AI                                                           │
│                                                                              │
│   NEW: More Tools                                                            │
│   ├── codesearch              Search code examples (Exa)                    │
│   ├── skill                   Load specialized prompts                      │
│   └── batch                   Parallel tool execution                       │
│                                                                              │
│   NEW: Event Bus Improvements                                                │
│   ├── Global bus              Cross-process events                          │
│   └── Type-safe events        Pydantic validation                           │
│                                                                              │
│   22 dependencies, ~15,000 lines (+7,000)                                   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### How Jetson v2 Builds This

With MCP and LSP, Jetson v2 can:
- Use external documentation servers
- Validate code changes automatically
- Delegate subtasks efficiently
- Research unfamiliar APIs

```
User: "Add support for Claude's extended thinking"

Jetson v2:
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  1. RESEARCH                                                                │
│     > task(prompt="Research Anthropic's extended thinking API")            │
│     > web_fetch("https://docs.anthropic.com/en/docs/build-with-claude/    │
│                  extended-thinking")                                       │
│                                                                             │
│  2. ANALYZE IMPACT                                                         │
│     > read_file("src/jetson/providers/anthropic.py")                       │
│     > read_file("src/jetson/models.py")                                    │
│     > lsp(action="references", symbol="MessagePart")                       │
│     → Find all places that need updates                                    │
│                                                                             │
│  3. IMPLEMENT (with LSP validation)                                        │
│     > edit_file("src/jetson/models.py")                                    │
│       + Add ReasoningPart model                                            │
│     (LSP catches: missing import)                                          │
│     > edit_file("src/jetson/models.py")                                    │
│       + Fix import                                                         │
│                                                                             │
│     > edit_file("src/jetson/providers/anthropic.py")                       │
│       + Handle thinking blocks in stream                                   │
│                                                                             │
│     > edit_file("src/jetson/tui/widgets/message.py")                       │
│       + Render reasoning parts                                             │
│                                                                             │
│  4. TEST                                                                    │
│     > bash("pytest tests/ -v")                                             │
│     (All pass - LSP caught errors during implementation)                   │
│                                                                             │
│  5. COMMIT                                                                  │
│     > bash("git add . && git commit -m 'feat: extended thinking'")         │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Files Changed/Added

```diff
src/jetson/
├── agent/                        # REFACTORED TO DIRECTORY
│   ├── __init__.py
│   ├── base.py                  # Agent base class
│   ├── primary.py               # Primary agent (full access)
│   ├── plan.py                  # Plan agent (read-only)
│   └── factory.py               # Agent creation
│
├── models.py                     # UPDATED
│   + ReasoningPart
│   + FilePart
│   + SourcePart
│   + DeltaUpdate
│
├── providers/
│   ├── ...existing...
│   ├── google.py                # NEW: Gemini
│   ├── groq.py                  # NEW: Fast inference
│   ├── mistral.py               # NEW
│   └── together.py              # NEW
│
├── tools/
│   ├── ...existing...
│   ├── codesearch.py            # NEW: Code search
│   ├── skill.py                 # NEW: Load skills
│   └── batch.py                 # NEW: Parallel execution
│
├── session/
│   ├── ...existing...
│   ├── fork.py                  # NEW: Session branching
│   └── share.py                 # NEW: Session sharing
│
└── bus/                          # REFACTORED
    ├── __init__.py
    ├── instance.py              # Per-directory bus
    ├── global.py                # NEW: Cross-process
    └── events.py                # Type-safe event definitions
```

### Why Human Only Reviews (5%)

At this point, Jetson v2 can:
- Research APIs autonomously
- Validate code with LSP
- Catch its own errors
- Delegate complex subtasks

Human only needed for:
- **Final review**: "Does this architecture make sense?"
- **Edge cases**: "What about this unusual scenario?"

---

## Phase 5: v4 Enterprise - Jetson Fully Autonomous

**Duration:** +4 weeks (cumulative: 15 weeks)
**Builder:** Jetson v3 (98%) + Human approval (2%)
**Lines:** ~25,000

### Goal
Add enterprise features, Web UI, and remaining OpenCode parity.

### What Gets Built

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              JETSON v4                                       │
│                         (Built by Jetson v3)                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   NEW: Web UI                                                                │
│   ├── React/SolidJS frontend                                                │
│   ├── Real-time SSE updates                                                 │
│   ├── File browser                                                          │
│   ├── Diff viewer                                                           │
│   ├── Session management                                                    │
│   └── Responsive design                                                     │
│                                                                              │
│   NEW: Plugin System                                                         │
│   ├── Plugin discovery         ~/.jetson/plugins/                           │
│   ├── Tool plugins             Custom tools                                 │
│   ├── Provider plugins         Custom providers                             │
│   ├── Event hooks              Hook into agent lifecycle                    │
│   └── Auth plugins             Custom authentication                        │
│                                                                              │
│   NEW: MCP Advanced                                                          │
│   ├── OAuth flow (PKCE)        Secure external auth                         │
│   ├── Resource reading         Read MCP resources                           │
│   ├── Prompt invocation        Use MCP prompts                              │
│   └── HTTP/SSE transport       Remote MCP servers                           │
│                                                                              │
│   NEW: LSP Advanced                                                          │
│   ├── Find references          All usages of symbol                         │
│   ├── Hover information        Type info on hover                           │
│   ├── Document symbols         Outline view                                 │
│   ├── Workspace symbols        Project-wide search                          │
│   └── 20+ built-in servers     More language support                        │
│                                                                              │
│   NEW: More Providers                                                        │
│   ├── Azure OpenAI                                                          │
│   ├── Amazon Bedrock                                                        │
│   ├── Google Vertex AI                                                      │
│   ├── GitHub Copilot                                                        │
│   └── 10+ more                                                              │
│                                                                              │
│   NEW: Storage Advanced                                                      │
│   ├── Read-write locking       Concurrent access                            │
│   ├── Writer priority          Prevent starvation                           │
│   ├── Cache versioning         Auto-invalidation                            │
│   └── Session diffs            Separate diff storage                        │
│                                                                              │
│   30 dependencies, ~25,000 lines (+10,000)                                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### How Jetson v3 Builds Web UI

```
User: "Add a web UI"

Jetson v3 (with reasoning enabled):
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  <thinking>                                                                │
│  Let me plan this carefully. A web UI needs:                               │
│  1. Frontend framework (React or SolidJS)                                  │
│  2. Build system (Vite)                                                    │
│  3. API client for backend                                                 │
│  4. Real-time updates (SSE)                                                │
│  5. Routing                                                                │
│  6. State management                                                       │
│                                                                             │
│  I should delegate the initial scaffolding to a subtask.                   │
│  </thinking>                                                               │
│                                                                             │
│  1. DELEGATE SETUP                                                          │
│     > task(prompt="Create a new Vite + React + TypeScript project          │
│            in web/ directory with Tailwind CSS")                           │
│                                                                             │
│  2. IMPLEMENT SSE HOOK (parallel with component work)                      │
│     > batch([                                                              │
│         task(prompt="Create useSSE hook for real-time events"),            │
│         task(prompt="Create Chat component with message list"),            │
│         task(prompt="Create SessionList component")                        │
│       ])                                                                   │
│                                                                             │
│  3. INTEGRATE                                                               │
│     > read_file("web/src/App.tsx")                                         │
│     > edit_file("web/src/App.tsx")                                         │
│       + Import and compose components                                      │
│                                                                             │
│  4. UPDATE BACKEND                                                          │
│     > edit_file("src/jetson/server/app.py")                                │
│       + Serve static files from web/dist                                   │
│       + Add CORS for development                                           │
│                                                                             │
│  5. TEST                                                                    │
│     > bash("cd web && npm run build && npm run test")                      │
│     > bash("pytest tests/test_server.py")                                  │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              JETSON v4                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   CLIENTS                                                                    │
│   ┌───────┐ ┌───────┐ ┌───────┐ ┌───────┐ ┌───────┐ ┌───────┐             │
│   │  CLI  │ │  TUI  │ │Web UI │ │Desktop│ │  MCP  │ │Plugin │             │
│   └───┬───┘ └───┬───┘ └───┬───┘ └───┬───┘ └───┬───┘ └───┬───┘             │
│       └─────────┴─────────┴─────────┴─────────┴─────────┘                  │
│                                │                                            │
│   ┌────────────────────────────┼────────────────────────────────────────┐  │
│   │                       API LAYER                                      │  │
│   │                            │                                         │  │
│   │  ┌───────────┐  ┌─────────┴─────────┐  ┌───────────┐               │  │
│   │  │   Auth    │  │     FastAPI       │  │    SSE    │               │  │
│   │  │  (OAuth)  │  │    + WebSocket    │  │  Events   │               │  │
│   │  └───────────┘  └───────────────────┘  └───────────┘               │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                         CORE LAYER                                   │   │
│   │                                                                      │   │
│   │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ │   │
│   │  │  Agent   │ │ Provider │ │   Tool   │ │  Plugin  │ │Permission│ │   │
│   │  │  Engine  │ │  (23+)   │ │ Registry │ │  Manager │ │  System  │ │   │
│   │  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘ │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐              │
│   │    MCP     │ │    LSP     │ │  Storage   │ │   Event    │              │
│   │  (OAuth)   │ │  (20+ srv) │ │  (SQLite)  │ │    Bus     │              │
│   └────────────┘ └────────────┘ └────────────┘ └────────────┘              │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Phase 6: Jetson Final - Feature Complete

**Duration:** +3 weeks (cumulative: 18 weeks)
**Builder:** Jetson v4 (99%) + Human sign-off (1%)
**Lines:** ~35,000

### Goal
Full OpenCode feature parity plus Jetson-specific improvements.

### What Gets Built

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            JETSON (Final)                                    │
│                         (Built by Jetson v4)                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   NEW: Desktop App                                                           │
│   ├── Tauri wrapper                                                         │
│   ├── System tray                                                           │
│   ├── Global hotkey                                                         │
│   └── Native notifications                                                  │
│                                                                              │
│   NEW: Enterprise Features                                                   │
│   ├── OAuth (GitHub, Google, SAML)                                          │
│   ├── Multi-user sessions                                                   │
│   ├── Audit logging                                                         │
│   ├── Rate limiting                                                         │
│   ├── Usage tracking                                                        │
│   └── Cost tracking per provider                                            │
│                                                                              │
│   NEW: Advanced Features                                                     │
│   ├── Image input (vision)                                                  │
│   ├── PDF input                                                             │
│   ├── Prompt caching                                                        │
│   ├── Response streaming optimization                                       │
│   └── Context window management                                             │
│                                                                              │
│   NEW: Distribution                                                          │
│   ├── PyPI package                                                          │
│   ├── Homebrew formula                                                      │
│   ├── Docker image                                                          │
│   └── Binary releases                                                       │
│                                                                              │
│   35 dependencies, ~35,000 lines (+10,000)                                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Final Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│                              JETSON                                          │
│                    (Full OpenCode Equivalent)                                │
│                                                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   INTERFACES                                                                 │
│   ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐       │
│   │  CLI   │ │  TUI   │ │ Web UI │ │Desktop │ │  MCP   │ │  API   │       │
│   │(Typer) │ │(Txtual)│ │(React) │ │(Tauri) │ │Server  │ │ Only   │       │
│   └───┬────┘ └───┬────┘ └───┬────┘ └───┬────┘ └───┬────┘ └───┬────┘       │
│       │          │          │          │          │          │             │
│       └──────────┴──────────┴──────────┴──────────┴──────────┘             │
│                                    │                                        │
│   ┌────────────────────────────────┼────────────────────────────────────┐  │
│   │                          API LAYER                                   │  │
│   │  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐    │  │
│   │  │    Auth    │  │   FastAPI  │  │    SSE     │  │ WebSocket  │    │  │
│   │  │   (OAuth)  │  │   Server   │  │   Events   │  │  (future)  │    │  │
│   │  └────────────┘  └────────────┘  └────────────┘  └────────────┘    │  │
│   │                                                                      │  │
│   │  ┌────────────┐  ┌────────────┐  ┌────────────┐                    │  │
│   │  │Rate Limit  │  │   Audit    │  │   Usage    │                    │  │
│   │  │            │  │  Logging   │  │  Tracking  │                    │  │
│   │  └────────────┘  └────────────┘  └────────────┘                    │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                          CORE LAYER                                  │   │
│   │                                                                      │   │
│   │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐      │   │
│   │  │  Agent  │ │Provider │ │  Tool   │ │ Plugin  │ │Permission│      │   │
│   │  │ Engine  │ │ (23+)   │ │Registry │ │ Manager │ │  System  │      │   │
│   │  │         │ │         │ │ (19+)   │ │         │ │          │      │   │
│   │  │ • build │ │• Anthro │ │• file   │ │• tools  │ │• ruleset │      │   │
│   │  │ • plan  │ │• OpenAI │ │• web    │ │• provdr │ │• patterns│      │   │
│   │  │ • custom│ │• Google │ │• exec   │ │• events │ │• arity   │      │   │
│   │  │         │ │• 20more │ │• MCP    │ │• auth   │ │          │      │   │
│   │  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘      │   │
│   │                                                                      │   │
│   │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐                   │   │
│   │  │ Session │ │ Message │ │  Event  │ │ Config  │                   │   │
│   │  │ Manager │ │  Parts  │ │   Bus   │ │ System  │                   │   │
│   │  │         │ │         │ │         │ │         │                   │   │
│   │  │• create │ │• text   │ │• local  │ │• global │                   │   │
│   │  │• fork   │ │• reason │ │• global │ │• project│                   │   │
│   │  │• revert │ │• tool   │ │• typed  │ │• remote │                   │   │
│   │  │• share  │ │• file   │ │         │ │         │                   │   │
│   │  └─────────┘ └─────────┘ └─────────┘ └─────────┘                   │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐          │
│   │     MCP     │ │     LSP     │ │   Storage   │ │   Vision    │          │
│   │             │ │             │ │             │ │             │          │
│   │ • client    │ │ • 50+ srvrs │ │ • SQLite    │ │ • images    │          │
│   │ • server    │ │ • diagnostc │ │ • locking   │ │ • PDFs      │          │
│   │ • OAuth     │ │ • symbols   │ │ • migration │ │ • caching   │          │
│   │ • resources │ │ • actions   │ │ • versioning│ │             │          │
│   └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Timeline Summary

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│   PHASE    VERSION   DURATION   BUILDER              LINES    AUTONOMY      │
│   ─────    ───────   ────────   ───────              ─────    ────────      │
│                                                                              │
│     1        v0      3 weeks    Claude Code          1,500    Human: 100%   │
│                                 (Human: 100%)                  Jetson: 0%   │
│                                                                              │
│     2        v1      2 weeks    Jetson v0 +          3,500    Human: 30%    │
│                                 Claude Code                   Jetson: 70%   │
│                                                                              │
│     3        v2      3 weeks    Jetson v1 +          8,000    Human: 15%    │
│                                 guidance                      Jetson: 85%   │
│                                                                              │
│     4        v3      3 weeks    Jetson v2 +         15,000    Human: 5%     │
│                                 review                        Jetson: 95%   │
│                                                                              │
│     5        v4      4 weeks    Jetson v3 +         25,000    Human: 2%     │
│                                 approval                      Jetson: 98%   │
│                                                                              │
│     6      Final     3 weeks    Jetson v4 +         35,000    Human: 1%     │
│                                 sign-off                      Jetson: 99%   │
│                                                                              │
│   ─────────────────────────────────────────────────────────────────────     │
│   TOTAL              18 weeks   Progressive          35,000                  │
│                      (~4.5 mo)  autonomy                                     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Feature Checklist by Phase

### v0 (Bootstrap)
- [x] CLI (run, chat, serve, tui, ui)
- [x] FastAPI + SSE
- [x] Textual TUI
- [x] 6 core tools
- [x] JSON storage
- [x] Basic config
- [x] Event bus
- [x] Anthropic provider

### v1 (Foundation)
- [ ] Multi-provider (OpenAI, Ollama)
- [ ] task tool (subagent)
- [ ] web_fetch tool
- [ ] SQLite storage
- [ ] Session compaction
- [ ] Doom loop detection
- [ ] tree tool
- [ ] multi_edit tool

### v2 (Professional)
- [ ] MCP client
- [ ] MCP server
- [ ] LSP diagnostics
- [ ] LSP go-to-definition
- [ ] Session revert
- [ ] Permission ruleset
- [ ] todowrite/todoread
- [ ] patch tool
- [ ] websearch tool

### v3 (Advanced)
- [ ] Agent types (build, plan)
- [ ] Extended thinking
- [ ] Session forking
- [ ] FilePart, SourcePart
- [ ] Delta streaming
- [ ] More providers (Google, Groq, Mistral)
- [ ] codesearch tool
- [ ] skill tool
- [ ] batch tool
- [ ] Global event bus

### v4 (Enterprise)
- [ ] Web UI (React)
- [ ] Plugin system
- [ ] MCP OAuth
- [ ] LSP advanced (references, symbols)
- [ ] 20+ providers
- [ ] Read-write locking
- [ ] Cache versioning

### Final (Complete)
- [ ] Desktop app (Tauri)
- [ ] OAuth (GitHub, Google)
- [ ] Audit logging
- [ ] Rate limiting
- [ ] Usage/cost tracking
- [ ] Image/PDF input
- [ ] Prompt caching
- [ ] Distribution packages

---

## The Self-Improvement Loop

Once complete, Jetson can continuously improve itself:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│                    THE CONTINUOUS IMPROVEMENT CYCLE                          │
│                                                                              │
│                         ┌─────────────────┐                                 │
│                         │   User Request   │                                 │
│                         │ "Add feature X"  │                                 │
│                         └────────┬────────┘                                 │
│                                  │                                           │
│                                  ▼                                           │
│                         ┌─────────────────┐                                 │
│                         │    Research     │                                 │
│                         │  (web_fetch,    │                                 │
│                         │   read_file)    │                                 │
│                         └────────┬────────┘                                 │
│                                  │                                           │
│                                  ▼                                           │
│                         ┌─────────────────┐                                 │
│                         │     Plan        │                                 │
│                         │  (reasoning,    │                                 │
│                         │   task tool)    │                                 │
│                         └────────┬────────┘                                 │
│                                  │                                           │
│                                  ▼                                           │
│                         ┌─────────────────┐                                 │
│                         │   Implement     │                                 │
│                         │  (write, edit,  │                                 │
│                         │   multi_edit)   │                                 │
│                         └────────┬────────┘                                 │
│                                  │                                           │
│                                  ▼                                           │
│                         ┌─────────────────┐                                 │
│                         │    Validate     │                                 │
│                         │  (LSP, tests,   │                                 │
│                         │   lint)         │                                 │
│                         └────────┬────────┘                                 │
│                                  │                                           │
│                    ┌─────────────┴─────────────┐                            │
│                    │                           │                            │
│                    ▼                           ▼                            │
│           ┌──────────────┐           ┌──────────────┐                       │
│           │   Errors?    │           │   Success    │                       │
│           │  (revert +   │           │  (commit +   │                       │
│           │   retry)     │           │   done)      │                       │
│           └──────────────┘           └──────────────┘                       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Conclusion

Jetson's roadmap demonstrates that:

1. **Minimal bootstrap** - Only 7 components need to be built by humans
2. **Progressive autonomy** - Each version enables faster development of the next
3. **Human in the loop** - User always approves changes (safety)
4. **Full feature parity** - Achieves OpenCode equivalence in ~4.5 months
5. **Continuous improvement** - Can keep improving itself indefinitely

The key insight: **An AI coding agent that can read, write, edit, and run code can build anything** - including better versions of itself.
