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
