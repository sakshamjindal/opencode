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

## Phase 6: HTTP Backend (Days 18-22)

### Goal: FastAPI server so TUI/Web can connect remotely

#### 6.1 Server Setup
```python
# src/opencode/server/__init__.py
from .app import app, run_server

# src/opencode/server/app.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from sse_starlette.sse import EventSourceResponse
from pydantic import BaseModel
from typing import Optional
import asyncio
import uuid

from ..models import Session, Message
from ..storage import save_session, load_session, list_sessions
from ..agent import run_agent_loop_async
from ..provider import AsyncProvider
from ..bus import bus

app = FastAPI(title="OpenCode", version="0.1.0")

# CORS for web frontend
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)

# Request/Response models
class CreateSessionRequest(BaseModel):
    project_id: str = "default"
    directory: str = "."

class SendMessageRequest(BaseModel):
    content: str

class SessionResponse(BaseModel):
    id: str
    title: str
    created_at: str
    message_count: int


# ============== Session Endpoints ==============

@app.post("/api/session", response_model=SessionResponse)
async def create_session(req: CreateSessionRequest):
    """Create a new session"""
    session = Session(
        id=str(uuid.uuid4())[:8],
        project_id=req.project_id,
        directory=req.directory
    )
    save_session(session)
    await bus.publish("session.created", session)

    return SessionResponse(
        id=session.id,
        title=session.title or "New Session",
        created_at=session.created_at.isoformat(),
        message_count=0
    )


@app.get("/api/session")
async def get_sessions():
    """List all sessions"""
    sessions = list_sessions()
    return [
        SessionResponse(
            id=s["id"],
            title=s.get("title", "Untitled"),
            created_at=s["created_at"],
            message_count=s["messages"]
        )
        for s in sessions
    ]


@app.get("/api/session/{session_id}")
async def get_session(session_id: str):
    """Get session with messages"""
    session = load_session(session_id)
    if not session:
        raise HTTPException(404, "Session not found")

    return {
        "id": session.id,
        "title": session.title,
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


# ============== Message Endpoints ==============

@app.post("/api/session/{session_id}/message")
async def send_message(session_id: str, req: SendMessageRequest):
    """Send a message and get streaming response"""
    session = load_session(session_id)
    if not session:
        raise HTTPException(404, "Session not found")

    # Return SSE stream
    return EventSourceResponse(
        stream_agent_response(session, req.content)
    )


async def stream_agent_response(session: Session, user_message: str):
    """Generator for SSE streaming"""
    import json

    # Emit user message
    yield {
        "event": "message",
        "data": json.dumps({
            "role": "user",
            "content": user_message
        })
    }

    provider = AsyncProvider()

    # Stream assistant response
    async for event in run_agent_loop_async(session, user_message, provider):
        if event["type"] == "text":
            yield {
                "event": "text",
                "data": json.dumps({"content": event["content"]})
            }
        elif event["type"] == "tool_call":
            yield {
                "event": "tool_call",
                "data": json.dumps({
                    "name": event["name"],
                    "input": event["input"]
                })
            }
        elif event["type"] == "tool_result":
            yield {
                "event": "tool_result",
                "data": json.dumps({
                    "name": event["name"],
                    "output": event["output"]
                })
            }

    # Save session
    save_session(session)

    yield {
        "event": "done",
        "data": json.dumps({"session_id": session.id})
    }


# ============== Event Stream ==============

@app.get("/api/events")
async def event_stream():
    """Global event stream for real-time updates"""
    return EventSourceResponse(bus_events())


async def bus_events():
    """Generator for bus events"""
    import json
    queue = asyncio.Queue()

    async def handler(event_type: str, data):
        await queue.put((event_type, data))

    unsubscribe = bus.subscribe_all(handler)

    try:
        while True:
            event_type, data = await queue.get()
            yield {
                "event": event_type,
                "data": data.model_dump_json() if hasattr(data, 'model_dump_json') else json.dumps(data)
            }
    finally:
        unsubscribe()


# ============== Tool Endpoints ==============

@app.get("/api/tools")
async def list_tools():
    """List available tools"""
    from ..tools import TOOLS
    return TOOLS


# ============== Run Server ==============

def run_server(host: str = "127.0.0.1", port: int = 8080):
    """Run the server"""
    import uvicorn
    uvicorn.run(app, host=host, port=port)
```

