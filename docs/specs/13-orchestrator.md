# 13 — Orchestrator (Pipeline Orchestrator)

> **Migrated from**: `docs/specs/13-orchestrator.md` (Overview, Design Principles, Project Structure, Core Components, Routing Table, Message Types, PostgreSQL Tables, Failure Scenarios, Configuration, Testing, Build Order) and `docs/specs/01-deployment.md` (Launcher Protocol, Logging for Ephemeral Agents, Secret Scoping, Orchestrator Health & Recovery)

## Overview

The **orchestrator** is a deterministic Python service (no LLM) that owns the end-to-end
pipeline flow. It watches Redis Streams for lifecycle events and launches/monitors agent
containers via the Docker SDK. It is the answer to "who is the conductor?"

The orchestrator is **not** an agent — it has no PydanticAI dependency, makes no LLM calls,
and contains no prompts. It is a lightweight event router + container launcher (~hundreds
of lines of code).

---

## Design Principles

1. **No LLM calls** — the orchestrator is pure orchestration logic.
2. **Single writer for pipeline state** — only the orchestrator transitions pipeline status.
3. **All agents are ephemeral** — every agent (Nelson, Julius, Sherlock, Leonard,
   Katherine, Richelieu) runs as a one-shot container. The orchestrator launches them,
   monitors them, and cleans up after them. No agents are defined in `docker-compose.yml`.
4. **Horizontal scaling via spawning** — when multiple consensus requests or git
   operations arrive, the orchestrator spawns multiple Nelson or Richelieu instances.
   Richelieu operations on the same branch are serialized; everything else runs in parallel.
5. **The dependency graph lives here** — on each `branch_merged` event the orchestrator
   evaluates `DependencyGraph.get_ready_tasks()` directly. No need to spin up Julius
   just to read state.
6. **Pluggable container runtime** — the Launcher has a backend interface (Docker SDK,
   Kubernetes Jobs, ECS Tasks) so the system scales from a single machine to a cluster
   with a config change.

---

## Project Structure

```
orchestrator/
├── pyproject.toml
├── Dockerfile
└── src/
    └── orchestrator/
        ├── __init__.py
        ├── main.py              # Entrypoint: connect to Redis, start router loop
        ├── router.py            # Event → handler dispatch table
        ├── launcher.py          # Docker SDK: run/stop/inspect containers
        ├── watchdog.py          # Heartbeat monitoring, dead container detection
        ├── state.py             # Pipeline + container state (PostgreSQL)
        └── config.py            # OrchestratorConfig (Pydantic settings)
```

The orchestrator is a **uv workspace member** alongside `core/` and each agent package.
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
                group="orchestrator",
                consumer="orchestrator-1",
                streams=self.watched_streams(),
                block=5000,  # ms
            )
            for stream, entries in messages:
                for msg_id, data in entries:
                    envelope = MessageEnvelope.model_validate_json(data[b"payload"])
                    handler = self.handlers[stream][envelope.message_type]
                    await handler(envelope)
                    await self.redis.xack(stream, "orchestrator", msg_id)
```

### 2. Launcher (`launcher.py`)

Manages ephemeral agent containers via the Launcher Protocol (defined below).
The orchestrator uses the Launcher to create, monitor, and clean up agent containers.
See spec 05 for the Docker Compose infrastructure that these containers run on.

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

## Launcher Protocol

> **Migrated from**: `docs/specs/01-deployment.md` — Launcher Protocol section

The orchestrator's `Launcher` class is the key abstraction for container management.
It has a pluggable backend to support different container runtimes.

**This is the canonical definition.**

```python
class Launcher(Protocol):
    """Interface for launching agent containers on any runtime."""

    async def launch(
        self,
        agent: AgentName,
        pipeline_id: str,
        task_id: str | None = None,
        env: dict[str, str] | None = None,
    ) -> str:
        """Launch an agent container. Returns container ID.

        The Launcher is responsible for:
        - Selecting the correct image (ai-team-{agent}:latest)
        - Applying the agent's resource limits (defaults + .ai-team.yaml overrides)
        - Filtering secrets via the agent's allowlist
        - Configuring the Loki logging driver
        - Connecting to the ai-team-net network
        - Setting container labels for identification
        """
        ...

    async def stop(self, container_id: str, timeout: int = 30) -> None:
        """Send SIGTERM, wait up to timeout, then SIGKILL."""
        ...

    async def inspect(self, container_id: str) -> ContainerStatus:
        """Get current container state (running, exited, etc.)."""
        ...

    async def wait(self, container_id: str) -> int:
        """Block until container exits. Returns exit code."""
        ...

    async def remove(self, container_id: str) -> None:
        """Remove container and its anonymous volumes."""
        ...
