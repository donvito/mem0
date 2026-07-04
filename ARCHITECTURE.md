# Mem0 Architecture

This document describes the architecture of the Mem0 repository: how the monorepo is organized, how the core memory pipeline works, and how the SDKs, servers, CLIs, and integrations fit together.

For contributor workflows (setup, build, lint, test commands), see [AGENTS.md](AGENTS.md) and [CONTRIBUTING.md](CONTRIBUTING.md).

## Overview

**Mem0** is an intelligent memory layer for AI agents and assistants. It extracts durable facts from conversations, stores them as vector-indexed memories, and retrieves them via semantic search so agents can stay personalized across sessions.

Mem0 ships in **two usage modes**, both available in Python and TypeScript:

| Mode | Python | TypeScript | Backing |
|------|--------|------------|---------|
| **Self-hosted (OSS)** | `Memory` / `AsyncMemory` | `Memory` (`mem0ai/oss`) | Your own LLM, embedder, and vector store |
| **Hosted Platform** | `MemoryClient` / `AsyncMemoryClient` | `MemoryClient` (`mem0ai`) | Mem0's managed API (`api.mem0.ai`) |

The OSS mode runs the full memory pipeline locally; the Platform mode is a thin HTTP client over the hosted API.

## High-Level Diagram

```
                        ┌────────────────────────────────────────────┐
                        │                Consumers                   │
                        │  apps · agents · editors (Claude Code,     │
                        │  Cursor, Codex) · Vercel AI SDK · CLIs     │
                        └───────┬───────────────────────┬────────────┘
                                │                       │
                     Self-hosted (OSS)             Hosted Platform
                                │                       │
        ┌───────────────────────▼──────────┐   ┌────────▼─────────────┐
        │   mem0/ (Python) · mem0-ts/oss   │   │  mem0/client/ (Py)   │
        │                                  │   │  mem0-ts/src/client/ │
        │  ┌────────────────────────────┐  │   └────────┬─────────────┘
        │  │       Memory pipeline      │  │            │ HTTPS
        │  │  extract → embed → store   │  │   ┌────────▼─────────────┐
        │  │  search → rerank → return  │  │   │   api.mem0.ai        │
        │  └───┬────────┬─────────┬─────┘  │   │  (managed platform)  │
        │      │        │         │        │   └──────────────────────┘
        │   LLMs    Embedders  Vector      │
        │  (~20)     (~14)     stores      │
        │                      (~26)       │
        └──────────────────────────────────┘
                        ▲
                        │ wraps the Python SDK
        ┌───────────────┴──────────────────┐
        │  server/ (FastAPI REST server)   │
        │  openmemory/ (API + MCP + UI)    │
        └──────────────────────────────────┘
```

## Monorepo Layout

This is a polyglot monorepo (Python + TypeScript). Each package is self-contained with its own build, lint, and test tooling.

