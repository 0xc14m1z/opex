# 08 — Error Recovery

> **Migrated from**: `docs/specs/08-error-recovery.md`

## Overview

Agents crash. LLM APIs go down. Tests flake. The system is designed to handle all of
these gracefully: checkpoint progress, restart from where it left off, retry with
configurable limits, and always escalate to a human on failure.

> **Design principle (P8 — No Silent Failures):** The system never gives up
> autonomously. All failures escalate to `NEEDS_HUMAN`. There is no automatic
> terminal `FAILED` state. See spec 01 for the full task lifecycle and human
> intervention flow.
>
> **Two levels of recovery:**
> 1. **Agent-level** (this spec): Within a single task attempt, agent crashes
>    are recovered via checkpoints. This handles OOM, container restarts, and
>    transient failures mid-execution.
> 2. **Task-level** (spec 01): When an entire attempt fails (agent completes
>    but produces bad output, or agent-level retries are exhausted), the task
>    is retried from scratch with a new attempt. After `max_task_retries`
>    attempts, the task escalates to `NEEDS_HUMAN`.

> **Note**: The orchestrator (spec 13) implements handler idempotency and startup
> reconciliation to ensure that events are never lost or double-processed after a
> crash. All event handlers in the orchestrator must be idempotent — see spec 13 for
> the idempotency rules table and startup reconciliation sequence.

---

## Recovery Strategy

```
Agent encounters error
        │
        ▼
┌─────────────────────┐
│ Save checkpoint     │  (current state persisted to PostgreSQL)
│ Log error           │  (full context including traceback)
│ Notify human (TUI)  │  (banner: "Leonard-1 crashed on task #3")
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│ Retry count < 3?    │
│                     │
│ YES → Restart from  │──── Resume from last checkpoint
│       checkpoint    │     (not from scratch)
│                     │
│ NO  → Escalate to   │──── Task attempt marked as "failed"
│       task-level     │     Orchestrator evaluates task-level retries
│       retry (spec 01)│     (see spec 01 — Failure Handling)
└─────────────────────┘
```

## Checkpoints

### What Gets Checkpointed

Each agent defines its own checkpoints at natural boundaries:

**Julius** (Task Decomposition):
| Checkpoint | State Saved |
|---|---|
| `plan_parsed` | Parsed plan from GitHub issue |
| `codebase_analyzed` | High-level codebase analysis result |
| `tasks_drafted` | Task list before validation |
| `tasks_validated` | Nelson-validated task list + dependency graph |

**Sherlock** (Task Enrichment):
| Checkpoint | State Saved |
|---|---|
| `files_identified` | Relevant files for this task |
| `patterns_extracted` | Coding patterns from the codebase |
| `plan_drafted` | Mini execution plan before validation |
| `plan_finalized` | Final execution plan |

**Leonard** (Implementation):
| Checkpoint | State Saved |
|---|---|
| `plan_received` | Execution plan from Sherlock |
| `code_generated` | Generated code (before testing) |
| `tests_passed` | Code + tests all passing |
| `lint_passed` | Code passing lint/format |
| `validation_done` | Acceptance criteria validated |
| `committed` | Changes committed to branch |

**Katherine** (Review):
| Checkpoint | State Saved |
|---|---|
| `diff_analyzed` | Diff read and understood |
| `consensus_requested` | Nelson review in progress |
| `review_complete` | Review decision made |
| `score_calculated` | Human review score computed |

### Checkpoint Storage

Checkpoints are stored in PostgreSQL:

```python
class Checkpoint(BaseModel):
    id: str                          # UUID
    agent: str                       # Agent name + instance
    task_id: str
    pipeline_id: str
    checkpoint_name: str             # e.g., "tests_passed"
    state: dict                      # Serialized agent state (JSON)
    created_at: datetime
    expires_at: datetime             # Auto-cleanup after pipeline completes
```

```sql
CREATE TABLE checkpoints (
    id TEXT PRIMARY KEY,
    agent TEXT NOT NULL,
    task_id TEXT NOT NULL,
    pipeline_id TEXT NOT NULL,
    checkpoint_name TEXT NOT NULL,
    state JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_checkpoint_lookup
    ON checkpoints(agent, task_id, pipeline_id);
```

### Checkpoint Resume

When an agent restarts, it:
1. Queries for the latest checkpoint for its task.
2. Deserializes the saved state.
3. Resumes from the step after the checkpoint.
4. Logs the resume event with checkpoint details.

```python
async def resume_or_start(self, task_id: str) -> AgentState:
    checkpoint = await self.store.get_latest_checkpoint(
        agent=self.name,
        task_id=task_id,
    )
    if checkpoint:
        log.info("resuming_from_checkpoint",
                 checkpoint=checkpoint.checkpoint_name,
                 created_at=checkpoint.created_at)
        return AgentState.from_checkpoint(checkpoint)
    else:
        log.info("starting_fresh", task_id=task_id)
        return AgentState.initial(task_id)
```

## Error Categories

### Transient Errors (Auto-Retry)

| Error | Handling |
|---|---|
| LLM API timeout | Retry with exponential backoff (1s, 2s, 4s) |
| LLM rate limit (429) | Respect `Retry-After` header, then retry |
| Redis connection lost | Reconnect with backoff, resume from checkpoint |
| GitHub API 5xx | Retry 3 times with backoff |
| Flaky test | Re-run tests once. If still failing, treat as real failure. |

### Recoverable Errors (Retry from Checkpoint)

