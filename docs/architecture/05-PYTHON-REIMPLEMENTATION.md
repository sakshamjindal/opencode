# Python Reimplementation Plan

## Executive Summary

This document outlines a comprehensive plan to reimplement OpenCode in Python. The reimplementation aims to maintain feature parity while leveraging Python's ecosystem strengths: rich ML/AI libraries, mature async frameworks, and extensive developer tooling.

## 1. Technology Stack Recommendations

### Core Runtime & Framework
| Component | TypeScript (Current) | Python (Proposed) | Rationale |
|-----------|---------------------|-------------------|-----------|
| Runtime | Bun | Python 3.11+ | Native async, typing, performance |
| Package Manager | Bun Workspaces | uv / Poetry | Modern dependency management |
| Type System | TypeScript | Pydantic + typing | Runtime validation + static types |
| CLI Framework | Yargs | Typer / Click | Modern CLI with type hints |
| HTTP Server | Hono | FastAPI / Litestar | Async-first, OpenAPI support |
| Async Runtime | Native | asyncio + uvloop | High-performance event loop |

### Frontend & UI
| Component | TypeScript (Current) | Python (Proposed) | Rationale |
|-----------|---------------------|-------------------|-----------|
| TUI Framework | OpenTUI + SolidJS | Textual / Rich | Python-native TUI |
| Desktop App | Tauri | PyQt6 / Flet | Cross-platform GUI |
| Web Frontend | SolidJS + Vite | Keep or HTMX + FastAPI | Minimal rewrite needed |

### Data & Storage
| Component | TypeScript (Current) | Python (Proposed) | Rationale |
|-----------|---------------------|-------------------|-----------|
| Validation | Zod | Pydantic v2 | Fastest Python validation |
| JSON Storage | Bun.file | aiofiles + orjson | Async I/O, fast JSON |
| Database | PlanetScale | SQLAlchemy + asyncpg | Async ORM support |
| Caching | In-memory + file | Redis / diskcache | Distributed caching |

### AI/LLM Integration
| Component | TypeScript (Current) | Python (Proposed) | Rationale |
|-----------|---------------------|-------------------|-----------|
| LLM SDK | Vercel AI SDK | LiteLLM / Anthropic SDK | Multi-provider support |
| Streaming | AsyncGenerator | AsyncGenerator | Direct equivalent |
| Tool Calling | AI SDK Tools | Function calling | Native provider support |

---

## 2. Project Structure

### Proposed Directory Layout
```
opencode-python/
├── pyproject.toml              # Project configuration (uv/Poetry)
├── src/
│   └── opencode/
│       ├── __init__.py
│       ├── __main__.py         # CLI entry point
│       │
│       ├── cli/                # CLI Commands
│       │   ├── __init__.py
│       │   ├── main.py         # Typer app
│       │   ├── run.py          # Run command
│       │   ├── serve.py        # Server command
│       │   └── tui.py          # TUI command
│       │
│       ├── core/               # Core Business Logic
│       │   ├── __init__.py
│       │   ├── agent.py        # Agent system
│       │   ├── session.py      # Session management
│       │   ├── message.py      # Message handling
│       │   └── processor.py    # Session processor
│       │
│       ├── tools/              # Tool System
│       │   ├── __init__.py
│       │   ├── base.py         # Tool base class
│       │   ├── registry.py     # Tool registry
│       │   ├── builtin/        # Built-in tools
│       │   │   ├── read.py
│       │   │   ├── write.py
│       │   │   ├── edit.py
│       │   │   ├── bash.py
│       │   │   └── ...
│       │   └── mcp.py          # MCP tool integration
│       │
│       ├── providers/          # LLM Providers
│       │   ├── __init__.py
│       │   ├── base.py         # Provider interface
│       │   ├── anthropic.py
│       │   ├── openai.py
│       │   └── ...
│       │
│       ├── server/             # HTTP Server
│       │   ├── __init__.py
│       │   ├── app.py          # FastAPI app
│       │   ├── routes/         # API routes
│       │   └── sse.py          # SSE handling
│       │
│       ├── storage/            # Persistence
│       │   ├── __init__.py
│       │   ├── base.py         # Storage interface
│       │   ├── json_store.py   # JSON file storage
│       │   └── lock.py         # Read-write locks
│       │
│       ├── bus/                # Event System
│       │   ├── __init__.py
│       │   └── events.py       # Event bus
│       │
│       ├── config/             # Configuration
│       │   ├── __init__.py
│       │   └── schema.py       # Pydantic models
│       │
│       ├── permission/         # Permission System
│       │   ├── __init__.py
│       │   └── rules.py        # Permission rules
│       │
│       ├── lsp/                # LSP Integration
│       │   ├── __init__.py
│       │   ├── client.py       # LSP client
│       │   └── servers.py      # Server definitions
│       │
│       ├── mcp/                # MCP Integration
│       │   ├── __init__.py
│       │   ├── client.py       # MCP client
│       │   └── oauth.py        # OAuth handling
│       │
│       └── tui/                # Terminal UI
│           ├── __init__.py
│           ├── app.py          # Textual app
│           ├── screens/        # UI screens
│           └── widgets/        # Custom widgets
│
├── tests/                      # Test suite
│   ├── conftest.py
│   ├── unit/
│   └── integration/
│
└── docs/                       # Documentation
```

