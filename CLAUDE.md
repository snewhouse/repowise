# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Repowise is a codebase intelligence system. It indexes a repository into four layers (dependency graph, git history, LLM-generated documentation, architectural decisions), stores the results in a database, and exposes them via 10 MCP tools so AI agents can query codebase knowledge instead of reading raw files.

**License:** AGPL-3.0-only. Embedding repowise requires a commercial license.

## Monorepo Structure

Four packages in a `uv` workspace (Python) + npm workspace (Node):

| Package | Language | Role | Depends on |
|---------|----------|------|------------|
| `packages/core` | Python | Engine: ingestion, AST parsing, graph building, generation, persistence, providers | - |
| `packages/server` | Python | FastAPI REST API (18 routers) + MCP server (10 tools) | core |
| `packages/cli` | Python | Click CLI (14 commands) | core, server |
| `packages/web` | TypeScript | Next.js 15 / React 19 frontend | server (via API proxy) |

All three Python packages are published as a single `repowise` PyPI package via explicit `setuptools.package-dir` mapping in the root `pyproject.toml`.

## Commands

```bash
# Setup
make install              # uv sync --all-packages
make install-dev          # uv sync --all-packages --all-extras
make install-web          # npm install for web package

# Testing
make test-fast            # Unit + provider tests (recommended for dev, no fixtures needed)
make test-unit            # Unit tests only
make test-providers       # Provider tests (no API keys required)
make test-integration     # Integration tests (needs fixtures)
make test                 # All tests

# Single test
uv run pytest tests/unit/test_foo.py -v
uv run pytest tests/unit/test_foo.py::test_bar -v

# Code quality
make check                # lint + format-check + typecheck (no modifications)
make fix                  # Auto-format + auto-fix lint
make lint                 # ruff check packages/ tests/
make format               # ruff format packages/ tests/
make typecheck            # mypy packages/core/src packages/cli/src packages/server/src

# Web UI
make dev-web              # Next.js dev server (port 3000)
make build-web            # Next.js production build
```

## Architecture

### Data Flow

```
Repository (filesystem/git)
  → FileTraverser (respects .gitignore, .repowiseIgnore)
  → ASTParser (tree-sitter, 10 languages)
  → GraphBuilder (NetworkX DiGraph: files, classes, functions, imports)
  → GenerationEngine (hierarchical, 9 page levels)
      ├─ ContextAssembler (source + git history + graph metrics + RAG)
      ├─ LLM Providers (Anthropic/OpenAI/Ollama/LiteLLM with prompt caching)
      └─ JobSystem (resumable checkpoints)
  → SQL Database (pages, metadata, versions)
  → Vector Store (embeddings for semantic search)
  → Consumers: CLI, Web UI, MCP Tools, Webhooks
```

Incremental updates (`repowise update`): ChangeDetector diffs against last synced commit, cascade analysis propagates through the graph, only affected pages regenerate.

### Key Source Paths

- **Pipeline orchestrator:** `packages/core/src/repowise/core/pipeline/orchestrator.py`
- **AST parsing + queries:** `packages/core/src/repowise/core/ingestion/` + `queries/*.scm` (10 languages)
- **LLM provider abstraction:** `packages/core/src/repowise/core/providers/llm/`
- **Generation templates:** `packages/core/src/repowise/core/generation/templates/*.j2` (11 Jinja2 templates)
- **SQLAlchemy models:** `packages/core/src/repowise/core/persistence/`
- **FastAPI app factory:** `packages/server/src/repowise/server/app.py`
- **API routers:** `packages/server/src/repowise/server/routers/` (18 modules)
- **MCP tools:** `packages/server/src/repowise/server/mcp_server/` (10 tools)
- **Frontend API clients:** `packages/web/src/lib/api/` (domain-organized)
- **Frontend hooks:** `packages/web/src/lib/hooks/` (SWR-based, SSE streaming)

### Database & Storage

- **SQL:** SQLite (default via aiosqlite) or PostgreSQL (asyncpg + pgvector). SQLAlchemy 2.0 async throughout.
- **Vector Store:** LanceDB (default) or pgvector.
- **Graph:** NetworkX DiGraph, persisted as JSON.

### Frontend → Backend Proxy

Next.js rewrites `/api/*` to the FastAPI backend (`REPOWISE_API_URL`, default `http://localhost:7337`). The frontend never calls the backend directly by port.

## Code Conventions

- **Python:** Async-first. Type-hinted. Ruff for lint+format (line-length 100). mypy with `check_untyped_defs`. structlog for logging.
- **TypeScript:** Strict mode. React Server Components by default, narrow client boundaries. SWR for data fetching. Radix UI + CVA for components. nuqs for URL search params.
- **Testing:** pytest with `asyncio_mode = "auto"`. respx for HTTP mocking. time-machine for time mocking.
- **Imports:** `known-first-party = ["repowise"]` — ruff handles isort.

## CI

GitHub Actions matrix: Python 3.11/3.12/3.13. Provider + unit tests on all PRs. Integration tests on push to main. Web lint + type-check.

## Docker

Multi-stage build: Node 20 (Next.js standalone) → Python 3.12-slim. Single image serves both backend (port 7337) and frontend (port 3000). Non-root user. Repos mounted read-only.

## Extension Points

- **Add language support:** Write a `.scm` tree-sitter query file in `packages/core/src/repowise/core/ingestion/queries/`
- **Add LLM provider:** Implement `BaseProvider` in `packages/core/src/repowise/core/providers/llm/`
- **Add API endpoint:** Create a router in `packages/server/src/repowise/server/routers/`
- **Add MCP tool:** Add to `packages/server/src/repowise/server/mcp_server/`
- **Add generation template:** Create `.j2` in `packages/core/src/repowise/core/generation/templates/`
