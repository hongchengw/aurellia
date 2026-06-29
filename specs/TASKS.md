# Aurellia — Task Breakdown

> **Generated from:** `AGENTS.md`, `specs/api-contracts.yaml`, `specs/bdd-scenarios.md`
> **Scaffold version:** 0.3.0
> Each task = one implementable unit. Grouped by feature phase in dependency order.
> Acceptance: every task's tests must pass, `ruff` + `mypy --strinct` clean, coverage ≥80%.

---

## Phase 1: Models (`src/aurellia/models/`)

### T1.1 — Repo model
- File: `models/__init__.py` (or `models/repo.py`)
- Pydantic `BaseModel` matching `RepoDTO` in `api-contracts.yaml`
- Required: `title` (str, 1–500), `url` (HttpUrl), `source` (Literal["github"])
- Optional: `description`, `repo_category` (enum), `language`, `stars`, `forks`, `stars_today`, `tags`, `skill_level` (enum), `relevance_score` (ge=0, le=1)
- Validators: URL must start with `https://`, strings stripped
- Tests: `tests/unit/test_models.py` — valid/invalid Repo construction, validator behavior

### T1.2 — Enums
- `RepoCategory`: llm, ai_agent, framework, tool, data, webdev, security, devtools, game, other
- `SkillLevel`: beginner, intermediate, advanced, any
- `DigestPeriod`: today, week, month
- Tests: enum serialization, invalid value rejection

### T1.3 — User model
- Pydantic model matching `UserPreferenceDTO`
- Fields: `interests: list[str]`, `languages: list[str]`, `skill_level: SkillLevel`
- Tests: valid/invalid construction

### T1.4 — Digest models
- `DigestEntry`: rank, title, url, stars, stars_today, language, relevance_score, reason
- `Digest`: entries (list[DigestEntry]), period (DigestPeriod), generated_at (datetime)
- Tests: model construction, serialization

---

## Phase 2: Sources (`src/aurellia/sources/`)

### T2.1 — SourceError exception
- Custom exception with `source: str` field
- Tests: exception attributes, string representation

### T2.2 — Base source ABC
- `Source` abstract base class with `fetch() -> list[Repo]` interface
- Tests: cannot instantiate ABC directly

### T2.3 — GitHub Trending scraper
- File: `sources/github.py`
- Scrape `https://github.com/trending` (and `/trending/python`, `/trending/typescript`)
- Parse with BeautifulSoup4: extract title, description, URL, language, stars, forks, stars_today
- Infer `repo_category` from title/description keywords (llm, ai_agent, etc.)
- Handle network failure → raise `SourceError(source="github")`
- Skip invalid/malformed entries gracefully
- Use `httpx.Client` (sync)
- Tests: mock HTML responses, parse valid page, handle network error, handle malformed entries, language-specific pages

---

## Phase 3: Scout Skill (`src/aurellia/agent/skills/scout.py`)

### T3.1 — Scout skill
- Wraps sources, calls `GitHubTrendingSource.fetch()`
- Returns `list[Repo]`
- Handles `SourceError` → returns empty list, logs error
- Tests: successful fetch, source failure returns empty list, all/python/typescript feeds

---

## Phase 4: Curate Skill (`src/aurellia/agent/skills/curate.py`)

### T4.1 — Deduplication
- Remove duplicate repos by URL (keep first, merge sources)
- Remove duplicate repos by title (case-insensitive)
- Tests: dedup by URL, dedup by title, keep distinct repos

### T4.2 — Language filtering
- Filter repos by user language preferences
- No filter when user has no language preference
- Tests: filter by language, no filter when empty

### T4.3 — Ranking
- Score by interest tag matching (higher = more matching tags)
- Boost for preferred language
- Boost for stars_today (trending)
- All scores normalized to [0.0, 1.0]
- Sort descending by relevance_score
- Tests: interest matching, language boost, trending boost, score bounds, sort order

---

## Phase 5: Courier Skill (`src/aurellia/agent/skills/courier.py`)

### T5.1 — Digest builder
- Takes curated repos + user preferences → `Digest`
- Max 10 entries (configurable)
- Each entry includes personalized reason string
- Tests: digest creation, entry limit, reason generation

### T5.2 — Markdown formatter
- Format `Digest` as markdown string with titles, stars, links
- Tests: valid markdown output, all entries present

### T5.3 — HTML formatter
- Format `Digest` as HTML string with repo cards
- Tests: valid HTML output, all entries present

### T5.4 — Email delivery (optional/stretch)
- Send digest via SMTP
- Handle failure gracefully (return False, log error)
- Tests: successful send, SMTP failure

---

## Phase 6: Learn Skill (`src/aurellia/agent/skills/learn.py`)

### T6.1 — Feedback processing
- Accept positive/negative feedback on topics
- Update topic weights (positive → increase, negative → decrease)
- Tests: positive feedback increases weight, negative decreases, weight bounds

---

## Phase 7: Agent Core (`src/aurellia/agent/`)

### T7.1 — AgentMemory
- Persistent state: user preferences, star history, topic weights
- Store/retrieve user preferences by user_id
- Track star history per user
- Delete all user data (right to be forgotten)
- Tests: remember preferences, track stars, learn from feedback, delete account

### T7.2 — AurelliaAgent orchestrator
- Compose skills: scout → curate → courier → learn
- Run pipeline with user context
- Welcome message for new users
- Tests: full pipeline execution, empty source handling, new user greeting