#### 6.2 Async Provider
```python
# src/opencode/provider.py - add async version

import anthropic
from typing import AsyncGenerator

class AsyncProvider:
    def __init__(self):
        self.client = anthropic.AsyncAnthropic()
        self.model = "claude-sonnet-4-20250514"

    async def stream(
        self,
        messages: list[dict],
        tools: list[dict] | None = None,
        system: str | None = None
    ) -> AsyncGenerator:
        """Async streaming response"""
        async with self.client.messages.stream(
            model=self.model,
            max_tokens=8192,
            system=system or "You are a helpful coding assistant.",
            messages=messages,
            tools=tools or []
        ) as stream:
            async for event in stream:
                yield event
```

#### 6.3 Async Agent Loop
```python
# src/opencode/agent.py - add async version

async def run_agent_loop_async(session: Session, user_message: str, provider: AsyncProvider):
    """Async agent loop that yields events for streaming"""
    from .tools import TOOLS, execute_tool_async

    session.messages.append(Message(role="user", content=user_message))

    while True:
        api_messages = format_messages(session.messages)

        assistant_content = []
        tool_calls = []
        current_text = ""

        async for event in provider.stream(api_messages, tools=TOOLS, system=SYSTEM_PROMPT):
            if event.type == "content_block_delta":
                if hasattr(event.delta, "text"):
                    current_text += event.delta.text
                    yield {"type": "text", "content": event.delta.text}

            elif event.type == "content_block_start":
                if event.content_block.type == "tool_use":
                    tool_calls.append({
                        "id": event.content_block.id,
                        "name": event.content_block.name,
                        "input": {}
                    })

            elif event.type == "message_stop":
                break

        # Add assistant message
        if current_text:
            assistant_content.append({"type": "text", "text": current_text})

        for tc in tool_calls:
            assistant_content.append({
                "type": "tool_use",
                "id": tc["id"],
                "name": tc["name"],
                "input": tc["input"]
            })

        session.messages.append(Message(role="assistant", content=assistant_content))

        # If no tool calls, done
        if not tool_calls:
            break

        # Execute tools
        tool_results = []
        for tc in tool_calls:
            yield {"type": "tool_call", "name": tc["name"], "input": tc["input"]}

            result = await execute_tool_async(tc["name"], tc["input"])
            yield {"type": "tool_result", "name": tc["name"], "output": result}

            tool_results.append({
                "type": "tool_result",
                "tool_use_id": tc["id"],
                "content": result
            })

        session.messages.append(Message(role="user", content=tool_results))
```

#### 6.4 Event Bus (Async)
```python
# src/opencode/bus.py
from typing import Callable, Any
from collections import defaultdict
import asyncio

class EventBus:
    def __init__(self):
        self._subscribers: dict[str, list[Callable]] = defaultdict(list)
        self._global_subscribers: list[Callable] = []

    def subscribe(self, event_type: str, callback: Callable):
        self._subscribers[event_type].append(callback)
        return lambda: self._subscribers[event_type].remove(callback)

    def subscribe_all(self, callback: Callable):
        self._global_subscribers.append(callback)
        return lambda: self._global_subscribers.remove(callback)

    async def publish(self, event_type: str, data: Any):
        # Notify specific subscribers
        for cb in self._subscribers[event_type]:
            if asyncio.iscoroutinefunction(cb):
                await cb(data)
            else:
                cb(data)

        # Notify global subscribers
        for cb in self._global_subscribers:
            if asyncio.iscoroutinefunction(cb):
                await cb(event_type, data)
            else:
                cb(event_type, data)

# Global instance
bus = EventBus()
```

#### 6.5 CLI Server Command
```python
# Add to cli.py

@app.command()
def serve(
    host: str = typer.Option("127.0.0.1", "--host", "-h"),
    port: int = typer.Option(8080, "--port", "-p"),
):
    """Start the HTTP server"""
    from .server import run_server
    console.print(f"[green]Starting server at http://{host}:{port}[/green]")
    run_server(host, port)
```