```

### Container creation parameters

| Parameter         | Value                                                        |
|-------------------|--------------------------------------------------------------|
| Image             | `ai-team-{agent}:latest` (built by `make build`)            |
| Network           | `ai-team-net`                                                |
| Volumes           | Per agent filesystem access rules (see spec 05 Volumes section) |
| Environment       | Filtered secrets (see Secret Scoping below) + `PIPELINE_ID` + `TASK_ID` + agent-specific vars |
| Name              | `ai-team-{agent}-{pipeline_id[:8]}-{task_id[:8]}`           |
| Restart policy    | `no` (the orchestrator handles retries)                        |
| Labels            | `ai-team.agent={agent}`, `ai-team.pipeline={pipeline_id}`, `ai-team.task={task_id}` |
| Resource limits   | Per-agent defaults merged with `.ai-team.yaml` overrides (see spec 05) |
| Log config        | Loki logging driver (see Logging for Ephemeral Agents below) |

### Launcher backends

| Backend              | Runtime           | Use case                      |
|----------------------|-------------------|-------------------------------|
| `DockerLauncher`     | Docker SDK        | Local dev (single machine)    |
| `KubernetesLauncher` | k8s Jobs API     | Production cluster            |
| `ECSLauncher`        | ECS RunTask API   | AWS production                |

The backend is selected via configuration. Agent images are pushed to a container
registry (ECR, GCR, GHCR) and referenced by the launcher.

---

## Logging for Ephemeral Agents

> **Migrated from**: `docs/specs/01-deployment.md` — Logging for Ephemeral Agents section

Ephemeral containers spawned by the Launcher do **not** inherit Docker Compose's
logging configuration. The Launcher must explicitly configure the Loki logging
driver on each `container.run()` call.

### Docker SDK logging configuration

```python
# In DockerLauncher.launch()
log_config = docker.types.LogConfig(
    type="loki",
    config={
        "loki-url": "http://loki:3100/loki/api/v1/push",
        "loki-retries": "2",
        "loki-batch-size": "400",
        "labels": f"service={agent},pipeline={pipeline_id},task={task_id}",
    },
)

