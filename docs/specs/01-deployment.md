# 01 — Deployment

## Overview

All agents run as isolated Docker containers orchestrated by Docker Compose.
A Makefile provides the developer-facing interface for building, running,
and debugging the system.

---

## Container Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Docker Compose Network                       │
│                                                                     │
│  ┌─ Always-running (docker-compose.yml) ─────────────────────────┐ │
│  │                                                                 │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │ │
│  │  │  Redis    │  │PostgreSQL│  │Controller │  │  TUI Client   │  │ │
│  │  │  :6379    │  │  :5432   │  │(orchestr.)│  │ (host, Redis) │  │ │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────────┘  │ │
│  │                                                                 │ │
│  │  ┌──────────┐  ┌──────────┐                                    │ │
│  │  │   Loki   │  │ Grafana  │                                    │ │
│  │  │  :3100   │  │  :3000   │                                    │ │
│  │  └──────────┘  └──────────┘                                    │ │
│  └─────────────────────────────────────────────────────────────────┘ │
│                                                                     │
│  ┌─ Ephemeral (spawned by Controller on demand) ─────────────────┐ │
│  │                                                                 │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │ │
│  │  │  Nelson   │  │  Julius  │  │ Sherlock │  │ Leonard (x N) │  │ │
│  │  │ (x N)    │  │          │  │ (x N)    │  │               │  │ │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────────┘  │ │
│  │                                                                 │ │
│  │  ┌──────────┐  ┌──────────┐                                    │ │
│  │  │Katherine │  │Richelieu │                                    │ │
│  │  │ (x N)    │  │ (x N)    │                                    │ │
│  │  └──────────┘  └──────────┘                                    │ │
│  └─────────────────────────────────────────────────────────────────┘ │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    Target Repo Volume                         │   │
│  │                    (shared, mostly RO)                        │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

## Networking

- **Internal network**: All containers share a Docker bridge network (`ai-team-net`).
- **No direct container-to-container calls**: All communication flows through Redis.
- **Exposed ports**: Redis (for TUI client on host), Grafana (:3000) for dashboards,
  and Loki (:3100) for log ingestion. In production, nothing is exposed externally.

## Volumes

| Volume               | Mount Path (container)         | Access    | Purpose                          |
|----------------------|--------------------------------|-----------|----------------------------------|
| `ai-team-redis`     | `/data`                        | Redis only| Redis persistence (AOF/RDB)      |
| `ai-team-postgres`  | `/var/lib/postgresql/data`     | PostgreSQL only | PostgreSQL data directory   |
| `ai-team-repo`      | `/workspace`                   | All agents| Target repo clone                |
| `ai-team-worktrees` | `/workspace/.worktrees`        | Leonard + Richelieu (RW), others (RO) | Git worktrees for parallel tasks |
| `ai-team-logs`      | `/data/logs`                   | All agents| Structured log files             |

### Filesystem Access Rules

- **All agents**: Read access to `/workspace` (the target repo) for codebase analysis.
- **Richelieu**: Read/write to `/workspace` and `/workspace/.worktrees`. Manages git state.
- **Leonard**: Read/write only within its assigned worktree path. Cannot modify the
  main repo checkout or other Leonard instances' worktrees.
- **All other agents**: Read-only access to the repo. They analyze but never modify.
- **Database access**: All agents connect to PostgreSQL over the network via `DATABASE_URL`.
  No filesystem mount is needed for database access.

> Enforcement: Use Docker volume mounts with `:ro` for read-only agents. Leonard
> containers mount only their specific worktree path as writable.

## Docker Compose Structure

```yaml
# docker-compose.yml — only infrastructure + controller
# ALL agents are ephemeral, spawned by the controller on demand.

x-logging: &default-logging
  driver: loki
  options:
    loki-url: "http://localhost:3100/loki/api/v1/push"
    loki-batch-size: "100"
    labels: "service={{.Name}}"

services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - ai-team-redis:/data
    command: redis-server --appendonly yes
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
    logging: *default-logging

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: aiteam
      POSTGRES_USER: aiteam
      POSTGRES_PASSWORD: aiteam
    ports:
      - "5432:5432"
    volumes:
      - ai-team-postgres:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U aiteam"]
      interval: 5s
    logging: *default-logging

  loki:
    image: grafana/loki:latest
    ports:
      - "3100:3100"

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    depends_on:
      - loki

  controller:
    build: ./controller
    env_file: .env
    volumes:
      - ai-team-logs:/data/logs
      - /var/run/docker.sock:/var/run/docker.sock  # Docker SDK access
    depends_on:
      redis:
        condition: service_healthy
      postgres:
        condition: service_healthy
    restart: unless-stopped
    logging: *default-logging

  # NO agent services here. All agents (Nelson, Julius, Sherlock,
  # Leonard, Katherine, Richelieu) are ephemeral containers launched
  # by the controller via Docker SDK. See spec 13.

volumes:
  ai-team-redis:
  ai-team-postgres:
  ai-team-repo:
  ai-team-worktrees:
  ai-team-logs:

networks:
  default:
    name: ai-team-net
```

## All-Ephemeral Agent Model

**Every agent** — including Nelson and Richelieu — is an ephemeral container
spawned by the controller on demand. No agents are defined in `docker-compose.yml`.

This gives us:
- **Uniform lifecycle**: Every agent follows the same spawn → work → exit pattern.
  The controller manages retries, health, and cleanup for all of them.
- **Horizontal scaling**: When multiple consensus requests arrive simultaneously,
  the controller spawns multiple Nelson instances. Same for parallel Richelieus
  on non-conflicting git operations.
