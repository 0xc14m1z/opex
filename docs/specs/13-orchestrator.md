# 13 — Pipeline Orchestrator (Controller)

## Overview

The **controller** is a deterministic Python service (no LLM) that owns the end-to-end
pipeline flow. It watches Redis Streams for lifecycle events and launches/monitors agent
containers via the Docker SDK. It is the answer to "who is the conductor?"

The controller is **not** an agent — it has no PydanticAI dependency, makes no LLM calls,
and contains no prompts. It is a lightweight event router + container launcher (~hundreds
of lines of code).

---

## Design Principles

1. **No LLM calls** — the controller is pure orchestration logic.
2. **Single writer for pipeline state** — only the controller transitions pipeline status.
3. **All agents are ephemeral** — every agent (Nelson, Julius, Sherlock, Leonard,
   Katherine, Richelieu) runs as a one-shot container. The controller launches them,
   monitors them, and cleans up after them. No agents are defined in `docker-compose.yml`.
4. **Horizontal scaling via spawning** — when multiple consensus requests or git
   operations arrive, the controller spawns multiple Nelson or Richelieu instances.
   Richelieu operations on the same branch are serialized; everything else runs in parallel.
5. **The dependency graph lives here** — on each `branch_merged` event the controller
   evaluates `DependencyGraph.get_ready_tasks()` directly. No need to spin up Julius
   just to read state.
6. **Pluggable container runtime** — the Launcher has a backend interface (Docker SDK,
   Kubernetes Jobs, ECS Tasks) so the system scales from a single machine to a cluster
   with a config change.

---

## Project Structure

```
controller/
├── pyproject.toml
├── Dockerfile
└── src/
    └── controller/
        ├── __init__.py
        ├── main.py              # Entrypoint: connect to Redis, start router loop
        ├── router.py            # Event → handler dispatch table
        ├── launcher.py          # Docker SDK: run/stop/inspect containers
        ├── watchdog.py          # Heartbeat monitoring, dead container detection
        ├── state.py             # Pipeline + container state (PostgreSQL)
        └── config.py            # ControllerConfig (Pydantic settings)
```

The controller is a **uv workspace member** alongside `core/` and each agent package.
It depends on `ai-team-core` for shared models (DependencyGraph, TaskStatus, MessageEnvelope).

---

## Core Components

### 1. Router (`router.py`)

A single async loop that reads from multiple Redis Streams and dispatches to handlers.

```python
class Router:
    """Listens on Redis Streams and dispatches events to handlers."""

    # stream → handler mapping (see routing table below)
    handlers: dict[str, dict[str, Callable]]

    async def run(self) -> None:
        """Main loop: XREADGROUP on all pipeline streams + system streams."""
        while True:
            messages = await self.redis.xreadgroup(
                group="controller",
                consumer="controller-1",
                streams=self.watched_streams(),
                block=5000,  # ms
            )
            for stream, entries in messages:
                for msg_id, data in entries:
                    envelope = MessageEnvelope.model_validate_json(data[b"payload"])
                    handler = self.handlers[stream][envelope.message_type]
                    await handler(envelope)
                    await self.redis.xack(stream, "controller", msg_id)
```

### 2. Launcher (`launcher.py`)

Manages ephemeral agent containers via the Docker SDK (`docker` Python package).

```python
class Launcher:
    """Creates and manages one-shot agent containers."""

    async def launch(
        self,
        agent: AgentName,
        pipeline_id: str,
        task_id: str | None,
        env: dict[str, str] | None = None,
    ) -> str:
        """Launch an agent container. Returns container ID."""
        ...

    async def stop(self, container_id: str, timeout: int = 30) -> None: ...
    async def inspect(self, container_id: str) -> ContainerStatus: ...
    async def wait(self, container_id: str) -> int:
        """Wait for container to exit. Returns exit code."""
        ...
    async def remove(self, container_id: str) -> None: ...
```

Container creation details:

| Parameter         | Value                                                    |
|-------------------|----------------------------------------------------------|
| Image             | `ai-team-{agent}:latest` (built by `make build`)        |
| Network           | `ai-team-net`                                            |
| Volumes           | Same as `docker-compose.yml` definitions per agent       |
| Environment       | `.env` + `PIPELINE_ID` + `TASK_ID` + agent-specific vars |
| Name              | `ai-team-{agent}-{pipeline_id[:8]}-{task_id[:8]}`       |
| Restart policy    | `no` (the controller handles retries)                    |
| Labels            | `ai-team.agent`, `ai-team.pipeline`, `ai-team.task`     |

### 3. Watchdog (`watchdog.py`)

Monitors container health via heartbeats and Docker inspect.

```python
class Watchdog:
    """Detects dead/stuck containers and triggers recovery."""

    async def run(self) -> None:
        """Periodic loop: check heartbeats and container status."""
        while True:
            await self.check_heartbeats()
            await self.check_containers()
            await asyncio.sleep(self.config.watchdog_interval_seconds)

    async def check_heartbeats(self) -> None:
        """Flag containers with heartbeat gaps > threshold."""
        ...

    async def check_containers(self) -> None:
        """Docker inspect all tracked containers, detect unexpected exits."""
        ...
```

Detection rules:

| Signal                                  | Threshold        | Action                      |
|-----------------------------------------|------------------|-----------------------------|
| No heartbeat from agent container       | 90 seconds       | Docker inspect → retry or escalate |
| Container exited with non-zero code     | Immediate        | Retry (up to 3) or escalate |
| Container running longer than deadline  | Configurable     | Log warning at 50%, kill at 100% |

### 4. State Tracker (`state.py`)

Reads/writes pipeline and container state to PostgreSQL.

```python
class StateTracker:
    """Pipeline and container lifecycle state."""

    async def create_pipeline(self, pipeline_id: str, plan: str) -> None: ...
    async def get_pipeline(self, pipeline_id: str) -> PipelineRecord: ...
    async def update_pipeline_status(self, pipeline_id: str, status: PipelineStatus) -> None: ...

    async def track_container(self, container_id: str, agent: str, pipeline_id: str, task_id: str | None) -> None: ...
    async def update_container(self, container_id: str, status: str, exit_code: int | None = None) -> None: ...
    async def remove_container(self, container_id: str) -> None: ...

    async def get_dependency_graph(self, pipeline_id: str) -> DependencyGraph: ...
    async def update_task_status(self, task_id: str, status: TaskStatus) -> None: ...
```

---

## Event → Container Routing Table

The controller's full dispatch table. Every message type from spec 10 is accounted for.

### Pipeline Events (controller acts on)

| Event (message_type)       | Source Stream                  | Controller Action                                         |
|----------------------------|-------------------------------|-----------------------------------------------------------|
| `pipeline_created`         | `system:pipelines`            | Create pipeline record; spawn Richelieu (`create_feature_branch`); then spawn Julius |
| `decomposition_complete`   | `pipeline:{id}:tasks`         | Store dependency graph; emit `batch_dispatched`; spawn Sherlocks for ready tasks |
| `task_enriched`            | `pipeline:{id}:enriched`      | Update task status → `ready`; spawn Richelieu (`create_worktree`); then spawn Leonard |
| `implementation_complete`  | `pipeline:{id}:reviews`       | Spawn Katherine for the completed task                    |
| `review_result:approved`   | `pipeline:{id}:reviews`       | Spawn Richelieu (`merge_branch`)                          |
| `review_result:changes_requested` | `pipeline:{id}:reviews` | Spawn Leonard again (rework) with feedback attached       |
| `review_result:escalated`  | `pipeline:{id}:reviews`       | Mark task `needs_human`; emit status event                |
| `consensus_request`        | `pipeline:{id}:consensus`     | Spawn Nelson for this request                             |
| `git_request`              | `pipeline:{id}:git_ops`       | Spawn Richelieu for this operation (serialize per-branch) |
| `branch_merged`            | `pipeline:{id}:git_ops`       | Mark task `completed`; check `get_ready_tasks()` → spawn Sherlocks for newly unblocked tasks; if all tasks done → emit `all_tasks_complete` |
| `all_tasks_complete`       | (self-emitted)                | Spawn Richelieu (`open_pr`)                               |
| `pr_opened`                | `pipeline:{id}:git_ops`       | Mark pipeline `completed`; emit `pipeline_completed`      |
| `pipeline_failed`          | (self-emitted)                | Mark pipeline `failed`; alert human; preserve state       |

