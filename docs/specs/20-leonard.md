# 20 — Leonard (Implementer)

> **Migrated from**: `PLAN.md` section 2 (Leonard description)

## Overview

Leonard is the coding agent. It takes a mini execution plan from Sherlock and
implements the actual code changes: writing code, running tests, running
linters, validating acceptance criteria, simplifying, and committing. Leonard
is the agent with the highest resource allocation because it executes code
(test suites, linters, formatters) inside its container.

Leonard can run in parallel -- Julius decides how many based on the dependency
graph. Each Leonard instance works in its own isolated git worktree provisioned
by Richelieu.

---

## Responsibilities

- Receive an execution plan from Sherlock and a worktree from Richelieu.
- Implement code changes following Sherlock's step-by-step plan.
- Run the test suite (commands from `.ai-team.yaml`).
- Run linting and formatting (commands from `.ai-team.yaml`).
- Validate output against the acceptance criteria from the execution plan.
- Run a code simplification pass (reduce unnecessary complexity).
- Commit changes with conventional commit messages.
- Handle rework: when Katherine requests changes, Leonard receives feedback and iterates.
- In learning mode, Leonard's implementation may be deleted and replayed from scratch with new principles.

---

## Lifecycle

- **Spawned when**: A task is enriched (Sherlock completed) and the worktree is ready (Richelieu completed). The controller (spec 13-controller) spawns Leonard with the worktree path and branch name.
- **Exits when**: Publishes `implementation_complete` to `pipeline:{id}:reviews`.
- **Parallelism**: Up to `max_parallel_leonards` (default: 3) running simultaneously. Each Leonard works in its own worktree.

### Rework Loop

When Katherine requests changes:
1. The controller receives `review_result:changes_requested`.
2. The controller respawns Leonard with Katherine's feedback attached.
3. Leonard reads the feedback, modifies the implementation, and re-publishes `implementation_complete`.
4. This loop continues until Katherine approves or the retry limit is reached.

### Learning Mode

When learning mode is active (see spec 03):
1. Human reviews Leonard's implementation and provides feedback.
2. Nelson extracts principles from the feedback.
3. Leonard's entire implementation is **deleted** (branch reset, worktree cleaned).
4. Sherlock re-enriches the task with the new principles.
5. Leonard re-implements **from scratch** (not amending -- full replay) with the new principles in its system prompt.
6. A new PR is opened (labeled `learning: replay-N`).

---

## Input / Output

### Input

- **Message type**: Spawned by controller with task context
- **Environment variables**: `PIPELINE_ID`, `TASK_ID`, `WORKTREE_PATH`, `BRANCH_NAME`
- **Data available**:
  - Sherlock's mini execution plan (from `task_enriched` in Redis/PostgreSQL).
  - Worktree at `WORKTREE_PATH` (read/write access to this path only).
  - `.ai-team.yaml` configuration (parsed).
  - Implementation principles from `.ai-team/principles/implementation/` (included in execution plan by Sherlock).
  - Katherine's feedback (if rework iteration).

### Output

- **Message type**: `implementation_complete`
- **Redis stream**: `pipeline:{id}:reviews`
- **Payload**: Branch name with passing tests, ready for Katherine's review.

---

## Algorithm / Process

1. **Read execution plan**: Parse Sherlock's mini execution plan from the task context.
2. **Read rework feedback** (if rework iteration): Parse Katherine's feedback and identify required changes.
3. **Implement code changes**:
   - Follow Sherlock's step-by-step plan.
   - Apply implementation principles included in the plan.
   - Write or modify the specified files.
   - Follow codebase conventions noted by Sherlock.
4. **Run tests**:
   - Execute the test command from `.ai-team.yaml` (`commands.test`).
   - If tests fail: analyze failures, fix the code, re-run (up to N attempts).
5. **Run linting and formatting**:
   - Execute lint command (`commands.lint`).
   - Execute format command (`commands.format`).
   - Execute type check command (`commands.type_check`) if configured.
   - Fix any issues found.
6. **Validate acceptance criteria**:
   - Check each acceptance criterion from Sherlock's execution plan.
   - If criteria are not met: iterate on the implementation.
7. **Simplification pass**:
   - Review the implementation for unnecessary complexity.
   - Remove dead code, simplify conditionals, reduce nesting where possible.
   - Re-run tests after simplification to ensure nothing broke.
8. **Secret scanning**:
   - Scan the diff for potential secrets (API keys, passwords, tokens).
   - If secrets are detected: remove them and log a warning.
9. **Commit**:
   - Stage changes.
   - Commit with a conventional commit message derived from the task description.
10. **Publish**: Publish `implementation_complete` to `pipeline:{id}:reviews`.

---

## Interaction with Other Agents

- **Sherlock (spec 19)**: Leonard's primary input is Sherlock's execution plan. The plan contains everything Leonard needs to implement the task.
- **Katherine (spec 21)**: Katherine reviews Leonard's implementation. If changes are requested, Leonard receives the feedback and iterates (rework loop).
- **Richelieu (spec 18)**: Richelieu provisions the worktree and branch before Leonard starts. Leonard commits to the task branch; Richelieu handles push, merge, and PR operations.
- **Nelson (spec 16)**: Leonard does not typically call Nelson directly. However, in rare cases where the implementation approach is unclear despite Sherlock's plan, Leonard may request consensus.
- **Controller (spec 13-controller)**: The controller spawns Leonard, monitors its health, and handles retries on failure.