| Error | Handling |
|---|---|
| LLM generates invalid code | Checkpoint → retry with error context in prompt |
| Tests fail after implementation | Checkpoint at `code_generated` → retry implementation |
| Lint/format fails | Auto-fix with formatter. If still failing, checkpoint → retry. |
| Merge conflict (auto-resolvable) | Richelieu resolves, Leonard re-validates |

### Fatal Errors (Escalate to Human)

| Error | Handling |
|---|---|
| 3 consecutive retries failed | Escalate with full crash history |
| Merge conflict (unresolvable) | Escalate with conflicting files and both versions |
| Budget exceeded (hard limit) | Halt task, escalate with cost summary |
| All LLM providers down | Halt pipeline, alert human |
| Target repo access denied | Halt pipeline, alert human to fix auth |
| `.ai-team.yaml` invalid | Halt pipeline, report validation errors |

## Retry Behavior

### Per-Task Retry Counter

```python
class RetryTracker:
    """Tracks retries per task per agent."""

    MAX_RETRIES: int = 3

    async def record_failure(
        self,
        task_id: str,
        agent: str,
        error: AiTeamError,
    ) -> RetryDecision:
        count = await self.store.increment_retry_count(task_id, agent)

        if count >= self.MAX_RETRIES:
            return RetryDecision(
                action="escalate",
                reason=f"Max retries ({self.MAX_RETRIES}) exceeded",
                history=await self.store.get_retry_history(task_id, agent),
            )

        return RetryDecision(
            action="retry",
            attempt=count + 1,
            resume_from=await self.store.get_latest_checkpoint(agent, task_id),
            error_context=str(error),  # Included in retry prompt
        )
```

### Error Context in Retries

When retrying, the previous error is included as context so the agent can avoid
making the same mistake:

```python
retry_context = f"""
Previous attempt #{attempt} failed with the following error:
{error_message}

The previous attempt reached checkpoint: {checkpoint_name}
Resuming from that point. Please avoid the same error.
"""
```

## Agent Health Monitoring

### Heartbeats

Every agent sends a heartbeat to Redis every 30 seconds:

```python
class Heartbeat(BaseModel):
    agent: str
    timestamp: datetime
    status: Literal["idle", "working", "error"]
    task_id: str | None
    memory_mb: float
    uptime_seconds: float
```

### Dead Agent Detection

The orchestrator's watchdog (see spec 13) monitors heartbeats for all ephemeral agent
containers (Nelson, Julius, Sherlock, Leonard, Katherine, Richelieu):

- If no heartbeat for 90 seconds → agent container is presumed dead.
- Log `agent_unresponsive` event.
- Respawn the container (same retry logic for all agents).
- If max retries exceeded, mark the task as stalled and alert human.

## Pipeline-Level Recovery

### Partial Pipeline Failure

If one task in a pipeline fails but others succeed:

1. Completed tasks remain merged into the feature branch.
2. The stuck task is marked as `NEEDS_HUMAN` (never auto-`FAILED`).
3. Other independent tasks continue (they don't depend on the stuck one).
4. Tasks that depend on the stuck task remain `PENDING` (dependencies unsatisfied).
5. The TUI shows which tasks are stuck and why.
6. Pipeline transitions to `PARTIALLY_FAILED` when all runnable work is done.

The human can then: add context and retry, take over the task manually, or
cancel the pipeline. See spec 01 — Human Intervention Flow for full details.

### Pipeline Cancellation

There is no automatic "full pipeline failure." Only the human can cancel a
pipeline (via TUI → `DELETE /pipelines/:id`). On cancellation:

1. Orchestrator stops spawning new agent containers.
2. All running containers are stopped gracefully.
3. All branches and worktrees are preserved (no auto-cleanup).
4. Cancellation reason is recorded in PostgreSQL.
5. Human can clean up later via TUI command or TTL auto-cleanup.

## Cleanup

### After Successful Pipeline

1. Orchestrator spawns Richelieu to remove all worktrees.
2. Orchestrator spawns Richelieu to delete merged task branches.
3. Expired checkpoints are purged from PostgreSQL.
4. Cost records and logs are retained (never auto-deleted).

### After Failed Pipeline

1. Worktrees and branches are **preserved** for debugging.
2. Checkpoints are preserved (not expired).
3. Human can manually clean up with `make clean-pipeline PIPELINE=<id>`.

## Idempotency

All agent operations should be idempotent where possible:

- Creating a branch that already exists → no-op (check first).
- Writing a file that already has the correct content → no-op.
- Sending a PR comment → check if identical comment exists first.
- Publishing a task to Redis → use message IDs to deduplicate.

This prevents double-execution issues when resuming from checkpoints.

> **Note**: The orchestrator itself also enforces idempotency in its event handlers.
> See spec 13 for the full handler idempotency table (e.g., `launch_agent` checks
> if agent already running, `open_pr` checks if PR already exists, etc.) and the
> startup reconciliation sequence that reconciles Docker container state with
> PostgreSQL state after a orchestrator crash.

---

## Cross-References

- **Spec 05** (Infrastructure): Resource limits prevent runaway containers.
- **Spec 06** (Orchestrator): Watchdog monitors heartbeats, handler idempotency, startup reconciliation, retry orchestration.
- **Spec 08** (TUI): Displays error banners, escalation alerts, blocked task indicators.
- **Spec 15** (Observability): Error events, checkpoint events, retry events in log stream.
- **Spec 16** (Cost Tracking): Budget exceeded is a fatal error category.
- **Spec 18** (Security): Container hardening limits blast radius of agent crashes.
