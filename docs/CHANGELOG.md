# Changelog

All notable changes to the Aurellia project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [Unreleased]

### Added
- Project scaffold with full directory structure
- AGENTS.md with naming conventions, coding style, and hard rules
- Technical specification (specs/SPECS.md)
- Database schema (specs/database-schema.yaml)
- API contracts (specs/api-contracts.yaml)
- BDD scenarios (specs/bdd-scenarios.md) — 30+ Gherkin scenarios
- Data models: Event, User, UserPreference, Digest, DigestEntry (Pydantic v2)
- Config module: environment-based settings with Settings.from_env()
- Security layer: Vault (Fernet encryption), Sanitizer (HTML/SQL injection prevention), RateLimiter (sliding window), PrivacyManager (data retention, anonymization)
- Source layer: LumaSource (HTML scraper), EventbriteSource (REST API), MLHSource (REST API)
- Agent layer: AurelliaAgent, AgentMemory (persistent state), 4 skills (Scout, Curate, Courier, Learn)
- MCP Server: 3 tools (list_events, get_digest, search_events)
- Workflow: Pipeline (scout→curate→courier→learn), Triggers (Daily, Interval, Manual)
- Web app: FastAPI with 7 endpoints, dark-themed dashboard with timeline view
- CLI entry point: `aurellia serve`, `aurellia run`, `aurellia digest`, `aurellia seed`, `aurellia mcp`
- Seed script with 8 sample NYC tech events
- GitHub Actions: CI test workflow + scheduled scraping workflow
- Claude Code skill: skills/scene-scout/SKILL.md
- 95 tests across unit, integration, and e2e layers
- Reproduction script (reproduce.py) for scaffold verification

### Security
- Vault: Fernet encryption (AES-128-CBC) with PBKDF2 key derivation
- Sanitizer: HTML tag stripping, SQL injection detection, URL validation
- RateLimiter: Sliding window algorithm, per-source instances
- PrivacyManager: Data anonymization, retention enforcement, right to deletion

---

## [0.1.0] - 2026-06-25

### Added
- Initial project scaffold and architecture
- Core models and agent pipeline
- Web dashboard and MCP server
- Security layer with encryption and rate limiting
- Test suite with 95 tests

---

## [0.2.0] - TBD

### Planned
- Partiful source integration
- Email digest delivery (SMTP)
- User authentication
- Deployment to Railway/Render
- Multi-city support
- Real-time event updates via webhook

---

*This changelog is maintained alongside AGENTS.md. Update with every PR merge.*
