# 17 — Julius (Task Decomposer)

> **Migrated from**: `PLAN.md` section 2 (Julius description)

## Overview

Julius takes a high-level plan from a human and decomposes it into minimal,
independent tasks suitable for parallel implementation. It is the first agent
to run in any pipeline, turning a vague or detailed plan into a structured
dependency graph of concrete work items. Each task is sized so that a single
Leonard can implement it and a single Katherine can review it in one pass.

---

## Responsibilities

- Parse and understand the human's plan (natural language or structured).
- Read the target repo's `.opex.yaml` configuration and guidelines.
- Analyze the codebase at a high level (directory structure, key modules, dependencies).
- Break the plan into tasks where each task touches the **minimum number of files**.
- Build a **dependency graph** identifying which tasks can run in parallel vs. which must be sequential.
- Estimate complexity per task.
- Use Nelson (spec 16) to validate the decomposition quality.
- Publish the task list and dependency graph for the orchestrator (spec 13-orchestrator) to dispatch.

---

## Lifecycle

- **Spawned when**: `pipeline_created` event is received by the orchestrator (after Richelieu creates the feature branch).
- **Exits when**: Publishes `decomposition_complete` to `pipeline:{id}:tasks`.
- **Parallelism**: 1 per pipeline. Only one Julius runs per pipeline.

---

## Input / Output

### Input

- **Message type**: `pipeline_created` (routed by the orchestrator)
- **Redis stream**: `system:pipelines`
- **Environment variables**: `PIPELINE_ID`, `REPO_URL`, `TARGET_BRANCH`
- **Data available**:
  - Plan description (human's plan text, from the pipeline record).
  - Target repo at `/workspace` (read-only access).
  - `.opex.yaml` configuration (parsed).

### Output

- **Message type**: `decomposition_complete`
- **Redis stream**: `pipeline:{id}:tasks`
- **Payload** (`DecompositionCompletePayload`):

```python
class DecompositionCompletePayload(BaseModel):
    pipeline_id: str
    task_count: int
    dependency_graph: DependencyGraph
    ready_task_ids: list[str]            # Tasks with no dependencies (can start now)
```

Additionally publishes individual `task_created` messages for each task before
the `decomposition_complete` message.

---

## Algorithm / Process

1. **Read plan**: Parse the human's plan text from the pipeline record.
2. **Read configuration**: Load `.opex.yaml` from the target repo. Extract guidelines, commands, knowledge references.
3. **Analyze codebase structure**:
   - Scan directory tree for key modules and packages.
   - Identify the project's language, framework, and primary patterns.
   - Read architecture docs and relevant knowledge files referenced in `.opex.yaml`.
   - Build a mental model of the codebase's structure and module boundaries.
4. **Decompose plan into tasks**:
   - Break the plan into the smallest logical units of work.
   - Each task should touch the minimum number of files.
   - Each task must be small enough for one Leonard to implement and one Katherine to review.
   - Assign a unique `task_id` to each task.
   - Write a clear task description with enough context for Sherlock to enrich.
5. **Build dependency graph**:
   - Identify dependencies between tasks (e.g., task-3 depends on task-1's API).
   - Mark independent tasks as parallelizable.
   - Detect implicit dependencies (e.g., shared database migrations, shared types).
   - Validate that the graph is a DAG (no cycles).
6. **Estimate complexity**: Assign a complexity estimate per task (used for resource planning).
7. **Validate via Nelson**: Submit the decomposition to Nelson (spec 16) for consensus validation with these review goals:
   - "Is each task truly independent where marked independent?"
   - "Are the tasks small enough for one agent to implement?"
   - "Does the dependency graph have any missing edges?"
   - "Are there implicit dependencies not captured?"
8. **Publish tasks**: Publish individual `task_created` messages, then publish `decomposition_complete` with the full dependency graph.

---

## Interaction with Other Agents

- **Nelson (spec 16)**: Julius uses Nelson to validate the decomposition before publishing. If Nelson identifies issues, Julius revises the decomposition.
- **Orchestrator (spec 13-orchestrator)**: The orchestrator spawns Julius and consumes its `decomposition_complete` output to begin dispatching tasks.
- **Sherlock (spec 19)**: Each task produced by Julius becomes input for a Sherlock instance that enriches it with codebase-specific details.
- **Richelieu (spec 18)**: Richelieu creates the feature branch before Julius runs. Julius does not interact with Richelieu directly.

---

## Principles Integration

Julius reads implementation principles from `.opex/principles/implementation/`
to understand what patterns and conventions the team values. This context helps
Julius size tasks appropriately -- for example, if there is a principle requiring
repository pattern usage, Julius ensures tasks that add data access include time
for that pattern.

<!-- TODO: Define how principles specifically influence decomposition decisions -->

---

## System Prompt

<!-- TODO: Define the system prompt template for Julius -->
<!-- Julius's system prompt should establish its role as a decomposer,
     emphasize minimal task sizing, dependency graph correctness, and
     the importance of each task being independently implementable. -->

---

## PydanticAI Tools

<!-- TODO: Define the tools available to Julius -->

Expected tools:

- **File system read**: Read files and directories in the target repo (read-only).
- **Directory listing**: List directory contents to understand project structure.
- **Dependency analysis**: Analyze import graphs and module relationships.
- **Configuration reader**: Parse `.opex.yaml` and extract relevant settings.
- **Nelson request**: Submit consensus requests for decomposition validation.
- **Redis publish**: Publish `task_created` and `decomposition_complete` messages.

---

## Error Handling

- **LLM failure during decomposition**: Retry up to 3 times. If Nelson validation
  fails repeatedly, escalate to human with the best decomposition so far.
- **Container crash**: The orchestrator's watchdog detects the crash and may respawn
  Julius. Since Julius is stateless (reads plan from Redis/PostgreSQL), a respawn
  starts the decomposition from scratch.
- **Invalid plan**: If the plan is too vague or ambiguous to decompose, Julius
  publishes a `pipeline_failed` event with a description of what is unclear,
  requesting human clarification.
- **Cyclic dependencies detected**: Julius must detect and resolve cycles before
  publishing. If cycles cannot be resolved, escalate to human.

---

## Resource Profile

| Metric             | Value  | Rationale                                |
|--------------------|--------|------------------------------------------|
| CPU limit          | 1.0    | Codebase analysis, LLM calls             |
| Memory limit       | 1G     | Directory scanning, dependency graphs    |
| CPU reservation    | 0.25   |                                          |
| Memory reservation | 256M   |                                          |

Overridable via `.opex.yaml` `resources.julius` section (see spec 12).

---

## Configuration

- **`.opex.yaml`**: Project description, knowledge references, guidelines, commands.
- **Environment variables**: `PIPELINE_ID`, `OPENROUTER_API_KEY`, `DATABASE_URL`, `REDIS_URL`.
- **LLM model**: Uses `llm.default_model` from `.opex.yaml` (or per-agent override `llm.overrides.julius`).

---

## Testing Strategy

- **Unit tests**: Test decomposition logic with sample plans and expected task lists.
  Mock LLM responses to verify the decomposition algorithm produces valid DAGs.
- **Integration tests**: Use VCR cassettes (see spec 10) to record real LLM
  decomposition calls. Verify the full flow from plan input to `decomposition_complete`
  output.
- **Edge cases**: Test with plans that have no parallelism (fully sequential),
  fully independent tasks, complex dependency chains, and ambiguous/vague plans.
- **Validation tests**: Verify that Nelson validation catches bad decompositions
  (missing dependencies, tasks that are too large).
