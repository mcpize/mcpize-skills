# Python MCP Server Patterns

These patterns come from the official mcpize Python template. Follow them so `mcpize dev` and `mcpize deploy` work without issues.

## Project Structure

```
src/[package_name]/
  __init__.py
  server.py        # FastMCP setup, middleware, tool registration, entry point
  tools.py         # Pure functions — all tool logic lives here
  api_client.py    # External API wrapper (if needed)
  py.typed         # PEP 561 type marker
```

> **Note**: `mcpize init` automatically renames `my_mcp_server` to your project name (e.g., `mcpize init whale-analytics` creates `src/whale_analytics/`). You don't need to rename anything manually.

## Server Setup (server.py)

```python
"""MCP server entry point."""

import contextlib
import json
import logging
import signal
import time
from os import getenv

from fastmcp import FastMCP
from fastmcp.server.middleware import Middleware, MiddlewareContext
from rich.console import Console
from starlette.requests import Request
from starlette.responses import JSONResponse

from .tools import my_tool_1, my_tool_2

logging.basicConfig(
    format="%(asctime)s %(levelname)s %(name)s: %(message)s",
    level=getenv("LOG_LEVEL", "INFO"),
)

# ============================================================================
# Dev Logging Middleware (auto-disabled in production)
# ============================================================================

console = Console()
IS_DEV = getenv("ENV", "development") != "production"


def truncate(s: str, max_len: int = 60) -> str:
    """Truncate string with ellipsis."""
    return s if len(s) <= max_len else s[: max_len - 3] + "..."


def format_latency(ms: float) -> str:
    """Color-code latency: green <100ms, yellow 100-500ms, red >500ms."""
    if ms < 100:
        return f"[green]{ms:.0f}ms[/green]"
    if ms < 500:
        return f"[yellow]{ms:.0f}ms[/yellow]"
    return f"[red]{ms:.0f}ms[/red]"


def timestamp() -> str:
    """Get current time as HH:MM:SS."""
    return time.strftime("%H:%M:%S")


class DevLoggingMiddleware(Middleware):
    """Colorized dev logging middleware for MCP requests/responses."""

    async def on_message(self, context: MiddlewareContext, call_next):
        """Log all MCP messages with timing."""
        if not IS_DEV:
            return await call_next(context)

        method = context.method
        message = context.message

        # Skip noisy notifications
        if method == "notifications/initialized":
            return await call_next(context)

        # Log request
        if method == "tools/call" and message:
            tool_name = getattr(message, "name", "unknown")
            tool_args = getattr(message, "arguments", {})
            args_str = truncate(json.dumps(tool_args)) if tool_args else ""
            console.print(
                f"[dim][{timestamp()}][/dim] [cyan]→[/cyan] tools/call [bold]{tool_name}[/bold]"
                + (f" [dim]{args_str}[/dim]" if args_str else "")
            )
        else:
            params_str = ""
            if message:
                with contextlib.suppress(Exception):
                    params_str = (
                        f" [dim]{truncate(json.dumps(message, default=str))}[/dim]"
                    )
            console.print(f"[dim][{timestamp()}][/dim] [cyan]→[/cyan] {method}{params_str}")

        # Execute and time
        start = time.time()
        result = await call_next(context)
        latency_ms = (time.time() - start) * 1000

        # Log response
        latency = format_latency(latency_ms)

        if hasattr(result, "isError") and result.isError:
            error_msg = truncate(str(result))
            console.print(
                f"[dim][{timestamp()}][/dim] [red]✖[/red] {method}"
                f" [red]{error_msg}[/red] ({latency})"
            )
        elif method == "tools/call":
            try:
                content = getattr(result, "content", [])
                if content and hasattr(content[0], "text"):
                    result_text = truncate(content[0].text)
                    console.print(
                        f"[dim][{timestamp()}][/dim] [green]←[/green] {result_text} ({latency})"
                    )
                else:
                    console.print(
                        f"[dim][{timestamp()}][/dim] [green]✓[/green] tools/call ({latency})"
                    )
            except Exception:
                console.print(
                    f"[dim][{timestamp()}][/dim] [green]✓[/green] tools/call ({latency})"
                )
        else:
            console.print(f"[dim][{timestamp()}][/dim] [green]✓[/green] {method} ({latency})")

        return result


# ============================================================================
# MCP Server Setup
# ============================================================================

mcp = FastMCP("your-server-name")  # Name from brief's slug — auto-set by post-init

# Add dev logging middleware
mcp.add_middleware(DevLoggingMiddleware())

# Health endpoint — required for MCPize Cloud
@mcp.custom_route("/health", methods=["GET"])
async def health_check(request: Request) -> JSONResponse:
    return JSONResponse({"status": "healthy"})

# Register tools
mcp.tool()(my_tool_1)
mcp.tool()(my_tool_2)


def main() -> None:
    """Run the MCP server with graceful shutdown."""
    port = int(getenv("PORT", "8080"))

    console.print()
    console.print("[bold]MCP Server running on[/bold]", f"[cyan]http://localhost:{port}[/cyan]")
    console.print(f"  [dim]Health:[/dim] http://localhost:{port}/health")
    console.print(f"  [dim]MCP:[/dim]    http://localhost:{port}/mcp")

    if IS_DEV:
        console.print()
        console.print("[dim]" + "─" * 50 + "[/dim]")
        console.print()

    def handle_sigterm(*_):
        console.print("[dim]Received SIGTERM, shutting down...[/dim]")
        raise SystemExit(0)

    signal.signal(signal.SIGTERM, handle_sigterm)

    # Run with Streamable HTTP transport — the modern standard
    mcp.run(
        transport="streamable-http",
        host="0.0.0.0",
        port=port,
    )


if __name__ == "__main__":
    main()
```

