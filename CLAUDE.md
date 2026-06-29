# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Aurellia is a single-user concierge agent that discovers NYC tech events from Luma, Eventbrite, and MLH, deduplicates and ranks them against user interests, and delivers a personalized digest via a FastAPI web dashboard, email (SMTP stub), or MCP server. v0.1.0 is feature-complete; integration testing is in progress.

## Project rules (project rules, not generic advice)

The authoritative project rules live in `AGENTS.md` and are **loaded into every session**. Read `AGENTS.md` before doing real work — the rest of this file is a summary, not a replacement.

The non-obvious rules enforced there:

- **Module responsibility is strict.** Dependency graph: `Web → Agent → Workflow → Sources → Models`. Models never import Agent/Web/Workflow. Sources never import Web or MCP. Web imports from Agent only, never Sources directly. Agent imports Models, Sources, Security, Utils — never Web.
- **All imports are absolute.** No relative imports (`from ..models import Event` is banned).
- **Async/sync split:** pipeline code is sync (`httpx.Client`); FastAPI endpoints are `async def`. Don't mix in the same function.
- **External HTTP is never called in tests.** Mock with `unittest.mock.patch`. Tests must be deterministic.
- **Validate at the boundary.** Pydantic models validate on construction — don't mutate them, make new instances.
- **Logging:** `logger = get_logger(__name__)` from `aurellia.utils.logger`. Never log secrets, API keys, or raw user PII. Anonymize first.

## Specification

`specs/SPECS.md` is the authoritative technical specification — data models, API contracts, pipeline stages, NFRs (≤5s pipeline, ≥80% coverage, 90% availability target, 90-day data retention), risk mitigations. Read it when touching architecture or data flow.

## Commands

```bash
# Setup
uv sync --extra dev              # install with dev extras

# Test (≥80% coverage enforced via --cov-fail-under=80)
uv run pytest                                    # full suite
uv run pytest tests/unit/test_curator.py          # single file
uv run pytest -k "test_fetch_events"              # single test by name
uv run pytest -m "not slow"                      # skip slow tests

# Static analysis
uv run ruff check src/ tests/                     # lint
uv run mypy src/aurellia/                        # strict type check (zero errors required)

# Run
uv run python reproduce.py                        # smoke-check imports + instantiation
uv run python -m aurellia serve                    # web dashboard on :8000
uv run python -m aurellia run --period 7          # one-shot pipeline → stdout
uv run python -m aurellia run --format html        # HTML digest
uv run python -m aurellia digest --period week    # digest for default user
uv run python -m aurellia seed                     # load 8 sample NYC events
uv run python -m aurellia mcp                      # MCP server on stdio
```

## Architecture

Four layers, top-down call order. The entry points are `AurelliaAgent.run_pipeline()` (`src/aurellia/agent/core.py:90`) and `Pipeline.run()` (`src/aurellia/workflow/pipeline.py:35`) — the latter wraps the former with TTL caching (`run_cached`, `scrape_interval_minutes`).

**Pipeline shape (Scout → Curate → Courier → Learn):**
- `ScoutSkill.execute()` fans out to all three `SourceBase` subclasses in `src/aurellia/sources/`, each returning normalized `Event` models. Sources degrade gracefully (one failure doesn't abort the pipeline). Selectors live in `config/sources.yaml`, not hardcoded.
- `CuratorSkill.execute(events, user)` deduplicates by URL then title similarity (`utils/nlp.py`), filters by borough/price/date, and ranks by relevance score (10 results default, configurable).
- `CourierSkill.build(user, curated, period_days)` produces a `Digest` with ranked `DigestEntry`s.
- `LearnSkill.execute()` reads RSVP history and adjusts topic weights in `AgentMemory`.

**Agent state** (`src/aurellia/agent/memory.py`): persists to `data/agent_state.json` as `topic_weights` + `rsvp_history`. The RSVP→weights loop is the feedback mechanism: RSVP'd event tags get +0.1 to their topic weight (capped at ±1.0).

**Persistence strategy:** Pydantic models (not ORM). The `database_url` setting defaults to SQLite at `data/aurellia.db`, but the codebase currently uses JSON-file state, not SQLAlchemy — SQLAlchemy is a declared dependency for future use, not the active persistence layer.

**Web layer:** `src/aurellia/web/app.py` is a FastAPI app with 7 endpoints rendered via Jinja2 templates (`web/templates/dashboard.html`) with inline CSS (`web/static/style.css`, ~99 lines). No auth by design (single-user).

**MCP server:** `src/aurellia/mcp/__init__.py` exposes `list_events`, `get_digest` (auth'd), `search_events` over stdio.

**Security:** `src/aurellia/security/vault.py` (Fernet + PBKDF2, master key from `AURELLIA_MASTER_KEY`), `sanitize.py` (HTML/SQL/URL validation + per-source sliding-window `RateLimiter`), `privacy.py` (anonymization, retention, deletion).

## Configuration

`src/aurellia/config/settings.py` exposes `Settings.from_env()`, a frozen dataclass read from env vars. `specs/SPECS.md` §6 is the full reference. Required: `AURELLIA_MASTER_KEY` (32+ chars, used by Vault). Optional but load-bearing: `EVENTBRITE_API_KEY`, `DATABASE_URL` (defaults to `sqlite:///data/aurellia.db`), `MAX_DIGEST_ENTRIES` (default 10), `MIN_RELEVANCE_SCORE` (default 0.3), `SMTP_*`. `DEBUG` accepts `1|true|yes`.

`config/sources.yaml` holds per-source CSS selectors and rate limits — if a scrape returns 0 events, check this before touching source code.

## Testing structure

- `tests/unit/` — models, curator, security, sources, NLP (isolated)
- `tests/integration/` — FastAPI routes (`test_api.py`), full pipeline (`test_pipeline.py`), digest generation (`test_digest_e2e.py`)
- `tests/e2e/` — end-to-end digest flow (`test_digest_e2e.py`)

`tests/conftest.py` provides four shared fixtures: `temp_db_path` (temp SQLite file), `sample_events` (4 events across all sources including Partiful), `sample_user` (with preferences), `sample_digest` (ranked entries). pytest markers: `@pytest.mark.slow`, `@pytest.mark.integration`, `@pytest.mark.e2e`.

## Commit style

Conventional Commits: `feat:`, `fix:`, `chore:`, `docs:`, `test:`, `refactor:`. Atomic commits — one logical change per commit. `AGENTS.md` §4.5 forbids direct commits to `main` and mandates `feat/`, `fix/`, `chore/`, etc. branches.

## CI / Deployment

- `.github/workflows/test.yml` — push/PR to `main` runs ruff, mypy, pytest with coverage (≥80% enforced).
- `.github/workflows/scrape.yml` — daily 8 AM UTC cron, runs `uv run python -m aurellia run --period 7`, uploads `data/` as artifact.
- `scripts/deploy.sh` — runs tests, seeds, then auto-detects Railway (`$RAILWAY_PROJECT_ID`) vs Render (`$RENDER_GIT_REPO`) for git-push deploy, else falls back to local serve.

## Files to be aware of

- `AGENTS.md` — project rules, loaded every session. Always read first.
- `specs/SPECS.md` — full technical specification.
- `README.md` — user-facing overview, commands reference.
- `pyproject.toml` — hatchling build, entry point `aurellia` → `aurellia.__main__:main`, ruff/mypy/pytest config.
- `docs/CHANGELOG.md` — release history; v0.2.0 roadmap (Partiful, SMTP, auth, multi-city, webhooks).
- `skills/scene-scout/SKILL.md` — Claude Code skill used by this project.
