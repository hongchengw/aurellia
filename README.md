# Aurellia

Your personal GitHub Trending repo scout. Aurellia discovers trending repositories from GitHub, filters and ranks them against your interests, and delivers a personalized morning digest — via web dashboard, email, or any MCP-compatible agent.

> **Project:** Aurellia — Kaggle x Google AI Agents Capstone (Concierge Track)

## Status

🔹 **v0.3.0 — scaffold ready for feature development.**

The project is structured as an empty package skeleton. All 30 source modules exist as stubs (empty files). No features are implemented yet. This is intentional — each feature will be built, tested, and reviewed individually before merging.

## Roadmap

Features will be implemented in dependency order:

| # | Feature | Description |
|---|---|---|
| 1 | **Models** | Pydantic data shapes: Repo, User, Digest, enums |
| 2 | **Sources** | GitHub Trending HTML scraper |
| 3 | **Scout Skill** | Discover trending repos from GitHub |
| 4 | **Curate Skill** | Deduplicate, filter by language, rank by relevance |
| 5 | **Courier Skill** | Build personalized digest (markdown/HTML) |
| 6 | **Learn Skill** | Feedback loop from stars/skips |
| 7 | **Agent Core** | Orchestrator + AgentMemory |
| 8 | **Security** | Vault (encryption), Sanitizer, PrivacyManager |
| 9 | **Web Dashboard** | FastAPI routes + UI |
| 10 | **MCP Server** | list_repos, get_digest, search_repos |
| 11 | **Workflow** | Pipeline, triggers, caching |
| 12 | **CLI** | serve, run, digest, seed, mcp commands |

## Package Structure

```
src/aurellia/
├── __init__.py
├── __main__.py
├── models/          ← Repo, User, Digest models
├── sources/         ← GitHub Trending scraper
├── agent/
│   ├── core.py      ← AurelliaAgent orchestrator
│   ├── memory.py    ← Persistent state (stars, weights)
│   └── skills/      ← scout, curate, courier, learn
├── workflow/         ← Pipeline + Triggers
├── mcp/              ← MCP server + tool definitions
├── web/              ← FastAPI app + routes
├── security/         ← vault, sanitize, privacy
├── config/           ← Settings from env vars
└── utils/            ← logger, nlp
```

## Configuration

All configuration via environment variables. Required: `AURELLIA_MASTER_KEY` (32+ chars). Optional: `GITHUB_TOKEN`, `DATABASE_URL`, `SCRAPE_INTERVAL`, `MAX_REPOS`, `DIGEST_PERIOD_DAYS`, `MAX_DIGEST_ENTRIES`, `SMTP_*`, `DEBUG`.

## Testing

Test infrastructure is configured but empty. Tests will be written alongside each feature.

```bash
uv sync --extra dev
uv run pytest
```

## Architecture (Target)

Four layers, strict dependency order:

```
Web (FastAPI)
     ↓
Agent (AurelliaAgent → Scout / Curate / Courier / Learn)
     ↓
Workflow (Pipeline + Triggers + Cache)
     ↓
Sources (GitHub Trending)  +  Models (Pydantic)
     ↓
Security · Config · Utils
```

## CLI (Target)

| Command | Purpose |
|---|---|
| `aurellia serve` | Start the FastAPI dashboard |
| `aurellia run --period N` | Run the pipeline |
| `aurellia digest` | Print current digest |
| `aurellia seed` | Insert sample repos |
| `aurellia mcp` | Start the MCP server |

## License

MIT