---

## Principles Integration

Leonard receives implementation principles through Sherlock's execution plan.
These principles are included in Leonard's system prompt context and guide
implementation decisions:

- **Pattern adherence**: Follow coding patterns mandated by principles (e.g., repository pattern, dependency injection).
- **Convention compliance**: Use naming conventions, error handling patterns, and test patterns specified by principles.
- **Negative examples**: Avoid anti-patterns called out in principle examples.

In learning mode, new principles extracted from human feedback are included in
subsequent replays, allowing Leonard to learn and improve.

---

## System Prompt

<!-- TODO: Define the system prompt template for Leonard -->
<!-- Leonard's system prompt should establish its role as an implementer,
     emphasize following the execution plan precisely, adhering to
     codebase conventions, writing clean and simple code, and ensuring
     all tests pass before committing. The system prompt should include
     implementation principles from Sherlock's plan. -->

---

## PydanticAI Tools

<!-- TODO: Define the tools available to Leonard -->

Expected tools:

- **File read**: Read files within the worktree and the main repo (for reference).
- **File write**: Write/modify files within the assigned worktree only.
- **Shell execution**: Run commands (tests, lint, format, type check) within the worktree.
- **Git commit**: Stage and commit changes with conventional commit messages.
- **Secret scanner**: Scan diff for potential secrets before committing.
- **Redis publish**: Publish `implementation_complete` messages.

---

## Safety

Leonard's access is restricted to prevent unintended side effects:

- **Filesystem**: Leonard can only write within its assigned worktree (`WORKTREE_PATH`). It cannot modify the main repo checkout or other Leonard instances' worktrees. Read access to the main repo is available for reference.
- **Network**: Leonard has outbound internet access (for package downloads, etc.) but no direct access to other containers (all communication via Redis).
- **Secret scanning**: Leonard scans its own diff for potential secrets before committing. If secrets are detected, they are removed and a warning is logged.
- **Resource limits**: Leonard has the highest resource allocation but is still bounded (2 CPU, 2G memory by default).

---

## Error Handling

- **Tests fail after N attempts**: If tests fail after the configured retry limit (default: 3 attempts), Leonard escalates to human with a summary of what went wrong, the failing tests, and the current state of the implementation.
- **Lint/format failures**: If linting or formatting cannot be fixed after retries, include the issues in the escalation.
- **Acceptance criteria not met**: If criteria cannot be satisfied, escalate with a description of which criteria failed and why.
- **Container crash**: The controller respawns Leonard. Since Leonard's work is committed to git incrementally (where possible), a respawn can resume from the last commit.
- **Timeout**: If Leonard exceeds its container deadline (default: 30 minutes), the controller kills the container and escalates.

---

## Resource Profile

| Metric             | Value  | Rationale                                         |
|--------------------|--------|---------------------------------------------------|
| CPU limit          | 2.0    | Code execution, tests, linters (CPU intensive)    |
| Memory limit       | 2G     | Test suites may be memory intensive               |
| CPU reservation    | 0.5    |                                                   |
| Memory reservation | 512M   |                                                   |

Leonard has the highest resource profile because it runs arbitrary code (test
suites, linters, formatters) inside its container.

Overridable via `.ai-team.yaml` `resources.leonard` section (see spec 12). For
projects with heavy test suites, the repo owner can increase limits:

```yaml
resources:
  leonard:
    cpu_limit: 4.0
    memory_limit: 4G
```

---

## Configuration

- **`.ai-team.yaml`**: Commands (test, lint, format, type_check, install), guidelines, knowledge references, resource overrides.
- **Environment variables**: `PIPELINE_ID`, `TASK_ID`, `WORKTREE_PATH`, `BRANCH_NAME`, `OPENROUTER_API_KEY`, `DATABASE_URL`, `REDIS_URL`.
- **LLM model**: Uses `llm.default_model` from `.ai-team.yaml` (or per-agent override `llm.overrides.leonard`).
- **Retry limit**: Configurable max attempts for test/lint failures (default: 3).

---

## Testing Strategy

- **Unit tests**: Test code generation logic with controlled codebases and expected
  outputs. Mock LLM responses to verify Leonard follows execution plans correctly.
- **Integration tests**: Use VCR cassettes (see spec 10) to record real LLM
  implementation calls. Test the full flow from execution plan to committed code
  in a test git repository.
- **Rework loop tests**: Verify that Leonard correctly applies Katherine's feedback
  in rework iterations.
- **Safety tests**: Verify filesystem isolation (cannot write outside worktree),
  secret scanning catches common patterns, and resource limits are respected.
- **Learning mode tests**: Verify that Leonard implements differently when new
  principles are added to the execution plan.
- **Edge cases**: Test with failing test suites, lint errors that cannot be
  auto-fixed, and tasks that exceed the retry limit.
