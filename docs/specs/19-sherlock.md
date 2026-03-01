# 19 — Sherlock (Task Enricher / Investigator)

> **Migrated from**: `PLAN.md` section 2 (Sherlock description)

## Overview

Sherlock takes a single task from Julius's decomposition and deeply inspects the
target codebase to produce a mini execution plan. This plan contains everything
Leonard needs to implement the task: exact files to modify, code patterns to
follow, test strategy, acceptance criteria, and relevant principles. Sherlock
bridges the gap between Julius's high-level task descriptions and Leonard's
concrete implementation work.

---

## Responsibilities

- Read and understand the task definition from Julius.
- Deeply inspect relevant source files, tests, and documentation in the target repo.
- Identify exact files to create or modify (with line-level precision where possible).
- Note existing conventions (naming, error handling, test patterns, imports).
- Produce a step-by-step execution plan with all context Leonard needs.
- Include relevant implementation principles from `.ai-team/principles/implementation/`.
- Use Nelson (spec 16) when there are ambiguous implementation approaches to resolve.
- Include acceptance criteria derived from the task definition.
- Reference relevant `.ai-team.yaml` guidelines in the execution plan.

---

## Lifecycle

- **Spawned when**: A task becomes `ready` in the dependency graph (all dependencies completed). The orchestrator (spec 13-orchestrator) spawns one Sherlock per task.
- **Exits when**: Publishes `task_enriched` to `pipeline:{id}:enriched`.
- **Parallelism**: Up to `max_parallel_sherlocks` (default: 5) running simultaneously.

---

## Input / Output

### Input

- **Message type**: Task definition (from `task_created` in `pipeline:{id}:tasks`)
- **Redis stream**: `pipeline:{id}:tasks`
- **Environment variables**: `PIPELINE_ID`, `TASK_ID`
- **Data available**:
  - Task definition from Julius (description, expected files, complexity estimate).
  - Target repo at `/workspace` (read-only access).
  - `.ai-team.yaml` configuration (parsed).
  - Principles from `.ai-team/principles/` (read-only).

### Output

- **Message type**: `task_enriched`
- **Redis stream**: `pipeline:{id}:enriched`
- **Payload**: Mini execution plan containing:
  - Files to create/modify (with line-level precision where possible).
  - Code patterns to follow (with examples from the codebase).
  - Test strategy (which tests to add/modify, how to run them).
  - Acceptance criteria derived from the task definition.
  - References to relevant `.ai-team.yaml` guidelines.
  - Relevant implementation principles.

---

## Algorithm / Process

1. **Read task definition**: Parse the task from Julius, understand the scope and intent.
2. **Identify relevant files**:
   - Scan the codebase for files related to the task (by name, imports, module structure).
   - Read relevant source files to understand current implementation.
   - Read relevant tests to understand testing patterns.
   - Read documentation referenced in `.ai-team.yaml` knowledge section.
3. **Note conventions**:
   - Identify naming patterns (variable names, function names, class names).
   - Identify error handling patterns (exception types, error messages, logging).
   - Identify test patterns (test structure, fixtures, mocking approach).
   - Identify import conventions and module boundaries.
4. **Gather principles**:
   - Query `.ai-team/principles/implementation/` for relevant principles.
   - Filter principles to those applicable to the task at hand.
   - Include principle content in the execution plan context.
5. **Resolve ambiguity (if needed)**:
   - If multiple implementation approaches exist, submit a `consensus_request` to Nelson (spec 16).
   - Nelson evaluates the approaches and returns a recommendation.
   - Incorporate the recommendation into the execution plan.
6. **Produce execution plan**:
   - List files to create/modify with specific locations.
   - Describe code changes step by step.
   - Specify which tests to add or modify.
   - Define acceptance criteria (what "done" looks like).
   - Include examples from the codebase to follow.
   - Reference relevant guidelines from `.ai-team.yaml`.
7. **Publish**: Publish `task_enriched` with the complete execution plan.

---

## Interaction with Other Agents

