# OpenCode Python MVP - Rapid Development Plan

## Philosophy

**Build something that works TODAY, improve it TOMORROW.**

This plan prioritizes:
- Working software at every phase
- Minimal abstractions initially
- Single-file starting points that expand
- Skip enterprise features entirely
- Use existing libraries aggressively

---

## MVP Scope

### What We're Building (v0.1)
```
┌─────────────────────────────────────────────────────────────────┐
│                         MVP Features                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ✅ INCLUDED                        ❌ DEFERRED                  │
│  ─────────────                      ──────────────               │
│  • Single LLM provider (Anthropic)  • Multi-provider support    │
│  • 5 core tools (read/write/edit/   • MCP integration           │
│    bash/glob)                       • LSP integration           │
│  • Simple file-based storage        • Plugin system             │
│  • Basic CLI interface              • OAuth flows               │
│  • Simple TUI (Rich-based)          • Web UI                    │
│  • Session persistence              • Desktop app               │
│  • Streaming responses              • Billing/enterprise        │
│  • Basic permission prompts         • Advanced permissions      │
│                                     • Session forking           │
│                                     • Compaction                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Target: 2-3 Weeks to Usable MVP

---

## Project Structure (Minimal)

```
opencode-py/
├── pyproject.toml
├── src/
│   └── opencode/
│       ├── __init__.py
│       ├── __main__.py      # Entry point
│       ├── cli.py           # CLI commands (~100 lines)
│       ├── agent.py         # Core agent loop (~200 lines)
│       ├── provider.py      # Anthropic provider (~100 lines)
│       ├── tools.py         # All tools in one file (~300 lines)
│       ├── storage.py       # Simple JSON storage (~100 lines)
│       ├── config.py        # Config loading (~50 lines)
│       ├── tui.py           # Rich-based TUI (~200 lines)
│       └── models.py        # Pydantic models (~100 lines)
└── tests/
    └── test_tools.py
```

**Total: ~1200 lines of code for MVP**

---

## Phase 1: Core Loop (Days 1-3)

### Goal: Chat with Claude, get responses

#### 1.1 Project Setup
```bash
# Create project
mkdir opencode-py && cd opencode-py
uv init
uv add anthropic pydantic typer rich

# Or with pip
python -m venv .venv
source .venv/bin/activate
pip install anthropic pydantic typer rich
```

#### 1.2 Minimal Models
```python
# src/opencode/models.py
from pydantic import BaseModel
from datetime import datetime
from typing import Literal

class Message(BaseModel):
    role: Literal["user", "assistant"]
    content: str
    timestamp: datetime = None

    def __init__(self, **data):
        super().__init__(**data)
        if self.timestamp is None:
            self.timestamp = datetime.now()

class Session(BaseModel):
    id: str
    messages: list[Message] = []
    created_at: datetime = None

    def __init__(self, **data):
        super().__init__(**data)
        if self.created_at is None:
            self.created_at = datetime.now()

class ToolCall(BaseModel):
    id: str
    name: str
    input: dict

class ToolResult(BaseModel):
    tool_use_id: str
    content: str
    is_error: bool = False
```

#### 1.3 Anthropic Provider (Minimal)
```python
# src/opencode/provider.py
import anthropic
from typing import AsyncGenerator
import os

class Provider:
    def __init__(self):
        self.client = anthropic.Anthropic(
            api_key=os.environ.get("ANTHROPIC_API_KEY")
        )
        self.model = "claude-sonnet-4-20250514"

    def stream(
        self,
        messages: list[dict],
        tools: list[dict] | None = None,
        system: str | None = None
    ):
        """Stream a response from Claude"""
        with self.client.messages.stream(
            model=self.model,
            max_tokens=8192,
            system=system or "You are a helpful coding assistant.",
            messages=messages,
            tools=tools or []
        ) as stream:
            for event in stream:
                yield event

    def complete(
        self,
        messages: list[dict],
        tools: list[dict] | None = None,
        system: str | None = None
    ):
        """Non-streaming completion"""
        return self.client.messages.create(
            model=self.model,
            max_tokens=8192,
            system=system or "You are a helpful coding assistant.",
            messages=messages,
            tools=tools or []
        )
