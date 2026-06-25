# Aurellia BDD Scenarios
# Behavior Driven Development scenarios using Gherkin syntax.

---

## Feature: Event Discovery

### Scenario: Successfully scrape events from Luma
Given the Luma source is configured with valid selectors
When the scout skill fetches events from Luma
Then events are returned with titles, URLs, and dates
And each event has source set to "luma"
And invalid events are skipped gracefully

### Scenario: Handle Luma network failure
Given the Luma source is unreachable
When the scout skill attempts to fetch events
Then a SourceError is raised with source="luma"
And the pipeline continues with other sources

### Scenario: Fetch events from Eventbrite API
Given a valid Eventbrite API key is configured
When the scout skill fetches events from Eventbrite
Then events are returned with normalized fields
And each event has source set to "eventbrite"
And prices are converted from cents to dollars

### Scenario: Handle missing Eventbrite API key
Given no Eventbrite API key is configured
When the scout skill attempts to fetch from Eventbrite
Then a SourceError is raised with message "Eventbrite API key is required"

### Scenario: Fetch events from MLH API
Given the MLH API is accessible
When the scout skill fetches events from MLH
Then only future events are returned
And each event has source set to "mlh"

### Scenario: All sources fail gracefully
Given all event sources are unreachable
When the pipeline runs
Then an empty event list is processed
And the pipeline completes without crashing
And errors are logged for each failed source

---

## Feature: Event Deduplication

### Scenario: Remove duplicate events by URL
Given two events with the same URL but different sources
When the curator skill deduplicates events
Then only one event is kept
And the event has both sources listed

### Scenario: Remove duplicate events by title
Given two events with the same title from different sources
When the curator skill deduplicates events
Then only one event is kept

### Scenario: Keep distinct events separate
Given three events with different titles and URLs
When the curator skill deduplicates events
Then all three events are kept

---

## Feature: Event Filtering

### Scenario: Filter events by maximum price
Given a user with max_price set to 20.0
When the curator filters events
Then events with price > 20.0 are excluded
And free events are included

### Scenario: Filter events by date range
Given a user requesting events within 7 days
When the curator filters events
Then events more than 7 days away are excluded
And past events are excluded

### Scenario: Filter events by borough
Given a user preferring Brooklyn
When the curator filters events
Then events in Brooklyn are prioritized

---

## Feature: Event Ranking

### Scenario: Rank events by interest matching
Given a user interested in "ai" and "hackathon"
When the curator ranks events
Then events tagged "ai" or "hackathon" have higher relevance scores
And events with no matching tags have lower scores

### Scenario: Rank free events higher for budget users
Given a user with max_price of 0
When the curator ranks events
Then free events are ranked higher than paid events

### Scenario: All relevance scores are valid
Given any set of events and user preferences
When the curator ranks events
Then all relevance scores are between 0.0 and 1.0

---

## Feature: Digest Generation

### Scenario: Generate morning digest
Given a user with interests in "ai" and "hackathon"
And 10 curated events
When the courier skill builds a digest
Then a digest is created with at most 10 entries
And entries are ranked by relevance
And each entry includes a personalized reason

### Scenario: Digest respects user budget
Given a user with max_price of 0
And events including some paid events
When the courier builds a digest
Then only free events are included in the digest

### Scenario: Digest covers correct date range
Given a user requesting a 7-day digest
And events spanning 30 days
When the courier builds a digest
Then only events within 7 days are included

### Scenario: Format digest as markdown
Given a valid digest with entries
When the courier formats the digest as markdown
Then a string is returned with event titles, dates, and links

### Scenario: Format digest as HTML
Given a valid digest with entries
When the courier formats the digest as HTML
Then an HTML string is returned with event cards

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

### Scenario: Reject invalid borough
Given a request with boroughs=["New Jersey"]
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
Given a string containing "'; DROP TABLE events; --"
When the sanitizer processes it
Then SQL injection characters are removed

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
Given a user sets interests to ["ai", "web3"]
When a new agent session is created
Then the user's interests are recalled as ["ai", "web3"]

### Scenario: Track RSVP history
Given a user RSVPs to an event
When RSVP history is queried
Then the event appears in the history

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
Then 3 tools are returned: list_events, get_digest, search_events

### Scenario: Call list_events tool
Given the MCP server is running
When list_events is called with topic="ai"
Then a list of AI-related events is returned

### Scenario: Call get_digest with authentication
Given a valid user_id
When get_digest is called
Then a personalized digest is returned

### Scenario: Reject get_digest without authentication
Given no user_id is provided
When get_digest is called
Then a ValueError is raised

---

## Feature: Web Dashboard

### Scenario: Health check returns 200
Given the server is running
When GET /health is called
Then status 200 is returned
And response contains {"status": "healthy"}

### Scenario: Events endpoint returns 200
Given the server is running
When GET /events is called
Then status 200 is returned
And response is a JSON array

### Scenario: Events endpoint supports topic filter
Given events exist with various topics
When GET /events?topic=ai is called
Then only events with "ai" tag are returned

### Scenario: Digest requires user_id
Given the server is running
When GET /digest is called without user_id
Then status 422 is returned

### Scenario: Invalid days_ahead returns error
Given the server is running
When GET /events?days_ahead=-1 is called
Then status 400 or 422 is returned

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
And AI events exist in sources
When the pipeline runs
Then the digest contains AI-related events ranked highest

### Scenario: User RSVP and agent learns
Given a user RSVPs to an AI event
When the next digest is generated
Then AI events have higher relevance scores

### Scenario: User deletes account
Given a user with data stored
When delete_account is called
Then all user data is removed
And subsequent queries return not found
