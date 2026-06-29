# Aurellia Technical Specification

**Version:** 0.3.0
**Status:** Draft (target architecture — not yet implemented)
**Author:** hongc
**Last Updated:** 2026-06-29

> **Note:** This spec describes the *target* architecture for v0.3.0. The current codebase is a scaffold with empty modules. Each section of this spec will be implemented incrementally and reviewed before merging.

---

## 1. Overview

### 1.1 Purpose

Aurellia is a personal concierge agent that discovers, curates, and delivers personalized trending GitHub repository recommendations. It scrapes GitHub Trending (daily/weekly), applies user preference learning, and delivers a daily morning digest of the most relevant repos.

### 1.2 Scope

**In Scope:**
- HTML scraping of GitHub Trending for trending repositories
- Repo deduplication and ranking based on user preferences (interests, languages, stars)
- Personalized morning digest delivery (web dashboard + email)
- MCP server exposing Aurellia as a tool for other agents
- Web dashboard with repo cards, filters, and search
- Feedback loop that learns from user stars/skips
- Automatic category inference (LLM, AI Agent, Framework, Tool, Data, Web, Security, DevTools, Game)

**Out of Scope (v3):**
- GitHub API integration (REST/GraphQL — using HTML scraping for simplicity)
- Multi-platform support (GitHub only for v3)
- Real-time webhook updates (polling-based)
- Repository contribution or code analysis

### 1.3 Key Design Decisions

| Decision | Rationale |
|---|---|
| SQLite over PostgreSQL | Zero-cost, zero-config, single-user. Appropriate for concierge agent. |
| Pydantic models (not ORM) | Simpler, validates at boundaries. No database migration complexity. |
| Single-agent over multi-agent | Multi-agent adds complexity without clear benefit for a single fast source. |
| In-memory caching | Avoids redundant scraping. Cache TTL configurable via settings. |
| Structured logging (JSON) | Enables log aggregation and debugging in production. |
| MCP as tool exposure layer | Standard protocol. Any MCP-compatible agent can consume Aurellia. |
| GitHub Trending over event sites | Event sites (Luma, Eventbrite, MLH) have aggressive bot protection. GitHub encourages scraping and provides richer data (stars, forks, languages, topics). |

---

## 2. System Architecture

### 2.1 High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                          EXTERNAL SOURCE                             │
│                    GitHub Trending (HTML pages)                      │
└────────────────────────┬─────────────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────────────────────┐
│                        SOURCE LAYER                                  │
│  GithubTrendingSource (all languages, Python, TypeScript)            │
│  [HTML scraping via BeautifulSoup4]                                 │
└────────────────────────┬─────────────────────────────────────────────┘
                         │ List[Repo]
                         ▼
┌──────────────────────────────────────────────────────────────────────┐
│                        AGENT LAYER                                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐            │
│  │  Scout   │→ │  Curate  │→ │  Courier │→ │  Learn   │            │
│  │  Skill   │  │  Skill   │  │  Skill   │  │  Skill   │            │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘            │
│                         ↑                                            │
│                    AgentMemory                                       │
│              (prefs, star history, weights)                          │
└────────────────────────┬─────────────────────────────────────────────┘
                         │ Digest
                         ▼
┌──────────────────────────────────────────────────────────────────────┐
│                     DELIVERY LAYER                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │  Web Dashboard│  │  Email SMTP  │  │  MCP Server  │              │
│  │  (FastAPI)   │  │              │  │  (stdio/SSE) │              │
│  └──────────────┘  └──────────────┘  └──────────────┘              │
└──────────────────────────────────────────────────────────────────────┘
```

### 2.2 Agentic Workflow Pipeline

```
Trigger (cron / manual / MCP call)
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│ Step 1: SCOUT                                           │
│   For each feed in [all, python, typescript]:           │
│     - Fetch trending repos from GitHub Trending        │
│     - Normalize to Repo model                           │
│     - Infer category from title/description            │
│     - Handle failures gracefully (continue on error)    │
│   Output: List[Repo] (~25-75 repos)                     │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│ Step 2: CURATE                                          │
│   2a. Deduplicate (by URL, then by title similarity)    │
│   2b. Filter (language preference)                      │
│   2c. Rank (interest match, language, stars, trending)  │
│   Output: List[Repo] sorted by relevance_score         │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│ Step 3: COURIER                                         │
│   - Take top N repos (configurable, default 10)        │
│   - Generate personalized reason for each              │
│   - Build Digest model                                  │
│   - Format as markdown / HTML                           │
│   Output: Digest                                        │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│ Step 4: LEARN                                           │
│   - Analyze star history for patterns                   │
│   - Update topic weights                                │
│   - Adjust future recommendations                       │
│   Output: Updated AgentMemory                           │
└─────────────────────────────────────────────────────────┘
```

### 2.3 Data Flow

```
GitHub Trending → Repo[] → Pipeline → Digest → Delivery
                                            │
                                        Web UI / Email / MCP