```

#### 1.4 Simple CLI Entry Point
```python
# src/opencode/__main__.py
from .cli import app

if __name__ == "__main__":
    app()
```

```python
# src/opencode/cli.py
import typer
from rich.console import Console
from rich.markdown import Markdown
from .provider import Provider
from .models import Message, Session
import uuid

app = typer.Typer()
console = Console()

@app.command()
def chat(message: str = typer.Argument(None)):
    """Start a chat session"""
    provider = Provider()
    session = Session(id=str(uuid.uuid4())[:8])

    console.print("[bold green]OpenCode MVP[/bold green]")
    console.print("Type 'exit' to quit\n")

    while True:
        # Get user input
        if message:
            user_input = message
            message = None  # Only use arg for first message
        else:
            user_input = console.input("[bold blue]You:[/bold blue] ")

        if user_input.lower() in ('exit', 'quit', 'q'):
            break

        session.messages.append(Message(role="user", content=user_input))

        # Convert to API format
        api_messages = [
            {"role": m.role, "content": m.content}
            for m in session.messages
        ]

        # Stream response
        console.print("\n[bold green]Assistant:[/bold green]")
        full_response = ""

        for event in provider.stream(api_messages):
            if event.type == "content_block_delta":
                if hasattr(event.delta, "text"):
                    console.print(event.delta.text, end="")
                    full_response += event.delta.text

        console.print("\n")
        session.messages.append(Message(role="assistant", content=full_response))

@app.command()
def version():
    """Show version"""
    console.print("opencode-py v0.1.0")

if __name__ == "__main__":
    app()
```

#### 1.5 Test It
```bash
export ANTHROPIC_API_KEY="your-key"
python -m opencode chat "Hello, who are you?"
```

**Checkpoint: You can now chat with Claude!**

---

## Phase 2: Tool System (Days 4-7)

### Goal: Claude can read/write files and run commands

#### 2.1 Tool Definitions (All in One File)
```python
# src/opencode/tools.py
from pathlib import Path
import subprocess
import glob as glob_module
from rich.console import Console

console = Console()

# Tool schemas for Claude
TOOLS = [
    {
        "name": "read_file",
        "description": "Read the contents of a file. Use this to examine code, configs, or any text file.",
        "input_schema": {
            "type": "object",
            "properties": {
                "path": {
                    "type": "string",
                    "description": "The file path to read"
                }
            },
            "required": ["path"]
        }
    },
    {
        "name": "write_file",
        "description": "Write content to a file. Creates the file if it doesn't exist.",
        "input_schema": {
            "type": "object",
            "properties": {
                "path": {
                    "type": "string",
                    "description": "The file path to write to"
                },
                "content": {
                    "type": "string",
                    "description": "The content to write"
                }
            },
            "required": ["path", "content"]
        }
    },
    {
        "name": "edit_file",
        "description": "Edit a file by replacing a specific string with another. The old_str must match exactly.",
        "input_schema": {
            "type": "object",
            "properties": {
                "path": {
                    "type": "string",
                    "description": "The file path to edit"
                },
                "old_str": {
                    "type": "string",
                    "description": "The exact string to find and replace"
                },
                "new_str": {
                    "type": "string",
                    "description": "The string to replace it with"
                }
            },
            "required": ["path", "old_str", "new_str"]
        }
    },
    {
        "name": "bash",
        "description": "Run a bash command. Use for git, npm, python, etc.",
        "input_schema": {
            "type": "object",
            "properties": {
                "command": {
                    "type": "string",
                    "description": "The bash command to run"
                }
            },
            "required": ["command"]
        }
    },
    {
        "name": "glob",
        "description": "Find files matching a pattern. Returns list of matching file paths.",
        "input_schema": {
            "type": "object",
            "properties": {
                "pattern": {
                    "type": "string",
                    "description": "Glob pattern like '**/*.py' or 'src/*.ts'"
                }
            },
            "required": ["pattern"]
        }
    },
    {
        "name": "grep",
        "description": "Search for a pattern in files. Returns matching lines with file paths.",
        "input_schema": {
            "type": "object",
            "properties": {
                "pattern": {
                    "type": "string",
                    "description": "The regex pattern to search for"
                },
                "path": {
                    "type": "string",
                    "description": "Directory or file to search in (default: current dir)"
                }
            },
            "required": ["pattern"]
        }
    }
]