---

## 3. Implementation Phases

### Phase 1: Core Foundation (4-6 weeks)

#### 1.1 Project Setup
```python
# pyproject.toml
[project]
name = "opencode"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "typer>=0.9.0",
    "pydantic>=2.0",
    "fastapi>=0.100",
    "uvicorn>=0.23",
    "aiofiles>=23.0",
    "orjson>=3.9",
    "httpx>=0.24",
    "sse-starlette>=1.6",
]

[project.optional-dependencies]
llm = [
    "anthropic>=0.25",
    "openai>=1.0",
    "litellm>=1.0",
]
tui = [
    "textual>=0.40",
    "rich>=13.0",
]
```

#### 1.2 Configuration System
```python
# src/opencode/config/schema.py
from pydantic import BaseModel, Field
from typing import Optional, Dict, Any
from pathlib import Path

class ProviderConfig(BaseModel):
    api_key: Optional[str] = None
    base_url: Optional[str] = None
    options: Dict[str, Any] = Field(default_factory=dict)

class AgentConfig(BaseModel):
    id: str
    type: str = "primary"  # primary | subagent | all
    model: Dict[str, str]  # {provider, model}
    permission: list = Field(default_factory=list)
    enabled: bool = True

class Config(BaseModel):
    provider: Dict[str, ProviderConfig] = Field(default_factory=dict)
    agent: Dict[str, AgentConfig] = Field(default_factory=dict)
    permission: list = Field(default_factory=list)
    mcp: Dict[str, Any] = Field(default_factory=dict)
    lsp: Dict[str, Any] | bool = Field(default_factory=dict)
    plugin: list[str] = Field(default_factory=list)

class ConfigLoader:
    @staticmethod
    async def load(directory: Path) -> Config:
        """Load config with precedence: project > global > remote"""
        ...
```

#### 1.3 Storage Layer
```python
# src/opencode/storage/json_store.py
import asyncio
import aiofiles
import orjson
from pathlib import Path
from typing import TypeVar, Generic, Callable
from contextlib import asynccontextmanager

T = TypeVar('T')

class RWLock:
    """Async read-write lock with writer priority"""
    def __init__(self):
        self._read_count = 0
        self._write_lock = asyncio.Lock()
        self._read_lock = asyncio.Lock()

    @asynccontextmanager
    async def read(self):
        async with self._read_lock:
            self._read_count += 1
            if self._read_count == 1:
                await self._write_lock.acquire()
        try:
            yield
        finally:
            async with self._read_lock:
                self._read_count -= 1
                if self._read_count == 0:
                    self._write_lock.release()

    @asynccontextmanager
    async def write(self):
        async with self._write_lock:
            yield

class Storage:
    def __init__(self, base_dir: Path):
        self.base_dir = base_dir
        self._locks: dict[str, RWLock] = {}

    def _get_lock(self, key: str) -> RWLock:
        if key not in self._locks:
            self._locks[key] = RWLock()
        return self._locks[key]

    async def read(self, *key: str) -> dict:
        path = self.base_dir.joinpath(*key).with_suffix('.json')
        lock = self._get_lock(str(path))
        async with lock.read():
            async with aiofiles.open(path, 'rb') as f:
                return orjson.loads(await f.read())

    async def write(self, *key: str, data: dict) -> None:
        path = self.base_dir.joinpath(*key).with_suffix('.json')
        path.parent.mkdir(parents=True, exist_ok=True)
        lock = self._get_lock(str(path))
        async with lock.write():
            async with aiofiles.open(path, 'wb') as f:
                await f.write(orjson.dumps(data, option=orjson.OPT_INDENT_2))

    async def update(self, *key: str, fn: Callable[[dict], None]) -> dict:
        path = self.base_dir.joinpath(*key).with_suffix('.json')
        lock = self._get_lock(str(path))
        async with lock.write():
            async with aiofiles.open(path, 'rb') as f:
                data = orjson.loads(await f.read())
            fn(data)
            async with aiofiles.open(path, 'wb') as f:
                await f.write(orjson.dumps(data, option=orjson.OPT_INDENT_2))
            return data
```