**Checkpoint: HTTP API server with SSE streaming!**

---

## Phase 7: Textual TUI (Days 23-26)

### Goal: Beautiful, feature-rich terminal UI with Textual

#### 7.1 Why Textual?
- CSS-like styling system
- Reactive widgets with automatic updates
- Mouse support and scrolling
- Async-first architecture
- Hot reload during development
- Professional look out of the box

#### 7.2 Install Dependencies
```bash
uv add textual httpx
# or
pip install textual httpx
```

#### 7.3 Project Structure for TUI
```
src/opencode/tui/
├── __init__.py
├── app.py              # Main Textual app
├── screens/
│   ├── __init__.py
│   ├── home.py         # Session list screen
│   └── chat.py         # Chat screen
├── widgets/
│   ├── __init__.py
│   ├── message.py      # Message display widget
│   ├── tool_call.py    # Tool call widget
│   └── input.py        # Enhanced input widget
├── styles.tcss         # Textual CSS styles
└── client.py           # API client for server communication
```

#### 7.4 API Client
```python
# src/opencode/tui/client.py
import httpx
import json
from typing import AsyncGenerator
from dataclasses import dataclass

@dataclass
class StreamEvent:
    type: str  # "text" | "tool_call" | "tool_result" | "done"
    data: dict

class OpenCodeClient:
    def __init__(self, base_url: str = "http://localhost:8080"):
        self.base_url = base_url
        self.client = httpx.AsyncClient(timeout=120)

    async def list_sessions(self) -> list[dict]:
        response = await self.client.get(f"{self.base_url}/api/session")
        return response.json()

    async def create_session(self) -> dict:
        response = await self.client.post(f"{self.base_url}/api/session", json={})
        return response.json()

    async def get_session(self, session_id: str) -> dict:
        response = await self.client.get(f"{self.base_url}/api/session/{session_id}")
        return response.json()

    async def send_message(self, session_id: str, content: str) -> AsyncGenerator[StreamEvent, None]:
        """Stream message response via SSE"""
        async with self.client.stream(
            "POST",
            f"{self.base_url}/api/session/{session_id}/message",
            json={"content": content}
        ) as response:
            async for line in response.aiter_lines():
                if line.startswith("event:"):
                    event_type = line[6:].strip()
                elif line.startswith("data:"):
                    data = json.loads(line[5:])
                    yield StreamEvent(type=event_type, data=data)

    async def close(self):
        await self.client.aclose()
```

#### 7.5 Message Widget
```python
# src/opencode/tui/widgets/message.py
from textual.widgets import Static
from textual.reactive import reactive
from rich.markdown import Markdown
from rich.panel import Panel
from rich.syntax import Syntax

class MessageWidget(Static):
    """Display a single message with role styling"""

    role = reactive("user")
    content = reactive("")

    def __init__(self, role: str, content: str = "", **kwargs):
        super().__init__(**kwargs)
        self.role = role
        self.content = content
        self.classes = role  # Apply CSS class

    def render(self):
        title = "You" if self.role == "user" else "Assistant"
        style = "blue" if self.role == "user" else "green"

        # Try to render markdown for assistant messages
        if self.role == "assistant":
            try:
                rendered = Markdown(self.content)
            except:
                rendered = self.content
        else:
            rendered = self.content

        return Panel(rendered, title=title, border_style=style)

    def append_content(self, text: str):
        """Append text to content (for streaming)"""
        self.content += text
        self.refresh()


class ToolCallWidget(Static):
    """Display a tool call"""

    def __init__(self, name: str, input_data: dict, **kwargs):
        super().__init__(**kwargs)
        self.tool_name = name
        self.input_data = input_data
        self.classes = "tool-call"

    def render(self):
        import json
        input_str = json.dumps(self.input_data, indent=2)[:500]
        return Panel(
            f"[cyan]{self.tool_name}[/cyan]\n{input_str}",
            title="🔧 Tool Call",
            border_style="yellow"
        )


class ToolResultWidget(Static):
    """Display a tool result"""

    def __init__(self, name: str, output: str, **kwargs):
        super().__init__(**kwargs)
        self.tool_name = name
        self.output = output[:2000]  # Truncate
        self.classes = "tool-result"

    def render(self):
        return Panel(
            self.output,
            title=f"✓ {self.tool_name}",
            border_style="green"
        )
```