def ask_permission(tool_name: str, details: str) -> bool:
    """Simple permission prompt"""
    console.print(f"\n[yellow]Tool:[/yellow] {tool_name}")
    console.print(f"[yellow]Details:[/yellow] {details}")
    response = console.input("[yellow]Allow? (y/n):[/yellow] ")
    return response.lower() in ('y', 'yes')


def execute_tool(name: str, input: dict) -> str:
    """Execute a tool and return the result"""

    try:
        if name == "read_file":
            path = Path(input["path"])
            if not path.exists():
                return f"Error: File not found: {path}"

            content = path.read_text()
            # Add line numbers
            lines = content.split('\n')
            numbered = '\n'.join(f"{i+1:4d} | {line}" for i, line in enumerate(lines))
            return numbered

        elif name == "write_file":
            path = Path(input["path"])

            if not ask_permission("write_file", f"Write to {path}"):
                return "Permission denied by user"

            path.parent.mkdir(parents=True, exist_ok=True)
            path.write_text(input["content"])
            return f"Successfully wrote to {path}"

        elif name == "edit_file":
            path = Path(input["path"])

            if not path.exists():
                return f"Error: File not found: {path}"

            content = path.read_text()
            old_str = input["old_str"]
            new_str = input["new_str"]

            if old_str not in content:
                return f"Error: Could not find the specified text in {path}"

            if content.count(old_str) > 1:
                return f"Error: Found multiple matches. Please provide more context."

            if not ask_permission("edit_file", f"Edit {path}\n- Remove: {old_str[:50]}...\n+ Add: {new_str[:50]}..."):
                return "Permission denied by user"

            new_content = content.replace(old_str, new_str, 1)
            path.write_text(new_content)
            return f"Successfully edited {path}"

        elif name == "bash":
            command = input["command"]

            # Block dangerous commands
            dangerous = ['rm -rf /', 'mkfs', ':(){:|:&};:']
            if any(d in command for d in dangerous):
                return "Error: Dangerous command blocked"

            if not ask_permission("bash", command):
                return "Permission denied by user"

            result = subprocess.run(
                command,
                shell=True,
                capture_output=True,
                text=True,
                timeout=120
            )

            output = result.stdout
            if result.stderr:
                output += f"\n[stderr]\n{result.stderr}"
            if result.returncode != 0:
                output += f"\n[exit code: {result.returncode}]"

            return output or "(no output)"

        elif name == "glob":
            pattern = input["pattern"]
            matches = glob_module.glob(pattern, recursive=True)

            if not matches:
                return "No files found matching pattern"

            # Sort by modification time, limit results
            matches = sorted(matches, key=lambda p: Path(p).stat().st_mtime, reverse=True)[:100]
            return '\n'.join(matches)

        elif name == "grep":
            pattern = input["pattern"]
            path = input.get("path", ".")

            try:
                result = subprocess.run(
                    ["grep", "-rn", "--include=*", pattern, path],
                    capture_output=True,
                    text=True,
                    timeout=30
                )
                return result.stdout or "No matches found"
            except FileNotFoundError:
                # Fallback if grep not available
                return "grep command not available"

        else:
            return f"Unknown tool: {name}"

    except Exception as e:
        return f"Error executing {name}: {str(e)}"
```

#### 2.2 Update Agent Loop with Tools
```python
# src/opencode/agent.py
from .provider import Provider
from .tools import TOOLS, execute_tool
from .models import Message, Session, ToolCall
from rich.console import Console
from rich.panel import Panel
from rich.syntax import Syntax
import json

console = Console()