container = client.containers.run(
    image=f"ai-team-{agent}:{tag}",
    log_config=log_config,
    # ... other params
)
```

### Label strategy

Every ephemeral container's logs are tagged with:
- `service` — the agent name (e.g., `leonard`, `nelson`)
- `pipeline` — the pipeline ID
- `task` — the task ID (empty for pipeline-level agents like Julius)

This allows Grafana/LogQL queries like:
```logql
{service="leonard", pipeline="pipe-abc123"} |= "test failed"
```

---

## Secret Scoping

> **Migrated from**: `docs/specs/01-deployment.md` — Secret Scoping section

Not all agents need all secrets. The Launcher filters environment variables
based on a per-agent allowlist before spawning containers.

### Default allowlists

| Agent      | Env vars received                                               |
|------------|----------------------------------------------------------------|
| Nelson     | `OPENROUTER_API_KEY`, `REDIS_URL`, `DATABASE_URL`             |
| Julius     | `OPENROUTER_API_KEY`, `REDIS_URL`, `DATABASE_URL`             |
| Sherlock   | `OPENROUTER_API_KEY`, `REDIS_URL`, `DATABASE_URL`             |
| Leonard    | `OPENROUTER_API_KEY`, `REDIS_URL`, `DATABASE_URL`             |
| Katherine  | `OPENROUTER_API_KEY`, `REDIS_URL`, `DATABASE_URL`             |
| Richelieu  | `GITHUB_PAT`, `GITHUB_APP_ID`, `GITHUB_APP_PRIVATE_KEY_PATH`, `REDIS_URL`, `DATABASE_URL` |

All agents also receive: `PIPELINE_ID`, `TASK_ID`, `LOG_LEVEL`.

No agent receives secrets it doesn't need. In particular:
- Only Richelieu gets GitHub credentials.
- If an agent doesn't call LLMs directly (communicates with Nelson via Redis
  instead), it can be removed from the `OPENROUTER_API_KEY` list.

### `.ai-team.yaml` overrides

The target repo owner can extend (but not reduce) the allowlists:

```yaml
# .ai-team.yaml
secrets:
  leonard:
    additional:
      - CUSTOM_API_KEY       # Leonard needs this for the target repo's test suite
```

### Implementation

```python
# In DockerLauncher
AGENT_SECRET_ALLOWLISTS: dict[AgentName, list[str]] = {
    "nelson": ["OPENROUTER_API_KEY", "REDIS_URL", "DATABASE_URL"],
    "julius": ["OPENROUTER_API_KEY", "REDIS_URL", "DATABASE_URL"],
    "sherlock": ["OPENROUTER_API_KEY", "REDIS_URL", "DATABASE_URL"],
    "leonard": ["OPENROUTER_API_KEY", "REDIS_URL", "DATABASE_URL"],
    "katherine": ["OPENROUTER_API_KEY", "REDIS_URL", "DATABASE_URL"],
    "richelieu": [
        "GITHUB_PAT", "GITHUB_APP_ID", "GITHUB_APP_PRIVATE_KEY_PATH",
        "REDIS_URL", "DATABASE_URL",
    ],
}

ALWAYS_PASSED = ["PIPELINE_ID", "TASK_ID", "LOG_LEVEL"]

def filter_env(self, agent: AgentName, full_env: dict) -> dict:
    allowed = set(AGENT_SECRET_ALLOWLISTS[agent] + ALWAYS_PASSED)
    allowed |= set(self.yaml_config.secrets.get(agent, {}).get("additional", []))
    return {k: v for k, v in full_env.items() if k in allowed}
```

---

## Event → Container Routing Table

The orchestrator's full dispatch table. Every message type from spec 04 is accounted for.

### Pipeline Events (orchestrator acts on)

| Event (message_type)       | Source Stream                  | Orchestrator Action                                         |
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

### Events the Orchestrator Observes (but doesn't act on)

| Event (message_type)       | Source Stream                  | Notes                                                     |
|----------------------------|-------------------------------|-----------------------------------------------------------|
| `git_response`             | `pipeline:{id}:git_ops`       | Consumed by orchestrator to track Richelieu results         |
| `consensus_response`       | `pipeline:{id}:consensus_resp`| Consumed by the agent that requested consensus            |
| `rework_requested`         | `pipeline:{id}:reviews`       | Orchestrator sees `review_result:changes_requested` instead |
| `human_decision`           | `human_input`                 | TUI → agents directly; orchestrator observes for state sync |

### System Events

| Event (message_type)       | Source Stream                  | Orchestrator Action                                         |
|----------------------------|-------------------------------|-----------------------------------------------------------|
| `heartbeat`                | `status`                      | Watchdog updates last-seen timestamps                     |
| `container_exited`         | (Docker event watcher)        | Watchdog evaluates exit code → retry or escalate          |

---

## New Message Types

Six new message types introduced by the orchestrator. All use the standard `MessageEnvelope`
from spec 04.

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

Published by the orchestrator when it launches a batch of Sherlocks.

```python
class BatchDispatchedPayload(BaseModel):
    pipeline_id: str
    task_ids: list[str]                  # Tasks being dispatched in this batch
    batch_number: int                    # 1-indexed
    total_tasks: int
    remaining_tasks: int
