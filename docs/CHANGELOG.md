# Changelog

All notable changes to the Aurellia project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [Unreleased]

---

## [0.3.0] - 2026-06-29

### Changed
- **Major pivot**: Project refactored from NYC tech event tracker to GitHub Trending repo scout
- Data models rewritten: `Event` → `Repo`, `EventType` → `RepoCategory`, `EventSource` → `RepoSource`
- Replaced Luma/Eventbrite/MLH event sources with `GithubTrendingSource` (scrapes GitHub Trending)
- Updated Scout to fetch from 3 GitHub Trending feeds (all languages, Python, TypeScript)
- Curator ranking now uses language preference, stars, and stars-today (trending signal)
- Courier now formats repo digest with stars, forks, language, and trending indicator
- Learn skill updated: `on_rsvp` → `on_star`, `on_skip` preserved for negative signals
- MCP tools renamed: `list_events` → `list_repos`, `search_events` → `search_repos`
- API endpoints: `/events` → `/repos`, borough filter → language filter
- Agent memory: `rsvp_history` → `star_history`, `boroughs` → `languages`

### Removed
- Luma, Eventbrite, and MLH source implementations (event sites with bot protection)
- NYC borough validation and geocoding (no longer needed)
- Event-specific concepts: RSVPs, ticket prices, venue locations, event types (hackathon, meetup, etc.)

### Added
- `GithubTrendingSource` with HTML scraping via BeautifulSoup4
- Category inference: LLM, AI Agent, Framework, Tool, Data, Web, Security, DevTools, Game
- GitHub star-based relevance scoring (logarithmic stars + stars-today boost)
- Programming language filtering in API, curator, and user preferences
- `CLAUDE.md` for AI coding assistant guidance

### Security
- Sanitizer: replaced borough validation with programming language validation

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

## [0.1.0] - 2026-06-25

### Added
- Initial project scaffold and architecture
- Core models and agent pipeline
- Web dashboard and MCP server
- Security layer with encryption and rate limiting
- Test suite with 95 tests

---

*This changelog is maintained alongside AGENTS.md. Update with every PR merge.*