SYSTEM_PROMPT = """You are an expert software engineer assistant. You help users with coding tasks by reading, writing, and editing files, running commands, and explaining code.

When asked to make changes:
1. First read the relevant files to understand the context
2. Make precise, minimal edits
3. Verify your changes work

Always explain what you're doing and why."""


def run_agent_loop(session: Session, user_message: str, provider: Provider):
    """Run the agent loop with tool use"""

    session.messages.append(Message(role="user", content=user_message))

    while True:
        # Convert messages for API
        api_messages = []
        for m in session.messages:
            if isinstance(m.content, str):
                api_messages.append({"role": m.role, "content": m.content})
            else:
                api_messages.append({"role": m.role, "content": m.content})

        # Get response
        response = provider.complete(api_messages, tools=TOOLS, system=SYSTEM_PROMPT)

        # Process response
        assistant_content = []
        tool_calls = []

        for block in response.content:
            if block.type == "text":
                console.print(f"\n[green]Assistant:[/green] {block.text}")
                assistant_content.append({"type": "text", "text": block.text})

            elif block.type == "tool_use":
                tool_calls.append(ToolCall(
                    id=block.id,
                    name=block.name,
                    input=block.input
                ))
                assistant_content.append({
                    "type": "tool_use",
                    "id": block.id,
                    "name": block.name,
                    "input": block.input
                })

                # Show tool call
                console.print(Panel(
                    f"[cyan]Tool:[/cyan] {block.name}\n[cyan]Input:[/cyan] {json.dumps(block.input, indent=2)[:500]}",
                    title="Tool Call",
                    border_style="cyan"
                ))

        # Add assistant message
        session.messages.append(Message(role="assistant", content=assistant_content))

        # If no tool calls, we're done
        if not tool_calls:
            break

        # Execute tools and collect results
        tool_results = []
        for tc in tool_calls:
            console.print(f"\n[yellow]Executing {tc.name}...[/yellow]")
            result = execute_tool(tc.name, tc.input)

            # Show result (truncated)
            display_result = result[:1000] + "..." if len(result) > 1000 else result
            console.print(Panel(display_result, title=f"Result: {tc.name}", border_style="green"))

            tool_results.append({
                "type": "tool_result",
                "tool_use_id": tc.id,
                "content": result
            })

        # Add tool results as user message (Claude's format)
        session.messages.append(Message(role="user", content=tool_results))

        # Check if we should stop
        if response.stop_reason == "end_turn":
            break
```

#### 2.3 Update CLI
```python
# src/opencode/cli.py (updated)
import typer
from rich.console import Console
from .provider import Provider
from .agent import run_agent_loop
from .models import Session
from .storage import save_session, load_session, list_sessions
import uuid
import os

app = typer.Typer()
console = Console()


@app.command()
def chat(
    message: str = typer.Argument(None, help="Initial message"),
    session_id: str = typer.Option(None, "--session", "-s", help="Resume session")
):
    """Start or resume a chat session"""
    provider = Provider()

    # Load or create session
    if session_id:
        session = load_session(session_id)
        if not session:
            console.print(f"[red]Session {session_id} not found[/red]")
            raise typer.Exit(1)
        console.print(f"[dim]Resumed session {session_id}[/dim]\n")
    else:
        session = Session(id=str(uuid.uuid4())[:8])
        console.print(f"[dim]New session: {session.id}[/dim]\n")

    console.print("[bold green]OpenCode MVP[/bold green]")
    console.print("Commands: 'exit' to quit, 'save' to save session\n")

    while True:
        # Get user input
        if message:
            user_input = message
            message = None
        else:
            user_input = console.input("\n[bold blue]You:[/bold blue] ")

        if not user_input.strip():
            continue

        if user_input.lower() in ('exit', 'quit', 'q'):
            save_session(session)
            console.print(f"[dim]Session saved: {session.id}[/dim]")
            break

        if user_input.lower() == 'save':
            save_session(session)
            console.print(f"[dim]Session saved: {session.id}[/dim]")
            continue

        # Run agent
        try:
            run_agent_loop(session, user_input, provider)
        except KeyboardInterrupt:
            console.print("\n[yellow]Interrupted[/yellow]")
            continue