#### 1.4 Event Bus
```python
# src/opencode/bus/events.py
from typing import TypeVar, Generic, Callable, Any
from pydantic import BaseModel
import asyncio
from collections import defaultdict

T = TypeVar('T', bound=BaseModel)

class EventBus:
    def __init__(self):
        self._subscribers: dict[str, list[Callable]] = defaultdict(list)
        self._global_subscribers: list[Callable] = []

    def subscribe(self, event_type: str, callback: Callable[[Any], None]):
        self._subscribers[event_type].append(callback)
        return lambda: self._subscribers[event_type].remove(callback)

    def subscribe_all(self, callback: Callable[[str, Any], None]):
        self._global_subscribers.append(callback)
        return lambda: self._global_subscribers.remove(callback)

    async def publish(self, event_type: str, data: BaseModel):
        # Notify specific subscribers
        for callback in self._subscribers[event_type]:
            if asyncio.iscoroutinefunction(callback):
                await callback(data)
            else:
                callback(data)

        # Notify global subscribers
        for callback in self._global_subscribers:
            if asyncio.iscoroutinefunction(callback):
                await callback(event_type, data)
            else:
                callback(event_type, data)

# Global bus instance
bus = EventBus()
```

---

### Phase 2: Tool System (3-4 weeks)

#### 2.1 Tool Base Class
```python
# src/opencode/tools/base.py
from abc import ABC, abstractmethod
from typing import TypeVar, Generic, Any
from pydantic import BaseModel
import asyncio

class ToolContext:
    def __init__(
        self,
        session_id: str,
        message_id: str,
        agent: str,
        abort_event: asyncio.Event
    ):
        self.session_id = session_id
        self.message_id = message_id
        self.agent = agent
        self.abort_event = abort_event
        self._metadata_callback: Callable | None = None

    def set_metadata_callback(self, cb: Callable):
        self._metadata_callback = cb

    async def metadata(self, title: str | None = None, **kwargs):
        if self._metadata_callback:
            await self._metadata_callback(title=title, metadata=kwargs)

    async def ask_permission(
        self,
        permission: str,
        patterns: list[str],
        always_patterns: list[str] | None = None
    ) -> None:
        """Request permission - raises PermissionDenied if denied"""
        from ..permission import check_permission
        await check_permission(
            self.session_id,
            permission,
            patterns,
            always_patterns
        )

P = TypeVar('P', bound=BaseModel)
M = TypeVar('M', bound=BaseModel)

class Tool(ABC, Generic[P, M]):
    """Base class for all tools"""

    id: str
    description: str
    parameters: type[P]

    @abstractmethod
    async def execute(self, args: P, ctx: ToolContext) -> tuple[str, M]:
        """Execute the tool and return (output, metadata)"""
        pass

    def format_validation_error(self, error: Any) -> str:
        """Optional: Custom error formatting"""
        return str(error)
```

#### 2.2 Built-in Tools
```python
# src/opencode/tools/builtin/read.py
from pathlib import Path
from pydantic import BaseModel, Field
from ..base import Tool, ToolContext

class ReadParams(BaseModel):
    path: str = Field(description="File path to read")
    offset: int | None = Field(None, description="Start line (1-based)")
    limit: int | None = Field(None, description="Max lines to read")

class ReadMetadata(BaseModel):
    path: str
    lines: int
    truncated: bool = False

class ReadTool(Tool[ReadParams, ReadMetadata]):
    id = "read"
    description = "Read file contents"
    parameters = ReadParams

    async def execute(self, args: ReadParams, ctx: ToolContext) -> tuple[str, ReadMetadata]:
        # Check permission
        await ctx.ask_permission("read", [args.path])

        # Read file
        path = Path(args.path)
        content = path.read_text()
        lines = content.split('\n')

        # Apply offset/limit
        if args.offset:
            lines = lines[args.offset - 1:]
        if args.limit:
            lines = lines[:args.limit]

        # Format with line numbers
        output = '\n'.join(
            f"{i + (args.offset or 1):4d}\t{line}"
            for i, line in enumerate(lines)
        )

        await ctx.metadata(title=f"Read {path.name}")

        return output, ReadMetadata(
            path=str(path),
            lines=len(lines),
            truncated=args.limit is not None and len(lines) >= args.limit
        )
```