#### 7.6 Chat Screen
```python
# src/opencode/tui/screens/chat.py
from textual.app import ComposeResult
from textual.screen import Screen
from textual.containers import ScrollableContainer, Horizontal
from textual.widgets import Header, Footer, Input, Static, Button
from textual.binding import Binding
from textual import work

from ..client import OpenCodeClient
from ..widgets.message import MessageWidget, ToolCallWidget, ToolResultWidget

class ChatScreen(Screen):
    """Main chat interface"""

    BINDINGS = [
        Binding("escape", "go_back", "Back"),
        Binding("ctrl+n", "new_session", "New"),
        Binding("ctrl+l", "clear", "Clear"),
    ]

    def __init__(self, session_id: str, client: OpenCodeClient, **kwargs):
        super().__init__(**kwargs)
        self.session_id = session_id
        self.client = client
        self.current_assistant_widget = None

    def compose(self) -> ComposeResult:
        yield Header()
        yield ScrollableContainer(id="messages")
        yield Horizontal(
            Input(placeholder="Type your message...", id="input"),
            Button("Send", id="send", variant="primary"),
            id="input-area"
        )
        yield Footer()

    async def on_mount(self):
        """Load existing messages"""
        session = await self.client.get_session(self.session_id)
        messages = self.query_one("#messages", ScrollableContainer)

        for msg in session.get("messages", []):
            if isinstance(msg["content"], str):
                messages.mount(MessageWidget(role=msg["role"], content=msg["content"]))

        # Focus input
        self.query_one("#input", Input).focus()

    async def on_input_submitted(self, event: Input.Submitted):
        """Handle message submission"""
        await self._send_message(event.value)
        event.input.value = ""

    async def on_button_pressed(self, event: Button.Pressed):
        """Handle send button"""
        if event.button.id == "send":
            input_widget = self.query_one("#input", Input)
            await self._send_message(input_widget.value)
            input_widget.value = ""
            input_widget.focus()

    @work
    async def _send_message(self, content: str):
        """Send message and stream response"""
        if not content.strip():
            return

        messages = self.query_one("#messages", ScrollableContainer)

        # Add user message
        messages.mount(MessageWidget(role="user", content=content))
        messages.scroll_end()

        # Create assistant message widget for streaming
        self.current_assistant_widget = MessageWidget(role="assistant", content="")
        messages.mount(self.current_assistant_widget)

        # Stream response
        async for event in self.client.send_message(self.session_id, content):
            if event.type == "text":
                self.current_assistant_widget.append_content(event.data.get("content", ""))
                messages.scroll_end()

            elif event.type == "tool_call":
                messages.mount(ToolCallWidget(
                    name=event.data.get("name", ""),
                    input_data=event.data.get("input", {})
                ))
                messages.scroll_end()

            elif event.type == "tool_result":
                messages.mount(ToolResultWidget(
                    name=event.data.get("name", ""),
                    output=event.data.get("output", "")
                ))
                messages.scroll_end()

            elif event.type == "done":
                self.current_assistant_widget = None

    def action_go_back(self):
        """Return to home screen"""
        self.app.pop_screen()

    def action_clear(self):
        """Clear messages"""
        messages = self.query_one("#messages", ScrollableContainer)
        messages.remove_children()
```