@app.command()
def sessions():
    """List saved sessions"""
    sessions = list_sessions()
    if not sessions:
        console.print("[dim]No saved sessions[/dim]")
        return

    for s in sessions:
        console.print(f"[cyan]{s['id']}[/cyan] - {s['created_at']} ({len(s['messages'])} messages)")


@app.command()
def run(
    prompt: str = typer.Argument(..., help="Task to perform"),
):
    """Run a one-shot task"""
    provider = Provider()
    session = Session(id=str(uuid.uuid4())[:8])

    console.print(f"[dim]Task: {prompt}[/dim]\n")
    run_agent_loop(session, prompt, provider)


if __name__ == "__main__":
    app()
```

#### 2.4 Simple Storage
```python
# src/opencode/storage.py
from pathlib import Path
import json
from .models import Session
from datetime import datetime

STORAGE_DIR = Path.home() / ".opencode" / "sessions"


def save_session(session: Session):
    """Save session to disk"""
    STORAGE_DIR.mkdir(parents=True, exist_ok=True)

    path = STORAGE_DIR / f"{session.id}.json"

    # Convert to serializable format
    data = {
        "id": session.id,
        "created_at": session.created_at.isoformat(),
        "messages": [
            {
                "role": m.role,
                "content": m.content,
                "timestamp": m.timestamp.isoformat()
            }
            for m in session.messages
        ]
    }

    path.write_text(json.dumps(data, indent=2))


def load_session(session_id: str) -> Session | None:
    """Load session from disk"""
    path = STORAGE_DIR / f"{session_id}.json"

    if not path.exists():
        return None

    data = json.loads(path.read_text())

    from .models import Message
    session = Session(
        id=data["id"],
        created_at=datetime.fromisoformat(data["created_at"]),
        messages=[
            Message(
                role=m["role"],
                content=m["content"],
                timestamp=datetime.fromisoformat(m["timestamp"])
            )
            for m in data["messages"]
        ]
    )

    return session


def list_sessions() -> list[dict]:
    """List all saved sessions"""
    if not STORAGE_DIR.exists():
        return []

    sessions = []
    for path in STORAGE_DIR.glob("*.json"):
        data = json.loads(path.read_text())
        sessions.append({
            "id": data["id"],
            "created_at": data["created_at"],
            "messages": len(data["messages"])
        })

    return sorted(sessions, key=lambda x: x["created_at"], reverse=True)
```

**Checkpoint: Claude can now read/write files and run commands!**

---

## Phase 3: Rich TUI (Days 8-10)

### Goal: Beautiful interactive terminal UI

#### 3.1 Rich-Based TUI
```python
# src/opencode/tui.py
from rich.console import Console, Group
from rich.panel import Panel
from rich.markdown import Markdown
from rich.syntax import Syntax
from rich.live import Live
from rich.spinner import Spinner
from rich.table import Table
from rich.layout import Layout
from rich.text import Text
from prompt_toolkit import prompt
from prompt_toolkit.history import FileHistory
from prompt_toolkit.auto_suggest import AutoSuggestFromHistory
from pathlib import Path
import json

from .provider import Provider
from .agent import run_agent_loop_streaming
from .models import Session
from .storage import save_session, load_session, list_sessions
import uuid

console = Console()
HISTORY_FILE = Path.home() / ".opencode" / "history"


def create_header():
    """Create app header"""
    return Panel(
        "[bold cyan]OpenCode[/bold cyan] [dim]v0.1.0[/dim]",
        style="cyan",
        padding=(0, 2)
    )


def format_message(role: str, content: str) -> Panel:
    """Format a message for display"""
    if role == "user":
        return Panel(
            content,
            title="[bold blue]You[/bold blue]",
            border_style="blue",
            padding=(0, 1)
        )
    else:
        # Try to render as markdown
        try:
            rendered = Markdown(content)
        except:
            rendered = content
        return Panel(
            rendered,
            title="[bold green]Assistant[/bold green]",
            border_style="green",
            padding=(0, 1)
        )