| Directory | Package | Language | Purpose |
|-----------|---------|----------|---------|
| `mem0/` | `mem0ai` (PyPI) | Python | Core SDK: memory pipeline + provider plugins |
| `mem0-ts/` | `mem0ai` (npm) | TypeScript | SDK: hosted client + OSS memory |
| `server/` | — | Python | FastAPI REST server wrapping the Python SDK |
| `openmemory/` | — | Python + TS | Self-hosted memory platform (API + MCP server + Next.js UI) |
| `cli/python/` | `mem0-cli` (PyPI) | Python | Typer-based CLI |
| `cli/node/` | `@mem0/cli` (npm) | TypeScript | Commander-based CLI |
| `integrations/` | various npm packages | TypeScript | Agent/editor integrations (one directory per integration) |
| `skills/` | — | Markdown | Claude Code skill definitions |
| `docs/` | — | MDX | Documentation site (Mintlify) |
| `tests/` | — | Python | Python SDK test suite (pytest) |
| `evaluation/` | — | submodule | Benchmarks (LOCOMO, LongMemEval, BEAM) → [`mem0ai/memory-benchmarks`](https://github.com/mem0ai/memory-benchmarks) |
| `examples/` | — | mixed | Sample apps, demos, notebooks |
| `scripts/` | — | Python | Repo-wide utilities (e.g. docs `llms.txt` coverage check) |

## Python SDK (`mem0/`)

The core package and reference implementation. Everything else (server, OpenMemory, CLI OSS mode) builds on it.

### Package structure

```
mem0/
├── memory/          # Core pipeline: Memory / AsyncMemory (main.py),
│                    # SQLite history store (storage.py), telemetry
├── client/          # MemoryClient / AsyncMemoryClient for the hosted platform
├── llms/            # LLM providers (OpenAI, Anthropic, Bedrock, Gemini, Groq,
│                    # Ollama, DeepSeek, vLLM, LiteLLM, xAI, ...)
├── embeddings/      # Embedding providers (OpenAI, Azure, Gemini, HuggingFace,
│                    # FastEmbed, Ollama, Vertex AI, ...)
├── vector_stores/   # Vector store providers (Qdrant, Pinecone, Chroma, pgvector,
│                    # Redis, Elasticsearch, Faiss, S3 Vectors, ...)
├── reranker/        # Rerankers (Cohere, HuggingFace, LLM-based,
│                    # SentenceTransformer, ZeroEntropy)
├── configs/         # Pydantic v2 config models (MemoryConfig + per-category configs)
├── proxy/           # OpenAI-compatible chat proxy with automatic memory
├── utils/           # Factories, entity extraction, scoring, HTTP helpers
└── exceptions.py    # Typed exception hierarchy
```

### Provider pattern

Every extensible capability follows the same plugin architecture:

1. An abstract base class in `mem0/<category>/base.py` defines the interface.
2. Each provider is one module in the category directory (e.g. `mem0/llms/anthropic.py`) implementing that interface.
3. A Pydantic config class in `mem0/<category>/configs.py` (or `mem0/configs/`) validates provider settings.
4. Factories in `mem0/utils/factory.py` (`LlmFactory`, `EmbedderFactory`, `VectorStoreFactory`, `RerankerFactory`) instantiate providers by name from config.

This means switching providers is a config change, not a code change:

```python
from mem0 import Memory

m = Memory.from_config({
    "llm": {"provider": "anthropic", "config": {"model": "claude-sonnet-4-5"}},
    "embedder": {"provider": "openai", "config": {"model": "text-embedding-3-small"}},
    "vector_store": {"provider": "qdrant", "config": {"host": "localhost"}},
})
```

Provider dependencies are **optional extras** in `pyproject.toml` — the core install stays lightweight, and each provider group pulls in only what it needs.

### The memory pipeline

`Memory.add()` and `Memory.search()` in `mem0/memory/main.py` implement the core loop:

**Add path** (`add(messages, user_id=..., ...)`):

1. **Fact extraction** — the configured LLM distills the incoming messages into candidate facts (prompts in `mem0/configs/prompts.py`).
2. **Retrieval of related memories** — each candidate fact is embedded and searched against the vector store within the same session scope (`user_id` / `agent_id` / `run_id`).
3. **Memory reconciliation** — a second LLM pass decides, per fact, whether to `ADD` a new memory, `UPDATE` or `DELETE` an existing one, or do nothing (`NONE`).
4. **Persistence** — vector store operations are applied, and every mutation is journaled in a local SQLite history database (`mem0/memory/storage.py`), which backs `history(memory_id)`.

**Search path** (`search(query, user_id=..., ...)`):

1. The query is embedded and run against the vector store with session-scoped filters.
2. If a reranker is configured, candidates are re-scored and reordered.
3. Results are returned with scores and metadata.

`AsyncMemory` mirrors the same pipeline with async providers and I/O.

### Hosted client (`mem0/client/`)

`MemoryClient` / `AsyncMemoryClient` expose the same method surface (`add`, `search`, `get_all`, `update`, `delete`, `history`, ...) as thin, typed HTTP wrappers over the platform REST API — no local LLM or vector store involved. Project/org management lives in `mem0/client/project.py`.

## TypeScript SDK (`mem0-ts/`)

Mirrors the Python SDK's dual-mode design with two entry points from one package:

```
mem0-ts/src/
├── client/     # MemoryClient — hosted platform client   → import from 'mem0ai'
├── oss/        # Memory — self-hosted pipeline           → import from 'mem0ai/oss'
│   └── src/
│       ├── memory/         # add/search pipeline (port of the Python design)
│       ├── llms/           # LLM providers
│       ├── embeddings/     # Embedding providers
│       ├── vector_stores/  # Vector store providers
│       ├── storage/        # History managers (SQLite, Supabase, in-memory)
│       ├── config/         # Config management
│       └── prompts/        # Extraction/update prompts
├── community/  # Community-maintained integrations (separate sub-package)
└── common/     # Shared types/utilities
```

Built with tsup (CJS + ESM), tested with jest, formatted with Prettier.

## Servers

### REST server (`server/`)

A FastAPI application that wraps the Python SDK's `Memory` class as a REST API for self-hosting. Deployed via Docker Compose with:

- **PostgreSQL + pgvector** — vector storage
- **Neo4j** — optional graph backend
- Auth (`auth.py`), rate limiting (`rate_limit.py`), Alembic migrations, and a dashboard

### OpenMemory (`openmemory/`)

A fuller self-hosted memory platform, also Docker Compose based:

- **`api/`** — FastAPI backend with SQLAlchemy models, Alembic migrations, REST routers, and an **MCP server** (`app/mcp_server.py`) so MCP-capable clients (Claude, Cursor, etc.) can read/write memories directly. Uses **Qdrant** as the vector store.
- **`ui/`** — Next.js 15 / React 19 dashboard (Radix UI, Redux Toolkit, Tailwind) for browsing and managing memories.

## CLIs (`cli/`)

Two functionally equivalent CLIs, defined by a shared spec (`cli/CLI_SPECIFICATION.md`, `cli/cli-spec.json`):

- **`cli/python/`** — Typer + Rich + httpx (`mem0-cli` on PyPI). Talks to the platform API; can drive the OSS SDK via the `[oss]` extra.
- **`cli/node/`** — Commander + Chalk (`@mem0/cli` on npm). Depends on the `mem0ai` npm package.

Both install a `mem0` entry point.

## Integrations (`integrations/`) and Skills (`skills/`)

Each integration is a self-contained npm package:

| Directory | Package | Purpose |
|-----------|---------|---------|
| `mem0-plugin/` | (marketplace plugin) | Editor plugins for Claude Code, Cursor, Codex: MCP server connection, lifecycle hooks for automatic memory capture, bundled skills. Contains nested `.opencode-plugin/` (`@mem0/opencode-plugin`). |
| `openclaw/` | `@mem0/openclaw-mem0` | OpenClaw plugin |
| `pi-agent-plugin/` | `@mem0/pi-agent-plugin` | Pi Agent plugin |
| `vercel-ai-sdk/` | `@mem0/vercel-ai-provider` | Vercel AI SDK memory provider |

MCP is the common protocol thread: the hosted MCP server (`mcp.mem0.ai`), the local OpenMemory MCP server, and the editor plugins all expose the same family of memory tools (`add_memory`, `search_memories`, `get_memories`, ...).

`skills/` holds Claude Code skill definitions in two categories:

- **Reference skills** (always-on SDK knowledge): `mem0/`, `mem0-cli/`, `mem0-vercel-ai-sdk/`
- **Pipeline skills** (run on demand): `mem0-integrate/`, `mem0-test-integration/`, `mem0-oss-to-platform/`

Marketplace plugins are registered in five `marketplace.json` files (repo root plus `.claude-plugin/`, `.cursor-plugin/`, `.codex-plugin/`, `.agents/plugins/`).

## Documentation (`docs/`)

Mintlify site organized by `platform/`, `open-source/`, `api-reference/` (with `openapi.json`), `integrations/`, `core-concepts/`, and `cookbooks/`. `docs/llms.txt` is a machine-readable index of every docs page; CI enforces that it stays in sync with the `.mdx` files (`scripts/check-llms-txt-coverage.py`).

## CI/CD Architecture

Both pipelines use a **single-router** pattern (workflows in `.github/workflows/`):

- **CI** — `ci-gate.yml` runs on every PR, detects which packages changed via path filters, invokes only the affected package workflows as reusable workflows (`workflow_call`), and aggregates results into one required **CI Gate** status check.
- **CD** — `release.yml` is the only workflow listening to `release: published`. It matches the release tag prefix (`v*` → Python SDK, `ts-v*` → TS SDK, `cli-v*`, `cli-node-v*`, `vercel-ai-v*`, `openclaw-v*`, `opencode-v*`, `pi-agent-v*`) and dispatches the matching package CD workflow. All publishing uses **OIDC trusted publishing** — no registry tokens are stored.

See the CI/CD tables in [AGENTS.md](AGENTS.md) for the full workflow-by-workflow breakdown.

## Cross-Cutting Concerns

- **Configuration** — Pydantic v2 models throughout Python (`mem0/configs/`); typed config objects in TypeScript. Everything is constructed from a single `MemoryConfig`-shaped dict/object.
- **Session scoping** — all memory operations are scoped by `user_id`, `agent_id`, and/or `run_id`, which map to vector-store filters.
- **History/auditability** — OSS mode journals every memory mutation (SQLite in Python, pluggable history managers in TS) to power `history()`.
- **Telemetry** — anonymized usage telemetry in both SDKs (`mem0/memory/telemetry.py`, `mem0-ts/src/client/telemetry.ts`).
- **Errors** — typed exception hierarchy in `mem0/exceptions.py`.
