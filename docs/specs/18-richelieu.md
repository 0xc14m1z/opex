# 18 — Richelieu (Git & Workspace Manager)

> **Migrated from**: `PLAN.md` section 2 (Richelieu description), `docs/specs/01-deployment.md` (Branching Model & Repo Lifecycle)

## Overview

Richelieu manages all git operations and workspace provisioning for the system.
No other agent touches git -- every branch creation, merge, push, PR opening,
worktree creation, and conflict resolution goes through Richelieu. This
centralization ensures git state remains consistent and that destructive
operations are guarded by safety checks.

Richelieu is one of only two agents with write access to the filesystem
(the other being Leonard, which writes only within its assigned worktree).

---

## Responsibilities

- **Feature branch creation**: Create a feature branch from the target branch at pipeline start.
- **Worktree creation**: Create a git worktree + task branch for each Leonard instance.
- **Worktree cleanup**: Remove worktrees and task branches after merge.
- **Task branch push**: Push task branches to GitHub.
- **PR opening**: Open task PRs (task branch -> feature branch) and feature PRs (feature branch -> default branch).
- **Branch merging**: Merge task branches into the feature branch after Katherine approves.
- **Conflict resolution**: Detect merge conflicts. Attempt rebase + auto-resolution. Escalate to human if manual intervention is needed.
- **Branch alignment**: When a task PR is merged, rebase other in-progress task branches onto the updated feature branch.
- **Repo fetching**: Run `git fetch origin` at pipeline start to ensure the latest state.

---

## Lifecycle

- **Spawned when**: A `git_request` message appears on `pipeline:{id}:git_ops` (see spec 13-orchestrator).
- **Exits when**: Publishes a `git_response` to `pipeline:{id}:git_ops`.
- **Parallelism**: 1 per branch (serialized). Multiple Richelieu instances can run in parallel across different branches, but operations on the same branch are serialized by the orchestrator to prevent race conditions.

---

## Input / Output

### Input

- **Message type**: `git_request`
- **Redis stream**: `pipeline:{id}:git_ops`
- **Operations** (specified in the request payload):
  - `create_feature_branch`
  - `create_worktree`
  - `push_branch`
  - `merge_branch`
  - `open_task_pr`
  - `open_feature_pr`
  - `rebase_branch`
  - `remove_worktree`
  - `fetch_origin`

### Output

- **Message type**: `git_response`
- **Redis stream**: `pipeline:{id}:git_ops`
- **Payload**: Operation result including success/failure, branch name, worktree path, PR URL/number, conflict details (if any).

---

## Algorithm / Process

Richelieu implements the following operations. For the full branching model,
see spec 01 (Branching Model & Repo Lifecycle). This spec describes **how**
Richelieu implements each operation.

### Operation: `create_feature_branch`

1. Run `git fetch origin` to pull latest state.
2. Create feature branch from `origin/{default_branch}` (default branch from `.opex.yaml`).
3. Branch naming: `feature/{feature-name}` (derived from the plan/pipeline name).
4. Push the feature branch to GitHub.
5. Publish `git_response` with branch name.

### Operation: `create_worktree`

1. Create a worktree at `/workspace/.worktrees/{feature}--{task_id}/`.
2. Create a task branch from the feature branch HEAD.
3. Branch naming: `opex/{feature-name}/{task-id}`.
4. Push the task branch to GitHub.
5. Publish `git_response` with worktree path and branch name.

### Operation: `merge_branch`

1. Checkout the feature branch.
2. Attempt merge of the task branch.
3. If merge succeeds: push the updated feature branch.
4. If merge conflicts:
   - Attempt rebase of task branch onto feature branch.
   - If rebase succeeds: force-push (task branch is AI-owned, safe).
   - If rebase conflicts: attempt auto-resolution.
   - If auto-resolve succeeds: force-push.
   - If auto-resolve fails: publish `git_response` with `success=false` and conflict details.
5. Publish `git_response` with merge result.

### Operation: `open_task_pr`