```python
# src/opencode/tools/builtin/bash.py
import asyncio
import subprocess
from pydantic import BaseModel, Field
from ..base import Tool, ToolContext

class BashParams(BaseModel):
    command: str = Field(description="Shell command to execute")
    timeout: int = Field(120_000, description="Timeout in ms")
    workdir: str | None = Field(None, description="Working directory")
    background: bool = Field(False, description="Run in background")

class BashMetadata(BaseModel):
    output: str
    exit_code: int | None
    description: str | None = None

class BashTool(Tool[BashParams, BashMetadata]):
    id = "bash"
    description = "Execute shell commands"
    parameters = BashParams

    async def execute(self, args: BashParams, ctx: ToolContext) -> tuple[str, BashMetadata]:
        # Extract command patterns for permission
        patterns = self._extract_patterns(args.command)
        await ctx.ask_permission("bash", patterns)

        # Execute command
        process = await asyncio.create_subprocess_shell(
            args.command,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.STDOUT,
            cwd=args.workdir
        )

        output = ""
        try:
            async def stream_output():
                nonlocal output
                while True:
                    line = await process.stdout.readline()
                    if not line:
                        break
                    output += line.decode()
                    await ctx.metadata(metadata={"output": output})

            await asyncio.wait_for(
                stream_output(),
                timeout=args.timeout / 1000
            )
            await process.wait()

        except asyncio.TimeoutError:
            process.kill()
            output += "\n[TIMEOUT]"

        return output, BashMetadata(
            output=output,
            exit_code=process.returncode
        )

    def _extract_patterns(self, command: str) -> list[str]:
        # Extract command patterns for permission checking
        parts = command.split()
        if not parts:
            return []
        # Simple arity handling
        return [parts[0]]
```

#### 2.3 Tool Registry
```python
# src/opencode/tools/registry.py
from typing import Dict, List
from pathlib import Path
import importlib.util
from .base import Tool
from .builtin import read, write, edit, bash, glob, grep, ...

class ToolRegistry:
    def __init__(self):
        self._tools: Dict[str, Tool] = {}
        self._register_builtins()

    def _register_builtins(self):
        builtins = [
            read.ReadTool(),
            write.WriteTool(),
            edit.EditTool(),
            bash.BashTool(),
            glob.GlobTool(),
            grep.GrepTool(),
            # ... more builtins
        ]
        for tool in builtins:
            self._tools[tool.id] = tool

    async def load_custom_tools(self, directories: List[Path]):
        """Load custom tools from tool/*.py files"""
        for directory in directories:
            tool_dir = directory / "tool"
            if not tool_dir.exists():
                continue

            for file in tool_dir.glob("*.py"):
                spec = importlib.util.spec_from_file_location(
                    file.stem, file
                )
                module = importlib.util.module_from_spec(spec)
                spec.loader.exec_module(module)

                for name, obj in vars(module).items():
                    if isinstance(obj, Tool):
                        self._tools[obj.id] = obj

    def get(self, tool_id: str) -> Tool | None:
        return self._tools.get(tool_id)

    def all(self) -> List[Tool]:
        return list(self._tools.values())

# Global registry
registry = ToolRegistry()
```

---

### Phase 3: Provider System (3-4 weeks)

#### 3.1 Provider Interface
```python
# src/opencode/providers/base.py
from abc import ABC, abstractmethod
from typing import AsyncGenerator, Any
from pydantic import BaseModel
from dataclasses import dataclass

@dataclass
class ToolCall:
    id: str
    name: str
    arguments: dict

@dataclass
class StreamPart:
    type: str  # "text" | "reasoning" | "tool-call" | "tool-result"
    content: str | ToolCall | None = None

class ModelInfo(BaseModel):
    id: str
    name: str
    provider: str
    supports_tools: bool = True
    supports_vision: bool = False
    supports_reasoning: bool = False
    cost_input: float | None = None
    cost_output: float | None = None

class Message(BaseModel):
    role: str  # "user" | "assistant" | "system" | "tool"
    content: str | list
    tool_call_id: str | None = None

class Provider(ABC):
    """Base class for LLM providers"""

    id: str
    name: str

    @abstractmethod
    async def models(self) -> list[ModelInfo]:
        """List available models"""
        pass

    @abstractmethod
    async def stream(
        self,
        model: str,
        messages: list[Message],
        tools: list[dict] | None = None,
        **kwargs
    ) -> AsyncGenerator[StreamPart, None]:
        """Stream a completion"""
        pass

    @abstractmethod
    async def complete(
        self,
        model: str,
        messages: list[Message],
        tools: list[dict] | None = None,
        **kwargs
    ) -> tuple[str, dict]:
        """Non-streaming completion"""
        pass
```

