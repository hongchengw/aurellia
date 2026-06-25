# Aurellia Technical Specification

**Version:** 0.1.0
**Status:** Draft
**Author:** hongc
**Last Updated:** 2026-06-25

---

## 1. Overview

### 1.1 Purpose

Aurellia is a personal concierge agent that discovers, curates, and delivers personalized tech event recommendations for New York City. It aggregates events from multiple sources (Luma, Eventbrite, MLH, Partiful), applies user preference learning, and delivers a daily morning digest.

### 1.2 Scope

**In Scope:**
- Web scraping of Luma.com for NYC tech events
- Eventbrite API integration for event discovery
- MLH API integration for fellowship/hackathon events
- Event deduplication and ranking based on user preferences
- Personalized morning digest delivery (web dashboard + email)
- MCP server exposing Aurellia as a tool for other agents
- Web dashboard with timeline view, filters, and search
- Feedback loop that learns from user RSVPs

**Out of Scope (v1):**
- Partiful integration (deferred — no public API)
- Mobile application
- Multi-city support (NYC only for v1)
- Real-time event updates (polling-based)
- Payment processing or ticket purchasing

### 1.3 Key Design Decisions

| Decision | Rationale |
|---|---|
| SQLite over PostgreSQL | Zero-cost, zero-config, single-user. Appropriate for concierge agent. |
| Pydantic models (not ORM) | Simpler, validates at boundaries. No database migration complexity. |
| Single-agent over multi-agent | Multi-agent adds complexity without clear benefit for 4 fast sources. |
| In-memory caching | Avoids redundant scraping. Cache TTL configurable via settings. |
| Structured logging (JSON) | Enables log aggregation and debugging in production. |
| MCP as tool exposure layer | Standard protocol. Any MCP-compatible agent can consume Aurellia. |

---

## 2. System Architecture

### 2.1 High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                          EXTERNAL SOURCES                            │
│  Luma.com (HTML)  │  Eventbrite API  │  MLH API  │  Partiful (HTML) │
└────────┬─────────┴────────┬─────────┴───────────┴──────────┬──────────┘
         │                  │                               │
         ▼                  ▼                               ▼
┌──────────────────────────────────────────────────────────────────────┐
│                        SOURCE LAYER                                  │
│  LumaSource  │  EventbriteSource  │  MLHSource  │  (PartifulSource) │
│  [scrape]    │  [REST API]        │  [REST API] │  [scrape]         │
└────────────────────────┬─────────────────────────────────────────────┘
                         │ List[Event]
                         ▼
┌──────────────────────────────────────────────────────────────────────┐
│                        AGENT LAYER                                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐            │
│  │  Scout   │→ │  Curate  │→ │  Courier │→ │  Learn   │            │
│  │  Skill   │  │  Skill   │  │  Skill   │  │  Skill   │            │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘            │
│                         ↑                                            │
│                    AgentMemory                                       │
│              (prefs, RSVP history, weights)                           │
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
│   For each source in [Luma, Eventbrite, MLH]:          │
│     - Fetch events (respecting rate limits)            │
│     - Normalize to Event model                          │
│     - Handle failures gracefully (continue on error)    │
│   Output: List[Event] (~50-200 events)                  │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│ Step 2: CURATE                                          │
│   2a. Deduplicate (by URL, then by title similarity)    │
│   2b. Filter (date range, price, borough)               │
│   2c. Rank (interest match, location, skill level)      │
│   Output: List[Event] sorted by relevance_score         │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│ Step 3: COURIER                                         │
│   - Take top N events (configurable, default 10)       │
│   - Generate personalized reason for each              │
│   - Build Digest model                                  │
│   - Format as markdown / HTML                           │
│   Output: Digest                                        │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│ Step 4: LEARN                                           │
│   - Analyze RSVP history for patterns                   │
│   - Update topic weights                                │
│   - Adjust future recommendations                       │
│   Output: Updated AgentMemory                           │
└─────────────────────────────────────────────────────────┘
```

### 2.3 Data Flow

```
Sources → Event[] → Pipeline → Digest → Delivery
                                      ↓
                                  Web UI / Email / MCP

