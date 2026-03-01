# 05 — Infrastructure

> **Migrated from**: `docs/specs/01-deployment.md` (Container Architecture, Networking, Volumes, Docker Compose, Ephemeral Model, Image Builds, Resource Limits, Docker Socket Security, Network Egress, Makefile, Environment, Health Checks, Scaling)

## Overview

This spec is the canonical reference for the Docker Compose infrastructure,
container architecture, networking, volume layout, image builds, resource
limits, security boundaries, and multi-machine scaling strategy. Other specs
reference this one for infrastructure details.

The orchestrator (spec 13) dynamically manages ephemeral agent containers on top
of this infrastructure. The API server (spec 14) is the sole external interface.

---

## Container Architecture

```mermaid
flowchart TD
    subgraph DockerNet["Docker Compose Network (ai-team-net)"]
        subgraph AlwaysRunning["Always-running (docker-compose.yml)"]
            Redis["Redis\n(internal)"]
            PG["PostgreSQL\n(internal)"]
            Orch["Orchestrator"]
            API["API Server\n:8080 (ext)"]
            Loki["Loki\n(internal)"]
            Grafana["Grafana\n:3000"]
            Proxy["Docker Socket\nProxy (internal)"]
        end
        subgraph Ephemeral["Ephemeral (spawned by Orchestrator on demand)"]
            Nelson["Nelson (x N)"]
            Julius["Julius"]
            Sherlock["Sherlock (x N)"]
            Leonard["Leonard (x N)"]
            Katherine["Katherine (x N)"]
            Richelieu["Richelieu (x N)"]
        end
        RepoVol[("Target Repo Volume\n(shared, mostly RO)")]
    end

    TUI["TUI Client\n(standalone, runs on host)"] -->|REST + SSE| API

    Orch -->|spawns| Ephemeral
    Orch -->|TCP:2375| Proxy
```

---

## Networking

- **Internal network**: All containers share a Docker bridge network (`ai-team-net`).
- **No direct container-to-container calls**: All communication flows through Redis.
- **No exposed database ports**: Redis and PostgreSQL are internal-only. No ports
  mapped to the host.
- **Exposed ports**:
  - API Server (`:8080`) — the sole external interface for the TUI client.
  - Grafana (`:3000`) — observability dashboards.
- **Egress**: No restrictions on outbound internet access from agent containers.
  Agents may need to download packages, access documentation, or call external APIs
  (LLM providers, GitHub). Outbound connections are logged for audit (see spec 09).

---

## Volumes

All data paths are mounted as named Docker volumes to support cloud volume providers
(DigitalOcean Block Storage, GCP Persistent Disks, etc.).

| Volume               | Mount Path (container)         | Access              | Purpose                          |
|----------------------|--------------------------------|---------------------|----------------------------------|
| `ai-team-redis`     | `/data`                        | Redis only          | Redis persistence (AOF/RDB)      |
| `ai-team-postgres`  | `/var/lib/postgresql/data`     | PostgreSQL only     | PostgreSQL data directory         |
| `ai-team-repo`      | `/workspace`                   | All agents (see below) | Target repo clone             |
| `ai-team-worktrees` | `/workspace/.worktrees`        | Leonard + Richelieu (RW), others (RO) | Git worktrees for parallel tasks |
| `ai-team-loki`      | `/loki`                        | Loki only           | Log index and chunk storage       |
| `ai-team-grafana`   | `/var/lib/grafana`             | Grafana only        | Dashboard definitions and state   |

> **Removed**: The `ai-team-logs` volume from the original design. Agents log to
> stdout via structlog. The Docker Loki logging driver ships logs to Loki
> automatically. No agent writes log files to disk.

### Filesystem Access Rules

- **All agents**: Read access to `/workspace` (the target repo) for codebase analysis.
- **Richelieu**: Read/write to `/workspace` and `/workspace/.worktrees`. Manages all
  git state: fetching, branching, merging, pushing, worktree lifecycle.
- **Leonard**: Read/write only within its assigned worktree path. Cannot modify the
  main repo checkout or other Leonard instances' worktrees.
- **All other agents** (Nelson, Julius, Sherlock, Katherine): Read-only access to
  the repo. They analyze but never modify.