### Events the Controller Observes (but doesn't act on)

| Event (message_type)       | Source Stream                  | Notes                                                     |
|----------------------------|-------------------------------|-----------------------------------------------------------|
| `git_response`             | `pipeline:{id}:git_ops`       | Consumed by controller to track Richelieu results         |
| `consensus_response`       | `pipeline:{id}:consensus_resp`| Consumed by the agent that requested consensus            |
| `rework_requested`         | `pipeline:{id}:reviews`       | Controller sees `review_result:changes_requested` instead |
| `human_decision`           | `human_input`                 | TUI → agents directly; controller observes for state sync |

### System Events

| Event (message_type)       | Source Stream                  | Controller Action                                         |
|----------------------------|-------------------------------|-----------------------------------------------------------|
| `heartbeat`                | `status`                      | Watchdog updates last-seen timestamps                     |
| `container_exited`         | (Docker event watcher)        | Watchdog evaluates exit code → retry or escalate          |

---

## New Message Types

Six new message types introduced by the controller. All use the standard `MessageEnvelope`
from spec 10.

### `pipeline_created`

Published by the TUI/CLI when a human starts a new pipeline.

```python
class PipelineCreatedPayload(BaseModel):
    pipeline_id: str
    plan: str                            # The human's plan text
    repo_url: str
    target_branch: str
    config: AiTeamConfig                 # Parsed .ai-team.yaml
```

### `decomposition_complete`

Published by Julius when it finishes decomposing the plan into tasks.

```python
class DecompositionCompletePayload(BaseModel):
    pipeline_id: str
    task_count: int
    dependency_graph: DependencyGraph
    ready_task_ids: list[str]            # Tasks with no dependencies (can start now)
```

### `batch_dispatched`

Published by the controller when it launches a batch of Sherlocks.

```python
class BatchDispatchedPayload(BaseModel):
    pipeline_id: str
    task_ids: list[str]                  # Tasks being dispatched in this batch
    batch_number: int                    # 1-indexed
    total_tasks: int
    remaining_tasks: int
```

### `all_tasks_complete`

Published by the controller when every task in the pipeline reaches `completed`.

```python
class AllTasksCompletePayload(BaseModel):
    pipeline_id: str
    task_count: int
    total_cost: float
    elapsed_seconds: float
```

### `pipeline_completed`

Published by the controller after the PR is opened successfully.

```python
class PipelineCompletedPayload(BaseModel):
    pipeline_id: str
    pr_url: str
    pr_number: int
    tasks_completed: int
    total_cost: float
    elapsed_seconds: float
    human_review_needed: bool
```

### `pipeline_failed`

Published by the controller when the pipeline cannot continue.

```python
class PipelineFailedPayload(BaseModel):
    pipeline_id: str
    reason: str
    failed_task_id: str | None           # None if pipeline-level failure
    tasks_completed: int
    tasks_remaining: int
    total_cost: float
    recoverable: bool                    # Can human fix and retry?
```

---

## New PostgreSQL Tables

Two new tables added to the schema from spec 12.

### `pipelines`

```sql
CREATE TABLE pipelines (
    id TEXT PRIMARY KEY,                 -- "pipe-{uuid}"
    plan TEXT NOT NULL,                  -- Original plan text
    repo_url TEXT NOT NULL,
    target_branch TEXT NOT NULL,
    status TEXT NOT NULL,                -- PipelineStatus enum
    task_count INTEGER,                  -- Set after decomposition
    tasks_completed INTEGER DEFAULT 0,
    pr_url TEXT,                         -- Set when PR is opened
    total_cost REAL DEFAULT 0.0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    started_at TIMESTAMPTZ,             -- When Julius was launched
    completed_at TIMESTAMPTZ,
    error TEXT                           -- Failure reason if failed
);

CREATE INDEX idx_pipelines_status ON pipelines(status);
```

### `active_containers`

