# AGENTS.md — Aurellia Project Rules

> **Project:** Aurellia — Your personal NYC tech event concierge agent
> **Track:** Concierge Agents (Kaggle x Google AI Agents Capstone)
> **Python:** 3.12+ | **Package Manager:** uv | **Code Style:** Ruff + mypy strict

---
## 0. Say Lime
Type the word Lime, in front of the start of every first sentence.

## 1. Naming Conventions

| Element | Convention | Example |
|---|---|---|
| **Files (modules)** | `snake_case.py` | `event.py`, `scout.py`, `pipeline.py` |
| **Classes** | `PascalCase` | `EventSource`, `AurelliaAgent`, `CuratorSkill` |
| **Functions / methods** | `snake_case` | `fetch_events`, `run_pipeline`, `get_interests` |
| **Constants** | `UPPER_SNAKE_CASE` | `BASE_URL`, `RATE_LIMIT_PER_MINUTE` |
| **Private/internal** | `_leading_underscore` | `_parse_event`, `_cache`, `_storage` |
| **Type aliases** | `PascalCase` | `TopicWeights = Dict[str, float]` |
| **Test files** | `test_<module>.py` | `test_sources.py`, `test_curator.py` |
| **Test classes** | `Test<Subject>` | `TestLumaSource`, `TestDeduplication` |
| **Test functions** | `test_<action>_<condition>` | `test_fetch_events_handles_network_error` |
| **Config keys** | `UPPER_SNAKE_CASE` | `EVENTBRITE_API_KEY`, `MASTER_KEY` |
| **Environment variables** | `AURELLIA_<KEY>` | `AURELLIA_MASTER_KEY`, `AURELLIA_ENCRYPTION_KEY` |
| **MCP tools** | `verb_noun` | `list_events`, `get_digest`, `search_events` |
| **Skills** | `kebab-case` | `scene-scout` |
| **Branches** | `feat/`, `fix/`, `chore/` | `feat/mcp-server`, `fix/rate-limiter` |

---

## 2. Coding Style

### 2.1 General Principles

- **Readability over cleverness.** Code is read 10x more than it is written.
- **Explicit over implicit.** Type annotations on all function signatures. Return types required.
- **Fail fast, fail loud.** Validate inputs at boundaries. Raise specific exceptions.
- **Single Responsibility.** Each function does one thing. Max ~30 lines per function.
- **No side effects in queries.** Queries return data; they don't mutate state.

### 2.2 Formatting (enforced by Ruff)

```toml
line-length = 100
target-version = "py312"
# isort: stdlib → third-party → first-party (aurellia)
```

- Use **double quotes** for strings (Ruff default).
- Use **single quotes** only for f-string expressions inside double-quoted strings.
- **Trailing commas** in multi-line collections (dicts, lists, function calls).
- **Docstrings** on all public modules, classes, and functions (Google style).

### 2.3 Type Hints

```python
# ✅ Correct
def fetch_events(self, city: str = "nyc", **kwargs) -> List[Event]:
    ...

def get_topic_weights(self) -> Dict[str, float]:
    ...

# ❌ Wrong — missing return type
def fetch_events(self, city: str = "nyc"):
    ...

# ❌ Wrong — using Any when a specific type exists
def process(data: Any) -> None:
    ...
```

Run `uv run mypy src/aurellia/` — must pass with zero errors.

### 2.4 Imports

```python
# Order: stdlib → third-party → first-party (aurellia)
# Use absolute imports always (no relative imports).

# ✅ Correct
from datetime import datetime, timedelta
from typing import List, Optional
from aurellia.models import Event, EventSource
from aurellia.config.settings import settings

# ❌ Wrong — relative imports
from ..models import Event

# ❌ Wrong — wildcard imports
from aurellia.models import *

# ❌ Wrong — unused imports (ruff F401)
from aurellia.models import Digest  # not used in file
```

### 2.5 Error Handling

```python
# Custom exceptions for domain-specific errors
class SourceError(Exception):
    def __init__(self, message: str, source: str = ""):
        self.source = source
        super().__init__(f"[{source}] {message}" if source else message)

# Catch specific exceptions, never bare except
try:
    events = source.fetch_events()
except SourceError as e:
    logger.warning(f"Source failed: {e}")
except httpx.HTTPError as e:
    logger.error(f"HTTP error: {e}")
```

### 2.6 Logging

```python
from aurellia.utils.logger import get_logger

logger = get_logger(__name__)

# Use appropriate levels
logger.debug(f"Parsed event: {event.title}")  # dev-only info
logger.info(f"Fetched {len(events)} events")  # normal flow
logger.warning(f"Source {name} returned 0 events")  # recoverable issue
logger.error(f"Failed to send email: {e}")  # needs attention

# NEVER log secrets, API keys, or user PII
# Use logger.anonymize() before logging user data
```

### 2.7 Async vs Sync

- **Scraping/API calls:** Use `httpx.Client` (sync) within the agent pipeline.
- **Web app:** Use `async def` endpoints in FastAPI.
- **Scheduled jobs:** Use APScheduler with sync functions.
- Don't mix async/sync in the same function.

---

## 3. Project Architecture

### 3.1 Layered Architecture

