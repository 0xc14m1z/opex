# 01 — Workflow

> **Migrated from**: `docs/specs/01-deployment.md` (Branching Model & Repo Lifecycle, Review Flow, Conflict Resolution, GitHub Labels) and `docs/specs/13-orchestrator.md` (Complete Event Flow: End-to-End Pipeline, 20-step walkthrough)

---

## Overview

This spec defines the complete pipeline lifecycle from plan submission to merged PR.
It covers the branching model, the worktree layout, the review flow, and the full
20-step event walkthrough that drives a pipeline from start to finish.

For the orchestrator that orchestrates these events, see spec 13.
For the communication protocol (Redis Streams), see spec 04.
For git operations (Richelieu), see spec 18.

---

## Pipeline Lifecycle

A pipeline transitions through these states:

```
CREATED → DECOMPOSING → IN_PROGRESS → COMPLETING → COMPLETED
                                    ↘ PARTIALLY_FAILED → COMPLETED (after retry)
                                                       → CANCELLED (human abandons)
                                        CANCELLED (from any state, by human)
```

| Status | Description |
|--------|-------------|
| `CREATED` | Plan received, not yet started |
| `DECOMPOSING` | Julius is running |
| `IN_PROGRESS` | Tasks are being processed |
| `COMPLETING` | All tasks done, opening PR |
| `COMPLETED` | PR opened |
| `PARTIALLY_FAILED` | Some tasks succeeded, some need human intervention. Pipeline can recover via retry or human takeover. |
| `CANCELLED` | Explicitly cancelled by human |

> **Design principle (P8 — No Silent Failures):** There is no automatic
> `FAILED` terminal state for pipelines. The system never gives up
> autonomously. Every failure escalates to a human who can retry with
> more context, take over manually, or explicitly cancel. The only way
> a pipeline terminates without success is by explicit human decision
> (`CANCELLED`).

---

## Branching Model

### Repo volume lifecycle

1. **`make connect REPO=<url>`** -- Clones the target repo into the `ai-team-repo`
   volume. This is a one-time setup operation. The clone persists across pipelines
   (warm cache).

2. **Pipeline starts** -- Richelieu runs `git fetch origin` to pull the latest
   changes, then creates a feature branch from `origin/{default_branch}` (the
   default branch is configured in `.ai-team.yaml`). The feature branch is pushed
   to GitHub.

3. **Task starts** -- Richelieu creates a worktree + task branch from the feature
   branch HEAD. The task branch is pushed to GitHub.

4. **Task PR** -- Richelieu opens a GitHub PR from the task branch targeting the
   feature branch. All review happens on this PR (see Review Flow below).

5. **Task complete** -- After the PR is merged on GitHub, Richelieu pulls the
   updated feature branch locally. The worktree is cleaned up.

6. **All tasks complete** -- Richelieu opens a **feature PR** from the feature
   branch targeting the default branch (`main`). This is the integration review.

7. **Between pipelines** -- The repo volume persists. The next pipeline starts
   with `git fetch`, so it has the latest upstream state.

### Branching structure

```
main ─────────────────────────────────────────────────────────
  │                              │
  │  Feature A                   │  Feature B (parallel, independent)
  │                              │
  └── feature/add-auth           └── feature/fix-perf
        │                              │
        ├── ai-team/add-auth/task-1    ├── ai-team/fix-perf/task-4
        │     PR #101 → feature/add-auth    PR #104 → feature/fix-perf
        │                              │
        ├── ai-team/add-auth/task-2    └── ai-team/fix-perf/task-5
        │     PR #102 → feature/add-auth    PR #105 → feature/fix-perf
        │
        └── ai-team/add-auth/task-3
              PR #103 → feature/add-auth
              (depends on task-1, branches
               AFTER PR #101 merges)

All task PRs merged → PR #106: feature/add-auth → main
All task PRs merged → PR #107: feature/fix-perf → main
```

### Branching rules

1. **All task branches fork from the feature branch HEAD** at the time the task
   starts.
2. **All task PRs target the feature branch** (not `main`).
3. **Dependent tasks wait**: If task-3 depends on task-1, task-3 doesn't start
   until task-1's PR is merged into the feature branch. This ensures task-3
   branches from a feature branch that already contains task-1's code.