#### 7.7 Home Screen
```python
# src/opencode/tui/screens/home.py
from textual.app import ComposeResult
from textual.screen import Screen
from textual.containers import Container, Vertical
from textual.widgets import Header, Footer, Static, Button, DataTable
from textual.binding import Binding
from textual import work

from ..client import OpenCodeClient

class HomeScreen(Screen):
    """Session list and selection"""

    BINDINGS = [
        Binding("n", "new_session", "New Session"),
        Binding("r", "refresh", "Refresh"),
        Binding("q", "quit", "Quit"),
    ]

    def __init__(self, client: OpenCodeClient, **kwargs):
        super().__init__(**kwargs)
        self.client = client

    def compose(self) -> ComposeResult:
        yield Header()
        yield Container(
            Static("[bold]OpenCode[/bold] - AI Coding Assistant", id="title"),
            DataTable(id="sessions"),
            Button("+ New Session", id="new-session", variant="primary"),
            id="main"
        )
        yield Footer()

    async def on_mount(self):
        """Load sessions on mount"""
        await self._load_sessions()

    @work
    async def _load_sessions(self):
        """Load sessions into table"""
        table = self.query_one("#sessions", DataTable)
        table.clear(columns=True)
        table.add_columns("ID", "Title", "Messages", "Created")
        table.cursor_type = "row"

        sessions = await self.client.list_sessions()
        for s in sessions:
            table.add_row(
                s["id"],
                s.get("title", "Untitled")[:30],
                str(s.get("message_count", 0)),
                s["created_at"][:16]
            )

    async def on_data_table_row_selected(self, event: DataTable.RowSelected):
        """Open selected session"""
        row_key = event.row_key
        table = self.query_one("#sessions", DataTable)
        session_id = table.get_cell(row_key, "ID")

        from .chat import ChatScreen
        self.app.push_screen(ChatScreen(session_id, self.client))

    async def on_button_pressed(self, event: Button.Pressed):
        """Handle new session button"""
        if event.button.id == "new-session":
            await self.action_new_session()

    @work
    async def action_new_session(self):
        """Create new session and open it"""
        session = await self.client.create_session()
        from .chat import ChatScreen
        self.app.push_screen(ChatScreen(session["id"], self.client))

    async def action_refresh(self):
        """Refresh session list"""
        await self._load_sessions()
```

#### 7.8 Main App
```python
# src/opencode/tui/app.py
from textual.app import App
from textual.binding import Binding

from .client import OpenCodeClient
from .screens.home import HomeScreen

class OpenCodeTUI(App):
    """OpenCode Terminal UI Application"""

    TITLE = "OpenCode"
    CSS_PATH = "styles.tcss"

    BINDINGS = [
        Binding("ctrl+q", "quit", "Quit", show=True),
        Binding("ctrl+d", "toggle_dark", "Dark Mode"),
    ]

    def __init__(self, server_url: str = "http://localhost:8080"):
        super().__init__()
        self.client = OpenCodeClient(server_url)

    async def on_mount(self):
        """Push home screen on start"""
        self.push_screen(HomeScreen(self.client))

    async def on_unmount(self):
        """Cleanup client on exit"""
        await self.client.close()

    def action_toggle_dark(self):
        """Toggle dark mode"""
        self.dark = not self.dark


def run_tui(server_url: str = "http://localhost:8080"):
    """Entry point for TUI"""
    app = OpenCodeTUI(server_url)
    app.run()
```

#### 7.9 Textual CSS Styles
```css
/* src/opencode/tui/styles.tcss */

Screen {
    background: $surface;
}

#title {
    text-align: center;
    padding: 1;
    text-style: bold;
    color: $text;
}

#main {
    padding: 1 2;
}

#messages {
    height: 1fr;
    border: solid $primary;
    padding: 1;
    margin-bottom: 1;
}

#input-area {
    height: auto;
    dock: bottom;
}

#input-area Input {
    width: 1fr;
}

#input-area Button {
    width: auto;
    margin-left: 1;
}

/* Message styling */
.user {
    margin: 1 0;
    background: $primary-darken-2;
}

.assistant {
    margin: 1 0;
    background: $surface-darken-1;
}

.tool-call {
    margin: 0 2;
    background: $warning-darken-2;
}

.tool-result {
    margin: 0 2;
    background: $success-darken-2;
}

/* Session table */
#sessions {
    height: auto;
    max-height: 70%;
    margin: 1 0;
}

DataTable > .datatable--cursor {
    background: $primary;
}

/* Buttons */
#new-session {
    margin-top: 1;
    width: 100%;
}
```