User Actions → Star / Skip / Feedback → AgentMemory → Topic Weights → Future Ranking
```

---

## 3. Component Specifications

### 3.1 Source Layer

#### GithubTrendingSource

- **Type:** HTML scraping
- **Library:** `httpx` + `BeautifulSoup4`
- **Rate Limit:** 30 req/min
- **Feeds:** All languages, Python-specific, TypeScript-specific
- **URL Pattern:** `https://github.com/trending[/{language}][?since={daily|weekly|monthly}]`
- **Error Handling:** `SourceError` on network failure; skip individual repo failures
- **Category Inference:** LLM, AI Agent, Framework, Tool, Data, Web, Security, DevTools, Game
- **Output:** `List[Repo]` with `source=RepoSource.GITHUB`

### 3.2 Agent Layer

#### AurelliaAgent

- **Responsibility:** Orchestrate skills, manage pipeline
- **State:** Holds `AgentMemory` and `User`
- **Skills:** ScoutSkill, CuratorSkill, CourierSkill, LearnSkill
- **Entry Point:** `run_pipeline(period_days) -> Digest`

#### AgentMemory

- **Persistence:** JSON file at `data/agent_state.json`
- **State:** Topic weights, star history, user preferences
- **Thread Safety:** Single-user, no concurrent access assumed
- **Serialization:** JSON with datetime as ISO strings

### 3.3 Security Layer

#### Vault

- **Encryption:** Fernet (AES-128-CBC) via `cryptography` library
- **Key Derivation:** PBKDF2 with SHA-256, 100k iterations
- **Storage:** In-memory dict, optional file persistence
- **Master Key:** `AURELLIA_MASTER_KEY` env var (required)

#### Sanitizer

- **HTML:** Strip all tags, remove `javascript:` and event handlers
- **SQL:** Remove quotes, dashes, semicolons; detect and strip SQL keywords
- **URL:** Enforce HTTPS, reject `data:` and `file:` schemes
- **Language:** Validate against known programming languages

#### RateLimiter

- **Algorithm:** Sliding window (list of timestamps)
- **Scope:** Per-instance (one per source)
- **Configurable:** `max_requests` and `window_seconds`

### 3.4 MCP Server

- **Protocol:** MCP (Model Context Protocol) via stdio transport
- **Tools Exposed:** 3 (list_repos, get_digest, search_repos)
- **Auth:** `get_digest` requires `user_id`; others are public
- **Discovery:** `list_tools()` returns tool schemas

### 3.5 Web Application

- **Framework:** FastAPI with Jinja2 templates
- **Endpoints:** `/health`, `/repos`, `/repos/{id}`, `/digest`, `POST /preferences`, `GET /preferences`, `/` dashboard
- **UI:** Dark-themed single-page dashboard with repo cards
- **Static Files:** Minimal CSS (inline in HTML for simplicity)
- **Auth:** None for v3 (single-user concierge)

---

## 4. Data Models

### 4.1 Repo

| Field | Type | Required | Validation |
|---|---|---|---|
| `title` | `str` | Yes | Non-empty |
| `description` | `str` | No | Default `""` |
| `url` | `str` | Yes | Must start with `https://` |
| `source` | `RepoSource` | Yes | Enum value (`github`) |
| `repo_category` | `RepoCategory` | No | Default `OTHER` |
| `language` | `str` | No | Default `""` |
| `stars` | `int` | No | Default `0`; ≥0 |
| `forks` | `int` | No | Default `0` |
| `stars_today` | `int` | No | Default `0` |
| `tags` | `List[str]` | No | Default `[]` |
| `skill_level` | `SkillLevel` | No | Default `ANY` |
| `relevance_score` | `float` | No | 0.0–1.0 |
| `sources` | `List[RepoSource]` | No | For merged repos |
| `created_at` | `datetime` | No | Auto-set |
| `updated_at` | `datetime` | No | Auto-set |
| `pushed_at` | `Optional[datetime]` | No | — |
| `readme_snippet` | `str` | No | Default `""` |

### 4.2 User

| Field | Type | Required | Validation |
|---|---|---|---|
| `id` | `str` | Yes | — |
| `email` | `str` | Yes | Valid email format |
| `name` | `str` | No | Default `""` |
| `preferences` | `UserPreference` | No | — |

### 4.3 UserPreference