- **Database access**: All agents connect to PostgreSQL over the network via
  `DATABASE_URL`. No filesystem mount is needed for database access.

> **Enforcement**: Use Docker volume mounts with `:ro` for read-only agents. Leonard
> containers mount only their specific worktree path as writable.

---

## Docker Compose Structure

This is the **canonical** `docker-compose.yml` definition. Spec 06 (orchestrator) and
other specs reference this section rather than duplicating it.

```yaml
# docker-compose.yml — infrastructure + orchestrator + API server
# ALL agents are ephemeral, spawned by the orchestrator on demand.

x-logging: &default-logging
  driver: loki
  options:
    loki-url: "http://loki:3100/loki/api/v1/push"
    loki-retries: "2"
    loki-batch-size: "400"
    labels: "service={{.Name}}"

services:
  redis:
    image: redis:latest
    volumes:
      - ai-team-redis:/data
    command: redis-server --appendonly yes
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
    logging: *default-logging
    # No ports exposed — internal only

  postgres:
    image: postgres:latest
    environment:
      POSTGRES_DB: ai_team
      POSTGRES_USER: ai_team
      POSTGRES_PASSWORD: ai_team
    volumes:
      - ai-team-postgres:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ai_team"]
      interval: 5s
    logging: *default-logging
    # No ports exposed — internal only

  loki:
    image: grafana/loki:latest
    volumes:
      - ai-team-loki:/loki
    command: -config.file=/etc/loki/local-config.yaml
    # No logging driver — Loki uses Docker default (json-file)
    # to avoid circular dependency (Loki logging to itself).
    # Loki logs are accessible via `docker logs loki`.

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    volumes:
      - ai-team-grafana:/var/lib/grafana
    depends_on:
      - loki
    # No Loki logging driver — same reason as Loki.
    # Grafana logs are accessible via `docker logs grafana`.

  docker-proxy:
    image: tecnativa/docker-socket-proxy:latest
    environment:
      CONTAINERS: 1          # Allow container operations
      POST: 1                # Allow create/start/stop
      IMAGES: 0              # Block image operations
      NETWORKS: 0            # Block network changes
      VOLUMES: 0             # Block volume creation
      SERVICES: 0            # Block swarm services
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    privileged: true
    # No Loki logging driver — infrastructure service.

  orchestrator:
    build: ./orchestrator
    env_file: .env
    environment:
      DOCKER_HOST: tcp://docker-proxy:2375
    depends_on:
      redis:
        condition: service_healthy
      postgres:
        condition: service_healthy
      docker-proxy:
        condition: service_started
    healthcheck:
      test: ["CMD", "python", "-c",
        "import redis; r=redis.Redis(host='redis'); assert r.get('orchestrator:heartbeat')"]
      interval: 15s
      timeout: 5s
      retries: 3
    restart: unless-stopped
    logging: *default-logging

  api:
    build: ./api
    ports:
      - "8080:8080"
    env_file: .env
    depends_on:
      redis:
        condition: service_healthy
      postgres:
        condition: service_healthy
    restart: unless-stopped
    logging: *default-logging

  # ALL agents (Nelson, Julius, Sherlock, Leonard, Katherine, Richelieu)
  # are ephemeral containers launched by the orchestrator on demand.
  # See "All-Ephemeral Agent Model" below and spec 13.

volumes:
  ai-team-redis:
  ai-team-postgres:
  ai-team-repo:
  ai-team-worktrees:
  ai-team-loki:
  ai-team-grafana:

networks:
  default:
    name: ai-team-net
```

### Logging driver notes

| Service        | Logging driver | Why                                          |
|----------------|----------------|----------------------------------------------|
| Redis          | Loki           | Capture Redis warnings and slow log entries  |
| PostgreSQL     | Loki           | Capture query errors and connection issues   |
| Orchestrator     | Loki           | Primary orchestration logs                   |
| API Server     | Loki           | Request/response logs                        |
| Loki           | json-file      | Cannot log to itself (circular dependency)   |
| Grafana        | json-file      | Infrastructure service, `docker logs` access |
| Docker Proxy   | json-file      | Infrastructure service, minimal output       |
| Ephemeral agents | Loki (see spec 13) | Launcher configures per container       |