#### 7.10 CLI Integration
```python
# Add to cli.py

@app.command()
def ui(
    server: str = typer.Option("http://localhost:8080", "--server", "-s",
                               help="Server URL to connect to"),
):
    """Launch Textual TUI (requires server running)"""
    from .tui.app import run_tui
    console.print(f"[dim]Connecting to {server}...[/dim]")
    run_tui(server)
```

#### 7.11 Standalone Mode (No Server)
```python
# src/opencode/tui/standalone.py
"""Standalone TUI that doesn't require a server"""

from textual.app import App, ComposeResult
from textual.containers import ScrollableContainer, Horizontal
from textual.widgets import Header, Footer, Input, Button
from textual.binding import Binding
from textual import work

from ..agent import run_agent_loop_async
from ..provider import AsyncProvider
from ..models import Session
from ..storage import save_session
from .widgets.message import MessageWidget, ToolCallWidget, ToolResultWidget
import uuid

class StandaloneTUI(App):
    """TUI that runs agent directly without server"""

    CSS_PATH = "styles.tcss"
    BINDINGS = [Binding("ctrl+q", "quit", "Quit")]

    def __init__(self):
        super().__init__()
        self.session = Session(id=str(uuid.uuid4())[:8])
        self.provider = AsyncProvider()

    def compose(self) -> ComposeResult:
        yield Header()
        yield ScrollableContainer(id="messages")
        yield Horizontal(
            Input(placeholder="Type your message...", id="input"),
            Button("Send", id="send", variant="primary"),
            id="input-area"
        )
        yield Footer()

    async def on_input_submitted(self, event: Input.Submitted):
        await self._send_message(event.value)
        event.input.value = ""

    @work
    async def _send_message(self, content: str):
        if not content.strip():
            return

        messages = self.query_one("#messages", ScrollableContainer)

        # User message
        messages.mount(MessageWidget(role="user", content=content))

        # Assistant streaming
        assistant_widget = MessageWidget(role="assistant", content="")
        messages.mount(assistant_widget)

        async for event in run_agent_loop_async(self.session, content, self.provider):
            if event["type"] == "text":
                assistant_widget.append_content(event["content"])
            elif event["type"] == "tool_call":
                messages.mount(ToolCallWidget(event["name"], event["input"]))
            elif event["type"] == "tool_result":
                messages.mount(ToolResultWidget(event["name"], event["output"]))

            messages.scroll_end()

        save_session(self.session)


def run_standalone():
    StandaloneTUI().run()
```

```python
# Add to cli.py

@app.command()
def tui():
    """Launch standalone Textual TUI (no server needed)"""
    from .tui.standalone import run_standalone
    run_standalone()
```

**Checkpoint: Beautiful Textual TUI with full features!**

---

## Final Project Structure

```
opencode-py/
├── pyproject.toml
├── README.md
├── src/
│   └── opencode/
│       ├── __init__.py           # Version info
│       ├── __main__.py           # Entry: python -m opencode
│       ├── cli.py                # Typer CLI (~150 lines)
│       ├── agent.py              # Agent loop (~300 lines)
│       ├── provider.py           # Anthropic client (~120 lines)
│       ├── tools.py              # All tools (~400 lines)
│       ├── storage.py            # JSON storage (~80 lines)
│       ├── config.py             # Configuration (~60 lines)
│       ├── models.py             # Pydantic models (~100 lines)
│       ├── bus.py                # Event bus (~60 lines)
│       │
│       ├── server/               # HTTP Backend
│       │   ├── __init__.py
│       │   └── app.py            # FastAPI app (~250 lines)
│       │
│       └── tui/                  # Textual TUI
│           ├── __init__.py
│           ├── app.py            # Main app (~50 lines)
│           ├── standalone.py     # Standalone mode (~80 lines)
│           ├── client.py         # API client (~60 lines)
│           ├── styles.tcss       # CSS styles (~80 lines)
│           ├── screens/
│           │   ├── __init__.py
│           │   ├── home.py       # Session list (~100 lines)
│           │   └── chat.py       # Chat screen (~120 lines)
│           └── widgets/
│               ├── __init__.py
│               └── message.py    # Message widgets (~80 lines)
│
└── tests/
    ├── test_tools.py
    ├── test_storage.py
    └── test_server.py

Total: ~2,100 lines of Python
```

---

