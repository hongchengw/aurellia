# Changelog

All notable changes to the Aurellia project will be documented in this file.

The format is based on [Keep a Changelog](https://achangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [Unreleased]

### Added
- v0.3.0 scaffold: empty package structure for feature-by-feature development
- 30 source module stubs across 10 subpackages
- Test infrastructure configured (pytest, coverage, markers)
- `pyproject.toml` with all tool configs (ruff, mypy, pytest, coverage)

### Removed
- All pre-scaffold implementation code (was local-only, never committed)
- Stale dependencies: sqlalchemy, pyyaml, mkdocs-material, pre-commit
- NYC-specific utilities (geocode, borough references)

---

## [0.3.0] - 2026-06-29

### Changed
- **Major pivot**: Project refactored from NYC tech event tracker to GitHub Trending repo scout
- Data models rewritten: `Event` → `Repo`, `EventType` → `RepoCategory`, `EventSource` → `RepoSource`
- Replaced Luma/Eventbrite/MLH event sources with `GithubTrendingSource`
- Updated Scout, Curate, Courier, Learn skills for repo domain
- MCP tools renamed: `list_events` → `list_repos`, `search_events` → `search_repos`
- API endpoints: `/events` → `/repos`

### Removed
- Luma, Eventbrite, and MLH source implementations
- NYC borough validation and geocoding
- Event-specific concepts: RSVPs, ticket prices, venue locations

---

## [0.1.0] - 2026-06-25

### Added
- Initial project scaffold and architecture
- Core models and agent pipeline
- Web dashboard and MCP server
- Security layer with encryption and rate limiting
- Test suite with 95 tests

---

*This changelog is maintained alongside AGENTS.md. Update with every PR merge.*
