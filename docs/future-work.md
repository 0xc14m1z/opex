# Future Work

> **Purpose**: Ideas and features that are explicitly out of scope for the initial
> implementation but worth considering for future iterations. Add new items as they
> come up during spec review or implementation.

---

## Pipeline Resumability

**Context**: Workflow spec deep-dive, Issue 2 (pipeline cancellation).

Currently, `CANCELLED` is a terminal state. Once a pipeline is cancelled, the human
must create a new pipeline to pursue the same feature. A future enhancement could
allow "un-cancelling" a pipeline:

- Resume from where it left off (completed tasks stay completed).
- Re-dispatch pending tasks from the dependency graph.
- Handle the case where the codebase has changed since cancellation (base branch
  moved, task branches may need rebasing).
- Potentially re-run Julius to re-validate the decomposition against the new
  codebase state.

**Complexity**: Medium-high. Requires careful handling of stale branches, changed
base branch, and potentially outdated task decomposition.

---

## GitHub Webhook for Real-Time Branch Detection

**Context**: Workflow spec deep-dive, Issue 5 (parallel pipelines).

The MONITORING state currently uses **polling** (every 60 seconds) to detect target
branch advancement, CI status changes, and PR merges. A GitHub webhook endpoint on
the API server would provide real-time detection:

- Push events on the target branch → immediate feature branch rebase.
- Check suite events → immediate CI failure detection.
- Pull request merge events → immediate MONITORING → MERGED transition.

This eliminates up to 60 seconds of latency on each detection, and reduces GitHub
API call volume.

**Requirements**:
- Public-facing API server endpoint (or a tunnel like ngrok/Tailscale funnel).
- Webhook secret management (HMAC verification).
- New `POST /webhooks/github` endpoint on the API server.

**Complexity**: Medium. The polling infrastructure stays as a fallback; webhooks
are an optimization layer on top.

---

## Pipeline Priority Levels

**Context**: Workflow spec deep-dive, Issue 5 (parallel pipelines).

The spawn queue is currently FIFO — all pipelines are treated equally. A future
enhancement could allow human-configurable priority levels:

- Priority levels per pipeline (e.g., `urgent`, `normal`, `background`).
- Higher-priority pipelines' spawn requests are served before lower-priority ones.
- Configurable from the TUI when creating a pipeline or after the fact.
- Priority can be changed while the pipeline is running.

The spawn queue already has a natural place to insert priority ordering without
architectural changes.

**Complexity**: Low. The spawn queue data structure changes from a simple FIFO to
a priority queue. No other system changes needed.

---

## Dependent Pipelines (Feature Chains)

**Context**: Workflow spec deep-dive, Issue 5 (parallel pipelines).

Currently, each pipeline is independent. If Feature B depends on Feature A's code,
the human must wait for Feature A to merge before starting Feature B. A future
enhancement could support pipeline dependencies:

- Feature B declares a dependency on Feature A.
- Feature B's pipeline doesn't start until Feature A enters MERGED.
- Feature B branches from the post-merge target branch (which includes Feature A).
- If Feature A is cancelled, Feature B is automatically paused with a human
  notification.

**Complexity**: Medium. Requires dependency tracking at the pipeline level
(currently only exists at the task level within a pipeline).