#### 3.2 Anthropic Provider
```python
# src/opencode/providers/anthropic.py
from anthropic import AsyncAnthropic
from .base import Provider, StreamPart, ModelInfo, Message, ToolCall
from typing import AsyncGenerator

class AnthropicProvider(Provider):
    id = "anthropic"
    name = "Anthropic"

    def __init__(self, api_key: str | None = None):
        self.client = AsyncAnthropic(api_key=api_key)

    async def models(self) -> list[ModelInfo]:
        return [
            ModelInfo(
                id="claude-sonnet-4-20250514",
                name="Claude Sonnet 4",
                provider=self.id,
                supports_reasoning=True,
                cost_input=3.0,
                cost_output=15.0
            ),
            ModelInfo(
                id="claude-3-5-sonnet-20241022",
                name="Claude 3.5 Sonnet",
                provider=self.id,
                supports_vision=True,
                cost_input=3.0,
                cost_output=15.0
            ),
            # ... more models
        ]

    async def stream(
        self,
        model: str,
        messages: list[Message],
        tools: list[dict] | None = None,
        **kwargs
    ) -> AsyncGenerator[StreamPart, None]:
        # Convert messages to Anthropic format
        anthropic_messages = self._convert_messages(messages)

        async with self.client.messages.stream(
            model=model,
            messages=anthropic_messages,
            tools=self._convert_tools(tools) if tools else None,
            max_tokens=kwargs.get("max_tokens", 8192),
            **kwargs
        ) as stream:
            async for event in stream:
                if event.type == "content_block_delta":
                    if hasattr(event.delta, "text"):
                        yield StreamPart(type="text", content=event.delta.text)
                    elif hasattr(event.delta, "thinking"):
                        yield StreamPart(type="reasoning", content=event.delta.thinking)

                elif event.type == "content_block_start":
                    if event.content_block.type == "tool_use":
                        yield StreamPart(
                            type="tool-call",
                            content=ToolCall(
                                id=event.content_block.id,
                                name=event.content_block.name,
                                arguments={}
                            )
                        )

    def _convert_messages(self, messages: list[Message]) -> list[dict]:
        # Convert to Anthropic format
        return [{"role": m.role, "content": m.content} for m in messages]

    def _convert_tools(self, tools: list[dict]) -> list[dict]:
        # Convert to Anthropic tool format
        return tools
```

#### 3.3 Provider Registry
```python
# src/opencode/providers/__init__.py
from typing import Dict
from .base import Provider
from .anthropic import AnthropicProvider
from .openai import OpenAIProvider
# ... more providers

class ProviderRegistry:
    def __init__(self):
        self._providers: Dict[str, type[Provider]] = {
            "anthropic": AnthropicProvider,
            "openai": OpenAIProvider,
            # ... more providers
        }
        self._instances: Dict[str, Provider] = {}

    def get(self, provider_id: str, config: dict | None = None) -> Provider:
        if provider_id not in self._instances:
            provider_class = self._providers.get(provider_id)
            if not provider_class:
                raise ValueError(f"Unknown provider: {provider_id}")
            self._instances[provider_id] = provider_class(**(config or {}))
        return self._instances[provider_id]

    def register(self, provider_id: str, provider_class: type[Provider]):
        self._providers[provider_id] = provider_class

providers = ProviderRegistry()
```

---

### Phase 4: Session & Agent System (3-4 weeks)