```sql
CREATE TABLE active_containers (
    container_id TEXT PRIMARY KEY,
    agent TEXT NOT NULL,                 -- AgentName
    pipeline_id TEXT NOT NULL,
    task_id TEXT,                        -- NULL for Julius (pipeline-level)
    status TEXT NOT NULL,                -- "running", "exited", "dead"
    exit_code INTEGER,                   -- Set on exit
    retry_count INTEGER DEFAULT 0,
    launched_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_heartbeat TIMESTAMPTZ,
    exited_at TIMESTAMPTZ,
    FOREIGN KEY (pipeline_id) REFERENCES pipelines(id)
);

CREATE INDEX idx_containers_pipeline ON active_containers(pipeline_id);
CREATE INDEX idx_containers_status ON active_containers(status);
```

### `PipelineStatus` enum

```python
class PipelineStatus(StrEnum):
    CREATED = "created"                  # Plan received, not yet started
    DECOMPOSING = "decomposing"          # Julius is running
    IN_PROGRESS = "in_progress"          # Tasks are being processed
    COMPLETING = "completing"            # All tasks done, opening PR
    COMPLETED = "completed"              # PR opened
    FAILED = "failed"                    # Unrecoverable failure
    CANCELLED = "cancelled"              # Cancelled by human
```

---

## Docker Compose Changes

The controller is the sole orchestrator. Only infrastructure services are defined in
`docker-compose.yml`. **All agents are ephemeral** — spawned by the controller via
the Launcher.

### Always-running services (in `docker-compose.yml`)

| Service      | Why always-running                                                |
|--------------|-------------------------------------------------------------------|
| `redis`      | Backbone — everything flows through it                            |
| `postgres`   | Durable state store for pipelines, containers, and tasks          |
| `loki`       | Log aggregation backend for all services                          |
| `grafana`    | Observability dashboards (connects to Loki)                       |
| `controller` | Orchestrates the entire pipeline lifecycle                        |

### Ephemeral containers (launched by controller via Launcher)

| Agent        | Launched when                          | Parallelism              | Exits when                          |
|--------------|----------------------------------------|--------------------------|-------------------------------------|
| `julius`     | `pipeline_created` event               | 1 per pipeline           | Publishes `decomposition_complete`  |
| `sherlock`   | Task becomes `ready` (per task)        | Up to max_parallel       | Publishes `task_enriched`           |
| `leonard`    | Task is enriched + worktree ready      | Up to max_parallel       | Publishes `implementation_complete` |
| `katherine`  | Implementation complete (per task)     | Up to max_parallel       | Publishes `review_result`           |
| `nelson`     | `consensus_request` on stream          | Up to max_parallel       | Publishes `consensus_response`      |
| `richelieu`  | `git_request` on stream                | 1 per branch (serialized)| Publishes `git_response`            |

### Updated `docker-compose.yml`

```yaml
# Only infrastructure + controller. No agents.

x-logging: &default-logging
  driver: loki
  options:
    loki-url: "http://localhost:3100/loki/api/v1/push"
    loki-retries: "2"
    loki-batch-size: "400"

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
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: ai_team
      POSTGRES_PASSWORD: ai_team
      POSTGRES_DB: ai_team
    volumes:
      - ai-team-postgres:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ai_team"]
      interval: 5s
    logging: *default-logging

  loki:
    image: grafana/loki:2.9.0
    ports:
      - "3100:3100"
    volumes:
      - ai-team-loki:/loki
    command: -config.file=/etc/loki/local-config.yaml

  grafana:
    image: grafana/grafana:10.2.0
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    volumes:
      - ai-team-grafana:/var/lib/grafana
    depends_on:
      - loki
    logging: *default-logging

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

  # ALL agents (Nelson, Julius, Sherlock, Leonard, Katherine, Richelieu)
  # are ephemeral containers launched by the controller on demand.
  # See "Ephemeral containers" table above.

volumes:
  ai-team-redis:
  ai-team-postgres:
  ai-team-loki:
  ai-team-grafana:
  ai-team-repo:
  ai-team-worktrees:
  ai-team-logs:

networks:
  default:
    name: ai-team-net
```