```

### `all_tasks_complete`

Published by the orchestrator when every task in the pipeline reaches `completed`.

```python
class AllTasksCompletePayload(BaseModel):
    pipeline_id: str
    task_count: int
    total_cost: float
    elapsed_seconds: float
```

### `pipeline_completed`

Published by the orchestrator after the PR is opened successfully.

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

Published by the orchestrator when the pipeline cannot continue.

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

Two new tables added to the schema from spec 02.

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

## Complete Event Flow: End-to-End Pipeline

A 20-step walkthrough of a typical pipeline from plan submission to PR.

```mermaid
sequenceDiagram
    participant Human
    participant TUI
    participant Orch as Orchestrator
    participant Redis
    participant PG as PostgreSQL
    participant Rich as Richelieu
    participant Jul as Julius
    participant Nel as Nelson
    participant Sher as Sherlock
    participant Leo as Leonard
    participant Kat as Katherine

    Note over Human, Kat: 1-2. Plan submission & pipeline creation
    Human->>TUI: Submit plan
    TUI->>Redis: pipeline_created
    Orch->>PG: Create pipeline (CREATED)
    Orch->>Rich: Spawn (create feature branch)
    Rich->>Redis: git_response (branch name)
    Rich--xRich: exits

    Note over Human, Kat: 3-4. Decomposition
    Orch->>PG: Status -> DECOMPOSING
    Orch->>Jul: Spawn (PIPELINE_ID)
    Jul->>Redis: consensus_request
    Orch->>Nel: Spawn for validation
    Nel->>Redis: consensus_response
    Nel--xNel: exits
    Jul->>Redis: task_created (x N)
    Jul->>Redis: decomposition_complete
    Jul--xJul: exits

    Note over Human, Kat: 5-6. Dispatch & enrichment
    Orch->>PG: Store DependencyGraph, status -> IN_PROGRESS
    Orch->>Sher: Spawn for ready tasks (up to max_parallel)
    Sher->>Redis: task_enriched
    Sher--xSher: exits

    Note over Human, Kat: 7-9. Worktree setup & implementation
    Orch->>PG: Task status -> READY
    Orch->>Rich: Spawn (create worktree)
    Rich->>Redis: git_response (worktree path)
    Rich--xRich: exits
    Orch->>Leo: Spawn (TASK_ID, worktree, branch)
    Leo->>Leo: Implement, test, lint, commit
    Leo->>Redis: implementation_complete
    Leo--xLeo: exits

    Note over Human, Kat: 10-12. Review
    Orch->>PG: Task status -> IN_REVIEW
    Orch->>Kat: Spawn (TASK_ID)
    Kat->>Redis: consensus_request
    Orch->>Nel: Spawn for review
    Nel->>Redis: consensus_response
    Nel--xNel: exits
    Kat->>Redis: review_result
    Kat--xKat: exits

    alt approved
        Orch->>Redis: git_request:merge_branch
    else changes_requested
        Orch->>PG: Task status -> REWORK
        Note right of Orch: Respawn Leonard (go to step 9)
    else escalated
        Orch->>PG: Task status -> NEEDS_HUMAN
        Orch->>Redis: escalation to human_input
    end

    Note over Human, Kat: 13-15. Merge & dependency dispatch
    Orch->>Rich: Spawn (merge task -> feature branch)
    Rich->>Redis: git_response (merge result)
    Rich--xRich: exits
    Orch->>PG: Task status -> COMPLETED
    Orch->>Orch: Evaluate get_ready_tasks()
    Note right of Orch: More ready -> spawn Sherlocks (step 6)<br/>All complete -> continue to step 16

    Note over Human, Kat: 16-19. Pipeline completion
    Orch->>Redis: all_tasks_complete
    Orch->>PG: Status -> COMPLETING
    Orch->>Rich: Spawn (open PR: feature -> target)
    Rich->>Redis: git_response (PR URL)
    Rich--xRich: exits
    Orch->>PG: Status -> COMPLETED
    Orch->>Redis: pipeline_completed

    Note over Human, Kat: 20. Human review
    TUI->>Human: Pipeline summary with link to PR
