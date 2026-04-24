# CLAUDE.md — MCP Server Template

This repo is a **template** for building Python MCP servers. It is not a runnable application itself — the `server/` directory contains placeholder code that the user replaces with their actual tools.

## Architecture

Every dev operation runs through Docker Compose. There is no expectation that Python or any tool is installed locally — Docker is the only requirement.

```
Dockerfile               ← root; builds the MCP server image
server/main.py           ← the MCP server (FastMCP, streamable-http, port 8000)
server/__init__.py       ← version string: __version__ = "x.y.z"
pyproject.toml           ← dependencies + tool config (black/ruff/isort/docformatter)
```

## Key Conventions

### Source directory
Server code lives in `server/`. All tooling (reformat, lint, test Dockerfiles) references `server/`. When the user renames this to their actual module name, they must update every path in: `Dockerfile`, `reformat/Dockerfile`, `lint/Dockerfile`, `docs-validate/Dockerfile`, `tests/Dockerfile`, `tests/docker-compose.yaml`, `reformat/reformat.sh`, `lint/lint.sh`, `pyproject.toml`.

### Placeholders to replace
- `<SERVER-NAME>` — hyphenated (Docker image name, GitHub repo name, badge URLs)
- `<ORGANIZATION>` — GitHub org or username

### Version
`server/__init__.py` holds `__version__`. CI reads it with a regex (no import needed). Bumping this above the last git tag triggers a stable Docker Hub release on the next main push.

### MCP transport
FastMCP with `transport="streamable-http"`. Endpoint at `/mcp` (POST). Port 8000. This is the current MCP standard — prefer it over the older SSE transport.

## Directory Structure

```
reformat/               docker-compose + Dockerfile + reformat.sh
                        runs black, docformatter, isort on server/ and tests/
                        volume-mounts project root so changes write back to host

lint/                   docker-compose + Dockerfile + lint.sh
                        runs ruff check on server/ and tests/ (read-only)

docs-validate/          docker-compose + Dockerfile
                        runs mkdocs build --strict
                        volume-mounts project root for docs/ and mkdocs.yml

tests/                  docker-compose with TWO services:
                        1. mcp-server  — root Dockerfile, health-checked on port 8000
                        2. test        — tests/Dockerfile, waits for server healthy
                        test command: pytest --mcp-tools=http://mcp-server:8000 tests/

inspector/              docker-compose with TWO services:
                        1. mcp-server  — same as above
                        2. mcp-inspector — ghcr.io/modelcontextprotocol/inspector
                        inspector pre-pointed at http://mcp-server:8000/mcp
                        UI accessible at http://localhost:6274 (check logs for token)

.github/workflows/ci.yaml   full pipeline (see below)
.vscode/tasks.json           tasks: test, lint, reformat, validate-docs, inspector
scripts/semver_compare.py    used by CI to compare version strings
```

## CI/CD Pipeline

Jobs and their dependencies:

```
reformat → lint     (parallel after reformat)
reformat → test
           ↓
         detect-changes
           ↓ (main only, when changed)
         publish  →  publish-docs
```

- **reformat**: on non-main branches, commits reformatted code back to the branch
- **publish**: pushes to Docker Hub using `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN` secrets
  - stable: tags both `x.y.z` and `latest`
  - dev: tags only `x.y.z.devYYYYMMDDHHMM`
- **publish-docs**: runs `mike deploy` to gh-pages (only if `docs/` dir and `mkdocs.yml` exist)

The old `.github/ci.yaml` is superseded by `.github/workflows/ci.yaml`. The old file can be deleted.

## Adding Tools

Edit `server/main.py`:

```python
@mcp.tool()
def my_tool(param: str) -> str:
    """What this tool does (shown in MCP clients)."""
    return f"result: {param}"
```

Run `inspector` task to test in the browser UI, or `test` task to run the automated suite.

## Testing

Tests use `pytest-mcp-tools==0.2.0`. The `--mcp-tools=http://mcp-server:8000` flag points the plugin at the running server. The test container depends on `mcp-server` being healthy before starting.

Add tests in `tests/test_*.py`. The `test_unit.py` placeholder just asserts True.

## Dependencies

Install order in Dockerfiles: `pip install -e ".[<group>]"` — the package is editable-installed from the copied `pyproject.toml`. Groups:
- `.[dev]` — reformat + lint containers
- `.[test]` — test container
- `.[docs]` — docs-validate container
- base (no extras) — root Dockerfile / production server
