# Aurellia

Your personal NYC tech event concierge agent. Aurellia discovers events from Luma, Eventbrite, and MLH, deduplicates and ranks them against your interests, and delivers a personalized morning digest — via web dashboard, email, or any MCP-compatible agent.

## Status

🔹 **v0.1.0 — feature-complete, integration testing in progress.**
The four agent layers (source → agent → workflow → web), the MCP server (~3 tools), and the security layer (Vault / Sanitizer / PrivacyManager) are all implemented. A seed script, two CI workflows, a deploy script, and a Claude Code skill (`scene-scout`) are in place. See `specs/SPECS.md` for the authoritative spec. See `docs/CHANGELOG.md` for release history.

Remaining milestones before a public cut: finish integration testing, deploy to a host (Railway / Render free tier — scripts exist), and write a short project writeup.

## Quick Start

```bash
# Install dependencies
uv sync --extra dev

# Run the full test suite (≥80% coverage enforced)
uv run pytest

# Verify every module imports and critical flows work
uv run python reproduce.py

# Run the web dashboard (http://localhost:8000)
uv run python -m aurellia serve

# Run a one‑shot pipeline for the next 7 days
uv run python -m aurellia run --period 7

# Seed local data with 8 sample NYC tech events
uv run python -m aurellia seed

# Start the MCP server (stdio)
uv run python -m aurellia mcp
```

Optional static analysis (configured in `pyproject.toml`):

```bash
uv run ruff check src/ tests/
uv run mypy src/aurellia/
```

## Architecture

Four layers, strict dependency order (top → bottom):

```
   Web (FastAPI + Jinja2 dashboard)
        ↓
   Agent (AurelliaAgent → Scout / Curate / Courier / Learn)
        ↓
   Workflow (Pipeline + Triggers + Cache)
        ↓
   Sources (Luma HTML · Eventbrite REST · MLH REST)  +  Models (Pydantic)

   Cross-cutting: Security (vault, sanitize, privacy) · Config · Utils
```

- **`src/aurellia/sources/`** — `LumaSource` (bs4 scraper, 30 req/min), `EventbriteSource` (OAuth REST, 50 req/min), `MLHSource` (open REST, 60 req/min). All normalize to a common `Event` model and degrade gracefully on failure. Selectors live in `config/sources.yaml` so a Luma redesign doesn't break the app.
- **`src/aurellia/agent/`** — `AurelliaAgent` orchestrates the four skills and holds `AgentMemory` (`data/agent_state.json`, topic weights + RSVP history). `ScoutSkill` fans out to sources; `CuratorSkill` deduplicates by URL then title similarity, filters, and ranks; `CourierSkill` builds the `Digest`; `LearnSkill` updates weights from RSVPs.
- **`src/aurellia/workflow/`** — `Pipeline` runs scout → curate → courier → learn; `Triggers` supports daily cron, interval, and manual runs.
- **`src/aurellia/web/`** — FastAPI app with 7 endpoints (`/health`, `/events`, `/events/{id}`, `/digest`, `POST /preferences`, `GET /preferences`, `/` dashboard) and a dark-themed Jinja2 dashboard with a timeline view.
- **`src/aurellia/mcp/`** — MCP server exposing `list_events`, `get_digest` (auth'd), `search_events` over stdio.
- **`src/aurellia/security/`** — `Vault` (Fernet AES-128-CBC, PBKDF2 100k iterations, master key from `AURELLIA_MASTER_KEY`), `Sanitizer` (HTML/SQL/URL validation + per-source `RateLimiter`), `PrivacyManager` (anonymization, retention, right to deletion).
- **`src/aurellia/config/`** — `Settings.from_env()` loads all config from env vars; `sources.yaml` holds CSS selectors.
- **`src/aurellia/utils/`** — `logger` (structlog JSON), `nlp` (text similarity for dedup), `geocode` (borough mapping).

See `AGENTS.md` for naming conventions, hard rules, and the dependency graph. See `specs/SPECS.md` for the full technical spec, data models, API contracts, and non-functional requirements.

## CLI

| Command | Purpose |
|---|---|
| `aurellia serve` | Start the FastAPI dashboard |
| `aurellia run --period N` | Run the pipeline for the next N days |
| `aurellia digest` | Print the current digest to stdout |
| `aurellia seed` | Insert sample NYC events |
| `aurellia mcp` | Start the MCP server on stdio |

## Configuration

All configuration is via environment variables — no secrets in code. Required: `AURELLIA_MASTER_KEY` (32+ chars). Optional: `EVENTBRITE_API_KEY`, `DATABASE_URL` (defaults to `sqlite:///data/aurellia.db`), `SCRAPE_INTERVAL`, `MAX_EVENTS`, `DIGEST_PERIOD_DAYS`, `MAX_DIGEST_ENTRIES`, `SMTP_*` for email delivery, `DEBUG`. Full list in `specs/SPECS.md` §6.

## Testing

95 tests across three layers, enforced by `pytest-cov` (≥80%):

- `tests/unit/` — models, curator, security, sources, NLP
- `tests/integration/` — API endpoints, full pipeline, digest generation
- `tests/e2e/` — end-to-end digest flow

External HTTP calls are mocked; tests are deterministic. Markers: `@pytest.mark.slow`, `@pytest.mark.integration`, `@pytest.mark.e2e`.

## CI / Deployment

- `.github/workflows/test.yml` — runs ruff, mypy, and pytest with coverage on push/PR.
- `.github/workflows/scrape.yml` — daily 8 AM UTC cron to refresh events.
- `scripts/deploy.sh` — runs tests, seeds, then deploys to Railway or Render (auto-detects via env vars) or falls back to a local `uv run python -m aurellia serve`.

## Roadmap

Planned for v0.2.0 (see `docs/CHANGELOG.md`):

- Partiful source integration
- Email digest delivery (SMTP)
- User authentication
- Deployment to Railway / Render
- Multi-city support (NYC only in v1)
- Real-time event updates via webhook (v1 is polling-based)

## License

MIT
