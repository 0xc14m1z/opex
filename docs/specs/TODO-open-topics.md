# Open Topics — To Be Specced

## Critical (blocks implementation)

### ~~1. Pipeline Orchestrator~~ — RESOLVED

Resolved in [13-orchestrator.md](13-orchestrator.md). A deterministic controller service
(no LLM) watches Redis Streams and launches ephemeral agent containers via Docker SDK.
All agents (including Nelson and Richelieu) are ephemeral one-shot containers.

### 2. Agent Prompts & Tool Definitions
The most important part of any agentic system. What system prompts does each agent get?
What tools (file read, shell exec, search) does each agent have in PydanticAI? How do we
version and iterate on prompts?

### ~~3. Model Selection Strategy~~ — RESOLVED

Resolved: Model selection is configured in `.ai-team.yaml` under the `llm:` section.
Each target repo specifies `default_model`, `consensus.models`, and optional per-agent
`overrides`. All calls go through OpenRouter via LiteLLM. See specs 02 and 12.

## Important (will cause problems later)

### 4. Rate Limiting
All calls go through OpenRouter, which has its own rate limits. Multiple Leonards and
Nelson's parallel consensus calls can hit these limits. How do we manage this across the
whole system? Options: LiteLLM's built-in rate limit handling, a central rate limiter in
the controller, or per-agent backoff.

### 5. Caching Strategy
Sherlock and Leonard may read the same files. Nelson may get similar prompts. Where do we
cache and how do we invalidate?

### ~~6. Nelson Concurrency~~ — RESOLVED
Nelson is ephemeral — the controller spawns a new Nelson container for each
`consensus_request`. Multiple Nelsons run in parallel (up to `max_parallel_nelsons`).
Requests queue in Redis if the limit is reached. See specs 01, 13.

## Resolved in Phase 0 Planning

### ~~7. Durable Storage~~ — RESOLVED
SQLite replaced with PostgreSQL (`postgres:16-alpine`). asyncpg driver. Solves concurrent
access from multiple containers. See specs 01, 12.

### ~~8. Log Inspection~~ — RESOLVED
Grafana + Loki added to Docker Compose. Docker Loki logging driver ships all container
logs automatically. Grafana provides LogQL querying and dashboards. See specs 01, 06.

### ~~9. Health Checks~~ — RESOLVED
Redis heartbeats only (every 30s to `status` stream). No HTTP health endpoints in agents.
Controller watchdog monitors heartbeats. See specs 08, 13.

### ~~10. Testing Infrastructure~~ — RESOLVED
testcontainers for real Redis and PostgreSQL per test session. VCR cassettes for LLM call
recording/replay. No mocks. See spec 11.

### ~~11. LLM Gateway~~ — RESOLVED
OpenRouter as the single LLM provider. Single `OPENROUTER_API_KEY`. LiteLLM as the Python
client layer (retry, cost callbacks, streaming). See specs 01, 04, 07.

### ~~12. Database Migrations~~ — RESOLVED
Numbered SQL files in `core/migrations/` + lightweight Python runner (~50 LOC) tracking
applied migrations in a `schema_migrations` table. See spec 12.

### ~~13. Redis Queue Abstraction~~ — RESOLVED
Medium-thickness typed wrapper: auto MessageEnvelope serialization, async generators for
consuming, idempotent consumer group creation, built-in dead letter handling. See spec 10.