#### 4.1 Session Management
```python
# src/opencode/core/session.py
from pydantic import BaseModel, Field
from datetime import datetime
from typing import Optional
from ..storage import Storage
from ..bus import bus

class SessionInfo(BaseModel):
    id: str
    project_id: str
    directory: str
    parent_id: Optional[str] = None
    title: str = ""
    created_at: datetime = Field(default_factory=datetime.utcnow)
    updated_at: datetime = Field(default_factory=datetime.utcnow)
    archived_at: Optional[datetime] = None

class Session:
    def __init__(self, storage: Storage):
        self.storage = storage

    async def create(self, project_id: str, directory: str) -> SessionInfo:
        session = SessionInfo(
            id=self._generate_id(),
            project_id=project_id,
            directory=directory
        )
        await self.storage.write(
            "session", project_id, session.id,
            data=session.model_dump(mode="json")
        )
        await bus.publish("session.created", session)
        return session

    async def get(self, session_id: str, project_id: str) -> SessionInfo:
        data = await self.storage.read("session", project_id, session_id)
        return SessionInfo.model_validate(data)

    async def update(self, session: SessionInfo) -> SessionInfo:
        session.updated_at = datetime.utcnow()
        await self.storage.write(
            "session", session.project_id, session.id,
            data=session.model_dump(mode="json")
        )
        await bus.publish("session.updated", session)
        return session

    async def list(self, project_id: str) -> list[SessionInfo]:
        keys = await self.storage.list("session", project_id)
        sessions = []
        for key in keys:
            data = await self.storage.read(*key)
            sessions.append(SessionInfo.model_validate(data))
        return sorted(sessions, key=lambda s: s.updated_at, reverse=True)

    def _generate_id(self) -> str:
        import uuid
        return f"session_{uuid.uuid4().hex[:12]}"
```

#### 4.2 Session Processor
```python
# src/opencode/core/processor.py
from typing import AsyncGenerator
from ..providers.base import Provider, StreamPart, Message, ToolCall
from ..tools import registry, ToolContext
from ..bus import bus
import asyncio

class SessionProcessor:
    def __init__(
        self,
        provider: Provider,
        model: str,
        session_id: str,
        agent: str
    ):
        self.provider = provider
        self.model = model
        self.session_id = session_id
        self.agent = agent
        self.abort_event = asyncio.Event()

    async def process(
        self,
        messages: list[Message],
        tools: list[dict]
    ) -> AsyncGenerator[StreamPart, None]:
        """Process a conversation turn with streaming"""

        pending_tool_calls: dict[str, ToolCall] = {}

        async for part in self.provider.stream(
            model=self.model,
            messages=messages,
            tools=tools
        ):
            yield part

            if part.type == "tool-call" and isinstance(part.content, ToolCall):
                pending_tool_calls[part.content.id] = part.content

        # Execute tool calls
        for call_id, tool_call in pending_tool_calls.items():
            tool = registry.get(tool_call.name)
            if not tool:
                yield StreamPart(
                    type="tool-result",
                    content=f"Unknown tool: {tool_call.name}"
                )
                continue

            ctx = ToolContext(
                session_id=self.session_id,
                message_id="",  # Set by caller
                agent=self.agent,
                abort_event=self.abort_event
            )

            try:
                # Validate and execute
                params = tool.parameters.model_validate(tool_call.arguments)
                output, metadata = await tool.execute(params, ctx)

                yield StreamPart(
                    type="tool-result",
                    content=output
                )

            except Exception as e:
                yield StreamPart(
                    type="tool-result",
                    content=f"Error: {str(e)}"
                )

    def abort(self):
        self.abort_event.set()
```

---

### Phase 5: HTTP Server (2-3 weeks)

#### 5.1 FastAPI Application
```python
# src/opencode/server/app.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from sse_starlette.sse import EventSourceResponse
from contextlib import asynccontextmanager
from .routes import session, message, tool, provider
from ..bus import bus

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    yield
    # Shutdown

app = FastAPI(
    title="OpenCode",
    version="0.1.0",
    lifespan=lifespan
)

# CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)

# Routes
app.include_router(session.router, prefix="/session", tags=["session"])
app.include_router(message.router, prefix="/message", tags=["message"])
app.include_router(tool.router, prefix="/tool", tags=["tool"])
app.include_router(provider.router, prefix="/provider", tags=["provider"])

# SSE Events
@app.get("/event")
async def event_stream():
    async def generate():
        queue = asyncio.Queue()

        def handler(event_type: str, data):
            queue.put_nowait((event_type, data))

        unsubscribe = bus.subscribe_all(handler)

        try:
            while True:
                event_type, data = await queue.get()
                yield {
                    "event": event_type,
                    "data": data.model_dump_json()
                }
        finally:
            unsubscribe()

    return EventSourceResponse(generate())
```

#### 5.2 Session Routes
```python
# src/opencode/server/routes/session.py
from fastapi import APIRouter, HTTPException
from pydantic import BaseModel
from ...core.session import Session, SessionInfo
from ...storage import get_storage

router = APIRouter()

class CreateSessionRequest(BaseModel):
    project_id: str
    directory: str

@router.post("/")
async def create_session(req: CreateSessionRequest) -> SessionInfo:
    session = Session(get_storage())
    return await session.create(req.project_id, req.directory)

@router.get("/{session_id}")
async def get_session(session_id: str, project_id: str) -> SessionInfo:
    session = Session(get_storage())
    try:
        return await session.get(session_id, project_id)
    except FileNotFoundError:
        raise HTTPException(404, "Session not found")

@router.get("/")
async def list_sessions(project_id: str) -> list[SessionInfo]:
    session = Session(get_storage())
    return await session.list(project_id)
```