- **Resource efficiency**: No idle agent containers burning memory.
- **Multi-machine ready**: The controller's Launcher is the only component that
  talks to the container runtime. Swapping Docker SDK for Kubernetes Jobs or
  ECS Tasks makes all agents run across a cluster (see Scaling below).

### Spawning rules per agent

| Agent      | Spawned when                           | Parallelism             | Exits when                          |
|------------|----------------------------------------|-------------------------|-------------------------------------|
| Julius     | `pipeline_created`                     | 1 per pipeline          | Publishes `decomposition_complete`  |
| Sherlock   | Task becomes `ready`                   | Up to N parallel        | Publishes `task_enriched`           |
| Leonard    | Task is enriched + worktree ready      | Up to N parallel        | Publishes `implementation_complete` |
| Katherine  | Implementation complete                | Up to N parallel        | Publishes `review_result`           |
| Nelson     | `consensus_request` on stream          | Up to N parallel        | Publishes `consensus_response`      |
| Richelieu  | `git_request` on stream                | 1 per branch (serialized) | Publishes `git_response`          |

## Makefile

```makefile
# Key targets
build:          # Build all agent images
up:             # Start the full stack (docker compose up -d)
down:           # Stop all services
logs:           # Tail logs from all agents
logs-agent:     # Tail logs from a specific agent (make logs-agent AGENT=nelson)
status:         # Show running containers and health
shell:          # Shell into a running agent (make shell AGENT=leonard-1)
test:           # Run full test suite
test-unit:      # Run unit tests only
test-integration: # Run integration tests (requires running stack)
lint:           # Run ruff + mypy on all packages
format:         # Run ruff format on all packages
clean:          # Remove all containers, volumes, and images
reset-db:       # Reset PostgreSQL database (confidence scores, weights, etc.)
tui:            # Launch the terminal UI dashboard
connect:        # Connect ai-team to a target repo (make connect REPO=<url>)
```

## Environment Configuration

```bash
# .env (never committed)
# --- LLM Provider ---
OPENROUTER_API_KEY=sk-or-...

# --- GitHub ---
GITHUB_APP_ID=12345
GITHUB_APP_PRIVATE_KEY_PATH=/secrets/github-app.pem
GITHUB_PAT=ghp_...                    # Fallback if no GitHub App

# --- Infrastructure ---
REDIS_URL=redis://redis:6379
DATABASE_URL=postgresql+asyncpg://aiteam:password@postgres:5432/aiteam

# --- General ---
LOG_LEVEL=DEBUG
MAX_PARALLEL_LEONARDS=3
DAILY_BUDGET_HARD=100.00              # USD per day across all tasks
```

## Health Checks

Every agent publishes heartbeats to the Redis `status` stream every 30 seconds.
The controller's watchdog monitors these (see spec 13). Docker Compose uses
simple process-alive checks.

## Scaling to Multiple Machines

The architecture is designed to scale from a single machine (Docker Compose) to a
cluster (Kubernetes, ECS, Swarm) with minimal code changes.

### Why it works

The system already has the right boundaries for multi-machine scaling:

1. **All communication is through Redis** — agents don't talk to each other directly.
   Redis can be a managed service (ElastiCache, Memorystore) reachable from any node.
2. **All durable state is in PostgreSQL** — swap for RDS, Cloud SQL, or any managed
   Postgres. Reachable from any node.
3. **All agents are ephemeral and stateless** — they receive input via Redis, do work,
   publish results, and exit. No local state between runs (checkpoints are in PostgreSQL).
4. **The controller is the only component that talks to the container runtime** — it's
   the single point where Docker SDK calls live.

### The Launcher abstraction

The controller's `Launcher` class is the key to multi-machine scaling. It has a
pluggable backend:

```python
class Launcher(Protocol):
    """Interface for launching agent containers on any runtime."""
    async def launch(self, agent: AgentName, pipeline_id: str, task_id: str | None, env: dict[str, str] | None = None) -> str: ...
    async def stop(self, container_id: str, timeout: int = 30) -> None: ...
    async def inspect(self, container_id: str) -> ContainerStatus: ...
    async def wait(self, container_id: str) -> int: ...
    async def remove(self, container_id: str) -> None: ...
```

| Backend              | Runtime           | Use case                      |
|----------------------|-------------------|-------------------------------|
| `DockerLauncher`     | Docker SDK        | Local dev (single machine)    |
| `KubernetesLauncher` | k8s Jobs API     | Production cluster            |
| `ECSLauncher`        | ECS RunTask API   | AWS production                |

The backend is selected via configuration. Agent images are pushed to a container
registry (ECR, GCR, GHCR) and referenced by the launcher.

### What scales automatically

| Component    | Single machine          | Multi-machine                         |
|-------------|-------------------------|---------------------------------------|
| Redis       | Local container         | ElastiCache / Memorystore             |
| PostgreSQL  | Local container         | RDS / Cloud SQL / managed Postgres    |
| Controller  | Local container         | Single pod/task (leader election for HA) |
| Agents      | Local containers        | Jobs/tasks across cluster nodes       |
| Loki+Grafana| Local containers        | Managed observability (CloudWatch, GCP Logging) or self-hosted in cluster |
| Images      | Local builds            | Container registry                    |

### Scaling the controller

The controller is lightweight (event router + container launcher). A single instance
handles hundreds of pipelines. For high availability:
- Run 2+ replicas with Redis-based leader election.
- Only the leader processes events and spawns containers.
- Followers monitor the leader's heartbeat and take over if it fails.

### Network considerations

In multi-machine setups, all containers must be able to reach Redis and PostgreSQL.
This is standard in k8s (Services) and ECS (Service Connect / service discovery).
No special networking is needed beyond what the orchestrator provides.