def format_tool_call(name: str, input: dict) -> Panel:
    """Format a tool call"""
    input_str = json.dumps(input, indent=2)
    return Panel(
        f"[cyan]{name}[/cyan]\n{input_str[:500]}",
        title="[yellow]Tool Call[/yellow]",
        border_style="yellow",
        padding=(0, 1)
    )


def format_tool_result(name: str, result: str) -> Panel:
    """Format a tool result"""
    # Truncate long results
    if len(result) > 2000:
        result = result[:2000] + "\n... (truncated)"

    return Panel(
        result,
        title=f"[green]{name} Result[/green]",
        border_style="green",
        padding=(0, 1)
    )


def show_sessions_menu():
    """Show interactive session selection"""
    sessions = list_sessions()

    if not sessions:
        console.print("[dim]No saved sessions[/dim]")
        return None

    table = Table(title="Saved Sessions")
    table.add_column("#", style="cyan")
    table.add_column("ID", style="green")
    table.add_column("Created", style="dim")
    table.add_column("Messages", justify="right")

    for i, s in enumerate(sessions, 1):
        table.add_row(str(i), s["id"], s["created_at"][:16], str(s["messages"]))

    console.print(table)

    choice = console.input("\n[cyan]Select session (number) or Enter for new:[/cyan] ")

    if choice.strip() and choice.isdigit():
        idx = int(choice) - 1
        if 0 <= idx < len(sessions):
            return sessions[idx]["id"]

    return None


def run_tui():
    """Main TUI loop"""
    console.clear()
    console.print(create_header())
    console.print()

    # Session selection
    console.print("[dim]Commands: /new, /sessions, /save, /exit, /help[/dim]\n")

    session_id = show_sessions_menu()

    if session_id:
        session = load_session(session_id)
        console.print(f"\n[dim]Resumed session {session_id}[/dim]\n")

        # Show recent messages
        for m in session.messages[-4:]:
            if isinstance(m.content, str):
                console.print(format_message(m.role, m.content))
    else:
        session = Session(id=str(uuid.uuid4())[:8])
        console.print(f"\n[dim]New session: {session.id}[/dim]\n")

    provider = Provider()
    HISTORY_FILE.parent.mkdir(parents=True, exist_ok=True)

    while True:
        try:
            # Get input with history
            user_input = prompt(
                "You: ",
                history=FileHistory(str(HISTORY_FILE)),
                auto_suggest=AutoSuggestFromHistory(),
            )
        except (KeyboardInterrupt, EOFError):
            break

        if not user_input.strip():
            continue

        # Handle commands
        if user_input.startswith("/"):
            cmd = user_input[1:].lower().split()[0]

            if cmd == "exit" or cmd == "quit":
                save_session(session)
                console.print(f"[dim]Session saved: {session.id}[/dim]")
                break

            elif cmd == "new":
                save_session(session)
                session = Session(id=str(uuid.uuid4())[:8])
                console.print(f"\n[dim]New session: {session.id}[/dim]\n")
                continue

            elif cmd == "sessions":
                save_session(session)
                session_id = show_sessions_menu()
                if session_id:
                    session = load_session(session_id)
                    console.print(f"\n[dim]Switched to session {session_id}[/dim]\n")
                continue

            elif cmd == "save":
                save_session(session)
                console.print(f"[dim]Session saved: {session.id}[/dim]")
                continue

            elif cmd == "help":
                console.print("""
[bold]Commands:[/bold]
  /new       - Start new session
  /sessions  - List/switch sessions
  /save      - Save current session
  /exit      - Save and exit
  /help      - Show this help
                """)
                continue

            else:
                console.print(f"[red]Unknown command: {cmd}[/red]")
                continue

        # Show user message
        console.print(format_message("user", user_input))

        # Run agent with streaming output
        try:
            with Live(Spinner("dots", text="Thinking..."), refresh_per_second=10):
                pass  # Spinner while waiting

            run_agent_loop(session, user_input, provider)

        except KeyboardInterrupt:
            console.print("\n[yellow]Interrupted[/yellow]")
            continue
        except Exception as e:
            console.print(f"[red]Error: {e}[/red]")
            continue

    console.print("\n[dim]Goodbye![/dim]")