> **Docker socket access**: The controller mounts `/var/run/docker.sock` to manage
> containers. This is a privileged operation — see spec 09 (security) for mitigations.
> In multi-machine deployments, the Docker SDK is replaced by a Kubernetes or ECS
> launcher backend (see spec 01 for the Launcher abstraction).

---

## Failure Scenarios & Recovery

### Scenario 1: Ephemeral agent crashes mid-task

```
Watchdog detects: no heartbeat for 90s OR container exited non-zero
    │
    ▼
Check retry count for (agent, task_id)
    │
    ├── retries < 3 → Relaunch container from latest checkpoint
    │                  (checkpoint data is passed via env/Redis)
    │
    └── retries >= 3 → Mark task as "needs_human"
                        If task blocks others → mark dependents as "blocked"
                        Emit status event for TUI
```

### Scenario 2: Nelson crashes mid-consensus

```
Watchdog detects: Nelson container exited non-zero OR heartbeat gap > 90s
    │
    ▼
Same as Scenario 1: check retry count
    │
    ├── retries < 3 → Respawn Nelson with same consensus_request
    │
    └── retries >= 3 → Return consensus_response with status="error"
                        The requesting agent handles the error
                        (typically: escalate to human)
```

### Scenario 3: Richelieu crashes mid-operation

```
Watchdog detects: Richelieu container exited non-zero OR heartbeat gap > 90s
    │
    ▼
Same as Scenario 1: check retry count
    │
    ├── retries < 3 → Respawn Richelieu with same git_request
    │                  Git operations are idempotent (create branch
    │                  that exists → no-op, etc.)
    │
    └── retries >= 3 → Mark task as "needs_human"
                        Preserve branches for manual intervention
```

> Since all agents are now ephemeral, Nelson and Richelieu follow the exact same
> retry/escalation logic as Julius, Sherlock, Leonard, and Katherine. Pending
> requests queue up in Redis (durable) while the controller respawns.

### Scenario 4: Merge conflict on branch_merged

```
Richelieu reports: git_response with success=false, conflicts=[...]
    │
    ▼
Controller checks: is conflict auto-resolvable?
    │
    ├── YES → Richelieu retries with auto-resolve
    │
    └── NO → Mark task as "needs_human"
             Preserve branches for manual resolution
             Emit status event with conflicting files
```

### Scenario 5: Budget exceeded during pipeline

```
Cost tracking (spec 07) fires: hard limit reached for task
    │
    ▼
Controller receives budget_exceeded event on status stream
    │
    ▼
Stop the container running that task
Mark task as "needs_human" with reason "budget_exceeded"
Other tasks continue if they have independent budgets
```

### Scenario 6: All LLM providers down

```
Nelson reports: consensus_response with status="error", all providers failed
    │
    ▼
Controller increments pipeline error counter
    │
    ├── < threshold → Wait, affected agents will retry via their own backoff
    │
    └── >= threshold → Pause pipeline (stop launching new containers)
                        Alert human
                        Resume when human confirms providers are back
```

---

## Configuration

```python
# controller/src/controller/config.py

class ControllerConfig(BaseSettings):
    """Controller configuration, loaded from environment."""

    # Redis
    redis_url: str = "redis://redis:6379"
    consumer_group: str = "controller"
    consumer_name: str = "controller-1"
    stream_block_ms: int = 5000

    # Parallelism (per agent type)
    max_parallel_nelsons: int = 5
    max_parallel_sherlocks: int = 5
    max_parallel_leonards: int = 3
    max_parallel_katherines: int = 3
    max_parallel_richelieus: int = 3       # Serialized per-branch, parallel across branches

    # Watchdog
    watchdog_interval_seconds: int = 15
    heartbeat_timeout_seconds: int = 90
    container_deadline_minutes: int = 30   # Per-container max runtime

    # Retries (per task per agent)
    max_retries: int = 3

    # Docker
    docker_network: str = "ai-team-net"
    image_prefix: str = "ai-team"          # Images: ai-team-julius, ai-team-sherlock, etc.
    image_tag: str = "latest"

    # Database
    database_url: str = "postgresql://ai_team:ai_team@postgres:5432/ai_team"

    # Pipeline error threshold (consecutive errors before pausing)
    pipeline_error_threshold: int = 5

    model_config = SettingsConfigDict(env_prefix="CONTROLLER_")
```

