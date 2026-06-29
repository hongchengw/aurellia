# Aurellia BDD Scenarios
# Behavior Driven Development scenarios using Gherkin syntax.

---

## Feature: Repo Discovery

### Scenario: Successfully scrape repos from GitHub Trending
Given the GitHub Trending source is configured
When the scout skill fetches repos from GitHub Trending
Then repos are returned with titles, URLs, and metadata
And each repo has source set to "github"
And repos include stars, forks, and language
And invalid repos are skipped gracefully

### Scenario: Handle GitHub network failure
Given GitHub Trending is unreachable
When the scout skill attempts to fetch repos
Then a SourceError is raised with source="github"
And the pipeline completes with an empty list
And the error is logged

### Scenario: Fetch repos from Python trending
Given the GitHub Trending source is configured for Python
When the scout skill fetches repos
Then all repos have language set to "Python"
And repos are returned with titles, URLs, and metadata

### Scenario: Fetch repos from TypeScript trending
Given the GitHub Trending source is configured for TypeScript
When the scout skill fetches repos
Then all repos have language set to "TypeScript"
And repos are returned with titles, URLs, and metadata

### Scenario: Infer LLM category from title/description
Given a repo titled "awesome-llm" with description "A large language model framework"
When the source parses the repo
Then the repo_category is set to "llm"

### Scenario: Infer AI Agent category from title/description
Given a repo titled "crewAI" with description "Multi-agent orchestration framework"
When the source parses the repo
Then the repo_category is set to "ai_agent"

### Scenario: All sources fail gracefully
Given GitHub Trending is unreachable
When the pipeline runs
Then an empty repo list is processed
And the pipeline completes without crashing
And errors are logged

---

## Feature: Repo Deduplication

### Scenario: Remove duplicate repos by URL
Given two repos with the same URL from different feeds
When the curator skill deduplicates repos
Then only one repo is kept
And the repo has both sources listed

### Scenario: Remove duplicate repos by title
Given two repos with the same title from different feeds
When the curator skill deduplicates repos
Then only one repo is kept

### Scenario: Keep distinct repos separate
Given three repos with different titles and URLs
When the curator skill deduplicates repos
Then all three repos are kept

---

## Feature: Repo Filtering

### Scenario: Filter repos by language preference
Given a user with languages set to ["python", "typescript"]
When the curator filters repos
Then only Python and TypeScript repos are included
And repos in other languages are excluded

### Scenario: No language filter when user has no preference
Given a user with no language preferences
When the curator filters repos
Then all repos are included regardless of language

---

## Feature: Repo Ranking

### Scenario: Rank repos by interest matching
Given a user interested in "ai" and "llm"
When the curator ranks repos
Then repos tagged "ai" or "llm" have higher relevance scores
And repos with no matching tags have lower scores

### Scenario: Rank preferred language higher
Given a user with language preference "rust"
When the curator ranks repos
Then Rust repos are ranked higher than other languages

### Scenario: Rank trending repos higher
Given two repos with equal interest matching
When the curator ranks repos
Then the repo with more stars_today is ranked higher

### Scenario: All relevance scores are valid
Given any set of repos and user preferences
When the curator ranks repos
Then all relevance scores are between 0.0 and 1.0

---

## Feature: Digest Generation

### Scenario: Generate morning digest
Given a user with interests in "ai" and "llm"
And 10 curated repos
When the courier skill builds a digest
Then a digest is created with at most 10 entries
And entries are ranked by relevance
And each entry includes a personalized reason

### Scenario: Digest covers correct date range
Given a user requesting a 7-day digest
When the courier builds a digest
Then period_start is now
And period_end is approximately 7 days from now

### Scenario: Format digest as markdown
Given a valid digest with entries
When the courier formats the digest as markdown
Then a string is returned with repo titles, stars, and links

### Scenario: Format digest as HTML
Given a valid digest with entries
When the courier formats the digest as HTML
Then an HTML string is returned with repo cards

---

## Feature: Email Delivery

### Scenario: Send digest via email
Given a valid digest and SMTP is configured
When the courier sends an email
Then the email is sent successfully
And the result is true

### Scenario: Handle email failure
Given SMTP server is unreachable
When the courier attempts to send an email
Then the result is false
And an error is logged

---

## Feature: User Preferences

### Scenario: Update user preferences
Given a valid user_id and preference data
When the preferences endpoint is called
Then preferences are updated successfully

### Scenario: Reject invalid skill level
Given a request with skill_level="master"
When the preferences endpoint validates
Then a 422 error is returned

### Scenario: Reject invalid language
Given a request with languages=["klingon"]
When the preferences endpoint validates
Then a 422 error is returned