---

### Phase 6: TUI Implementation (3-4 weeks)

#### 6.1 Textual Application
```python
# src/opencode/tui/app.py
from textual.app import App, ComposeResult
from textual.widgets import Header, Footer, Static
from textual.containers import Container, Horizontal, Vertical
from textual.binding import Binding
from .screens import HomeScreen, SessionScreen
from .widgets import MessageList, InputBox, StatusBar

class OpenCodeApp(App):
    CSS_PATH = "styles.tcss"
    BINDINGS = [
        Binding("q", "quit", "Quit"),
        Binding("n", "new_session", "New Session"),
        Binding("s", "sessions", "Sessions"),
        Binding("?", "help", "Help"),
    ]

    def __init__(self, sdk_url: str):
        super().__init__()
        self.sdk_url = sdk_url
        self.current_session = None

    def compose(self) -> ComposeResult:
        yield Header()
        yield Container(id="main")
        yield Footer()

    async def on_mount(self):
        await self.push_screen(HomeScreen())

    def action_new_session(self):
        # Create new session
        pass

    def action_sessions(self):
        # Show session list
        pass
```

#### 6.2 Message Widget
```python
# src/opencode/tui/widgets/message.py
from textual.widgets import Static
from textual.reactive import reactive
from rich.markdown import Markdown
from rich.panel import Panel

class MessageWidget(Static):
    content = reactive("")
    role = reactive("user")

    def __init__(self, role: str, content: str, **kwargs):
        super().__init__(**kwargs)
        self.role = role
        self.content = content

    def render(self) -> Panel:
        style = "blue" if self.role == "user" else "green"
        title = "You" if self.role == "user" else "Assistant"

        return Panel(
            Markdown(self.content),
            title=title,
            border_style=style
        )
```

---

### Phase 7: LSP & MCP Integration (2-3 weeks)

#### 7.1 LSP Client
```python
# src/opencode/lsp/client.py
import asyncio
import json
from typing import Any
from dataclasses import dataclass

@dataclass
class Diagnostic:
    severity: int
    message: str
    line: int
    character: int

class LSPClient:
    def __init__(self, command: list[str], root_uri: str):
        self.command = command
        self.root_uri = root_uri
        self.process = None
        self.request_id = 0
        self.pending: dict[int, asyncio.Future] = {}

    async def start(self):
        self.process = await asyncio.create_subprocess_exec(
            *self.command,
            stdin=asyncio.subprocess.PIPE,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE
        )
        asyncio.create_task(self._read_loop())
        await self._initialize()

    async def _initialize(self):
        await self._request("initialize", {
            "rootUri": self.root_uri,
            "capabilities": {}
        })
        await self._notify("initialized", {})

    async def definition(self, uri: str, line: int, char: int) -> list[dict]:
        return await self._request("textDocument/definition", {
            "textDocument": {"uri": uri},
            "position": {"line": line, "character": char}
        })

    async def references(self, uri: str, line: int, char: int) -> list[dict]:
        return await self._request("textDocument/references", {
            "textDocument": {"uri": uri},
            "position": {"line": line, "character": char},
            "context": {"includeDeclaration": True}
        })

    async def diagnostics(self, uri: str) -> list[Diagnostic]:
        # Return cached diagnostics for URI
        return self._diagnostics.get(uri, [])

    async def _request(self, method: str, params: dict) -> Any:
        self.request_id += 1
        req_id = self.request_id

        future = asyncio.get_event_loop().create_future()
        self.pending[req_id] = future

        message = {
            "jsonrpc": "2.0",
            "id": req_id,
            "method": method,
            "params": params
        }
        await self._send(message)

        return await future

    async def _notify(self, method: str, params: dict):
        message = {
            "jsonrpc": "2.0",
            "method": method,
            "params": params
        }
        await self._send(message)

    async def _send(self, message: dict):
        content = json.dumps(message)
        header = f"Content-Length: {len(content)}\r\n\r\n"
        self.process.stdin.write(header.encode() + content.encode())
        await self.process.stdin.drain()

    async def _read_loop(self):
        while True:
            # Read header
            header = await self.process.stdout.readline()
            if not header:
                break
            # Parse and handle response
            # ...
```

