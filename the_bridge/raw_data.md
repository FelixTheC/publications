# Cluster the Bridge MCP
### A Technical Deep-Dive into Dynamic LLM Tool Orchestration

---

## 1. The Problem: Too Many Tools, Too Little Context

Modern AI assistants face a scaling challenge: enterprises use dozens of services (Atlassian, Google, Microsoft, ...) each exposing hundreds of tools. Naively loading all tools into the LLM context is:

- **Expensive** — tokens are not free
- **Noisy** — irrelevant tools confuse the planner
- **Insecure** — unused tools still hold attack surface

**The Bridge** solves this with a principle: *only load what you need, when you need it.*

---

## 2. What Is The Bridge?

The Bridge is a **PoC orchestration layer** built on the [Model Context Protocol (MCP)](https://modelcontextprotocol.io/). It sits between the user and the actual service integrations:

```
User Prompt
    │
    ▼
[ Bridge API ]  ──► ArangoDB (tool registry)
    │
    ▼
[ LLM Planner ] ──► Structured execution plan (ActionItems)
    │
    ▼
[ JIT MCP Server ] ── only the required tools loaded ──►  Plugins
    │                                                     ├── atlassian_mcp
    ▼                                                     ├── google_mcp
[ MCP Client ]                                            └── microsoft_mcp
    │
    ▼
Synthesized Response
```

**Core architectural principle:** Core routing logic is entirely decoupled from service implementations.

---

## 3. The Brain: Declarative Planning with ArangoDB + LLMs

### Tool Discovery (`src/tools/get_tools.py`)

Plugins expose tools via simple decorators — no manual registration:

```python
from pexon_mcp_utils.utils import ause_tool

@ause_tool(description="Create a Jira issue in a given project")
async def create_issue(project_key: str, summary: str, description: str) -> dict:
    """Creates a Jira issue and returns the created issue data."""
    ...
```

At build/sync time, `get_tools.py`:
1. Scans all plugin packages for `@use_tool` / `@ause_tool` decorators
2. Extracts `__annotations__` and docstrings
3. Emits a structured `tool_data.json` → seeded into ArangoDB's `mcp_tools` collection

### Intent Matching with AQL (`src/llm/prompt.py`)

Rather than dumping the whole registry into the LLM prompt, the Bridge filters first:

```aql
FOR t IN mcp_tools
  FILTER REGEX_TEST(t.service.name, @pattern, true)
  RETURN { name: t.name, description: t.description, input_schema: t.input_schema }
```

Only the tools relevant to the request reach the LLM — keeping context lean and focused.

### Structured Output via Pydantic

The LLM is constrained to a strict schema using LangChain's `with_structured_output`:

```python
class ActionItem(BaseModel):
    type: Literal["tool", "llm"]
    tool_name: str | None
    arguments: dict | None
    prompt: str | None

class ActionItems(BaseModel):
    steps: list[ActionItem]
```

The result is a **machine-executable plan**, not a natural language description.  
Steps can reference prior results: `{{ steps.0.result.issue_key }}`.

---

## 4. The Execution: Just-in-Time (JIT) Server Provisioning

### Request Lifecycle

```
1. POST /prompt  →  Bridge API (main.py)
2. AQL query     →  ArangoDB (find relevant tools)
3. LLM planning  →  ActionItems (what to call, in what order)
4. start_server.sh →  spawns src/server_main.py via `uv run`
5. Dynamic import  →  ONLY required tools loaded
6. SSE transport  →  MCP Client connects on port 9090
7. Tool execution →  Plugins called with injected auth state
8. kill_local_mcp_server() →  process cleaned up immediately
```

### Dynamic Import (`src/server_main.py`)

The server boots with **zero tools**. It receives the required tool list via CLI args and imports them at runtime:

```python
module = importlib.import_module(
    tool.path_to_function.replace(f".{tool.name}", ""), "*"
)
obj = getattr(module, tool.name)
mcp_obj.add_tool(obj)
```

**Result:** minimal attack surface, minimal memory footprint, per-request isolation.

---

## 5. The Extension: Dynamic Middleware Injection

### Plugin-as-a-Module Pattern

Plugins can contribute behavior to the server's ASGI stack, not just tools.

`src/tools/get_additional_middlewares.py` scans plugin directories for any class with `Middleware` in its name, then automatically injects it into the FastMCP ASGI stack at startup.

### Core: `PexonMiddleware` (`src/config/pexon_middleware.py`)

All incoming `x-` headers are parsed, grouped by service prefix, and injected into Starlette's `request.state`:

```
x-atlassian-token: abc123
x-atlassian-domain: mycompany.atlassian.net
x-google-token: xyz789
           │
           ▼
request.state.atlassian_service_headers = {
    "token": "abc123",
    "domain": "mycompany.atlassian.net"
}
request.state.google_service_headers = { "token": "xyz789" }
```

Individual tool implementations call `get_http_request().get("state")` — **no manual plumbing required**.

### Writing a Custom Plugin Middleware

```python
class MyServiceMiddleware:
    def __init__(self, app: ASGIApp, parent: object = None):
        self.app = app

    async def __call__(self, scope: Scope, receive: Receive, send: Send):
        if scope["type"] == "http":
            # inject custom logic: rate limiting, per-service logging, etc.
            pass
        await self.app(scope, receive, send)
```

Drop this in your plugin directory — it's discovered and registered automatically.

---

## 6. The Environment: uv Workspace & Isolation

The project is a **monorepo with true package isolation**, managed by `uv`:

```
mcp_playaround/           ← workspace root (shared uv.lock)
├── pyproject.toml
├── src/                  ← Bridge core
└── plugins/
    ├── atlassian_mcp/    ← standalone package, own pyproject.toml
    ├── google_mcp/       ← own deps (e.g. google-auth)
    ├── microsoft_mcp/
    ├── pexon_mcp_utils/  ← shared library used by all plugins
    └── plugin_template/  ← cookiecutter scaffold
```

- `atlassian_mcp` and `google_mcp` can have **conflicting dependencies** — fully isolated
- `pexon_mcp_utils` acts as a **shared contract** (decorators, base types)
- `uv sync` at the root reconciles the whole tree, making any plugin safely importable at runtime

**Adding a new integration:** `cookiecutter plugins/plugin_template -o plugins/` — then register it in `pyproject.toml`.

---

## 7. End-to-End Example

**User prompt:**
> "Create a Jira ticket in project INTERNAL titled 'Setup MCP Bridge' and link the related Google Drive doc"

**What happens:**
1. AQL finds `create_issue` (atlassian), `search_files` (google)
2. LLM produces:
   ```json
   [
     { "type": "tool", "tool_name": "create_issue", "arguments": { "project_key": "INTERNAL", "summary": "Setup MCP Bridge" } },
     { "type": "tool", "tool_name": "search_files",  "arguments": { "query": "MCP Bridge" } },
     { "type": "llm",  "prompt": "Summarize the result and confirm both actions to the user." }
   ]
   ```
3. JIT server loads **only** `create_issue` and `search_files`
4. Auth tokens from `x-atlassian-*` and `x-google-*` headers are already in `request.state`
5. Results injected between steps via `{{ steps.N.result }}`
6. Server killed, synthesized response returned

---

## 8. Architectural Highlights & Strategic Trade-offs

*   **Security-First Isolation:** JIT process spawning ensures absolute isolation. *Trade-off:* We prioritize maximum security over sub-second startup latency.
*   **Intelligent Semantic Discovery:** ArangoDB + Vector search matches intent to tools. *Trade-off:* Operational power is gained through sophisticated external dependencies.
*   **Stateless & Privacy-Conscious:** Header-based auth avoids secret storage. *Trade-off:* Shifts token orchestration to the client to maintain a lean, neutral core.
*   **Developer-Centric Velocity:** "Zero-config" discovery enables rapid scaling. *Trade-off:* Relies on development conventions rather than rigid manifests to fuel speed.
*   **Industrial-Grade Reliability:** Aggressive cleanup guarantees resource availability. *Trade-off:* Prioritizes system uptime and port readiness over graceful shutdown hooks.



## 9. Summary: What Makes This Architecture Interesting

**Decoupled** — Core routing knows nothing about Jira or Google Drive. Plugins know nothing about routing.

**Stateless & On-Demand** — MCP servers exist only for the duration of a single task. No idle processes, no shared state between requests.

**Type-Safe by Design** — Pydantic and type hints are not just validation — they *are* the discovery mechanism. The Bridge reads your types to understand your tools.

**Zero-Config Extensibility** — Drop a decorated function in the right place. The Bridge discovers, registers, and plans with it automatically.

---

*Questions? The best starting point is `src/llm/prompt.py` (the planner) and `src/server_main.py` (the JIT server) — together they are the heart of the Bridge.*