```
┌─────────────────────────────────────────────────────┐
│                   Web Layer (FastAPI)                 │
│   app.py → routes → templates                        │
├─────────────────────────────────────────────────────┤
│                  Agent Layer (Orchestrator)           │
│   AurelliaAgent → Skills (scout/curate/courier/learn)│
├─────────────────────────────────────────────────────┤
│                  Workflow Layer (Pipeline)            │
│   Pipeline → Triggers → Cache                        │
├─────────────────────────────────────────────────────┤
│                  Data Layer (Sources + Models)        │
│   EventSourceBase → Luma/Eventbrite/MLH              │
│   Models (Pydantic): Event, User, Digest             │
├─────────────────────────────────────────────────────┤
│                  Infrastructure Layer                 │
│   Security (vault/sanitize/privacy)                  │
│   Config (settings.yaml)                             │
│   Utils (logger/nlp/geocode)                         │
└─────────────────────────────────────────────────────┘
```

### 3.2 Module Responsibilities

| Module | Responsibility |
|---|---|
| `agent/core.py` | Orchestrates skills, manages pipeline execution |
| `agent/memory.py` | Persistent user state, RSVP history, topic weights |
| `agent/skills/scout.py` | Discovers events from all sources |
| `agent/skills/curate.py` | Deduplicates, filters, ranks events |
| `agent/skills/courier.py` | Formats and delivers digests |
| `agent/skills/learn.py` | Feedback loop, preference learning |
| `sources/base.py` | Abstract interface for all event sources |
| `sources/luma.py` | Luma.com HTML scraper |
| `sources/eventbrite.py` | Eventbrite REST API client |
| `sources/mlh.py` | MLH REST API client |
| `mcp/__init__.py` | MCP server + tool definitions |
| `workflow/pipeline.py` | Full pipeline orchestration |
| `workflow/triggers.py` | Scheduling triggers |
| `web/app.py` | FastAPI application + routes |
| `security/vault.py` | Encrypted secret storage |
| `security/sanitize.py` | Input validation + rate limiting |
| `security/privacy.py` | User data privacy + retention |
| `models/__init__.py` | All Pydantic data models |
| `config/settings.py` | Environment-based configuration |

### 3.3 Dependency Rules

```
Web → Agent → Workflow → Sources → Models
                   ↓
              Security (used by any layer)
                   ↓
              Config + Utils (used by all layers)
```

- **Models** never import from Agent, Web, or Workflow.
- **Sources** never import from Web or MCP.
- **Agent** imports from Models, Sources, Security, Utils — never from Web.
- **Web** imports from Agent — never from Sources directly.

---

## 4. Hard Rules

### 4.1 Security

- **NEVER** hardcode API keys, passwords, or secrets in source code.
- **NEVER** commit `.env` files or `aurellia.db` to git.
- **ALWAYS** use `Vault` for secret storage in production.
- **ALWAYS** sanitize user input before processing (via `Sanitizer`).
- **ALWAYS** anonymize user identifiers before logging.
- **NEVER** include API keys in error messages or logs.
- Rate limiting must be enforced on all external API calls.

### 4.2 Data

- **ALWAYS** validate data at the boundary (source → model).
- **ALWAYS** use Pydantic models for data validation.
- **NEVER** mutate model instances in place — create new instances.
- User preference data must be encrypted at rest.
- Implement right to be forgotten (full data deletion on request).

### 4.3 Testing

- **ALWAYS** write tests for new features before merging.
- **ALWAYS** mock external HTTP calls (use `unittest.mock.patch`).
- **NEVER** make real API calls in tests.
- Tests must be deterministic — no flaky tests.
- Coverage target: ≥80% (`--cov-fail-under=80`).
- Test file naming: `test_<module>.py`.
- Use `pytest` markers: `@pytest.mark.slow`, `@pytest.mark.integration`.

### 4.4 Code Quality

- **ALWAYS** run `uv run ruff check src/ tests/` before committing.
- **ALWAYS** run `uv run mypy src/aurellia/` before committing.
- **ALWAYS** run `uv run pytest` before committing.
- Max function length: 50 lines (excluding docstrings).
- Max file length: 300 lines (excluding tests).
- Every public function must have a docstring.

### 4.5 Git

- **NEVER** commit to `main` directly.
- **ALWAYS** use feature branches: `feat/<description>`.
- Commit messages: `feat:`, `fix:`, `chore:`, `docs:`, `test:`, `refactor:`.
- Keep commits atomic — one logical change per commit.

### 4.6 Deployment

- **NEVER** expose the app without HTTPS in production.
- **ALWAYS** set `AURELLIA_MASTER_KEY` as a strong random string.
- **ALWAYS** set `DEBUG=false` in production.
- Use environment variables for all configuration in production.
- Database migrations must be backward-compatible.

---

## 5. Tool Versions

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
| SQLAlchemy | 2.0.x | ORM (future) |
| pytest | 8.x | Test framework |
| pytest-asyncio | 0.24.x | Async tests |
| pytest-cov | 5.x | Coverage |
| Ruff | 0.6.x | Linter/formatter |
| mypy | 1.13.x | Type checker |
| mkdocs-material | 9.x | Documentation |

---

## 6. MCP Server Exposed Tools

| Tool | Description | Auth Required |
|---|---|---|
| `list_events` | List upcoming NYC tech events with filters | No |
| `get_digest` | Get personalized morning digest | Yes (user_id) |
| `search_events` | Natural language event search | No |

---

## 7. Agentic Workflow

```
Trigger (cron/manual/MCP)
  │
  ▼
Scout Skill ─── Fetch from Luma, Eventbrite, MLH (parallel)
  │
  ▼
Curator Skill ── Deduplicate → Filter → Rank
  │
  ▼
Courier Skill ── Build Digest (markdown/HTML/email)
  │
  ▼
Learn Skill ─── Update topic weights from context
  │
  ▼
Deliver ── Web dashboard / Email / MCP response
```

---

*Last updated: 2026-06-25 | Version: 0.1.0*