```

---

## Failure Scenarios & Recovery

### Scenario 1: Ephemeral agent crashes mid-task

```mermaid
flowchart TD
    A["Watchdog detects: no heartbeat for 90s\nOR container exited non-zero"] --> B{"Check retry count\nfor (agent, task_id)"}
    B -->|"retries < 3"| C["Relaunch container from latest checkpoint\n(checkpoint data passed via env/Redis)"]
    B -->|"retries >= 3"| D["Mark task as needs_human\nIf task blocks others: mark dependents as blocked\nEmit status event for TUI"]
```

### Scenario 2: Nelson crashes mid-consensus

```mermaid
flowchart TD
    A["Watchdog detects: Nelson exited non-zero\nOR heartbeat gap > 90s"] --> B{Check retry count}
    B -->|"retries < 3"| C[Respawn Nelson with same consensus_request]
    B -->|"retries >= 3"| D["Return consensus_response with status=error\nRequesting agent handles the error\n(typically: escalate to human)"]
```

### Scenario 3: Richelieu crashes mid-operation

```mermaid
flowchart TD
    A["Watchdog detects: Richelieu exited non-zero\nOR heartbeat gap > 90s"] --> B{Check retry count}
    B -->|"retries < 3"| C["Respawn Richelieu with same git_request\n(git ops are idempotent)"]
    B -->|"retries >= 3"| D["Mark task as needs_human\nPreserve branches for manual intervention"]
```

> Since all agents are now ephemeral, Nelson and Richelieu follow the exact same
> retry/escalation logic as Julius, Sherlock, Leonard, and Katherine. Pending
> requests queue up in Redis (durable) while the orchestrator respawns.

### Scenario 4: Merge conflict on branch_merged

```mermaid
flowchart TD
    A["Richelieu reports: git_response\nsuccess=false, conflicts=[...]"] --> B{Is conflict auto-resolvable?}
    B -->|YES| C[Richelieu retries with auto-resolve]
    B -->|NO| D["Mark task as needs_human\nPreserve branches for manual resolution\nEmit status event with conflicting files"]
```

### Scenario 5: Budget exceeded during pipeline

```mermaid
flowchart TD
    A["Cost tracking (spec 07) fires:\nhard limit reached for task"] --> B[Orchestrator receives budget_exceeded\nevent on status stream]
    B --> C["Stop container running that task"]
    C --> D["Mark task as needs_human\nreason: budget_exceeded"]
    D --> E["Other tasks continue if they\nhave independent budgets"]
```

### Scenario 6: All LLM providers down

```mermaid
flowchart TD
    A["Nelson reports: consensus_response\nstatus=error, all providers failed"] --> B[Orchestrator increments\npipeline error counter]
    B --> C{"Error count vs threshold"}
    C -->|"< threshold"| D["Wait, affected agents retry\nvia their own backoff"]
    C -->|">= threshold"| E["Pause pipeline\n(stop launching new containers)\nAlert human\nResume when human confirms providers are back"]
```

---

## Orchestrator Health & Recovery

> **Migrated from**: `docs/specs/01-deployment.md` — Orchestrator Health & Recovery section

### Heartbeat

The orchestrator writes a heartbeat to Redis every 10 seconds:

```python
async def heartbeat_loop(redis: Redis) -> None:
    while True:
        await redis.set("orchestrator:heartbeat", str(time.time()), ex=30)
        await asyncio.sleep(10)
