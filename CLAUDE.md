# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Aurellia is a single-user concierge agent that discovers trending GitHub repositories, filters and ranks them against user interests, and delivers a personalized digest via a FastAPI web dashboard, email (SMTP), or MCP server. v0.3.0 is a clean scaffold — no features are implemented yet. Features will be built one at a time and reviewed individually.

## Current State

The project directory structure is in place (`src/aurellia/` with all subpackages), but every `.py` file is empty (0 bytes). Tests are configured but empty. The `pyproject.toml` defines all dependencies and tool configurations. No code exists to reference yet.

## Project Rules

Read `AGENTS.md` before any real work. Key rules:

- **Dependency graph:** `Web → Agent → Workflow → Sources → Models`. Strict — lower layers never import higher layers.
- **All imports are absolute.** No relative imports.
- **Async/sync split:** pipeline code is sync (`httpx.Client`); FastAPI endpoints are `async def`. Don't mix.
- **Validate at the boundary.** Pydantic models validate on construction.
- **Logging:** `logger = get_logger(__name__)` from `aurellia.utils.logger`.
- **No external HTTP in tests.** Mock everything.

## Specification

`specs/SPECS.md` is the authoritative technical specification — data models, API contracts, pipeline stages, NFRs. The subdirectories contain:
- `database-schema.yaml` — Pydantic models + DDL
- `api-contracts.yaml` — OpenAPI-style endpoint specs
- `bdd-scenarios.md` — Gherkin scenarios

**When building a feature, read the corresponding spec first.**

## Feature Build Order

Each feature should be built in its own branch with tests, then reviewed:

1. Models → 2. Sources → 3. Scout → 4. Curate → 5. Courier → 6. Learn → 7. Agent Core → 8. Security → 9. Web → 10. MCP → 11. Workflow → 12. CLI

## Commands

```bash
uv sync --extra dev              # install dependencies
uv run pytest                    # run tests (currently 0)
uv run ruff check src/ tests/     # lint
uv run mypy src/aurellia/        # type check
```

## Commit Style

Conventional Commits required: `feat:`, `fix:`, `chore:`, `docs:`, `test:`, `refactor:`. Atomic commits — one logical change per commit.

## License

MIT