#### 7.2 MCP Client
```python
# src/opencode/mcp/client.py
import httpx
from typing import Any
from dataclasses import dataclass

@dataclass
class MCPTool:
    name: str
    description: str
    input_schema: dict

class MCPClient:
    def __init__(self, url: str, auth_token: str | None = None):
        self.url = url
        self.auth_token = auth_token
        self.client = httpx.AsyncClient()

    async def list_tools(self) -> list[MCPTool]:
        response = await self._request("tools/list", {})
        return [
            MCPTool(
                name=t["name"],
                description=t.get("description", ""),
                input_schema=t.get("inputSchema", {})
            )
            for t in response.get("tools", [])
        ]

    async def call_tool(self, name: str, arguments: dict) -> Any:
        return await self._request("tools/call", {
            "name": name,
            "arguments": arguments
        })

    async def _request(self, method: str, params: dict) -> dict:
        headers = {}
        if self.auth_token:
            headers["Authorization"] = f"Bearer {self.auth_token}"

        response = await self.client.post(
            self.url,
            json={
                "jsonrpc": "2.0",
                "id": 1,
                "method": method,
                "params": params
            },
            headers=headers
        )
        result = response.json()
        if "error" in result:
            raise Exception(result["error"])
        return result.get("result", {})
```

---

## 4. Migration Strategy

### Parallel Development
```
Phase 1-2: Foundation
├── Core systems running in Python
├── Can import/export sessions from TypeScript version
└── CLI subset working

Phase 3-4: Feature Parity
├── All built-in tools implemented
├── Provider support matching TypeScript
└── TUI functional

Phase 5-6: Integration
├── LSP/MCP support
├── Plugin system
└── Full API compatibility

Phase 7: Cutover
├── Data migration tools
├── Documentation update
└── Deprecation of TypeScript version
```

### Compatibility Considerations
1. **Session Format**: Use same JSON schema for session/message storage
2. **Config Format**: Parse same opencode.jsonc format
3. **API Compatibility**: Match OpenAPI spec for HTTP server
4. **Plugin System**: Design Python plugin format, provide migration guide

---

## 5. Testing Strategy

### Test Structure
```
tests/
├── unit/
│   ├── test_config.py
│   ├── test_storage.py
│   ├── test_tools/
│   │   ├── test_read.py
│   │   ├── test_bash.py
│   │   └── ...
│   ├── test_providers/
│   └── test_session.py
├── integration/
│   ├── test_server.py
│   ├── test_lsp.py
│   └── test_mcp.py
└── e2e/
    ├── test_cli.py
    └── test_tui.py
```

### Testing Tools
- **pytest**: Test framework
- **pytest-asyncio**: Async test support
- **httpx**: API testing
- **pytest-cov**: Coverage reporting
- **respx**: HTTP mocking

---

## 6. Timeline Summary

| Phase | Duration | Deliverables |
|-------|----------|--------------|
| 1. Core Foundation | 4-6 weeks | Config, Storage, Events, CLI scaffold |
| 2. Tool System | 3-4 weeks | Tool framework, all built-in tools |
| 3. Provider System | 3-4 weeks | Provider interface, 5+ providers |
| 4. Session & Agent | 3-4 weeks | Session management, processor |
| 5. HTTP Server | 2-3 weeks | FastAPI server, SSE |
| 6. TUI | 3-4 weeks | Textual-based TUI |
| 7. LSP & MCP | 2-3 weeks | LSP client, MCP client |
| **Total** | **20-28 weeks** | Full feature parity |

---

## 7. Key Dependencies

```toml
[project.dependencies]
# Core
pydantic = ">=2.0"
typer = ">=0.9.0"
orjson = ">=3.9"
aiofiles = ">=23.0"
httpx = ">=0.24"

# Server
fastapi = ">=0.100"
uvicorn = ">=0.23"
sse-starlette = ">=1.6"

# LLM
anthropic = ">=0.25"
openai = ">=1.0"
litellm = ">=1.0"

# TUI
textual = ">=0.40"
rich = ">=13.0"

# LSP
pygls = ">=1.0"

# Testing
pytest = ">=7.0"
pytest-asyncio = ">=0.21"
```

---

## 8. Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Performance gap vs Bun | Medium | Use uvloop, orjson, async throughout |
| TUI feature gap | Medium | Textual is mature; custom widgets as needed |
| Provider SDK differences | Low | LiteLLM provides unified interface |
| Plugin ecosystem | Medium | Provide TypeScript → Python migration guide |
| LSP complexity | Medium | Use pygls library for heavy lifting |