```

#### 3.2 Add TUI Command
```python
# Add to cli.py

@app.command()
def tui():
    """Launch interactive TUI"""
    from .tui import run_tui
    run_tui()
```

#### 3.3 Dependencies Update
```bash
uv add prompt_toolkit
# or
pip install prompt_toolkit
```

**Checkpoint: Beautiful interactive TUI with session management!**

---

## Phase 4: Polish & Config (Days 11-14)

### Goal: Config file support, better UX

#### 4.1 Configuration
```python
# src/opencode/config.py
from pydantic import BaseModel
from pathlib import Path
import json
import os

class Config(BaseModel):
    model: str = "claude-sonnet-4-20250514"
    api_key: str | None = None
    auto_approve_read: bool = True  # Don't ask for read permission
    max_output_lines: int = 100

    @classmethod
    def load(cls) -> "Config":
        """Load config from file and env"""
        config_path = Path.home() / ".opencode" / "config.json"

        data = {}
        if config_path.exists():
            data = json.loads(config_path.read_text())

        # Env overrides
        if api_key := os.environ.get("ANTHROPIC_API_KEY"):
            data["api_key"] = api_key

        if model := os.environ.get("OPENCODE_MODEL"):
            data["model"] = model

        return cls(**data)

    def save(self):
        """Save config to file"""
        config_path = Path.home() / ".opencode" / "config.json"
        config_path.parent.mkdir(parents=True, exist_ok=True)

        # Don't save API key to file
        data = self.model_dump(exclude={"api_key"})
        config_path.write_text(json.dumps(data, indent=2))


# Global config
_config: Config | None = None

def get_config() -> Config:
    global _config
    if _config is None:
        _config = Config.load()
    return _config
```

#### 4.2 Better Tool Output
```python
# Update tools.py - smarter permission handling

def execute_tool(name: str, input: dict) -> str:
    config = get_config()

    # Auto-approve reads if configured
    if name == "read_file" and config.auto_approve_read:
        pass  # No permission needed
    elif name in ("write_file", "edit_file", "bash"):
        if not ask_permission(name, str(input)):
            return "Permission denied by user"

    # ... rest of execution
```

#### 4.3 Streaming Response Display
```python
# src/opencode/agent.py - add streaming support

def run_agent_loop_streaming(session: Session, user_message: str, provider: Provider):
    """Agent loop with streaming display"""
    from rich.live import Live
    from rich.text import Text

    session.messages.append(Message(role="user", content=user_message))

    while True:
        api_messages = format_messages(session.messages)

        # Stream response
        assistant_text = ""
        tool_calls = []

        console.print("\n[green]Assistant:[/green] ", end="")

        with Live(Text(""), refresh_per_second=15, transient=True) as live:
            for event in provider.stream(api_messages, tools=TOOLS, system=SYSTEM_PROMPT):
                if event.type == "content_block_delta":
                    if hasattr(event.delta, "text"):
                        assistant_text += event.delta.text
                        live.update(Text(assistant_text))

                elif event.type == "content_block_start":
                    if event.content_block.type == "tool_use":
                        tool_calls.append({
                            "id": event.content_block.id,
                            "name": event.content_block.name,
                            "input": {}
                        })

                elif event.type == "content_block_delta":
                    if hasattr(event.delta, "partial_json"):
                        # Accumulate tool input
                        if tool_calls:
                            # Parse partial JSON (simplified)
                            pass

        # Print final text
        console.print(assistant_text)

        # Handle tool calls...
```

---

## Phase 5: Additional Tools (Days 15-17)

### Add More Useful Tools

```python
# Add to tools.py

# Web fetch tool
{
    "name": "web_fetch",
    "description": "Fetch a web page and extract its text content",
    "input_schema": {
        "type": "object",
        "properties": {
            "url": {"type": "string", "description": "URL to fetch"}
        },
        "required": ["url"]
    }
}