## Dependencies

```toml
# pyproject.toml
[project]
name = "opencode"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    # Core
    "anthropic>=0.25.0",
    "typer>=0.9.0",
    "rich>=13.0.0",
    "pydantic>=2.0.0",
    "prompt-toolkit>=3.0.0",

    # Server
    "fastapi>=0.109.0",
    "uvicorn>=0.27.0",
    "sse-starlette>=1.8.0",

    # TUI
    "textual>=0.47.0",
    "httpx>=0.26.0",
]

[project.optional-dependencies]
web = ["beautifulsoup4>=4.12.0"]  # For web_fetch tool
dev = ["pytest>=7.0.0", "pytest-asyncio>=0.23.0", "ruff>=0.1.0"]

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

# CLI mode - one-shot task
opencode run "Create a hello world Python script"

# CLI mode - interactive chat
opencode chat

# Standalone TUI (no server needed)
opencode tui

# Resume session
opencode chat --session abc123

# Start HTTP server (for remote TUI access)
opencode serve --port 8080

# Textual TUI (connects to server)
opencode ui --server http://localhost:8080
```

---

## What's Next (Post-MVP)

After MVP is working, add in order of value:

1. **More Providers** - OpenAI, local models via Ollama
2. **grep Tool** - Proper ripgrep integration
3. **LSP Integration** - Show errors after edits
4. **MCP Support** - External tool servers
5. **Authentication** - OAuth for multi-user support
6. **Plugin System** - Extensible tool loading

---

## Timeline Summary

| Phase | Days | Milestone |
|-------|------|-----------|
| 1 | 1-3 | Basic chat working |
| 2 | 4-7 | Tools working (read/write/bash/glob) |
| 3 | 8-10 | Rich TUI with sessions |
| 4 | 11-14 | Config, polish, more tools |
| 5 | 15-17 | Web fetch, tree, testing |
| 6 | 18-22 | **HTTP Backend (FastAPI + SSE)** |
| 7 | 23-26 | **Textual TUI (standalone + server mode)** |
| **26** | | **Full MVP Complete** |

**~4.5 weeks to a fully functional AI coding assistant with CLI, TUI, and API!**

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        OpenCode Python                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Clients                                                        │
│   ┌─────────┐  ┌─────────┐  ┌────────────┐                     │
│   │   CLI   │  │Rich TUI │  │Textual TUI │                     │
│   │ (Typer) │  │ (Rich)  │  │ (Textual)  │                     │
│   └────┬────┘  └────┬────┘  └─────┬──────┘                     │
│        │            │             │                              │
│        │    Direct  │             │  HTTP (optional)            │
│        └────────────┤             ├────────────┐                │
│                     │             │            │                 │
│                     ▼             ▼            ▼                 │
│   ┌─────────────────────────┐  ┌───────────────────────────┐   │
│   │      Core Layer         │  │     FastAPI Server        │   │
│   │  (Standalone Mode)      │  │  • REST API endpoints     │   │
│   │                         │  │  • SSE streaming          │   │
│   └───────────┬─────────────┘  └─────────────┬─────────────┘   │
│               │                              │                   │
│               └──────────────┬───────────────┘                  │
│                              │                                   │
│   ┌──────────────────────────┼───────────────────────────────┐  │
│   │                    Core Layer                             │  │
│   │  ┌──────────┐  ┌──────────┐  ┌──────────┐               │  │
│   │  │  Agent   │  │ Session  │  │   Tool   │               │  │
│   │  │  Loop    │  │ Manager  │  │ Registry │               │  │
│   │  └────┬─────┘  └────┬─────┘  └────┬─────┘               │  │
│   │       │             │             │                       │  │
│   │       ▼             ▼             ▼                       │  │
│   │  ┌──────────┐  ┌──────────┐  ┌──────────┐               │  │
│   │  │ Provider │  │ Storage  │  │Event Bus │               │  │
│   │  │(Anthropic│  │  (JSON)  │  │  (Pub/   │               │  │
│   │  │   SDK)   │  │          │  │   Sub)   │               │  │
│   │  └──────────┘  └──────────┘  └──────────┘               │  │
│   └──────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```