---

## Complete Event Flow: End-to-End Pipeline

A 20-step walkthrough of a typical pipeline from plan submission to PR.

```
 1. Human submits plan via TUI/CLI
    → TUI publishes `pipeline_created` to `system:pipelines`

 2. Controller receives `pipeline_created`
    → Creates pipeline record in PostgreSQL (status: CREATED)
    → Spawns Richelieu to create feature branch
    → Richelieu publishes `git_response` with branch name, exits

 3. Controller receives `git_response` (feature branch ready)
    → Updates pipeline status → DECOMPOSING
    → Spawns Julius container (PIPELINE_ID env var)

 4. Julius analyzes codebase, decomposes plan into tasks
    → Publishes `consensus_request` to `pipeline:{id}:consensus`
    → Controller spawns Nelson for validation
    → Nelson publishes `consensus_response`, exits
    → Julius publishes all `task_created` messages to `pipeline:{id}:tasks`
    → Julius publishes `decomposition_complete` to `pipeline:{id}:tasks`
    → Julius container exits

 5. Controller receives `decomposition_complete`
    → Stores DependencyGraph in PostgreSQL
    → Updates pipeline status → IN_PROGRESS
    → Calls `graph.get_ready_tasks()` to find tasks with no dependencies
    → Spawns Sherlock containers for ready tasks (up to max_parallel)
    → Publishes `batch_dispatched` to `pipeline:{id}:status`

 6. Sherlock (per task) inspects codebase, produces execution plan
    → May publish `consensus_request` → controller spawns Nelson → Nelson responds, exits
    → Publishes `task_enriched` to `pipeline:{id}:enriched`
    → Container exits

 7. Controller receives `task_enriched`
    → Updates task status → READY
    → Spawns Richelieu to create worktree
    → Richelieu publishes `git_response` with worktree path and branch name, exits

 8. Controller receives `git_response` (worktree ready)
    → Spawns Leonard container with TASK_ID, worktree path, branch name

 9. Leonard implements code following Sherlock's execution plan
    → Runs tests, lint, format (from .ai-team.yaml commands)
    → Commits to task branch
    → Publishes `implementation_complete` to `pipeline:{id}:reviews`
    → Container exits

10. Controller receives `implementation_complete`
    → Updates task status → IN_REVIEW
    → Spawns Katherine container with TASK_ID

11. Katherine reviews the diff using Nelson consensus
    → Publishes `consensus_request` → controller spawns Nelson → Nelson responds, exits
    → Publishes `review_result` to `pipeline:{id}:reviews`
    → Container exits

12. Controller receives `review_result`
    ├── decision: "approved"
    │   → Publishes `git_request:merge_branch` to `pipeline:{id}:git_ops`
    │
    ├── decision: "changes_requested"
    │   → Updates task status → REWORK
    │   → Respawns Leonard with feedback attached
    │   → Go to step 9
    │
    └── decision: "escalated"
        → Updates task status → NEEDS_HUMAN
        → Publishes escalation to `human_input`

13. Controller receives `git_request:merge_branch`
    → Spawns Richelieu to merge task branch → feature branch
    → Richelieu publishes `git_response` with merge result, exits

14. Controller receives `git_response` (merge success)
    → Updates task status → COMPLETED
    → Spawns Richelieu to remove worktree (`git_request:remove_worktree`)

15. Controller evaluates dependency graph
    → Calls `graph.get_ready_tasks()` with updated statuses
    ├── More tasks ready → Spawn Sherlocks (go to step 6)
    └── No tasks ready + all tasks complete → Continue to step 16

16. Controller detects all tasks completed
    → Publishes `all_tasks_complete` to `pipeline:{id}:status`

17. Controller transitions pipeline → COMPLETING
    → Publishes `git_request:open_pr` to `pipeline:{id}:git_ops`

18. Controller spawns Richelieu to open PR
    → Richelieu opens PR from feature branch → target branch
    → Includes task summaries, review scores, cost summary in PR body
    → Publishes `git_response` with PR URL and number
    → Container exits

19. Controller receives `git_response` (PR opened)
    → Updates pipeline status → COMPLETED
    → Publishes `pipeline_completed` to `system:pipelines`

20. Human reviews PR (if Katherine flagged it) or it auto-merges
    → TUI shows pipeline summary with link to PR
```