# In execute_tool:
elif name == "web_fetch":
    import httpx
    from bs4 import BeautifulSoup

    url = input["url"]
    response = httpx.get(url, follow_redirects=True, timeout=30)
    soup = BeautifulSoup(response.text, 'html.parser')

    # Remove scripts and styles
    for tag in soup(['script', 'style', 'nav', 'footer']):
        tag.decompose()

    text = soup.get_text(separator='\n', strip=True)
    return text[:10000]  # Limit output
```

```python
# Tree tool for directory structure
{
    "name": "tree",
    "description": "Show directory tree structure",
    "input_schema": {
        "type": "object",
        "properties": {
            "path": {"type": "string", "description": "Directory path"},
            "depth": {"type": "integer", "description": "Max depth (default 3)"}
        },
        "required": []
    }
}

elif name == "tree":
    path = Path(input.get("path", "."))
    depth = input.get("depth", 3)

    def tree_str(p: Path, prefix: str = "", level: int = 0) -> str:
        if level >= depth:
            return ""

        lines = []
        items = sorted(p.iterdir(), key=lambda x: (x.is_file(), x.name))

        for i, item in enumerate(items):
            is_last = i == len(items) - 1
            connector = "└── " if is_last else "├── "
            lines.append(f"{prefix}{connector}{item.name}")

            if item.is_dir() and not item.name.startswith('.'):
                extension = "    " if is_last else "│   "
                lines.append(tree_str(item, prefix + extension, level + 1))

        return '\n'.join(filter(None, lines))

    return tree_str(path)
```

---

## Final Project Structure

```
opencode-py/
├── pyproject.toml
├── README.md
├── src/
│   └── opencode/
│       ├── __init__.py          # Version info
│       ├── __main__.py          # Entry: python -m opencode
│       ├── cli.py               # Typer CLI (~150 lines)
│       ├── tui.py               # Rich TUI (~250 lines)
│       ├── agent.py             # Agent loop (~200 lines)
│       ├── provider.py          # Anthropic client (~80 lines)
│       ├── tools.py             # All tools (~400 lines)
│       ├── storage.py           # JSON storage (~80 lines)
│       ├── config.py            # Configuration (~60 lines)
│       └── models.py            # Pydantic models (~80 lines)
└── tests/
    ├── test_tools.py
    └── test_storage.py

Total: ~1,300 lines of Python
```

---

## Dependencies (Minimal)

```toml
# pyproject.toml
[project]
name = "opencode"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "anthropic>=0.25.0",
    "typer>=0.9.0",
    "rich>=13.0.0",
    "pydantic>=2.0.0",
    "prompt-toolkit>=3.0.0",
]

[project.optional-dependencies]
web = ["httpx>=0.24.0", "beautifulsoup4>=4.12.0"]
dev = ["pytest>=7.0.0", "ruff>=0.1.0"]

[project.scripts]
opencode = "opencode.cli:app"
```

---

## Quick Start Commands

```bash
# Install
pip install -e .

# Or with uv
uv pip install -e .

# Run
export ANTHROPIC_API_KEY="your-key"

# One-shot task
opencode run "Create a hello world Python script"

# Interactive chat
opencode chat

# TUI mode
opencode tui

# Resume session
opencode chat --session abc123
```

---

## What's Next (Post-MVP)

After MVP is working, add in order of value:

1. **More Providers** - OpenAI, local models via Ollama
2. **Better Streaming** - True streaming in TUI
3. **Textual TUI** - Upgrade from Rich to Textual for better UX
4. **grep Tool** - Proper ripgrep integration
5. **LSP Integration** - Show errors after edits
6. **MCP Support** - External tool servers
7. **Web UI** - FastAPI + HTMX

---

## Timeline Summary

| Day | Milestone |
|-----|-----------|
| 1-3 | Basic chat working |
| 4-7 | Tools working (read/write/bash) |
| 8-10 | Rich TUI with sessions |
| 11-14 | Config, polish, more tools |
| 15-17 | Web fetch, tree, testing |
| **17** | **MVP Complete** |

**2.5 weeks to a fully functional AI coding assistant!**