- **Julius (spec 17)**: Sherlock receives task definitions produced by Julius. Julius provides the "what"; Sherlock provides the "how".
- **Nelson (spec 16)**: Sherlock uses Nelson when there are ambiguous implementation approaches. Nelson evaluates and resolves the ambiguity via consensus.
- **Leonard (spec 20-leonard)**: Sherlock's execution plan is the primary input for Leonard. The plan must be detailed enough for Leonard to implement without further codebase exploration.
- **Orchestrator (spec 13-orchestrator)**: The orchestrator spawns Sherlock when a task is ready and consumes the `task_enriched` output to proceed with Richelieu (worktree creation) and then Leonard.

---

## Principles Integration

Sherlock is the primary channel through which principles reach Leonard's
implementation context:

1. Queries `.ai-team/principles/implementation/` for all active principles.
2. Filters to principles relevant to the current task (by file patterns, module, agent).
3. Includes relevant principles directly in the execution plan, with examples.
4. Leonard's system prompt will include these principles when implementing.

For example, if there is a principle requiring the repository pattern for all data
access, and the task involves adding a new data source, Sherlock includes:
- The principle text.
- Examples of existing repository implementations from the codebase.
- A note that Leonard must follow this pattern.

---

## System Prompt

<!-- TODO: Define the system prompt template for Sherlock -->
<!-- Sherlock's system prompt should establish its role as a codebase investigator,
     emphasize thoroughness in identifying relevant files and patterns,
     and stress that the execution plan must be complete enough for
     Leonard to implement without additional codebase exploration. -->

---

## PydanticAI Tools

<!-- TODO: Define the tools available to Sherlock -->

Expected tools:

- **File system read**: Read files in the target repo (read-only).
- **Directory listing**: List directory contents to navigate the project structure.
- **Search/grep**: Search for patterns, function names, class references across the codebase.
- **AST parsing** (optional): Parse source files into ASTs for precise function/class identification.
- **Configuration reader**: Parse `.ai-team.yaml` and extract relevant settings.
- **Principles reader**: Read and filter principles from `.ai-team/principles/`.
- **Nelson request**: Submit consensus requests for ambiguous approach resolution.
- **Redis publish**: Publish `task_enriched` messages.

---

## Error Handling

- **Insufficient context**: If Sherlock cannot find enough information in the codebase
  to produce a complete execution plan, it flags the task with a note about what
  is missing and proceeds with the best plan possible. Leonard will encounter the
  gap and can escalate.
- **Nelson failure**: If Nelson is unavailable for ambiguity resolution, Sherlock
  picks the most conservative approach and documents the alternatives for Leonard.
- **Container crash**: The orchestrator respawns Sherlock with the same task. Since
  Sherlock is stateless (reads from repo and Redis), a respawn starts enrichment
  from scratch.
- **Large codebase**: If the codebase is very large, Sherlock focuses on the files
  and directories most relevant to the task, using the task definition and module
  structure to narrow scope.

---

## Resource Profile

| Metric             | Value  | Rationale                                |
|--------------------|--------|------------------------------------------|
| CPU limit          | 1.0    | File reading, context building           |
| Memory limit       | 1G     | May need to hold many file contents      |
| CPU reservation    | 0.25   |                                          |
| Memory reservation | 256M   |                                          |

Overridable via `.ai-team.yaml` `resources.sherlock` section (see spec 12).

---

## Configuration

- **`.ai-team.yaml`**: Knowledge references (architecture docs, style guide, ADRs), guidelines, commands.
- **Environment variables**: `PIPELINE_ID`, `TASK_ID`, `OPENROUTER_API_KEY`, `DATABASE_URL`, `REDIS_URL`.
- **LLM model**: Uses `llm.default_model` from `.ai-team.yaml` (or per-agent override `llm.overrides.sherlock`).

---

## Testing Strategy

- **Unit tests**: Test execution plan generation with sample tasks and mock codebase
  structures. Verify that Sherlock identifies the correct files and patterns.
- **Integration tests**: Use VCR cassettes (see spec 10) to record real LLM calls
  during codebase analysis. Verify the full flow from task input to `task_enriched` output.
- **Principle inclusion tests**: Verify that relevant principles are correctly filtered
  and included in execution plans.
- **Edge cases**: Test with tasks that span multiple modules, tasks in unfamiliar
  codebases (no prior principles), and tasks where Nelson is needed for ambiguity resolution.