| Field | Type | Required | Validation |
|---|---|---|---|
| `interests` | `List[str]` | No | Default `[]` |
| `languages` | `List[str]` | No | Lowercased |
| `skill_level` | `SkillLevel` | No | Default `BEGINNER` |
| `sources` | `List[RepoSource]` | No | Default `[]` |

### 4.4 Digest

| Field | Type | Required | Validation |
|---|---|---|---|
| `user` | `User` | Yes | — |
| `generated_at` | `datetime` | Yes | — |
| `period_start` | `datetime` | Yes | — |
| `period_end` | `datetime` | Yes | — |
| `entries` | `List[DigestEntry]` | No | Default `[]` |

### 4.5 DigestEntry

| Field | Type | Required | Validation |
|---|---|---|---|
| `rank` | `int` | Yes | — |
| `repo` | `Repo` | Yes | — |
| `relevance_score` | `float` | Yes | 0.0–1.0 |
| `reason` | `str` | No | Default `""` |

---

## 5. API Endpoints

### 5.1 GET /repos

List trending repos with optional filters.

**Query Parameters:**
| Param | Type | Default | Description |
|---|---|---|---|
| `language` | `string` | — | Filter by programming language |
| `topic` | `string` | — | Filter by topic tag |
| `min_stars` | `int` | 0 | Minimum star count |

**Response:** `200 OK` — `List[RepoDTO]`

### 5.2 GET /repos/{repo_id}

**Response:** `501 Not Implemented` (v3)

### 5.3 GET /digest

**Query Parameters:**
| Param | Type | Required | Description |
|---|---|---|---|
| `user_id` | `string` | Yes | User identifier |
| `period` | `string` | "week" | "today", "week", "month" |

**Response:** `200 OK` — `DigestDTO`

### 5.4 POST /preferences

**Body:** `UserPreferenceDTO`
**Response:** `200 OK` — `{status: "ok"}`

### 5.5 GET /health

**Response:** `200 OK` — `{status: "healthy", version: "0.3.0"}`

---

## 6. Configuration

| Env Var | Required | Default | Description |
|---|---|---|---|
| `AURELLIA_MASTER_KEY` | Yes | — | Master encryption key (32+ chars) |
| `GITHUB_TOKEN` | No | — | GitHub token (higher rate limits) |
| `OPENROUTER_API_KEY` | No | — | LLM API key (future) |
| `DATABASE_URL` | No | `sqlite:///data/aurellia.db` | Database URL |
| `SCRAPE_INTERVAL` | No | 60 | Minutes between scrapes |
| `MAX_REPOS` | No | 100 | Max repos per source |
| `DIGEST_PERIOD_DAYS` | No | 7 | Default digest period |
| `MAX_DIGEST_ENTRIES` | No | 10 | Repos in digest |
| `SMTP_HOST` | No | — | Email delivery |
| `SMTP_PORT` | No | 587 | SMTP port |
| `SMTP_USER` | No | — | SMTP username |
| `SMTP_PASSWORD` | No | — | SMTP password |
| `DEBUG` | No | false | Debug mode |

---

## 7. Non-Functional Requirements

| Requirement | Target |
|---|---|
| **Availability** | 99% (single-instance, free tier) |
| **Latency** | <5s for pipeline run (cached: <500ms) |
| **Throughput** | 75 repos/minute scraping capacity |
| **Data Privacy** | User data encrypted at rest, anonymized in logs |
| **Retention** | 90-day data retention default |
| **Test Coverage** | ≥80% |
| **Security** | No secrets in code, HTTPS enforced, rate limited |

---

## 8. Risks & Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| GitHub changes HTML structure | High | Graceful parsing (skip failures), monitor for breakage |
| GitHub rate limits | Medium | Respect 30 req/min, use GITHUB_TOKEN for higher limits |
| User data breach | High | Encryption at rest, anonymized logs |
| Source downtime | Low | Graceful degradation (GitHub is highly available) |
| Cold start (no user history) | Low | Default interests, learn from behavior |

---

## 9. Milestones

| Milestone | Status | Target |
|---|---|---|
| Scaffold & architecture | ✅ Complete | — |
| Source layer (GitHub Trending) | ✅ Complete | — |
| Agent pipeline (scout→curate→courier→learn) | ✅ Complete | — |
| Web dashboard | ✅ Complete | — |
| MCP server | ✅ Complete | — |
| Security layer | ✅ Complete | — |
| Integration testing | ✅ Complete | — |
| Deployment | ⏳ Pending | — |
| Documentation (writeup) | ⏳ Pending | — |

---

*This document is a living specification. Update it as design decisions change.*
