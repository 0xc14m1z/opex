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
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐   │
│  │  Nelson   │  │  Julius  │  │ Sherlock │  │ Leonard (x N)    │   │
│  │  :8001    │  │  :8002   │  │  :8003   │  │ :8004, :8005...  │   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └───────┬──────────┘   │
│       │              │              │                │              │
│  ┌────┴─────┐  ┌────┴─────┐                                       │
│  │Katherine │  │Richelieu │                                        │
│  │  :8006   │  │  :8007   │                                        │
│  └────┬─────┘  └────┬─────┘                                        │
│       │              │                                              │
│       └──────┬───────┘                                              │
│              │                                                      │
│       ┌──────┴──────┐    ┌──────────┐    ┌──────────────────────┐  │
│       │    Redis     │    │  SQLite   │    │  Target Repo Volume  │  │
│       │   :6379      │    │ (volume)  │    │   (shared, mostly RO)│  │
│       └─────────────┘    └──────────┘    └──────────────────────┘  │
│                                                                     │
│       ┌─────────────┐                                               │
│       │  TUI Client  │  (runs on host, connects to Redis)           │
│       └─────────────┘                                               │
└─────────────────────────────────────────────────────────────────────┘
```

## Networking

- **Internal network**: All containers share a Docker bridge network (`ai-team-net`).
- **No direct container-to-container calls**: All communication flows through Redis.
- **Exposed ports**: Only Redis (for TUI client on host) and optionally agent health
  check ports for debugging. In production, nothing is exposed externally.

## Volumes

| Volume               | Mount Path (container)    | Access    | Purpose                          |
|----------------------|---------------------------|-----------|----------------------------------|
| `ai-team-redis`     | `/data`                   | Redis only| Redis persistence (AOF/RDB)      |
| `ai-team-sqlite`    | `/data/db`                | All agents| SQLite database file             |
| `ai-team-repo`      | `/workspace`              | All agents| Target repo clone                |
| `ai-team-worktrees` | `/workspace/.worktrees`   | Leonard + Richelieu (RW), others (RO) | Git worktrees for parallel tasks |
| `ai-team-logs`      | `/data/logs`              | All agents| Structured log files             |

### Filesystem Access Rules

- **All agents**: Read access to `/workspace` (the target repo) for codebase analysis.
- **Richelieu**: Read/write to `/workspace` and `/workspace/.worktrees`. Manages git state.
- **Leonard**: Read/write only within its assigned worktree path. Cannot modify the
  main repo checkout or other Leonard instances' worktrees.
- **All other agents**: Read-only access to the repo. They analyze but never modify.

> Enforcement: Use Docker volume mounts with `:ro` for read-only agents. Leonard
> containers mount only their specific worktree path as writable.

## Docker Compose Structure

```yaml
# docker-compose.yml (simplified)
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

  nelson:
    build: ./agents/nelson
    env_file: .env
    volumes:
      - ai-team-repo:/workspace:ro
      - ai-team-sqlite:/data/db
      - ai-team-logs:/data/logs
    depends_on:
      redis:
        condition: service_healthy

  julius:
    build: ./agents/julius
    env_file: .env
    volumes:
      - ai-team-repo:/workspace:ro
      - ai-team-sqlite:/data/db:ro
      - ai-team-logs:/data/logs
    depends_on:
      redis:
        condition: service_healthy

  # sherlock, katherine: similar to julius (read-only workspace)

  richelieu:
    build: ./agents/richelieu
    env_file: .env
    volumes:
      - ai-team-repo:/workspace        # read-write
      - ai-team-worktrees:/workspace/.worktrees
      - ai-team-sqlite:/data/db
      - ai-team-logs:/data/logs
    depends_on:
      redis:
        condition: service_healthy

  # Leonard is NOT defined statically — Richelieu spawns Leonard containers
  # dynamically via the Docker API (see below)

volumes:
  ai-team-redis:
  ai-team-sqlite:
  ai-team-repo:
  ai-team-worktrees:
  ai-team-logs:

networks:
  default:
    name: ai-team-net
```

## Dynamic Leonard Scaling

Leonard instances are **not** defined in `docker-compose.yml`. Instead:

1. Julius determines how many parallel tasks can run (from the dependency graph).
2. Richelieu creates a worktree for each task.
3. Richelieu sends a "provision Leonard" message to a **Leonard Spawner** sidecar
   (or Richelieu itself uses the Docker SDK to spin up Leonard containers).
4. Each Leonard container gets:
   - Its own worktree mounted as writable.
   - The main repo mounted as read-only (for reference).
   - A unique task ID passed as an environment variable.
5. When Leonard finishes (or crashes), the container is stopped and removed.

> Alternative: Use a fixed pool of Leonard containers (e.g., `leonard-1`, `leonard-2`,
> `leonard-3`) that pick tasks from the queue. Simpler but less elastic.

We'll decide between dynamic spawning and a fixed pool during Phase 5 implementation.

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
reset-db:       # Reset SQLite database (confidence scores, weights, etc.)
tui:            # Launch the terminal UI dashboard
connect:        # Connect ai-team to a target repo (make connect REPO=<url>)
```

## Environment Configuration

```bash
# .env (never committed)
# --- LLM Providers ---
ANTHROPIC_API_KEY=sk-ant-...
OPENAI_API_KEY=sk-...
GOOGLE_API_KEY=AIza...

# --- GitHub ---
GITHUB_APP_ID=12345
GITHUB_APP_PRIVATE_KEY_PATH=/secrets/github-app.pem
GITHUB_PAT=ghp_...                    # Fallback if no GitHub App

# --- Redis ---
REDIS_URL=redis://redis:6379

# --- General ---
LOG_LEVEL=DEBUG
MAX_PARALLEL_LEONARDS=3
CONSENSUS_MAX_ROUNDS=3
DEFAULT_BUDGET_LIMIT_SOFT=5.00        # USD per task (soft limit)
DEFAULT_BUDGET_LIMIT_HARD=20.00       # USD per task (hard limit)
DAILY_BUDGET_HARD=100.00              # USD per day across all tasks
```

## Health Checks

Every agent container exposes a `/health` endpoint (or equivalent) that reports:
- Agent status (idle, working, error).
- Current task ID (if working).
- Redis connectivity.
- Last heartbeat timestamp.

Docker Compose health checks use these to detect unresponsive agents.

## Future: Cloud Deployment

The Docker Compose setup is designed to be portable:
- All state is in volumes (Redis, SQLite, logs) — easily mapped to cloud storage.
- Agent images are self-contained — push to any container registry.
- Redis can be swapped for a managed service (ElastiCache, Memorystore).
- SQLite can be swapped for PostgreSQL if concurrent write contention becomes an issue.
- Leonard dynamic scaling maps naturally to Kubernetes Jobs or ECS tasks.
