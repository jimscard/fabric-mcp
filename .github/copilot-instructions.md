# Fabric MCP Server - AI Agent Instructions

## Project Overview
This is an MCP (Model Context Protocol) server that bridges Daniel Miessler's [Fabric framework](https://github.com/danielmiessler/fabric) with MCP-compatible clients. It translates MCP requests into REST API calls to a running `fabric --serve` instance and streams responses back.

**Key Architecture Pattern**: Adapter + Facade - MCP tools in [core.py](../src/fabric_mcp/core.py) adapt client requests → [api_client.py](../src/fabric_mcp/api_client.py) calls Fabric API → responses streamed back via SSE.

## Critical Development Workflows

### Build & Test Commands (via Make)
```bash
make bootstrap          # Initial setup: uv sync, pre-commit install
make test              # Full test suite with linting (uses pytest-xdist)
make test-fast         # Skip linting, parallel execution only
make lint              # Ruff, Pylint, Pyright (strict mode), Vulture
make format            # Ruff + isort formatting
make coverage          # Must maintain ≥95% coverage (see COVERAGE_FAIL_UNDER)
make dev               # Start fastmcp dev server for MCP Inspector debugging
```

**Important**: Always run `make bootstrap` after fresh clone. Tests use `pytest-xdist` for parallel execution (`-n auto`).

### Dependency Management
- Use `uv` exclusively (not pip/poetry)
- Add deps to [pyproject.toml](../pyproject.toml) under `[project.dependencies]` or `[dependency-groups]`
- Run `uv sync --dev` to update lockfile
- Pin versions for reproducibility: prefer `~=` or exact versions

### Testing Strategy (see [overall-testing-strategy.md](../docs/architecture/overall-testing-strategy.md))
- **Unit tests** (`tests/unit/`): Mock ALL external dependencies (`httpx` calls, filesystem)
- **Integration tests** (`tests/integration/`): Test against live `fabric --serve` or mock Fabric API boundary
- **Coverage target**: ≥90% (enforced at 95% in Makefile)
- **Naming**: `test_*.py` files, `test_*` functions, mirror `src/fabric_mcp/` structure

## Project-Specific Conventions

### Type Safety (MANDATORY)
- **Strict Pyright mode** enabled in [pyproject.toml](../pyproject.toml) - all type errors must resolve
- Full type hints required: functions, methods (including `self`/`cls`), variables
- Avoid `Any` - use `object`, `TypeVar`, or specific types; justify in comments if unavoidable
- See [coding-standards.md](../docs/architecture/coding-standards.md) for details

### Error Handling Pattern (see [error-handling-strategy.md](../docs/architecture/error-handling-strategy.md))
1. **Fabric API errors**: Catch in `FabricToolsMixin._make_fabric_api_request()` ([fabric_tools.py](../src/fabric_mcp/fabric_tools.py))
   - Translate HTTP status codes to MCP error types (e.g., `404` → `INVALID_PARAMS` for pattern not found)
   - Use `raise_mcp_error()` helper from [utils.py](../src/fabric_mcp/utils.py)
2. **MCP error format**: Return `McpError` with `ErrorData(code, message)` from `mcp.types`
3. **SSE stream interruptions**: Detect in pattern execution, send `urn:fabric-mcp:error:fabric-stream-interrupted`
4. **Logging**: Use `logging.getLogger(__name__)` with RichHandler; sanitize API keys/sensitive data

### Core Components & Data Flow
```
MCP Client
    ↓ (stdio/http transport)
FabricMCP (core.py) - FastMCP[None] with mixins
    ├─ FabricToolsMixin (fabric_tools.py) - 6 MCP tools
    ├─ SSEParserMixin (sse_parser.py) - Parse Fabric SSE streams
    └─ ValidationMixin (validation.py) - Input validation
    ↓
FabricApiClient (api_client.py) - httpx async client
    ↓ (REST API)
fabric --serve (external Fabric instance)
```

**Key Files**:
- [core.py](../src/fabric_mcp/core.py): `FabricMCP` class with `_load_default_config()` loads `~/.config/fabric/.env`
- [fabric_tools.py](../src/fabric_mcp/fabric_tools.py): 6 MCP tools (`fabric_list_patterns`, `fabric_run_pattern`, etc.)
- [server_stdio.py](../src/fabric_mcp/server_stdio.py): Entry point for `make dev` (MCP Inspector)
- [cli.py](../src/fabric_mcp/cli.py): Production entry point with Click (`fabric-mcp --transport stdio|http`)

