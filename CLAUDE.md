# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the App

```powershell
# Primary entry point — starts the CLI chat app
python main.py

# With uv (set USE_UV=1 in .env to make main.py use uv for the server subprocess too)
uv run main.py
```

Requires a `.env` file with:
```
ANTHROPIC_API_KEY=...
CLAUDE_MODEL=claude-opus-4-8   # or any claude-* model ID
```

## Running the MCP Inspector (for server development)

```powershell
$env:DANGEROUSLY_OMIT_AUTH="true"; mcp dev mcp_server.py
```

In the browser, connect with: Transport = **stdio**, Command = `python`, Arguments = `"<full path to mcp_server.py>"` (quotes required — path contains spaces).

## Architecture

The app is a CLI chat client backed by Claude and one or more MCP servers.

**Request flow:**
1. `main.py` boots a `CliApp` (prompt-toolkit UI) and a `CliChat` agent, both wired to an `MCPClient` that talks to `mcp_server.py` via stdio.
2. User input goes to `CliChat.run()` → `Chat.run()` in `core/chat.py`, which drives a tool-use loop with Claude via `core/claude.py`.
3. Tool calls are dispatched by `core/tools.py` (`ToolManager`), which fans out across all connected `MCPClient` instances to find the right server for each tool.

**MCP layer (`mcp_server.py` + `mcp_client.py`):**
- `mcp_server.py` defines tools (`@mcp.tool`), resources (`@mcp.resource`), and prompts (`@mcp.prompt`) using FastMCP.
- Resources use the `docs://` URI scheme: `docs://documents` returns the list of doc IDs (JSON), `docs://documents/{doc_id}` returns a single doc's text.
- `MCPClient` wraps `ClientSession` and is used as an async context manager. Every `session()` method call is async — all `list_tools`, `call_tool`, `list_prompts`, `get_prompt`, and `read_resource` calls must be `await`ed.
- `main.py` supports loading extra MCP servers by passing their script paths as CLI arguments; each gets its own `MCPClient` instance and its tools become available to Claude automatically.

**`@` and `/` syntax in the CLI:**
- `@doc_id` in a message injects that document's content into the prompt via `CliChat._extract_resources`.
- `/command doc_id` runs an MCP prompt (e.g. `/format deposition.md`) via `CliChat._process_command`.
- Tab completion for both is driven by `core/cli.py` (`UnifiedCompleter`).

## Key Dependency

`mcp[cli]==1.8.0` (pinned in README; `pyproject.toml` uses `>=1.8.0`). The `base.UserMessage` type used in server prompts lives at `mcp.server.fastmcp.prompts.base`.