1. Open a GitHub PR from the task branch targeting the feature branch.
2. Include task description, execution plan summary, and acceptance criteria in PR body.
3. Apply labels: `opex`, and `learning-mode` if learning mode is active.
4. Publish `git_response` with PR URL and number.

### Operation: `open_feature_pr`

1. Open a GitHub PR from the feature branch targeting the default branch.
2. Include: task summaries, review scores, cost summary, human review recommendations.
3. Apply labels: `opex`, `needs-human-review` if any task was flagged.
4. Publish `git_response` with PR URL and number.

### Operation: `rebase_branch`

Triggered when a task PR merges and other in-progress task branches need updating.

1. Fetch the updated feature branch.
2. Rebase the target task branch onto the updated feature branch.
3. If rebase succeeds: force-push (task branch is AI-owned).
4. If rebase conflicts: attempt auto-resolution; escalate if needed.
5. If code changed (not just fast-forward), flag for Katherine re-review.

### Operation: `remove_worktree`

1. Remove the git worktree at the specified path.
2. Delete the task branch locally (remote branch is preserved until PR is merged/closed).
3. Publish `git_response` confirming cleanup.

### Operation: `fetch_origin`

1. Run `git fetch origin --prune`.
2. Publish `git_response` confirming fetch completed.

---

## Branching Structure

For full details, see spec 01 (Branching Model & Repo Lifecycle). Summary:

```
main -------------------------------------------------------
  |                              |
  |  Feature A                   |  Feature B (parallel)
  |                              |
  +-- feature/add-auth           +-- feature/fix-perf
        |                              |
        +-- opex/add-auth/task-1    +-- opex/fix-perf/task-4
        |     PR #101 -> feature       |     PR #104 -> feature
        |                              |
        +-- opex/add-auth/task-2    +-- opex/fix-perf/task-5
        |     PR #102 -> feature       |     PR #105 -> feature
        |
        +-- opex/add-auth/task-3
              PR #103 -> feature
              (depends on task-1)

All task PRs merged -> PR #106: feature/add-auth -> main
All task PRs merged -> PR #107: feature/fix-perf -> main
```

### Worktree Layout

```
/workspace/                              # Main clone, checked out on default branch
/workspace/.worktrees/
  add-auth--task-1/                      # Worktree on opex/add-auth/task-1
  add-auth--task-2/                      # Worktree on opex/add-auth/task-2
  fix-perf--task-4/                      # Worktree on opex/fix-perf/task-4
```

### Branching Rules

1. All task branches fork from the feature branch HEAD at the time the task starts.
2. All task PRs target the feature branch (not `main`).
3. Dependent tasks wait: if task-3 depends on task-1, task-3 does not start until task-1's PR is merged into the feature branch.
4. Independent tasks run in parallel. Their PRs may need rebasing (see conflict resolution).
5. One feature PR at the end: when all task PRs are merged, Richelieu opens a single PR from the feature branch to the default branch.
6. Multiple features are independent: different features use different feature branches.

---

## Conflict Resolution

When parallel task PRs target the same feature branch, merging one may cause
the other to become outdated or have conflicts.

```
Task-1 PR merges into feature/add-auth
    |
    v
Orchestrator detects task-2 PR is outdated
    |
    v
Spawns Richelieu to rebase task-2 onto feature/add-auth
    |
    +-- Rebase succeeds -> force-push (task branch is AI-owned, safe)
    |                       PR auto-updates on GitHub
    |
    +-- Rebase conflicts -> Richelieu attempts auto-resolution
                           |
                           +-- Auto-resolve succeeds -> force-push
                           |
                           +-- Auto-resolve fails -> mark task "needs_human"
                                                      Preserve branches
                                                      Emit status event with conflicting files
```

Katherine re-reviews if the rebase changed code (not just a clean fast-forward).

---

## Interaction with Other Agents

- **Orchestrator (spec 13-orchestrator)**: The orchestrator dispatches all `git_request` messages and consumes `git_response` results. The orchestrator serializes operations per-branch.
- **Leonard (spec 20-leonard)**: Leonard works within the worktree that Richelieu provisions. Leonard commits to the task branch; Richelieu handles everything else (push, merge, PR).
- **Katherine (spec 21)**: After Katherine approves, the orchestrator tells Richelieu to merge. If rebase changes code, Katherine re-reviews.
- **Nelson (spec 16)**: Richelieu may request Nelson consensus on conflict resolution strategy in complex cases (rare).

