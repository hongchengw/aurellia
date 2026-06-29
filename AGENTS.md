# AGENTS.md — Aurellia Project Rules

> **Project:** Aurellia — GitHub Trending repo scout agent (Kaggle x Google AI Agents Capstone)
> **Stack:** Python 3.12+ · uv · FastAPI · Pydantic v2 · httpx · BeautifulSoup4 · Ruff + mypy strict

---

## 1. Role & Boundaries

Aurellia is a **concierge agent** that discovers trending GitHub repos, curates them to user interests, and delivers personalized digests. It does NOT:
- Make unsolicited external API calls outside the pipeline
- Store user data without encryption
- Expose internal agent state through web endpoints

---

## 2. Naming Conventions

| Element | Convention | Example |
|---|---|---|
| **Files** | `snake_case.py` | `repo.py`, `scout.py` |
| **Classes** | `PascalCase` | `RepoSource`, `AurelliaAgent` |
| **Functions** | `snake_case` | `fetch_repos`, `run_pipeline` |
| **Constants** | `UPPER_SNAKE_CASE` | `BASE_URL`, `RATE_LIMIT_PER_MINUTE` |
| **Private** | `_leading_underscore` | `_parse_repo`, `_cache` |
| **Tests** | `test_<module>.py` / `Test<Subject>` / `test_<action>_<condition>` | `test_sources.py`, `TestGithubSource` |
| **Env vars** | `AURELLIA_<KEY>` | `AURELLIA_MASTER_KEY` |
| **MCP tools** | `verb_noun` | `list_repos`, `get_digest` |
| **Branches** | `feat/`, `fix/`, `chore/` | `feat/mcp-server` |

---

## 3. Coding Style

- **Type annotations** on all function signatures. Return types required. `mypy --strict` must pass.
- **Google-style docstrings** on all public modules, classes, and functions.
- **Absolute imports** only. Order: stdlib → third-party → first-party (`aurellia`).
- **Ruff** enforced: `line-length = 100`, `target-version = "py312"`, double quotes, trailing commas.
- **Fail fast, fail loud.** Validate inputs at boundaries. Raise specific exceptions (`SourceError`).
- **Catch specific exceptions**, never bare `except`. Never catch `Exception` silently.
- **Never log secrets, API keys, or user PII.** Use `logger.anonymize()` before logging user data.
- **Scraping:** `httpx.Client` (sync). **Web:** `async def` in FastAPI. **Don't mix.**

---

## 4. Architecture

### 4.1 Layers

```
Web (FastAPI) → Agent (AurelliaAgent + 4 skills) → Workflow (Pipeline + Triggers) → Sources (GitHub Trending) → Models (Pydantic)
Cross-cutting: Security · Config · Utils
```

### 4.2 Module Map

| Module | Responsibility |
|---|---|
| `agent/core.py` | Orchestrates skills, runs pipeline |
| `agent/memory.py` | Persistent state: star history, topic weights |
| `agent/skills/scout.py` | Fetches from GitHub Trending (all, python, typescript) |
| `agent/skills/curate.py` | Dedup → filter → rank |
| `agent/skills/courier.py` | Build digest (markdown/HTML/email) |
| `agent/skills/learn.py` | Feedback loop, preference learning |
| `sources/github.py` | GitHub Trending HTML scraper (bs4) |
| `mcp/__init__.py` | MCP server: `list_repos`, `get_digest`, `search_repos` |
| `workflow/pipeline.py` | Scout → Curate → Courier → Learn |
| `web/app.py` | FastAPI: `/repos`, `/digest`, `/preferences` |

### 4.3 Dependency Rules

- **Models** never import from Agent, Web, or Workflow.
- **Sources** never import from Web or MCP.
- **Agent** imports from Models, Sources, Security, Utils — never from Web.
- **Web** imports from Agent — never from Sources directly.

---

## 5. Build & Workflow Commands

```bash
uv sync --extra dev          # Install dependencies
uv run pytest                # Run tests (≥80% coverage enforced)
uv run ruff check src/ tests/ # Lint
uv run mypy src/aurellia/    # Type check
uv run python -m aurellia serve   # Start web dashboard
uv run python -m aurellia run --period 7  # Run pipeline
uv run python -m aurellia seed      # Seed sample repos
uv run python -m aurellia mcp       # Start MCP server
```

---

## 6. Hard Rules

### 6.1 Security
- **NEVER** hardcode secrets. **NEVER** commit `.env` or `aurellia.db`.
- **ALWAYS** use `Vault` for secrets. **ALWAYS** sanitize input. **ALWAYS** anonymize user IDs in logs.
- Rate limiting on all external API calls.

### 6.2 Data
- **ALWAYS** validate at boundary (source → model). **ALWAYS** use Pydantic.
- **NEVER** mutate model instances in place. Right to be forgotten enforced.

### 6.3 Testing
- **ALWAYS** write tests before merging. **ALWAYS** mock HTTP calls. **NEVER** make real API calls in tests.
- Coverage target: ≥80%. Tests must be deterministic.

### 6.4 Code Quality
- **ALWAYS** run `ruff` + `mypy` + `pytest` before committing.
- Max function length: 50 lines. Max file length: 300 lines (excluding tests).

### 6.5 Context Loading Order
1. Read `README.md` → `AGENTS.md` → source code in `src/aurellia/`
2. Do NOT read `specs/`, `docs/CHANGELOG.md`, or YAML schemas unless explicitly asked.
3. Specs may be stale. Source code is the source of truth.

### 6.6 Changelog
- **ALWAYS** log changes in `docs/CHANGELOG.md` before merging. Use [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) format.
- Every behavior-changing PR MUST include a changelog entry.

### 6.7 Git
- **NEVER** commit to `main` directly. **ALWAYS** use `feat/<description>` branches.
- Conventional commits: `feat:`, `fix:`, `chore:`, `docs:`, `test:`, `refactor:`.

---

## 7. Skills Catalog

| Skill | Trigger | Description |
|---|---|---|
| `scene-scout` | Manual / Claude Code | Guided repo discovery workflow |

*(Skills loaded on demand — not auto-loaded into context)*

---

## 8. Tool Versions

| Tool | Version | Purpose |
|---|---|---|
| Python | 3.12.x | Runtime |
| uv | 0.7.x | Package manager |
| FastAPI | 0.115.x | Web framework |
| Pydantic | 2.9.x | Data models |
| httpx | 0.27.x | HTTP client |
| BeautifulSoup4 | 4.12.x | HTML parsing |
| APScheduler | 3.10.x | Task scheduling |
| structlog | 24.x | Structured logging |
| MCP SDK | 0.1.x | MCP server |
| pytest | 8.x | Test framework |
| Ruff | 0.6.x | Linter/formatter |
| mypy | 1.13.x | Type checker |

---

*Last updated: 2026-06-29 | Version: 0.3.0*
