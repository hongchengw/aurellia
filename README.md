# Aurellia

Your personal GitHub Trending repo scout. Aurellia discovers trending repositories from GitHub, filters and ranks them against your interests, and delivers a personalized morning digest — via web dashboard, email, or any MCP-compatible agent.

> **Project:** Aurellia — Kaggle x Google AI Agents Capstone (Concierge Track)
> **Why GitHub Trending?** Event sites (Luma, Eventbrite, MLH) have aggressive bot protection. GitHub encourages scraping and provides far richer data — stars, forks, languages, topics — that reveals real patterns in how AI/LLM systems are built.

## Status

🔹 **v0.3.0 — pivot to GitHub Trending complete, 96 tests passing.**

The four agent layers (source → agent → workflow → web), the MCP server (3 tools), and the security layer (Vault / Sanitizer / PrivacyManager) are all implemented. A seed script, CI workflows, and a Claude Code skill are in place. See `specs/SPECS.md` for the authoritative spec. See `docs/CHANGELOG.md` for release history.

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

# Run a one-shot pipeline to get today's trending repos
uv run python -m aurellia run --period 7

# Seed local data with sample repos
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
   Sources (GitHub Trending HTML)  +  Models (Pydantic)

   Cross-cutting: Security (vault, sanitize, privacy) · Config · Utils
```

- **`src/aurellia/sources/`** — `GithubTrendingSource` (BeautifulSoup4 scraper, 30 req/min). Scrapes GitHub Trending across all languages, Python, and TypeScript. Infers category (LLM, AI Agent, Framework, Tool, etc.) from title/description. All repos normalize to a common `Repo` model.
- **`src/aurellia/agent/`** — `AurelliaAgent` orchestrates the four skills and holds `AgentMemory` (`data/agent_state.json`, topic weights + star history). `ScoutSkill` fetches from GitHub Trending; `CuratorSkill` deduplicates by URL, filters by language/preferences, and ranks by relevance; `CourierSkill` builds the `Digest`; `LearnSkill` updates weights from stars/skips.
- **`src/aurellia/workflow/`** — `Pipeline` runs scout → curate → courier → learn; `Triggers` supports daily cron, interval, and manual runs.
- **`src/aurellia/web/`** — FastAPI app with endpoints for `/repos`, `/digest`, `/preferences`, `/health`, and `/` dashboard.
- **`src/aurellia/mcp/`** — MCP server exposing `list_repos`, `get_digest` (auth'd), `search_repos` over stdio.
- **`src/aurellia/security/`** — `Vault` (Fernet AES-128-CBC, PBKDF2), `Sanitizer` (HTML/SQL/URL validation + language validation), `PrivacyManager` (anonymization, retention, right to deletion).
- **`src/aurellia/config/`** — `Settings.from_env()` loads all config from env vars.
- **`src/aurellia/utils/`** — `logger` (structlog JSON), `nlp` (keyword extraction, summarization, category enrichment).

See `AGENTS.md` for naming conventions, hard rules, and the dependency graph. See `specs/SPECS.md` for the full technical spec, data models, API contracts, and non-functional requirements.

## CLI

| Command | Purpose |
|---|---|
| `aurellia serve` | Start the FastAPI dashboard |
| `aurellia run --period N` | Run the pipeline for the next N days |
| `aurellia digest` | Print the current digest to stdout |
| `aurellia seed` | Insert sample repos |
| `aurellia mcp` | Start the MCP server on stdio |

## Configuration

All configuration is via environment variables — no secrets in code. Required: `AURELLIA_MASTER_KEY` (32+ chars). Optional: `GITHUB_TOKEN` (for higher API rate limits), `DATABASE_URL` (defaults to `sqlite:///data/aurellia.db`), `SCRAPE_INTERVAL`, `MAX_REPOS`, `DIGEST_PERIOD_DAYS`, `MAX_DIGEST_ENTRIES`, `SMTP_*` for email delivery`, `DEBUG`. Full list in `specs/SPECS.md` §6.

## Testing

96 tests across three layers, enforced by `pytest-cov` (≥80%):

- `tests/unit/` — models, curator, security, sources, NLP
- `tests/integration/` — API endpoints, full pipeline, digest generation
- `tests/e2e/` — full user journey

External HTTP calls are mocked; tests are deterministic. Markers: `@pytest.mark.slow`, `@pytest.mark.integration`, `@pytest.mark.e2e`.

## Categories

Aurellia automatically categorizes trending repos:

| Category | Example Keywords |
|---|---|
| LLM | llm, gpt, claude, gemini, llama, rag, chatbot, embeddings |
| AI Agent | agent, agentic, autonomous, multi-agent, crew, swarm, langchain |
| Framework | framework, nextjs, react, django, fastapi, spring |
| Tool | tool, cli, utility, helper, automation |
| Data | data, pandas, spark, analytics, sql |
| Web | web, frontend, backend, javascript, typescript |
| Security | security, crypto, vulnerability, auth |
| DevTools | devops, docker, kubernetes, deploy |
| Game | game, unity, godot, unreal |

## CI / Deployment

- `.github/workflows/test.yml` — runs ruff, mypy, and pytest with coverage on push/PR.
- `.github/workflows/scrape.yml` — daily cron to refresh trending repos.

## License

MIT