---

## Principles Integration

Richelieu does not apply implementation or review principles directly. However,
it follows git conventions configured in `.opex.yaml` under the `git:` section:

- `default_branch`: Which branch to target for feature PRs.
- `branch_prefix`: Prefix for all AI-created branches (default: `opex/`).
- `require_pr`: Whether PRs are required (always true in practice).
- `auto_merge`: Whether to auto-merge feature PRs when Katherine approves all tasks.

---

## Safety

All destructive git operations are guarded:

- **Force push**: Only on AI-owned task branches (never on feature branches or default branch).
- **Branch delete**: Only for task branches after PR is merged/closed. Feature branches are preserved until the feature PR is merged.
- **Rebase**: Only on AI-owned task branches. Never rebase feature branches or the default branch.
- **Confirmation**: Configurable safeguards in `.opex.yaml` can require human confirmation for specific operations.

---

## System Prompt

<!-- TODO: Define the system prompt template for Richelieu -->
<!-- Richelieu's system prompt should emphasize safety-first git operations,
     never operating on branches it doesn't own, and always preserving
     state for human intervention on failure. -->

---

## PydanticAI Tools

<!-- TODO: Define the tools available to Richelieu -->

Expected tools:

- **Git commands**: `git fetch`, `git checkout`, `git branch`, `git merge`, `git rebase`, `git push`, `git worktree add/remove`, `git log`, `git diff`.
- **GitHub API**: Open PRs, add labels, post comments (via PyGithub or `gh` CLI).
- **File system**: Read/write access to `/workspace` and `/workspace/.worktrees/`.
- **Redis publish**: Publish `git_response` messages.

---

## Error Handling

- **Merge conflict (unresolvable)**: Mark task as `needs_human`. Preserve all branches and worktrees for manual intervention. Emit status event with conflicting file list.
- **GitHub API failure**: Retry with exponential backoff up to 3 times. If persistent, escalate to human.
- **Container crash**: The orchestrator respawns Richelieu with the same `git_request`. Git operations are idempotent (create branch that exists -> no-op, etc.).
- **Push rejected**: If a push is rejected (e.g., branch protection), report the error and escalate to human.

---

## Resource Profile

| Metric             | Value  | Rationale                                |
|--------------------|--------|------------------------------------------|
| CPU limit          | 0.5    | Git operations only                      |
| Memory limit       | 512M   | Git operations only                      |
| CPU reservation    | 0.25   |                                          |
| Memory reservation | 128M   |                                          |

Overridable via `.opex.yaml` `resources.richelieu` section (see spec 12).

---

## Configuration

- **`.opex.yaml`** (`git:` section): `default_branch`, `branch_prefix`, `require_pr`, `auto_merge`.
- **Environment variables**: `PIPELINE_ID`, `TASK_ID`, `GITHUB_APP_ID`, `GITHUB_APP_PRIVATE_KEY_PATH`, `GITHUB_WEBHOOK_SECRET`, `REDIS_URL`, `DATABASE_URL`.
- **GitHub authentication**: GitHub App (preferred for orgs, auto-refresh tokens) or PAT fallback (for personal repos). See spec 12 for auth details.

---

## Testing Strategy

- **Unit tests**: Test each git operation against a test git repository. Mock GitHub API calls for PR operations.
- **Integration tests**: Use testcontainers with a real git repository to test the full worktree lifecycle: create feature branch -> create worktree -> commit -> merge -> cleanup.
- **Conflict resolution tests**: Set up deliberate merge conflicts and verify auto-resolution behavior and escalation paths.
- **Safety tests**: Verify that Richelieu refuses to force-push to non-AI-owned branches, refuses to delete feature branches prematurely, and preserves state on failure.
- **Idempotency tests**: Verify that re-running the same git operation (e.g., after a crash and respawn) produces the correct result without corruption.