---

## All-Ephemeral Agent Model

**Every agent** — including Nelson and Richelieu — is an ephemeral container
spawned by the orchestrator on demand. No agents are defined in `docker-compose.yml`.

This gives us:
- **Uniform lifecycle**: Every agent follows the same spawn → work → exit pattern.
  The orchestrator manages retries, health, and cleanup for all of them.
- **Horizontal scaling**: When multiple consensus requests arrive simultaneously,
  the orchestrator spawns multiple Nelson instances. Same for parallel Richelieus
  on non-conflicting git operations.
- **Resource efficiency**: No idle agent containers burning memory.
- **Multi-machine ready**: The orchestrator's Launcher is the only component that
  talks to the container runtime. Swapping Docker SDK for Kubernetes Jobs or
  ECS Tasks makes all agents run across a cluster (see Scaling below).

### Spawning rules per agent

| Agent      | Spawned when                           | Parallelism              | Exits when                          |
|------------|----------------------------------------|--------------------------|-------------------------------------|
| Julius     | `pipeline_created`                     | 1 per pipeline           | Publishes `decomposition_complete`  |
| Sherlock   | Task becomes `ready`                   | Up to N parallel         | Publishes `task_enriched`           |
| Leonard    | Task is enriched + worktree ready      | Up to N parallel         | Publishes `implementation_complete` |
| Katherine  | Implementation complete                | Up to N parallel         | Publishes `review_result`           |
| Nelson     | `consensus_request` on stream          | Up to N parallel         | Publishes `consensus_response`      |
| Richelieu  | `git_request` on stream                | 1 per branch (serialized)| Publishes `git_response`            |

---

## Image Build Strategy

All agents share the `core/` package. To avoid rebuilding shared dependencies
six times, a **shared base image** is used.

### Build hierarchy

```
python:3.12-slim
    │
    └── ai-team-base:latest
        │   Installs: core/ package + all shared dependencies
        │   Built by: make build-base
        │
        ├── ai-team-julius:latest
        │   Adds: agents/julius/ package
        │
        ├── ai-team-sherlock:latest
        │   Adds: agents/sherlock/ package
        │
        ├── ai-team-leonard:latest
        │   Adds: agents/leonard/ package + code execution tools
        │
        ├── ai-team-katherine:latest
        │   Adds: agents/katherine/ package
        │
        ├── ai-team-nelson:latest
        │   Adds: agents/nelson/ package
        │
        └── ai-team-richelieu:latest
            Adds: agents/richelieu/ package + git tools
```

### Base image Dockerfile

```dockerfile
# docker/Dockerfile.base
FROM python:3.12-slim

# System dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    git \
    && rm -rf /var/lib/apt/lists/*

# Install uv
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

# Install core package
WORKDIR /app
COPY core/ core/
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev --package ai-team-core
```

### Agent Dockerfile template

```dockerfile
# agents/{agent}/Dockerfile
FROM ai-team-base:latest

COPY agents/{agent}/ agents/{agent}/
RUN uv sync --frozen --no-dev --package {agent}

USER nobody
ENTRYPOINT ["uv", "run", "python", "-m", "{agent}"]
```

### Build commands

```makefile
build-base:    docker build -f docker/Dockerfile.base -t ai-team-base .
build-agent:   docker build -f agents/$(AGENT)/Dockerfile -t ai-team-$(AGENT) .
build:         make build-base && for agent in julius sherlock leonard katherine nelson richelieu; do make build-agent AGENT=$$agent; done
```

---

## Resource Limits

Every ephemeral agent container is launched with resource limits to prevent
runaway processes. Defaults are defined here; they can be overridden per agent
in the target repo's `.ai-team.yaml`.

### Default resource profiles

| Agent      | CPU limit | Memory limit | CPU reservation | Memory reservation | Rationale                           |
|------------|-----------|-------------|-----------------|--------------------|------------------------------------|
| Nelson     | 1.0       | 1G          | 0.25            | 256M               | Parallel LLM API calls (I/O bound) |
| Julius     | 1.0       | 1G          | 0.25            | 256M               | Codebase analysis, LLM calls       |
| Sherlock   | 1.0       | 1G          | 0.25            | 256M               | File reading, context building      |
| Leonard    | 2.0       | 2G          | 0.5             | 512M               | Code execution, tests, linters      |
| Katherine  | 1.0       | 1G          | 0.25            | 256M               | Diff analysis, LLM calls           |
| Richelieu  | 0.5       | 512M        | 0.25            | 128M               | Git operations only                 |