---

## Feature: Security

### Scenario: Store and retrieve secrets
Given a vault with a master key
When a secret is stored
Then it can be retrieved later
And the stored value is encrypted

### Scenario: Reject missing master key
Given no AURELLIA_MASTER_KEY is set
When a vault is initialized
Then an EnvironmentError is raised

### Scenario: Sanitize HTML injection
Given a string containing "<script>alert('xss')</script>Hello"
When the sanitizer processes it
Then script tags are removed
And "Hello" remains

### Scenario: Sanitize SQL injection
Given a string containing "'; DROP TABLE repos; --"
When the sanitizer processes it
Then SQL injection characters and keywords are removed

### Scenario: Validate programming language
Given a language name "Python"
When the sanitizer validates it
Then "python" is returned

### Scenario: Reject invalid language
Given a language name "NotALanguage"
When the sanitizer validates it
Then a ValueError is raised

### Scenario: Enforce rate limit
Given a rate limiter with max_requests=5
When 6 requests are made
Then the 6th request is blocked
And remaining() returns 0

### Scenario: Rate limit resets after window
Given a rate limiter with window_seconds=1
When the limit is exceeded and 1 second passes
Then requests are allowed again

### Scenario: Enforce data retention
Given user data older than retention period
When retention is enforced
Then old data is purged

### Scenario: Anonymize user data in logs
Given a log message containing user email
When privacy manager anonymizes it
Then the email is replaced with "[EMAIL]"

---

## Feature: Agent Memory

### Scenario: Remember user preferences across sessions
Given a user sets interests to ["ai", "llm"]
When a new agent session is created
Then the user's interests are recalled as ["ai", "llm"]

### Scenario: Track star history
Given a user stars a repo
When star history is queried
Then the repo appears in the history

### Scenario: Learn from feedback
Given a user gives positive feedback on "ai"
And negative feedback on "web3"
When topic weights are queried
Then "ai" has a higher weight than "web3"

### Scenario: Delete user account
Given a user requests account deletion
When delete_account is called
Then all user data is removed

---

## Feature: MCP Server

### Scenario: List available tools
Given the MCP server is running
When list_tools is called
Then 3 tools are returned: list_repos, get_digest, search_repos

### Scenario: Call list_repos tool
Given the MCP server is running
When list_repos is called with language="python"
Then a list of Python repos is returned

### Scenario: Call get_digest with authentication
Given a valid user_id
When get_digest is called
Then a personalized digest is returned

### Scenario: Reject get_digest without authentication
Given no user_id is provided
When get_digest is called
Then a ValueError is raised

### Scenario: Call search_repos tool
Given the MCP server is running
When search_repos is called with query="python LLM agent"
Then a list of matching repos is returned

---

## Feature: Web Dashboard

### Scenario: Health check returns 200
Given the server is running
When GET /health is called
Then status 200 is returned
And response contains {"status": "healthy"}

### Scenario: Repos endpoint returns 200
Given the server is running
When GET /repos is called
Then status 200 is returned
And response is a JSON array

### Scenario: Repos endpoint supports language filter
Given repos exist with various languages
When GET /repos?language=python is called
Then only Python repos are returned

### Scenario: Repos endpoint supports topic filter
Given repos exist with various topics
When GET /repos?topic=ai is called
Then only repos with "ai" tag are returned

### Scenario: Repos endpoint supports min_stars filter
Given repos exist with various star counts
When GET /repos?min_stars=1000 is called
Then only repos with at least 1000 stars are returned

### Scenario: Digest requires user_id
Given the server is running
When GET /digest is called without user_id
Then status 422 is returned

---

## Feature: Scheduled Execution

### Scenario: Daily trigger fires at correct time
Given a daily trigger configured for 08:00
When the current time is 08:00
Then should_fire returns true

### Scenario: Daily trigger does not fire at wrong time
Given a daily trigger configured for 08:00
When the current time is 14:00
Then should_fire returns false

### Scenario: Interval trigger respects elapsed time
Given an interval trigger of 60 minutes
When 60 minutes have passed since last fire
Then should_fire returns true

---

## Feature: End-to-End User Journey

### Scenario: New user onboarding
Given a new user with email "user@example.com"
When the agent greets the user
Then a welcome message is returned with Aurellia branding

### Scenario: User receives personalized digest
Given a user interested in "ai"
And AI repos exist in sources
When the pipeline runs
Then the digest contains AI-related repos ranked highest

### Scenario: User stars repo and agent learns
Given a user stars an AI repo
When the next digest is generated
Then AI repos have higher relevance scores

### Scenario: User deletes account
Given a user with data stored
When delete_account is called
Then all user data is removed