User Actions → RSVP / Feedback → AgentMemory → Topic Weights → Future Ranking
```

---

## 3. Component Specifications

### 3.1 Source Layer

#### LumaSource

- **Type:** HTML scraping
- **Library:** `httpx` + `BeautifulSoup4`
- **Rate Limit:** 30 req/min
- **Selectors:** Configurable via `sources.yaml`
- **Error Handling:** `SourceError` on network failure; skip individual card failures
- **Output:** `List[Event]` with `source=EventSource.LUMA`

#### EventbriteSource

- **Type:** REST API (OAuth Bearer token)
- **Library:** `httpx`
- **Rate Limit:** 50 req/min
- **Auth:** `EVENTBRITE_API_KEY` env var
- **Pagination:** Server-side (page_size param)
- **Error Handling:** `SourceError` on missing key or HTTP errors
- **Output:** `List[Event]` with `source=EventSource.EVENTBRITE`

#### MLHSource

- **Type:** REST API (no auth)
- **Library:** `httpx`
- **Rate Limit:** 60 req/min
- **Filtering:** Client-side date filter (upcoming only)
- **Error Handling:** `SourceError` on HTTP errors
- **Output:** `List[Event]` with `source=EventSource.MLH`

### 3.2 Agent Layer

#### AurelliaAgent

- **Responsibility:** Orchestrate skills, manage pipeline
- **State:** Holds `AgentMemory` and `User`
- **Skills:** ScoutSkill, CuratorSkill, CourierSkill, LearnSkill
- **Entry Point:** `run_pipeline(period_days) -> Digest`

#### AgentMemory

- **Persistence:** JSON file at `data/agent_state.json`
- **State:** Topic weights, RSVP history, user preferences
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
- **SQL:** Remove quotes, dashes, semicolons; detect keywords
- **URL:** Enforce HTTPS, reject `data:` and `file:` schemes
- **City:** Validate against known NYC boroughs

#### RateLimiter

- **Algorithm:** Sliding window (list of timestamps)
- **Scope:** Per-instance (one per source)
- **Configurable:** `max_requests` and `window_seconds`

### 3.4 MCP Server

- **Protocol:** MCP (Model Context Protocol) via stdio transport
- **Tools Exposed:** 3 (list_events, get_digest, search_events)
- **Auth:** `get_digest` requires `user_id`; others are public
- **Discovery:** `list_tools()` returns tool schemas

### 3.5 Web Application

- **Framework:** FastAPI with Jinja2 templates
- **Endpoints:** 7 (health, list events, get event, digest, update prefs, get prefs, dashboard)
- **UI:** Dark-themed single-page dashboard with timeline layout
- **Static Files:** Minimal CSS (inline in HTML for simplicity)
- **Auth:** None for v1 (single-user concierge)

---

## 4. Data Models

### 4.1 Event

| Field | Type | Required | Validation |
|---|---|---|---|
| `title` | `str` | Yes | Non-empty |
| `description` | `str` | No | Default `""` |
| `url` | `str` | Yes | Must start with `https://` |
| `source` | `EventSource` | Yes | Enum value |
| `event_type` | `EventType` | No | Default `OTHER` |
| `location` | `str` | No | Default `"NYC"` |
| `starts_at` | `datetime` | Yes | Must be future |
| `ends_at` | `Optional[datetime]` | No | — |
| `is_free` | `bool` | No | Default `True` |
| `price` | `Optional[float]` | No | ≥0; null if free |
| `currency` | `str` | No | Default `"USD"` |
| `tags` | `List[str]` | No | Default `[]` |
| `skill_level` | `SkillLevel` | No | Default `ANY` |
| `relevance_score` | `float` | No | 0.0–1.0 |
| `sources` | `List[EventSource]` | No | For merged events |

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
| `boroughs` | `List[str]` | No | NYC borough names only |
| `max_price` | `float` | No | ≥0 |
| `skill_level` | `SkillLevel` | No | Default `BEGINNER` |
| `preferred_days` | `List[str]` | No | Default `[]` |

### 4.4 Digest

| Field | Type | Required | Validation |
|---|---|---|---|
| `user` | `User` | Yes | — |
| `generated_at` | `datetime` | Yes | — |
| `period_start` | `datetime` | Yes | — |
| `period_end` | `datetime` | Yes | — |
| `entries` | `List[DigestEntry]` | Yes | Non-empty |

### 4.5 DigestEntry

| Field | Type | Required | Validation |
|---|---|---|---|
| `rank` | `int` | Yes | — |
| `event` | `Event` | Yes | — |
| `relevance_score` | `float` | Yes | 0.0–1.0 |
| `reason` | `str` | No | Default `""` |

---

## 5. API Endpoints

### 5.1 GET /events

List upcoming tech events with optional filters.

**Query Parameters:**
| Param | Type | Default | Description |
|---|---|---|---|
| `topic` | `string` | — | Filter by tag |
| `borough` | `string` | — | Filter by borough |
| `days_ahead` | `int` | 7 | Days to look ahead (1-90) |
| `free_only` | `bool` | false | Only free events |

**Response:** `200 OK` — `List[EventDTO]`

### 5.2 GET /events/{event_id}

**Response:** `501 Not Implemented` (v1)

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

**Response:** `200 OK` — `{status: "healthy", version: "0.1.0"}`

---

## 6. Configuration

| Env Var | Required | Default | Description |
|---|---|---|---|
| `AURELLIA_MASTER_KEY` | Yes | — | Master encryption key (32+ chars) |
| `EVENTBRITE_API_KEY` | No | — | Eventbrite OAuth token |
| `OPENROUTER_API_KEY` | No | — | LLM API key (future) |
| `DATABASE_URL` | No | `sqlite:///data/aurellia.db` | Database URL |
| `SCRAPE_INTERVAL` | No | 60 | Minutes between scrapes |
| `MAX_EVENTS` | No | 100 | Max events per source |
| `DIGEST_PERIOD_DAYS` | No | 7 | Default digest period |
| `MAX_DIGEST_ENTRIES` | No | 10 | Events in digest |
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
| **Throughput** | 100 events/minute scraping capacity |
| **Data Privacy** | User data encrypted at rest, anonymized in logs |
| **Retention** | 90-day data retention default |
| **Test Coverage** | ≥80% |
| **Security** | No secrets in code, HTTPS enforced, rate limited |

---

## 8. Risks & Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Luma changes HTML structure | High | Configurable selectors in sources.yaml |
| Eventbrite API rate limits | Medium | Respect 50 req/min, cache results |
| User data breach | High | Encryption at rest, anonymized logs |
| Source downtime | Medium | Graceful degradation (continue on failure) |
| Cold start (no user history) | Low | Default interests, learn from behavior |

---

## 9. Milestones

| Milestone | Status | Target |
|---|---|---|
| Scaffold & architecture | ✅ Complete | — |
| Source layer (Luma, Eventbrite, MLH) | ✅ Complete | — |
| Agent pipeline (scout→curate→courier→learn) | ✅ Complete | — |
| Web dashboard | ✅ Complete | — |
| MCP server | ✅ Complete | — |
| Security layer | ✅ Complete | — |
| Integration testing | 🔄 In Progress | — |
| Deployment | ⏳ Pending | — |
| Documentation (writeup) | ⏳ Pending | — |

---

*This document is a living specification. Update it as design decisions change.*