4. **Independent tasks run in parallel**: Tasks with no dependency edges can
   start simultaneously. Their PRs may need rebasing (see Conflict Resolution).
5. **One feature PR at the end**: When all task PRs are merged, Richelieu opens
   a single PR from the feature branch to the default branch.
6. **Multiple features are independent**: Different features use different
   feature branches. They share the same git clone (objects are shared) but
   their worktrees and branches don't interfere.

---

## Worktree Layout

```
/workspace/                              # Main clone, checked out on default branch
/workspace/.worktrees/
  add-auth--task-1/                      # Worktree on ai-team/add-auth/task-1
  add-auth--task-2/                      # Worktree on ai-team/add-auth/task-2
  fix-perf--task-4/                      # Worktree on ai-team/fix-perf/task-4
```

---

## Conflict Resolution

When parallel task PRs target the same feature branch, merging one may cause
the other to become outdated or have conflicts.

```
Task-1 PR merges into feature/add-auth
    │
    ▼
Orchestrator detects task-2 PR is outdated
    │
    ▼
Spawns Richelieu to rebase task-2 onto feature/add-auth
    │
    ├── Rebase succeeds → force-push (task branch is AI-owned, safe)
    │                      PR auto-updates on GitHub
    │
    └── Rebase conflicts → Richelieu attempts auto-resolution
                           │
                           ├── Auto-resolve succeeds → force-push
                           │
                           └── Auto-resolve fails → mark task "needs_human"
                                                      Preserve branches
                                                      Emit status event with conflicting files
```

Katherine re-reviews if the rebase changed code (not just a clean fast-forward).

---

## Review Flow on Task PRs

Every task PR goes through this review flow on GitHub:

1. **Katherine** posts structured review comments on the PR (correctness,
   style, safety, simplicity). See spec 21 for Katherine's full review process.
2. **If changes needed**: Katherine requests changes. Leonard pushes fixes
   to the task branch. Katherine re-reviews. The full conversation is visible
   on the PR.
3. **Katherine calculates human review score** via Nelson consensus (see spec 16).
4. **If score < threshold**: Katherine approves the PR.
5. **If score >= threshold**: PR stays open, flagged for human review.
   A `learning-mode` label is added if learning mode is active (see spec 03).
6. **Human reviews** (if flagged): Approves, requests changes, or discusses.
   Human comments feed back into the principle system (see spec 03).

---

## GitHub Labels

| Label                    | Applied when                                    |
|--------------------------|-------------------------------------------------|
| `ai-team`               | All PRs created by the system                   |
| `learning-mode`         | Feature is in learning mode                     |
| `learning: replay-N`    | Task PR is the Nth replay in learning mode      |
| `autonomous`            | Task completed without human review             |
| `needs-human-review`    | Katherine flagged for human review              |

---

## Dependency Dispatch

The orchestrator evaluates the dependency graph after every task completion to
determine which tasks can be launched next. This is the core scheduling logic.

### How it works

1. When `branch_merged` event arrives (a task's PR was merged into the feature
   branch), the orchestrator marks that task as `COMPLETED`.

2. The orchestrator calls `DependencyGraph.get_ready_tasks()` which returns tasks
   whose dependencies are all completed and whose status is still `PENDING`:

   ```python
   def get_ready_tasks(self) -> list[Task]:
       """Returns tasks whose dependencies are all completed."""
       completed = {t.id for t in self.tasks if t.status == TaskStatus.COMPLETED}
       return [
           t for t in self.tasks
           if t.status == TaskStatus.PENDING
           and all(dep in completed for dep in t.depends_on)
       ]
   ```

3. For each newly ready task, the orchestrator spawns a Sherlock container to
   enrich it (beginning the Sherlock → Leonard → Katherine pipeline for that
   task).

4. The orchestrator publishes a `batch_dispatched` event tracking which tasks
   were launched in this batch.

5. If `get_ready_tasks()` returns empty AND all tasks are completed, the
   orchestrator transitions the pipeline to `COMPLETING` and spawns Richelieu
   to open the feature PR.

### Parallelism limits

The orchestrator respects per-agent parallelism limits (see spec 13 for
`OrchestratorConfig`):

- `max_parallel_sherlocks: 5`
- `max_parallel_leonards: 3`
- `max_parallel_katherines: 3`
- `max_parallel_nelsons: 5`
- `max_parallel_richelieus: 3` (serialized per-branch, parallel across branches)