### Configuration & Environment
```bash
FABRIC_BASE_URL="http://127.0.0.1:8080"  # Default Fabric API endpoint
FABRIC_API_KEY="..."                      # Optional auth for Fabric API
FABRIC_MCP_LOG_LEVEL="INFO"              # Server log level
```
Loaded via [config.py](../src/fabric_mcp/config.py) - reads `~/.config/fabric/.env` for DEFAULT_MODEL/DEFAULT_VENDOR.

### MCP Tools Implementation Pattern
All tools in `FabricToolsMixin` follow this pattern:
1. Validate inputs (using `ValidationMixin` methods)
2. Call `_make_fabric_api_request(endpoint, pattern_name, operation)` - handles all error translation
3. Parse/validate response structure
4. Return typed result (list[str], dict, or stream AsyncGenerator)

**Example**: See `fabric_list_patterns()` in [fabric_tools.py#L90-110](../src/fabric_mcp/fabric_tools.py)

### Streaming Pattern (SSE)
`fabric_run_pattern()` returns `AsyncGenerator[str, None]`:
- Uses `httpx_sse.aconnect_sse()` for SSE connections
- Parses with `SSEParserMixin._parse_sse_response()`
- Yields text chunks; handles errors mid-stream
- **Critical**: Must handle `CancelledError` for client disconnects

### Code Style Enforcement
- **Ruff**: Primary formatter/linter (config in [pyproject.toml](../pyproject.toml))
- **Pylint**: Additional checks (`.pylintrc` at root)
- **isort**: Import sorting (configured via Ruff in pyproject.toml)
- **Naming**: `snake_case` (vars/functions), `PascalCase` (classes), `UPPER_SNAKE_CASE` (constants)
- **Pre-commit hooks**: Auto-run on commit; bypass only for `make merge`

## Common Pitfalls to Avoid
1. **Don't mock `FabricMCP` directly** - mock `FabricApiClient.get()` or `httpx` calls instead
2. **Never log API keys** - use `SENSITIVE_CONFIG_PATTERNS` from [constants.py](../src/fabric_mcp/constants.py) to filter
3. **Async/await required** - All `httpx` calls, FastMCP handlers must be `async`
4. **Resource cleanup** - Use `try/finally` to close `FabricApiClient` in `_make_fabric_api_request()`
5. **Transport-specific CLI options** - Validate in [cli.py](../src/fabric_mcp/cli.py) callbacks (e.g., `--host` only valid with `--transport http`)

## Integration Points
- **External Dependency**: Requires running `fabric --serve` instance (default `http://127.0.0.1:8080`)
- **MCP Protocol**: Uses `fastmcp` + `modelcontextprotocol` libraries
- **Transports**: stdio (default), Streamable HTTP
- **No persistence** - Fully stateless; all state in external Fabric instance

## Documentation Structure
- **PRD**: [docs/PRD/](../docs/PRD/) - Product requirements
- **Architecture**: [docs/architecture/](../docs/architecture/) - Design patterns, tech stack, testing strategy
- **Stories**: [docs/stories/](../docs/stories/) - User story backlog (numbered 1.1-4.2)
- **Contributing**: [docs/contributing.md](../docs/contributing.md) - Detailed dev guide

## Version & Releases
- Version stored in [src/fabric_mcp/__about__.py](../src/fabric_mcp/__about__.py)
- Managed via `hatch` (configured in [pyproject.toml](../pyproject.toml))
- CI/CD: See `.github/workflows/` for test/publish pipelines

## Quick Reference
```bash
# Start development server with MCP Inspector
make dev

# Run with stdio transport (production)
uv run fabric-mcp --transport stdio

# Run with HTTP transport
uv run fabric-mcp --transport http --host 0.0.0.0 --port 8000

# Debug: Check if Fabric API is reachable
curl http://127.0.0.1:8080/patterns/names
```

---
**When modifying code**: Always update corresponding tests, maintain ≥95% coverage, run `make lint` before committing. The pre-commit hooks will enforce formatting, but strict type checking must pass manually.