### `.ai-team.yaml` overrides

The target repo owner can override resource limits for agents that need more
(e.g., Leonard running a heavy test suite):

```yaml
# .ai-team.yaml
resources:
  leonard:
    cpu_limit: 4.0
    memory_limit: 4G
  sherlock:
    memory_limit: 2G
```

The Launcher merges `.ai-team.yaml` overrides with the defaults above. The
override values take precedence.

---

## Docker Socket Security

The orchestrator needs Docker API access to spawn agent containers. Instead of
mounting the raw Docker socket (which grants full host-level Docker access), a
**socket proxy** restricts the orchestrator to only the operations it needs.

### Architecture

```mermaid
flowchart LR
    Orch[Orchestrator] -->|TCP:2375| Proxy["Docker Socket Proxy\n(tecnativa/docker-socket-proxy)"]
    Proxy -->|unix socket| Sock["/var/run/docker.sock"]

    Proxy -. "Allows:\n- Container create/start/stop/inspect/wait/remove" .-> Sock
    Proxy -. "Blocks:\n- Image build/push/pull\n- Network create/delete\n- Volume create/delete\n- Swarm operations\n- Exec into containers" .-x Blocked[ ]

    style Blocked fill:none,stroke:none
```

### Risk mitigation

| Risk                                 | Mitigation                                    |
|--------------------------------------|-----------------------------------------------|
| Compromised orchestrator spawns rogue container | Socket proxy blocks unauthorized operations |
| Orchestrator accesses host filesystem  | Socket proxy blocks volume creation           |
| Orchestrator modifies network topology | Socket proxy blocks network operations        |
| Proxy itself is compromised          | Proxy runs with minimal attack surface, no exposed ports |

---

## Network Egress

Agent containers have **unrestricted outbound internet access**. This is an
intentional design decision:

- **Leonard** may need to `pip install`, `npm install`, or download dependencies
  when running the target repo's test suite or build commands.
- **Agents with web search tools** may need to access documentation, package
  registries, or reference materials to do their work effectively.
- **LLM API calls** go through OpenRouter (external endpoint).
- **Richelieu** calls the GitHub API for PR operations.

### Audit

All outbound connections are logged for audit purposes via the standard
structlog output. The audit trail is queryable in Grafana/Loki and stored in
the `audit_log` PostgreSQL table (see spec 09).

### Future hardening

If the threat model evolves (e.g., the system handles repos with proprietary code
where exfiltration is a concern), a forward proxy with domain allowlisting can be
added as a Docker Compose service. This is explicitly deferred — see spec 09.

---

## Makefile

```makefile
# ─── Build ───────────────────────────────────────────────
build:              # Build all images (base + all agents + orchestrator + api)
build-base:         # Build shared base image only
build-agent:        # Build single agent image (make build-agent AGENT=leonard)
build-orchestrator:   # Build orchestrator image
build-api:          # Build API server image

# ─── Stack ───────────────────────────────────────────────
up:                 # Start the full stack (docker compose up -d)
down:               # Stop all services (including ephemeral agents)
restart:            # Restart a service (make restart SVC=orchestrator)

# ─── Debugging ───────────────────────────────────────────
logs:               # Tail logs from all services
logs-agent:         # Tail logs from a specific agent (make logs-agent AGENT=nelson)
status:             # Show running containers and health
shell:              # Shell into a running container (make shell SVC=orchestrator)
db-shell:           # psql into PostgreSQL
redis-cli:          # Connect to Redis CLI

# ─── Maintenance ─────────────────────────────────────────
prune:              # Clean orphaned agent containers + dangling images
clean:              # Remove all containers, volumes, and images
reset-db:           # Reset PostgreSQL database

# ─── Development ─────────────────────────────────────────
test:               # Run full test suite
test-unit:          # Run unit tests only
test-integration:   # Run integration tests (requires running stack)
lint:               # Run ruff + mypy on all packages
format:             # Run ruff format on all packages

# ─── Operations ──────────────────────────────────────────
connect:            # Connect ai-team to a target repo (make connect REPO=<url>)
tui:                # Launch the TUI client (standalone, not Docker)
```