If more tasks are ready than the parallelism limit allows, the orchestrator
queues them and launches as slots become available.

---

## Task Lifecycle

Each task in a pipeline transitions through these states:

```
PENDING
  │
  ▼
ENRICHING (Sherlock running)
  │
  ▼
READY (worktree created)
  │
  ▼
IMPLEMENTING (Leonard running)
  │
  ▼
IN_REVIEW (Katherine running)
  │
  ├── approved → MERGING → COMPLETED → (unblock dependents, evaluate graph)
  │
  ├── changes_requested → REWORK → IMPLEMENTING (review loop)
  │
  └── escalated (auto-retries exhausted) → NEEDS_HUMAN
                                              │
                                ┌──────────────┼───────────────────┐
                                ▼              ▼                   ▼
                          [Investigate]   [Take over]        [Cancel pipeline]
                               │              │
                               ▼              ▼
                       Diagnostic chat   ABANDONED_HUMAN_TAKEOVER
                               │              │
                     ┌─────────┴──────┐    Human: "I'm done"
                     ▼                ▼       │
               Add context      Edit task    Katherine verifies
                     │                │       │
                     ▼                ▼    ┌──┴──┐
                  RETRYING         RETRYING │    │
                     │                │   pass  fail
                     ▼                ▼    │    │
                  ENRICHING       ENRICHING │    └→ stays, report back
                  (new attempt)            ▼
                                    COMPLETED_EXTERNALLY
                                    (unblocks dependents)

Terminal states:
  COMPLETED              — system finished the task
  COMPLETED_EXTERNALLY   — human finished the task, Katherine verified
  CANCELLED              — human cancelled the pipeline

Non-terminal waiting states:
  NEEDS_HUMAN                — waiting for human decision
  ABANDONED_HUMAN_TAKEOVER   — human took over, doing work externally
```

> **Design principle (P8 — No Silent Failures):** There is no automatic
> `FAILED` terminal state for tasks. When auto-retries are exhausted, the
> task moves to `NEEDS_HUMAN`, not `FAILED`. The system always escalates
> to a human rather than silently giving up.

---

## Failure Handling

### Failure granularity: task-level

A single task failure does **not** fail the pipeline or kill sibling tasks.
Independent tasks continue running. The pipeline only enters
`PARTIALLY_FAILED` when all runnable work is done and at least one task
is stuck in `NEEDS_HUMAN` or `ABANDONED_HUMAN_TAKEOVER`.

### Retry mechanics

- **Retry count**: Configurable in `.ai-team.yaml` under `retries.max_task_retries`
  (default: 2 retries = 3 total attempts). Overridable per-pipeline from the TUI.
- **Retry strategy: from scratch.** Leonard makes a **single commit** at task
  completion — no intermediate commits. Julius creates tasks small enough
  that restarting is cheap. On retry, the previous attempt's branch work is
  discarded (branch reset to feature branch HEAD).
- **Retry trigger**: After the Leonard → Katherine review loop exhausts max
  review cycles (see Issue 11), or after an agent container crashes/OOMs
  and agent-level retries (spec 08) are exhausted.
- **After auto-retries exhausted**: Task → `NEEDS_HUMAN`. Always escalates,
  never auto-fails.

### Attempt history

Each task attempt is a separate record in PostgreSQL — never overwritten.
When retrying, a new attempt is created linked to the same task.

```
task_attempts table:
  id                 -- UUID
  task_id            -- FK to tasks
  attempt_number     -- 1-indexed
  status             -- 'running' | 'succeeded' | 'failed' | 'abandoned'
  started_at
  finished_at
  failure_reason     -- human-readable
  exit_code          -- agent container exit code (if applicable)
  agent_logs_ref     -- reference to logs in Loki
  diff_snapshot      -- git diff at time of failure (for inspection)
```

### External dependency failures

- **Redis/PostgreSQL down**: System-level concern, owned by spec 08 (Error
  Recovery). Orchestrator startup reconciliation handles recovery.
- **GitHub API down**: Task-level concern. Richelieu retries with exponential
  backoff (handled by agent retry mechanism in spec 08). After max agent-level
  retries → task escalates to `NEEDS_HUMAN`.

### Cleanup on failure