---

## Testing Approach

### Unit Tests

| Component     | What to test                                                   |
|---------------|----------------------------------------------------------------|
| Router        | Correct handler dispatch for each message type; unknown types go to dead letter; XACK on success |
| Launcher      | Container creation params (image, network, volumes, env); mock Docker SDK |
| Watchdog      | Heartbeat timeout detection; exit code handling; retry vs. escalate logic |
| State Tracker | Pipeline CRUD; container lifecycle; dependency graph evaluation |

### Integration Tests

| Scenario                        | What to verify                                          |
|---------------------------------|---------------------------------------------------------|
| Happy path (3 independent tasks)| All 20 steps complete; pipeline status transitions are correct |
| Task with dependencies          | Dependent tasks launch only after blockers complete     |
| Agent crash + retry             | Watchdog detects crash; container relaunched; resumes from checkpoint |
| Max retries exceeded            | Task marked `needs_human`; dependents blocked; pipeline continues for independent tasks |
| Merge conflict                  | Controller handles Richelieu's conflict report correctly |

### E2E Test

One end-to-end test with real containers (but mocked LLM responses via VCR cassettes
from spec 11):

1. Submit a plan with 3 tasks (2 parallel + 1 dependent).
2. Verify Julius decomposes correctly.
3. Verify Sherlocks launch for the 2 parallel tasks.
4. Verify Leonards implement and Katherines review.
5. Verify the dependent task launches after its blocker completes.
6. Verify PR is opened with correct content.

---

## Build Order

The controller is **Phase 1** — built immediately after the foundation (Phase 0) and
before any agents. It needs core infrastructure (Redis, PostgreSQL, Pydantic models) but
does not need any agent to exist yet. The router handlers reference message types that
agents will eventually produce, but can be unit-tested with mock messages from day one.

### Phase 1 — Controller (Complete)

| Step | Component                                         | Depends on                     |
|------|---------------------------------------------------|--------------------------------|
| 1.1  | `controller/config.py`                            | —                              |
| 1.2  | `controller/state.py` + migrations                | PostgreSQL schema (spec 12)    |
| 1.3  | `controller/launcher.py`                          | Docker SDK                     |
| 1.4  | `controller/watchdog.py`                          | Heartbeat model (spec 08)      |
| 1.5  | `controller/router.py` — full dispatch table      | 1.2–1.4                        |
| 1.6  | `controller/main.py` — entrypoint                 | 1.1–1.5                        |
| 1.7  | Docker Compose with controller service            | 1.6                            |
| 1.8  | Unit tests (mock Docker SDK, mock Redis)           | 1.1–1.6                        |
| 1.9  | Integration tests (real Redis, mock containers)    | 1.7                            |

The controller is fully functional after Phase 1 — it can receive events and launch
containers. As each agent is built in Phases 2–7, it immediately plugs into the
controller's existing routing table.

### Phase 8 — E2E Verification

| Step | Component                                         | Depends on                     |
|------|---------------------------------------------------|--------------------------------|
| 8.1  | E2E test (real containers, VCR cassettes)          | All agents built (Phases 2–7)  |

---

## Relationship to Other Specs

| Spec | Relationship                                                                |
|------|-----------------------------------------------------------------------------|
| 01   | Controller replaces the implicit orchestration. Updates Docker Compose layout (only infra + controller always-running; all agents are ephemeral). Launcher abstraction enables multi-machine scaling. |
| 08   | Controller implements the watchdog described in 08 (previously "part of Richelieu or a dedicated sidecar"). Checkpoint/retry logic is per-agent; the controller manages the retry counter and relaunch decision. |
| 10   | Controller consumes all message types from 10. Adds 6 new types. The routing table is the definitive "who reacts to what" reference. |
| 12   | Adds `PipelineStatus` enum, `PipelineCreatedPayload` and 5 other payload models to `models/messages.py`. Adds `pipelines` and `active_containers` tables to the PostgreSQL schema. |