---

## Phase 8: Security (`src/aurellia/security/`)

### T8.1 — Vault
- Encrypt/decrypt secrets using `AURELLIA_MASTER_KEY`
- Raise `EnvironmentError` if key missing
- Tests: store/retrieve secret, encryption verified, missing key raises error

### T8.2 — Sanitizer
- Strip HTML tags (XSS prevention)
- Remove SQL injection characters/keywords
- Validate programming language names (normalize "Python" → "python")
- Raise `ValueError` for invalid languages
- Tests: HTML sanitization, SQL sanitization, valid language, invalid language rejection

### T8.3 — PrivacyManager
- Anonymize user data in logs (email → `[EMAIL]`)
- Rate limiter: max_requests per window, block excess, reset after window
- Data retention: purge data older than retention period
- Tests: email anonymization, rate limit enforcement, rate limit reset, data retention purge

---

## Phase 9: Web Dashboard (`src/aurellia/web/`)

### T9.1 — FastAPI app + health endpoint
- Create FastAPI app
- `GET /health` → `{"status": "healthy", "version": "0.3.0"}`
- Tests: health check returns 200

### T9.2 — Repos endpoint
- `GET /repos` with query params: `language`, `topic`, `min_stars`
- Returns JSON array of Repo objects
- Tests: 200 response, language filter, topic filter, min_stars filter

### T9.3 — Digest endpoint
- `GET /digest` with required `user_id`, optional `period`
- Returns personalized digest
- 422 if `user_id` missing
- Tests: valid request, missing user_id returns 422

### T9.4 — Preferences endpoints
- `GET /preferences?user_id=...` → user preferences
- `POST /preferences` → update preferences
- 422 for invalid skill_level or language
- Tests: get preferences, update preferences, invalid skill_level, invalid language

---

## Phase 10: MCP Server (`src/aurellia/mcp/`)

### T10.1 — MCP server setup
- Initialize MCP server with server name "aurellia"
- Tests: server starts, server name correct

### T10.2 — list_repos tool
- Parameters: `language` (optional), `topic` (optional), `min_stars` (default 0)
- Returns list of RepoDTO
- Tests: list all repos, filter by language

### T10.3 — get_digest tool
- Parameters: `user_id` (required), `period` (default "week")
- Returns DigestDTO
- Raise `ValueError` if `user_id` missing
- Tests: valid request, missing user_id raises error

### T10.4 — search_repos tool
- Parameters: `query` (required, natural language)
- Returns list of matching RepoDTO
- Tests: search returns results

---

## Phase 11: Workflow (`src/aurellia/workflow/`)

### T11.1 — Pipeline
- Orchestrate: scout → curate → courier → learn
- Accept user context, return Digest
- Handle empty results gracefully
- Tests: full pipeline, empty sources, pipeline with feedback

### T11.2 — Triggers
- DailyTrigger: fires at configured time (e.g., 08:00)
- IntervalTrigger: fires after N minutes elapsed
- `should_fire()` method for each
- Tests: daily trigger fires at correct time, doesn't fire at wrong time, interval trigger respects elapsed time

---

## Phase 12: Config & Utils

### T12.1 — Settings
- Load config from env vars: `AURELLIA_MASTER_KEY`, `GITHUB_TOKEN`, `DATABASE_URL`, `SCRAPE_INTERVAL`, `MAX_REPOS`, `DIGEST_PERIOD_DAYS`, `MAX_DIGEST_ENTRIES`, `SMTP_*`, `DEBUG`
- Validate required settings
- Tests: load from env, missing required raises error, defaults applied

### T12.2 — Logger
- structlog configuration
- Anonymize helper for user data in log messages
- Tests: logger configuration, anonymize helper

### T12.3 — NLP utilities
- Keyword extraction for category inference
- Tag matching for relevance scoring
- Tests: keyword extraction, tag matching

---

## Phase 13: CLI (`src/aurellia/__main__.py`)

### T13.1 — CLI entry point
- `aurellia serve` → start FastAPI dashboard
- `aurellia run --period N` → run pipeline
- `aurellia digest` → print current digest
- `aurellia seed` → insert sample repos
- `aurellia mcp` → start MCP server
- Tests: each command invokes correct function

---

## Phase 14: Integration & E2E Tests (`tests/integration/`, `tests/e2e/`)

### T14.1 — Integration tests
- Test agent + skills together with mocked sources
- Test web endpoints with TestClient
- Test MCP tools with mock agent

### T14.2 — E2E tests
- New user onboarding flow
- Personalized digest delivery
- Star repo → agent learns → next digest improved
- Account deletion

---

## Summary

| Phase | Feature | Tasks |
|-------|---------|-------|
| 1 | Models | 4 |
| 2 | Sources | 3 |
| 3 | Scout Skill | 1 |
| 4 | Curate Skill | 3 |
| 5 | Courier Skill | 4 |
| 6 | Learn Skill | 1 |
| 7 | Agent Core | 2 |
| 8 | Security | 3 |
| 9 | Web Dashboard | 4 |
| 10 | MCP Server | 4 |
| 11 | Workflow | 2 |
| 12 | Config & Utils | 3 |
| 13 | CLI | 1 |
| 14 | Integration & E2E | 2 |
| **Total** | | **37** |

---

*Last updated: 2026-06-29 | Generated for Aurellia v0.3.0*