Failed attempts preserve branches and worktrees for human inspection.
Cleanup is manual via a TUI/API command, or automatic via configurable
TTL (default: 48 hours).

---

## Human Intervention Flow

When a task reaches `NEEDS_HUMAN`, the human has three options: investigate
and retry, take over manually, or cancel the entire pipeline.

### Option 1: Investigate and retry (diagnostic chat)

A two-phase process mediated by the TUI and API server.

**Phase 1 — Diagnostic Chat**

The human enters a chat session in the TUI (via API server → LLM). This is
**not** a new agent — it's a stateless conversation handled by the API server
calling the LLM directly (via LiteLLM/OpenRouter). The chat has access to:

- The failed task's attempt history (logs, diffs, failure reasons)
- The original task description and all context entries
- The current state of the codebase (relevant files)

The human and LLM discuss what went wrong, what's missing, and whether the
task is scoped correctly. No system state changes during this phase.

**At any point during the chat, the human can invoke Nelson consensus** to
get multi-model validation on a decision before committing to it. Nelson is
spawned through the normal orchestrator mechanism — the API server publishes
a `consensus_request` to Redis.

**Phase 2 — Context Distillation**

The human (with LLM assistance) distills the chat into a concise context
addition. The TUI switches to a **review mode** showing:

- The proposed context addition (what will be added)
- A **diff view**: task context before vs. after
- If the task description was edited: a diff of the description changes

The human reviews, can edit further, and explicitly approves. Only on
approval:

1. The context entry is written to PostgreSQL (`task_context_entries` table)
2. The task moves to `RETRYING`
3. The retry counter resets (human context makes this effectively a new problem)
4. Sherlock is re-spawned with the full context chain

**All chat messages are logged** to PostgreSQL (`diagnostic_chat_sessions`
and `diagnostic_chat_messages` tables). This ensures full auditability and
the ability to reconstruct the human's reasoning.

### Human context history

Context additions are **not** just appended — they form a timeline linked
to the failures that prompted them. When Sherlock enriches a retry, it
receives all entries in order: the original context, plus each human
addition tagged with which failure prompted it.

```
task_context_entries table:
  id
  task_id
  entry_type         -- 'original' | 'human_addition' | 'task_edit'
  content            -- the actual context text
  triggered_by       -- FK to task_attempts (which attempt failed; null for 'original')
  chat_session_id    -- FK to diagnostic_chat_sessions (null for 'original')
  created_at
  created_by         -- 'julius' | 'human:{user_id}'
```

### Task description editing

The human can also edit the task description itself, using the same
chat-mediated workflow: describe the changes in chat → LLM makes the
edits → TUI shows a **diff** (before/after) → human approves → stored
as `entry_type: 'task_edit'` in the context history.

### Option 2: Human takeover

The human decides to do the work themselves outside the system.

```
Task → ABANDONED_HUMAN_TAKEOVER
    │
    (human works outside the system)
    │
    Human returns to TUI: "I'm done"
    │
    ▼
Katherine runs in verify_prerequisites mode:
    ├── Does the expected branch exist?
    ├── Are the changes committed and pushed?
    ├── Do the tests pass? (from .ai-team.yaml commands)
    ├── Does lint pass?
    ├── Are the dependent tasks' requirements met?
    │
    ├── All checks pass → COMPLETED_EXTERNALLY (dependents unblocked)
    └── Some checks fail → stays ABANDONED_HUMAN_TAKEOVER (report to human)
```

Katherine's `verify_prerequisites` mode is a reusable capability — a
validation-only run that checks code quality and completeness without
performing a full review cycle. See spec 21 for Katherine's modes.

### Option 3: Cancel the pipeline

The human decides the whole feature isn't worth pursuing.

```
Pipeline → CANCELLED
    │
    All running tasks → CANCELLED
    All pending tasks → CANCELLED
    │
    Cancellation record stored:
      - reason (human-provided text via TUI)
      - which tasks completed vs. were in-progress
      - what branches/PRs exist
      - timestamp
    │
    Branches/PRs are NOT auto-deleted (human may want to salvage).
    Cleanup available via TUI command or TTL auto-cleanup.
```

---

## Retry Configuration

### In `.ai-team.yaml`

```yaml
retries:
  max_task_retries: 2          # 2 retries = 3 total attempts (default)
  cleanup_ttl_hours: 48        # Auto-cleanup failed branches after 48h
```