```

Docker Compose's health check verifies this key exists. If the orchestrator hangs
(event loop frozen), the heartbeat expires after 30 seconds, Docker marks the
container unhealthy, and `restart: unless-stopped` triggers a restart.

### Crash recovery

The architecture provides crash recovery through existing infrastructure:

1. **Redis Streams + XACK**: Messages read but not acknowledged remain in the
   Pending Entries List (PEL). On restart, `XREADGROUP` automatically returns
   unacknowledged messages. No event is ever lost.

2. **PostgreSQL state**: Pipeline and container state survives orchestrator
   restarts. The orchestrator reads its last known state from PostgreSQL on startup.

3. **Docker labels**: All agent containers are labeled with `ai-team.*` labels.
   The orchestrator can discover running containers via Docker API queries.

### Handler idempotency

Since events may be re-processed after a crash, **all event handlers must be
idempotent**:

| Handler                 | Idempotency rule                                              |
|-------------------------|---------------------------------------------------------------|
| `launch_agent`          | Check if agent already running for (pipeline, task) before launching |
| `update_pipeline_status`| Conditional update: `WHERE status = {expected_current_status}` |
| `open_pr`               | Check if PR already exists for the branch before creating     |
| `merge_branch`          | Check if branch is already merged before merging              |
| `track_container`       | Upsert by container_id (INSERT ON CONFLICT UPDATE)            |

### Startup reconciliation

On startup, the orchestrator reconciles its state before entering the event loop:

```mermaid
flowchart TD
    A["1. Connect to PostgreSQL\nLoad pipeline and container state"] --> B["2. List Docker containers\nwith ai-team.* labels"]
    B --> C{"3. For each container"}
    C -->|"In DB + running"| D["Adopt\n(update last_heartbeat)"]
    C -->|"In DB + stopped"| E["Check exit code\nProcess result if not already processed"]
    C -->|"Not in DB + running"| F["Stop and remove\n(orphan from previous crash)"]
    D & E & F --> G["4. Process pending Redis entries\n(automatic via XREADGROUP with PEL)"]
    G --> H["5. For PAUSED pipelines:\ncheck interrupted tasks,\nresume from checkpoint"]
    H --> I["6. Start normal event loop\n+ heartbeat loop"]
```

### Graceful SIGTERM (nice-to-have)

On `make down` or intentional shutdown:
1. Stop reading new events from Redis
2. Wait for current handler to finish
3. Exit cleanly

This is an optimization — the system handles hard crashes correctly via the
mechanisms above. Graceful shutdown simply avoids wasting in-flight agent work.

---

## Configuration

```python
# orchestrator/src/orchestrator/config.py

class OrchestratorConfig(BaseSettings):
    """Orchestrator configuration, loaded from environment."""

    # Redis
    redis_url: str = "redis://redis:6379"
    consumer_group: str = "orchestrator"
    consumer_name: str = "orchestrator-1"
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

## Docker Compose Changes

The orchestrator is the sole orchestrator. Only infrastructure services are defined in
`docker-compose.yml` (see spec 05). **All agents are ephemeral** — spawned by the
orchestrator via the Launcher.

### Always-running services (in `docker-compose.yml`)

| Service      | Why always-running                                                |
|--------------|-------------------------------------------------------------------|
| `redis`      | Backbone — everything flows through it                            |
| `postgres`   | Durable state store for pipelines, containers, and tasks          |
| `loki`       | Log aggregation backend for all services                          |
| `grafana`    | Observability dashboards (connects to Loki)                       |
| `orchestrator` | Orchestrates the entire pipeline lifecycle                        |

### Ephemeral containers (launched by orchestrator via Launcher)