## Writing Tools (tools.py)

In FastMCP, tools are just Python functions. The magic:
- **Type hints** become the input schema automatically
- **Docstrings** become the tool description
- **Argument docstrings** describe each parameter

```python
"""MCP server tools - pure functions for testing."""

import asyncio
from datetime import UTC, datetime


def hello(name: str) -> dict[str, str]:
    """Returns a greeting message.

    Args:
        name: Name to greet.

    Returns:
        Dictionary with greeting message.
    """
    return {"message": f"Hello, {name}! Welcome to MCP."}


def echo(text: str) -> dict[str, str]:
    """Echoes back the input text with a timestamp.

    Args:
        text: Text to echo back.

    Returns:
        Dictionary with echoed text and ISO timestamp.
    """
    return {
        "echo": text,
        "timestamp": datetime.now(UTC).isoformat(),
    }


async def delayed_echo(text: str, delay: float = 1.0) -> dict[str, str]:
    """Echoes text after a delay (async example).

    Args:
        text: Text to echo back.
        delay: Seconds to wait before responding (default: 1.0).

    Returns:
        Dictionary with echoed text and actual delay.
    """
    await asyncio.sleep(delay)
    return {
        "echo": text,
        "delay_seconds": str(delay),
    }
```

### Key patterns for Python tools

- **Tools are pure functions** — they live in `tools.py`, separate from server setup. This makes them easy to test with pytest.
- **Return dicts** — FastMCP serializes them to JSON automatically. No need to manually create content arrays like in TypeScript.
- **Type hints are required** — they generate the input schema. Use `str`, `int`, `float`, `bool`, `list[str]`, `dict[str, str]`, etc.
- **Docstrings are required** — the function docstring becomes the tool description. Arg descriptions in the docstring become parameter descriptions.
- **Optional params**: use `param: str = "default"` for optional parameters with defaults.
- **Async tools**: use `async def` for tools that call external APIs (httpx, etc.). FastMCP handles both sync and async.

## Error Handling

Never raise exceptions from tools — catch everything and return an error dict:

```python
import os
import httpx


async def get_data(query: str) -> dict:
    """Fetches data from the API.

    Args:
        query: What to search for.
    """
    try:
        api_key = os.getenv("API_KEY")
        if not api_key:
            return {
                "error": "API_KEY is not configured",
                "suggestion": "Set the API_KEY secret in MCPize Dashboard or via: mcpize secrets set API_KEY your-key",
            }

        async with httpx.AsyncClient(timeout=30.0) as client:
            response = await client.get(
                "https://api.example.com/search",
                params={"q": query},
                headers={"Authorization": f"Bearer {api_key}"},
            )
            response.raise_for_status()
            return response.json()

    except httpx.HTTPStatusError as e:
        return {
            "error": f"API returned {e.response.status_code}",
            "suggestion": "Check your API key or try again later.",
        }
    except httpx.TimeoutException:
        return {
            "error": "API request timed out after 30 seconds",
            "suggestion": "The external service might be slow. Try again in a moment.",
        }
    except Exception as e:
        return {
            "error": str(e),
            "suggestion": "An unexpected error occurred. Check the server logs.",
        }
```

## Caching

Use `cachetools` for in-memory caching:

```python
from cachetools import TTLCache

# Cache with 5-minute TTL, max 1000 entries
_cache = TTLCache(maxsize=1000, ttl=300)


def get_cached(key: str):
    """Get value from cache, or None if not found/expired."""
    return _cache.get(key)


def set_cached(key: str, value):
    """Store value in cache."""
    _cache[key] = value


async def get_price(token: str) -> dict:
    """Gets the current price of a token.

    Args:
        token: Token symbol (e.g., ETH, BTC).
    """
    cache_key = f"price:{token}"
    cached = get_cached(cache_key)
    if cached:
        return {**cached, "cached": True}

    # Fetch fresh data
    result = await fetch_price_from_api(token)
    set_cached(cache_key, result)
    return {**result, "cached": False}
```

## API Client Wrapper

For servers calling external APIs:

```python
# src/[package]/api_client.py
import os
import httpx
from cachetools import TTLCache

_cache = TTLCache(maxsize=500, ttl=300)


class ApiClient:
    def __init__(self, base_url: str, api_key_env: str):
        self.base_url = base_url
        self.api_key = os.getenv(api_key_env, "")

    async def get(self, path: str, params: dict | None = None) -> dict:
        cache_key = f"{path}:{params}"
        cached = _cache.get(cache_key)
        if cached:
            return cached

        async with httpx.AsyncClient(timeout=30.0) as client:
            response = await client.get(
                f"{self.base_url}{path}",
                params=params,
                headers={"Authorization": f"Bearer {self.api_key}"},
            )
            response.raise_for_status()
            data = response.json()

        _cache[cache_key] = data
        return data
```

## Required Dependencies

From the template's `pyproject.toml`:

```toml
[project]
requires-python = ">=3.11"
dependencies = [
    "fastmcp>=2.13,<3",
    "rich>=14.0",
]

[dependency-groups]
dev = [
    "pytest>=8.0",
    "pytest-asyncio>=0.24",
    "ruff>=0.8",
]
```

Add these if your server needs them:
- `httpx` — async HTTP client for external API calls
- `cachetools` — in-memory caching with TTL
- Other packages from the brief's Technical Stack

## Dev Logging Middleware

The template includes `DevLoggingMiddleware` that logs requests/responses with timing:
- **Color-coded latency** — green <100ms, yellow 100-500ms, red >500ms
- **Tool call details** — shows tool name, arguments, and result text
- **Auto-disabled** in production (`ENV=production`)
- Uses `rich` for colorized console output

Keep this code when editing `server.py` — it's the block between `# Dev Logging Middleware` and `# MCP Server Setup`.

## Entry Point

The server runs via `server.py` directly — it has `if __name__ == "__main__": main()` at the bottom.

Two ways to run it:
- **CLI script** (from `pyproject.toml`): `your-server-name` (configured in `[project.scripts]`)
- **Module**: `python -m package_name.server` (note the `.server` suffix — there is no `__main__.py`)

The `mcpize.yaml` uses: `.venv/bin/python -m package_name.server`