### Per-pipeline override from TUI

The human can override `max_task_retries` when creating a pipeline or when
a task reaches `NEEDS_HUMAN` (e.g., "give this one 5 tries"). The override
is stored in the pipeline record in PostgreSQL.

---

## Complete Event Flow: End-to-End Pipeline

A 20-step walkthrough of a typical pipeline from plan submission to PR.

```
 1. Human submits plan via TUI/CLI
    → TUI publishes `pipeline_created` to `system:pipelines`

 2. Orchestrator receives `pipeline_created`
    → Creates pipeline record in PostgreSQL (status: CREATED)
    → Spawns Richelieu to create feature branch
    → Richelieu publishes `git_response` with branch name, exits

 3. Orchestrator receives `git_response` (feature branch ready)
    → Updates pipeline status → DECOMPOSING
    → Spawns Julius container (PIPELINE_ID env var)

 4. Julius analyzes codebase, decomposes plan into tasks
    → Publishes `consensus_request` to `pipeline:{id}:consensus`
    → Orchestrator spawns Nelson for validation
    → Nelson publishes `consensus_response`, exits
    → Julius publishes all `task_created` messages to `pipeline:{id}:tasks`
    → Julius publishes `decomposition_complete` to `pipeline:{id}:tasks`
    → Julius container exits

 5. Orchestrator receives `decomposition_complete`
    → Stores DependencyGraph in PostgreSQL
    → Updates pipeline status → IN_PROGRESS
    → Calls `graph.get_ready_tasks()` to find tasks with no dependencies
    → Spawns Sherlock containers for ready tasks (up to max_parallel)
    → Publishes `batch_dispatched` to `pipeline:{id}:status`

 6. Sherlock (per task) inspects codebase, produces execution plan
    → May publish `consensus_request` → orchestrator spawns Nelson → Nelson responds, exits
    → Publishes `task_enriched` to `pipeline:{id}:enriched`
    → Container exits

 7. Orchestrator receives `task_enriched`
    → Updates task status → READY
    → Spawns Richelieu to create worktree
    → Richelieu publishes `git_response` with worktree path and branch name, exits

 8. Orchestrator receives `git_response` (worktree ready)
    → Spawns Leonard container with TASK_ID, worktree path, branch name

 9. Leonard implements code following Sherlock's execution plan
    → Runs tests, lint, format (from .ai-team.yaml commands)
    → Commits to task branch
    → Publishes `implementation_complete` to `pipeline:{id}:reviews`
    → Container exits

10. Orchestrator receives `implementation_complete`
    → Updates task status → IN_REVIEW
    → Spawns Katherine container with TASK_ID

11. Katherine reviews the diff using Nelson consensus
    → Publishes `consensus_request` → orchestrator spawns Nelson → Nelson responds, exits
    → Publishes `review_result` to `pipeline:{id}:reviews`
    → Container exits

12. Orchestrator receives `review_result`
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

13. Orchestrator receives `git_request:merge_branch`
    → Spawns Richelieu to merge task branch → feature branch
    → Richelieu publishes `git_response` with merge result, exits

14. Orchestrator receives `git_response` (merge success)
    → Updates task status → COMPLETED
    → Spawns Richelieu to remove worktree (`git_request:remove_worktree`)

15. Orchestrator evaluates dependency graph
    → Calls `graph.get_ready_tasks()` with updated statuses
    ├── More tasks ready → Spawn Sherlocks (go to step 6)
    └── No tasks ready + all tasks complete → Continue to step 16

16. Orchestrator detects all tasks completed
    → Publishes `all_tasks_complete` to `pipeline:{id}:status`

17. Orchestrator transitions pipeline → COMPLETING
    → Publishes `git_request:open_pr` to `pipeline:{id}:git_ops`

18. Orchestrator spawns Richelieu to open PR
    → Richelieu opens PR from feature branch → target branch
    → Includes task summaries, review scores, cost summary in PR body
    → Publishes `git_response` with PR URL and number
    → Container exits

19. Orchestrator receives `git_response` (PR opened)
    → Updates pipeline status → COMPLETED
    → Publishes `pipeline_completed` to `system:pipelines`

20. Human reviews PR (if Katherine flagged it) or it auto-merges
    → TUI shows pipeline summary with link to PR
```