| Agent        | Launched when                          | Parallelism              | Exits when                          |
|--------------|----------------------------------------|--------------------------|-------------------------------------|
| `julius`     | `pipeline_created` event               | 1 per pipeline           | Publishes `decomposition_complete`  |
| `sherlock`   | Task becomes `ready` (per task)        | Up to max_parallel       | Publishes `task_enriched`           |
| `leonard`    | Task is enriched + worktree ready      | Up to max_parallel       | Publishes `implementation_complete` |
| `katherine`  | Implementation complete (per task)     | Up to max_parallel       | Publishes `review_result`           |
| `nelson`     | `consensus_request` on stream          | Up to max_parallel       | Publishes `consensus_response`      |
| `richelieu`  | `git_request` on stream                | 1 per branch (serialized)| Publishes `git_response`            |

> **Docker socket access**: The orchestrator connects to the Docker Socket Proxy
> (see spec 05) to manage containers. In multi-machine deployments, the Docker
> SDK is replaced by a Kubernetes or ECS launcher backend (see Launcher Protocol above).

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
| Merge conflict                  | Orchestrator handles Richelieu's conflict report correctly |

### E2E Test

One end-to-end test with real containers (but mocked LLM responses via VCR cassettes
from spec 10):

1. Submit a plan with 3 tasks (2 parallel + 1 dependent).
2. Verify Julius decomposes correctly.
3. Verify Sherlocks launch for the 2 parallel tasks.
4. Verify Leonards implement and Katherines review.
5. Verify the dependent task launches after its blocker completes.
6. Verify PR is opened with correct content.

---

## Build Order

The orchestrator is **Phase 1** — built immediately after the foundation (Phase 0) and
before any agents. It needs core infrastructure (Redis, PostgreSQL, Pydantic models) but
does not need any agent to exist yet. The router handlers reference message types that
agents will eventually produce, but can be unit-tested with mock messages from day one.

### Phase 1 — Orchestrator (Complete)

| Step | Component                                         | Depends on                     |
|------|---------------------------------------------------|--------------------------------|
| 1.1  | `orchestrator/config.py`                            | —                              |
| 1.2  | `orchestrator/state.py` + migrations                | PostgreSQL schema (spec 02)    |
| 1.3  | `orchestrator/launcher.py`                          | Docker SDK                     |
| 1.4  | `orchestrator/watchdog.py`                          | Heartbeat model (spec 08)      |
| 1.5  | `orchestrator/router.py` — full dispatch table      | 1.2–1.4                        |
| 1.6  | `orchestrator/main.py` — entrypoint                 | 1.1–1.5                        |
| 1.7  | Docker Compose with orchestrator service            | 1.6                            |
| 1.8  | Unit tests (mock Docker SDK, mock Redis)           | 1.1–1.6                        |
| 1.9  | Integration tests (real Redis, mock containers)    | 1.7                            |

The orchestrator is fully functional after Phase 1 — it can receive events and launch
containers. As each agent is built in Phases 2–7, it immediately plugs into the
orchestrator's existing routing table.

### Phase 8 — E2E Verification

| Step | Component                                         | Depends on                     |
|------|---------------------------------------------------|--------------------------------|
| 8.1  | E2E test (real containers, VCR cassettes)          | All agents built (Phases 2–7)  |

---

## Relationship to Other Specs

| Spec | Relationship                                                                |
|------|-----------------------------------------------------------------------------|
| 05   | Infrastructure spec defines Docker Compose layout, volumes, networking, image builds, resource limits. The orchestrator runs on top of this infrastructure. |
| 14   | API server provides the external interface; orchestrator handles internal orchestration. |
| 15   | TUI consumes pipeline events that the orchestrator publishes. |
| 04   | Orchestrator consumes all message types from spec 04. Adds 6 new types. The routing table is the definitive "who reacts to what" reference. |
| 01   | Adds `PipelineStatus` enum, `PipelineCreatedPayload` and 5 other payload models to `models/messages.py`. Adds `pipelines` and `active_containers` tables to the PostgreSQL schema. |
