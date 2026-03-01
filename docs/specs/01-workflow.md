# 01 — Workflow

> **Migrated from**: `docs/specs/01-deployment.md` (Branching Model & Repo Lifecycle, Review Flow, Conflict Resolution, GitHub Labels) and `docs/specs/13-orchestrator.md` (Complete Event Flow: End-to-End Pipeline, 20-step walkthrough)

---

## Overview

This spec defines the complete pipeline lifecycle from plan submission to merged PR.
It covers the branching model, the worktree layout, the review flow, and the full
20-step event walkthrough that drives a pipeline from start to finish.

For the controller that orchestrates these events, see spec 13.
For the communication protocol (Redis Streams), see spec 04.
For git operations (Richelieu), see spec 18.

---

## Pipeline Lifecycle

A pipeline transitions through these states:

```
CREATED → DECOMPOSING → IN_PROGRESS → COMPLETING → COMPLETED
                                                  ↘ FAILED
                                        CANCELLED (from any state)
```

| Status | Description |
|--------|-------------|
| `CREATED` | Plan received, not yet started |
| `DECOMPOSING` | Julius is running |
| `IN_PROGRESS` | Tasks are being processed |
| `COMPLETING` | All tasks done, opening PR |
| `COMPLETED` | PR opened |
| `FAILED` | Unrecoverable failure |
| `CANCELLED` | Cancelled by human |

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
Controller detects task-2 PR is outdated
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

The controller evaluates the dependency graph after every task completion to
determine which tasks can be launched next. This is the core scheduling logic.

### How it works

1. When `branch_merged` event arrives (a task's PR was merged into the feature
   branch), the controller marks that task as `COMPLETED`.

2. The controller calls `DependencyGraph.get_ready_tasks()` which returns tasks
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

3. For each newly ready task, the controller spawns a Sherlock container to
   enrich it (beginning the Sherlock → Leonard → Katherine pipeline for that
   task).

4. The controller publishes a `batch_dispatched` event tracking which tasks
   were launched in this batch.

5. If `get_ready_tasks()` returns empty AND all tasks are completed, the
   controller transitions the pipeline to `COMPLETING` and spawns Richelieu
   to open the feature PR.

### Parallelism limits

The controller respects per-agent parallelism limits (see spec 13 for
`ControllerConfig`):

- `max_parallel_sherlocks: 5`
- `max_parallel_leonards: 3`
- `max_parallel_katherines: 3`
- `max_parallel_nelsons: 5`
- `max_parallel_richelieus: 3` (serialized per-branch, parallel across branches)

If more tasks are ready than the parallelism limit allows, the controller
queues them and launches as slots become available.

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
