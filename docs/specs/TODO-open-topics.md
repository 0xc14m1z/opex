# Open Topics — To Be Specced

## Critical (blocks implementation)

### 1. Agent Prompts & Tool Definitions

The most important part of any agentic system. What system prompts does each agent get?
What tools (file read, shell exec, search) does each agent have in PydanticAI? How do we
version and iterate on prompts? These need to be defined in specs 16-21 (one per agent).

### ~~2. Pipeline Orchestrator~~ — RESOLVED

Resolved in [13-controller.md](13-controller.md). A deterministic controller service
(no LLM) watches Redis Streams and launches ephemeral agent containers via Docker SDK.
All agents (including Nelson and Richelieu) are ephemeral one-shot containers.

### ~~3. Model Selection Strategy~~ — RESOLVED

Resolved: Model selection is configured in `.ai-team.yaml` under the `llm:` section.
Each target repo specifies `default_model`, `consensus.models`, and optional per-agent
`overrides`. All calls go through OpenRouter via LiteLLM. See specs 12 and 01.

## Important (will cause problems later)

### 4. Rate Limiting

All calls go through OpenRouter, which has its own rate limits. Multiple Leonards and
Nelson's parallel consensus calls can hit these limits. How do we manage this across the
whole system? Options: LiteLLM's built-in rate limit handling, a central rate limiter in
the controller, or per-agent backoff.

> **Note**: The API server (spec 14) could also handle rate limiting for inbound TUI
> requests, which is a separate concern from LLM API rate limits.

### 5. Caching Strategy

Sherlock and Leonard may read the same files. Nelson may get similar prompts. Where do we
cache and how do we invalidate?

### 6. API Server Detail

Request/response schemas, error format, and rate limiting for the API server (spec 14).
The endpoint list is defined in spec 05 but the detail is not yet specced:
- Pydantic request/response models for each endpoint
- Error response format (structured JSON errors)
- Rate limiting strategy for TUI requests
- Authentication token management

### 7. TUI Detail

Detailed view specifications, keybindings, and learning chat UX for the TUI (spec 15):
- Textual widget hierarchy and layout
- Keybinding map for navigation
- Learning mode chat interface design
- Multi-team switching UX
- Log filtering and search interface

## Resolved in Phase 0 Planning

### ~~8. Durable Storage~~ — RESOLVED
SQLite replaced with PostgreSQL (`postgres:16-alpine`). asyncpg driver. Solves concurrent
access from multiple containers. See specs 05, 01.

### ~~9. Log Inspection~~ — RESOLVED
Grafana + Loki added to Docker Compose. Docker Loki logging driver ships all container
logs automatically. Grafana provides LogQL querying and dashboards. See specs 05, 06.

### ~~10. Health Checks~~ — RESOLVED
Redis heartbeats only (every 30s to `status` stream). No HTTP health endpoints in agents.
Controller watchdog monitors heartbeats. See specs 08, 13.

### ~~11. Testing Infrastructure~~ — RESOLVED
testcontainers for real Redis and PostgreSQL per test session. VCR cassettes for LLM call
recording/replay. No mocks. See spec 10.

### ~~12. LLM Gateway~~ — RESOLVED
OpenRouter as the single LLM provider. Single `OPENROUTER_API_KEY`. LiteLLM as the Python
client layer (retry, cost callbacks, streaming). See specs 05, 04, 07.

### ~~13. Database Migrations~~ — RESOLVED
Numbered SQL files in `core/migrations/` + lightweight Python runner (~50 LOC) tracking
applied migrations in a `schema_migrations` table. See spec 01.

### ~~14. Redis Queue Abstraction~~ — RESOLVED
Medium-thickness typed wrapper: auto MessageEnvelope serialization, async generators for
consuming, idempotent consumer group creation, built-in dead letter handling. See spec 04.

### ~~15. Nelson Concurrency~~ — RESOLVED
Nelson is ephemeral — the controller spawns a new Nelson container for each
`consensus_request`. Multiple Nelsons run in parallel (up to `max_parallel_nelsons`).
Requests queue in Redis if the limit is reached. See specs 05, 13.

### ~~16. Secret Scoping~~ — RESOLVED
Per-agent secret allowlists implemented in the Launcher. Each agent only receives the
env vars it needs. Target repo can extend (not reduce) allowlists via `.ai-team.yaml`.
See spec 09.

### ~~17. Docker Socket Security~~ — RESOLVED
Socket proxy (tecnativa/docker-socket-proxy) restricts controller to container
operations only. Blocks image, network, volume, and swarm operations. See spec 09.

### ~~18. Network Egress~~ — RESOLVED
No restrictions, audit only. Agents need internet for packages, docs, LLM APIs. All
outbound connections logged for audit. Forward proxy deferred. See spec 09.

### ~~19. Repo Volume Lifecycle~~ — RESOLVED
`make connect REPO=<url>` clones into ai-team-repo volume. Persists across pipelines
(warm cache). Richelieu fetches latest before each pipeline. See spec 12.