---

## Environment Configuration

```bash
# .env (never committed — in .gitignore and .dockerignore)

# --- LLM Provider ---
OPENROUTER_API_KEY=sk-or-...

# --- GitHub ---
GITHUB_APP_ID=12345
GITHUB_APP_PRIVATE_KEY_PATH=/secrets/github-app.pem
GITHUB_PAT=ghp_...                       # Fallback if no GitHub App

# --- Infrastructure ---
REDIS_URL=redis://redis:6379
DATABASE_URL=postgresql+asyncpg://ai_team:ai_team@postgres:5432/ai_team

# --- API Server ---
API_TOKEN=at-...                          # Token for TUI authentication

# --- General ---
LOG_LEVEL=DEBUG
DAILY_BUDGET_HARD=100.00                  # USD per day across all tasks
```

---

## Health Checks

| Component   | Health check mechanism                              | Interval |
|-------------|-----------------------------------------------------|----------|
| Redis       | `redis-cli ping` (Docker Compose healthcheck)       | 5s       |
| PostgreSQL  | `pg_isready -U ai_team` (Docker Compose healthcheck)| 5s       |
| Orchestrator  | Redis heartbeat key (`orchestrator:heartbeat`)        | 10s write, 15s check |
| API Server  | `/health` endpoint (Docker Compose healthcheck)     | 15s      |
| Agents      | Redis `status` stream heartbeats (every 30s)        | 30s      |
| Loki        | Process-alive (Docker Compose default)              | —        |
| Grafana     | Process-alive (Docker Compose default)              | —        |

The orchestrator's watchdog monitors agent heartbeats and Docker container status.
See spec 13 for watchdog details.

---

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
4. **The orchestrator is the only component that talks to the container runtime** — it's
   the single point where Docker SDK calls live (via the Launcher abstraction).
5. **The API server is stateless** — it reads from Redis/PostgreSQL. Can be replicated
   behind a load balancer.

### What scales automatically

| Component    | Single machine          | Multi-machine                         |
|-------------|-------------------------|---------------------------------------|
| Redis       | Local container         | ElastiCache / Memorystore             |
| PostgreSQL  | Local container         | RDS / Cloud SQL / managed Postgres    |
| Orchestrator  | Local container         | Single pod/task (leader election for HA) |
| API Server  | Local container         | Multiple replicas behind load balancer |
| Agents      | Local containers        | Jobs/tasks across cluster nodes       |
| Loki+Grafana| Local containers        | Managed observability (CloudWatch, GCP Logging) or self-hosted in cluster |
| Images      | Local builds            | Container registry (GHCR, ECR, GCR)   |

### Scaling the orchestrator

The orchestrator is lightweight (event router + container launcher). A single instance
handles hundreds of pipelines. For high availability:
- Run 2+ replicas with Redis-based leader election.
- Only the leader processes events and spawns containers.
- Followers monitor the leader's heartbeat and take over if it fails.

### Network considerations

In multi-machine setups, all containers must be able to reach Redis and PostgreSQL.
This is standard in k8s (Services) and ECS (Service Connect / service discovery).
No special networking is needed beyond what the orchestrator provides.

---

## Relationship to Other Specs

| Spec | Relationship |
|------|-------------|
| 05 (this) | Docker Compose, volumes, networking, image builds, resource limits, Docker socket security, egress policy, Makefile, environment, health checks, scaling |
| 06 | Orchestrator owns the Launcher Protocol, ephemeral container lifecycle, and dynamic container management on top of this infrastructure |
| 07 | API server deployment details (service definition in Docker Compose above) |
| 08 | TUI is a standalone client connecting to the API server exposed by this infrastructure |
| 09 | Threat model, container hardening, prompt injection defense — builds on socket security and egress policy defined here |
| 10 | Redis Streams protocol — Redis infrastructure defined here |
| 12 | PostgreSQL schema — database infrastructure defined here |
