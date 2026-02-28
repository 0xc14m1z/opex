# AI-Team: Autonomous Agentic Development System

## 1. Vision

A multi-agent system that autonomously develops features for any codebase. The system
breaks down plans into tasks, enriches them with codebase context, implements them in
parallel, reviews the output through multi-LLM consensus, and learns from human feedback
to decide when to escalate.

The system is **provider-agnostic** — every critical decision passes through a consensus
loop where Claude, GPT, and Gemini cross-review each other until they converge.

---

## 2. Agents

### Nelson — Consensus Orchestrator
- **Purpose**: Runs the multi-LLM consensus loop for any decision or evaluation.
- **Used by**: All other agents when they need a high-confidence decision.
- **Mechanism**:
  1. Sends the same prompt to all 3 LLMs (Claude, GPT-4o, Gemini) in parallel.
  2. Each LLM reviews the other two responses and provides feedback.
  3. Cross-review continues until convergence or max rounds (default: 3).
  4. **Termination strategy (hybrid)**:
     - Round 1–2: Check for majority agreement (2 of 3).
     - Round 3: Apply weighted voting (weights learned from historical accuracy).
     - No convergence: Escalate to human with all positions summarized.
- **Outputs**: Consensus result, confidence score, dissenting opinions, round count.
- **State tracked**: Per-provider accuracy weights (SQLite), consensus history for auditing.

### Julius — Task Decomposer
- **Purpose**: Takes a high-level plan (from humans) and decomposes it into minimal,
  independent tasks suitable for parallel implementation.
- **Input**: Plan description + target repo reference.
- **Process**:
  1. Reads the plan and the `.ai-team.yaml` config from the target repo.
  2. Analyzes the codebase at a high level (directory structure, key modules, dependencies).
  3. Breaks the plan into tasks where each task touches the **minimum number of files**.
  4. Builds a **dependency graph** — identifies which tasks can run in parallel vs. which
     must be sequential.
  5. Uses Nelson to validate the decomposition makes sense.
- **Output**: Ordered task list with dependency graph, estimated complexity per task.
- **Key constraint**: Each task must be small enough for a single Leonard to complete and
  for Katherine to review in one pass.

### Sherlock — Task Enricher / Investigator
- **Purpose**: Takes a single task from Julius and deeply inspects the codebase to produce
  a mini execution plan.
- **Input**: Task definition + target repo reference.
- **Process**:
  1. Reads relevant source files, tests, documentation.
  2. Identifies exact files to modify, functions to change, patterns to follow.
  3. Notes existing conventions (naming, error handling, test patterns).
  4. Produces a step-by-step execution plan with all context Leonard needs.
  5. Uses Nelson if there are ambiguous approaches to resolve.
- **Output**: Mini execution plan containing:
  - Files to create/modify (with line-level precision where possible).
  - Code patterns to follow (with examples from the codebase).
  - Test strategy (which tests to add/modify, how to run them).
  - Acceptance criteria derived from the task definition.
  - References to relevant `.ai-team.yaml` guidelines.

### Leonard — Implementer
- **Purpose**: Takes a mini execution plan from Sherlock and implements it.
- **Can run in parallel**: Yes — Julius decides how many based on the dependency graph.
- **Process**:
  1. Richelieu provisions a worktree and branch for this task.
  2. Implements the code changes following Sherlock's plan.
  3. Runs the test suite (commands from `.ai-team.yaml`).
  4. Runs linting/formatting (commands from `.ai-team.yaml`).
  5. Validates output against the acceptance criteria.
  6. Runs code simplification pass (reduce unnecessary complexity).
  7. Commits changes with conventional commit messages.
- **Output**: A branch with passing tests, ready for Katherine's review.
- **On failure**: If tests fail or criteria aren't met after N attempts (default: 3),
  escalate to human with a summary of what went wrong.

### Katherine — Code Reviewer
- **Purpose**: Reviews Leonard's implementation. Decides whether it needs human review.
- **Process**:
  1. Reads the diff, the original task, and Sherlock's execution plan.
  2. Uses Nelson to get multi-LLM consensus on code quality:
     - Correctness (does it match the task requirements?).
     - Style (does it follow codebase conventions?).
     - Safety (security, edge cases, error handling).
     - Simplicity (is there unnecessary complexity?).
  3. If changes needed: sends feedback to Leonard for rework (loop).
  4. If approved: calculates a **human review score** via Nelson:
     - **Novelty** — Does this introduce new patterns, dependencies, or architectural changes?
     - **Complexity** — Cyclomatic complexity delta, number of files changed, logic depth.
     - **AI confidence** — Nelson's consensus confidence from the review.
  5. If score exceeds threshold → flag PR for human review.
  6. If score is below threshold → auto-approve.
- **Learning**: When a human approves or requests changes on PRs that Katherine auto-approved
  or flagged, the scoring weights are adjusted. Over time, Katherine calibrates what truly
  needs human attention.

### Richelieu — Git & Workspace Manager
- **Purpose**: Manages all git operations and workspace provisioning.
- **Responsibilities**:
  1. **Feature start**: Creates a feature branch from the target branch.
  2. **Task start**: Creates a worktree + sub-branch for each Leonard instance.
  3. **Task complete**: Merges Leonard's branch into the feature branch (after Katherine approves).
  4. **Conflict resolution**: Detects merge conflicts. Attempts auto-resolution. Escalates
     to human if manual intervention needed.
  5. **Feature complete**: Opens a PR from the feature branch to the target branch.
  6. **Cleanup**: Removes worktrees and branches after merge.
  7. **Branch alignment**: When a PR is merged upstream, rebases any in-progress feature
     branches.
- **Safety**: All destructive git operations (force push, branch delete) require confirmation
  or are behind configurable safeguards.

---

## 3. Consensus Mechanism (Nelson) — Deep Dive

```
┌─────────────────────────────────────────────────────────────────┐
│                         NELSON LOOP                             │
│                                                                 │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐                     │
│  │  Claude  │    │  GPT-4o │    │ Gemini  │                     │
│  └────┬────┘    └────┬────┘    └────┬────┘                      │
│       │              │              │                            │
│       ▼              ▼              ▼                            │
│   Response A     Response B     Response C                      │
│       │              │              │                            │
│       ▼              ▼              ▼                            │
│  ┌─────────────────────────────────────────┐                    │
│  │         CROSS-REVIEW ROUND              │                    │
│  │  A reviews B+C                          │                    │
│  │  B reviews A+C                          │                    │
│  │  C reviews A+B                          │                    │
│  └─────────────────────────────────────────┘                    │
│       │                                                         │
│       ▼                                                         │
│  ┌─────────────────┐     ┌──────────────┐     ┌──────────────┐ │
│  │ Majority (2/3)? │─YES─▶│ Return result │     │              │ │
│  └───────┬─────────┘     └──────────────┘     │              │ │
│          NO                                    │              │ │
│          ▼                                     │              │ │
│  ┌─────────────────┐                           │              │ │
│  │ Max rounds?     │─NO──▶ Loop again          │              │ │
│  └───────┬─────────┘                           │              │ │
│          YES                                   │              │ │
│          ▼                                     │              │ │
│  ┌─────────────────┐     ┌──────────────┐     │              │ │
│  │ Weighted vote   │─WIN─▶│ Return result │     │              │ │
│  └───────┬─────────┘     └──────────────┘     │              │ │
│          TIE                                   │              │ │
│          ▼                                     │              │ │
│  ┌─────────────────────────────────────────┐  │              │ │
│  │ ESCALATE TO HUMAN                       │  │              │ │
│  │ (with all positions + reasoning)        │  │              │ │
│  └─────────────────────────────────────────┘  │              │ │
└─────────────────────────────────────────────────────────────────┘
```

### Weight Learning
- Each provider starts with equal weight (1.0).
- When a human overrides a consensus decision, the providers that agreed with the human
  get a small weight boost; those that disagreed get a small penalty.
- Weights are normalized so they always sum to 3.0 (for 3 providers).
- Weights are stored in SQLite and evolve over time.
- A minimum weight floor (0.5) prevents any provider from being completely silenced.

---

## 4. Confidence Scoring & Adaptive Learning

### Score Components (Katherine's Human Review Decision)
| Signal              | Source          | Description                                        |
|---------------------|----------------|----------------------------------------------------|
| Novelty             | Nelson          | New patterns, deps, or architectural changes       |
| Complexity          | Heuristics      | Files changed, cyclomatic complexity delta          |
| AI Confidence       | Nelson          | Consensus confidence from the review loop           |
| Historical Accuracy | SQLite          | How often similar PRs needed human intervention    |

### Adaptive Threshold
- Initial threshold is set conservatively (flag most PRs for human review).
- Each human decision (approve / request changes) is recorded.
- A lightweight model (logistic regression or similar) is periodically retrained on
  the decision history to recalibrate the threshold.
- The system trends toward flagging fewer PRs as it learns the team's standards.

---

## 5. Pipeline Flow

```
Human creates plan
        │
        ▼
   ┌─────────┐
   │ JULIUS  │──── Reads .ai-team.yaml, analyzes codebase
   │ Decompose│──── Produces task list + dependency graph
   └────┬────┘     Uses Nelson to validate decomposition
        │
        ▼
   ┌─────────┐
   │RICHELIEU│──── Creates feature branch
   └────┬────┘
        │
        ▼  (for each task, respecting dependency graph)
   ┌─────────┐
   │SHERLOCK │──── Inspects codebase deeply
   │ Enrich  │──── Produces mini execution plan
   └────┬────┘     Uses Nelson if approaches are ambiguous
        │
        ▼  (parallel where dependency graph allows)
   ┌─────────┐
   │RICHELIEU│──── Creates worktree + branch per Leonard
   └────┬────┘
        │
        ▼
   ┌─────────┐
   │LEONARD  │──── Implements, tests, validates, simplifies
   │Implement│──── Commits to task branch
   └────┬────┘
        │
        ▼
   ┌──────────┐
   │KATHERINE │──── Reviews via Nelson consensus
   │  Review  │──── Approves or requests rework (→ loop to Leonard)
   └────┬─────┘     Calculates human review score
        │
        ▼
   ┌─────────┐
   │RICHELIEU│──── Merges task branch → feature branch
   └────┬────┘     Resolves conflicts or escalates
        │
        ▼  (when all tasks complete)
   ┌─────────┐
   │RICHELIEU│──── Opens PR from feature branch → target branch
   └────┬────┘     Cleans up worktrees
        │
        ▼
   Human reviews (if Katherine flagged it) or auto-merges
```

---

## 6. Tech Stack

| Layer               | Technology                          | Purpose                                |
|---------------------|-------------------------------------|----------------------------------------|
| Agent framework     | PydanticAI                          | Typed agents, tool use, structured I/O |
| LLM routing         | LiteLLM                             | Unified API for Claude/GPT/Gemini      |
| Task queue          | Redis (streams or pub/sub)          | Pipeline communication, task dispatch  |
| Durable state       | SQLite                              | Confidence scores, weights, history    |
| Git operations      | GitPython + subprocess              | Branch/worktree management             |
| GitHub integration  | PyGithub / `gh` CLI                 | Issue intake, PR creation, comments    |
| Containerization    | Docker + Docker Compose             | Agent isolation                        |
| Dev workflow        | Makefile                            | Build, test, run, logs                 |
| Package management  | uv                                  | Fast dependency management             |
| Logging             | structlog                           | Structured JSON logging                |
| Dashboard           | FastAPI + HTMX (or similar)         | Observability web UI                   |
| Testing             | pytest                              | Agent and integration testing          |

---

## 7. Project Structure

```
ai-team/
├── PLAN.md
├── Makefile
├── docker-compose.yml
├── pyproject.toml                  # Root workspace config (uv)
├── uv.lock
│
├── core/                           # Shared library (installable package)
│   ├── pyproject.toml
│   └── src/
│       └── ai_team_core/
│           ├── __init__.py
│           ├── llm/
│           │   ├── __init__.py
│           │   ├── client.py       # LiteLLM wrapper, provider config
│           │   └── consensus.py    # Nelson's consensus engine
│           ├── models/
│           │   ├── __init__.py
│           │   ├── task.py         # Task, ExecutionPlan, DependencyGraph
│           │   ├── review.py       # ReviewResult, HumanReviewScore
│           │   └── config.py       # .ai-team.yaml schema (Pydantic)
│           ├── queue/
│           │   ├── __init__.py
│           │   └── redis.py        # Redis queue/pub-sub abstraction
│           ├── state/
│           │   ├── __init__.py
│           │   └── store.py        # SQLite state management
│           ├── git/
│           │   ├── __init__.py
│           │   └── workspace.py    # Git/worktree operations
│           ├── scoring/
│           │   ├── __init__.py
│           │   └── confidence.py   # Adaptive confidence scoring
│           └── logging/
│               ├── __init__.py
│               └── structured.py   # structlog configuration
│
├── agents/                          # Each agent is its own package
│   ├── nelson/
│   │   ├── pyproject.toml
│   │   ├── Dockerfile
│   │   └── src/
│   │       └── nelson/
│   │           ├── __init__.py
│   │           └── agent.py        # Consensus loop orchestration
│   │
│   ├── julius/
│   │   ├── pyproject.toml
│   │   ├── Dockerfile
│   │   └── src/
│   │       └── julius/
│   │           ├── __init__.py
│   │           └── agent.py        # Task decomposition + dependency graph
│   │
│   ├── sherlock/
│   │   ├── pyproject.toml
│   │   ├── Dockerfile
│   │   └── src/
│   │       └── sherlock/
│   │           ├── __init__.py
│   │           └── agent.py        # Codebase analysis + execution planning
│   │
│   ├── leonard/
│   │   ├── pyproject.toml
│   │   ├── Dockerfile
│   │   └── src/
│   │       └── leonard/
│   │           ├── __init__.py
│   │           └── agent.py        # Implementation + testing + validation
│   │
│   ├── katherine/
│   │   ├── pyproject.toml
│   │   ├── Dockerfile
│   │   └── src/
│   │       └── katherine/
│   │           ├── __init__.py
│   │           └── agent.py        # Code review + human review scoring
│   │
│   └── richelieu/
│       ├── pyproject.toml
│       ├── Dockerfile
│       └── src/
│           └── richelieu/
│               ├── __init__.py
│               └── agent.py        # Git/workspace management
│
├── dashboard/                       # Observability UI
│   ├── pyproject.toml
│   ├── Dockerfile
│   └── src/
│       └── dashboard/
│           ├── __init__.py
│           ├── app.py              # FastAPI app
│           ├── routes/
│           └── templates/
│
└── tests/
    ├── unit/                        # Unit tests per agent
    │   ├── test_nelson.py
    │   ├── test_julius.py
    │   ├── test_sherlock.py
    │   ├── test_leonard.py
    │   ├── test_katherine.py
    │   └── test_richelieu.py
    ├── integration/                 # Cross-agent pipeline tests
    │   ├── test_pipeline.py
    │   └── test_consensus.py
    └── conftest.py
```

---

## 8. `.ai-team.yaml` Specification

This file lives in the root of any target repository and tells the agents how to work
with that codebase.

```yaml
# .ai-team.yaml
version: "1"

project:
  name: "my-app"
  description: "Brief description of what this project does"
  language: "python"                  # Primary language
  framework: "fastapi"               # Primary framework (optional)

# Where to find codebase knowledge
knowledge:
  architecture_docs: "docs/architecture.md"
  api_docs: "docs/api/"
  style_guide: "docs/STYLE_GUIDE.md"
  adr_directory: "docs/adr/"          # Architecture Decision Records
  additional:
    - "CONTRIBUTING.md"
    - "docs/patterns.md"

# Commands the agents should use
commands:
  install: "uv sync"
  test: "uv run pytest"
  lint: "uv run ruff check ."
  format: "uv run ruff format ."
  type_check: "uv run mypy src/"
  build: "uv run python -m build"

# Rules the agents must follow
guidelines:
  - "All functions must have type annotations"
  - "Test coverage must not decrease"
  - "No direct database queries outside the repository layer"
  - "Use structured logging, never print()"
  - "Follow conventional commits for commit messages"

# Branches
git:
  default_branch: "main"
  branch_prefix: "ai-team/"          # All AI-created branches use this prefix
  require_pr: true
  auto_merge: false                   # Even if Katherine approves, don't auto-merge

# Scoring overrides
review:
  human_review_threshold: 0.7         # Score above this → human review required
  always_human_review:
    - "migrations/"                   # Always flag changes to these paths
    - "infrastructure/"
    - "*.sql"
```

---

## 9. Implementation Phases

### Phase 0 — Foundation (Week 1-2)
> Get the skeleton standing. No AI yet — just infrastructure.

- [ ] Initialize uv workspace with root `pyproject.toml`.
- [ ] Set up `core/` package structure.
- [ ] Implement Redis queue abstraction (`core/queue/`).
- [ ] Implement SQLite state store (`core/state/`).
- [ ] Implement LiteLLM client wrapper (`core/llm/client.py`).
- [ ] Implement structlog configuration (`core/logging/`).
- [ ] Define Pydantic models for tasks, reviews, configs (`core/models/`).
- [ ] Parse `.ai-team.yaml` into typed config.
- [ ] Set up Docker Compose (Redis, SQLite volume).
- [ ] Set up Makefile (build, test, up, down, logs).
- [ ] Set up pytest with basic test infrastructure.

### Phase 1 — Nelson: The Consensus Engine (Week 3)
> The heart of the system. Everything else depends on this.

- [ ] Implement parallel LLM prompt dispatch (Claude + GPT + Gemini).
- [ ] Implement cross-review round logic.
- [ ] Implement hybrid termination (majority → weighted → escalate).
- [ ] Implement provider weight storage and learning.
- [ ] Implement consensus result model with confidence score.
- [ ] Add structured logging for every consensus round.
- [ ] Write unit tests with mocked LLM responses.
- [ ] Write integration test with real LLM calls.
- [ ] Dockerize Nelson.

### Phase 2 — Richelieu: Git Backbone (Week 4)
> Without git management, no agent can do real work.

- [ ] Implement feature branch creation.
- [ ] Implement worktree creation/cleanup per task.
- [ ] Implement branch merging (task → feature).
- [ ] Implement conflict detection and escalation.
- [ ] Implement PR creation via GitHub API.
- [ ] Implement branch alignment on upstream merge.
- [ ] Add safety guards on destructive operations.
- [ ] Write tests against a test git repo.
- [ ] Dockerize Richelieu.

### Phase 3 — Julius: Task Decomposition (Week 5-6)
> Turn plans into parallelizable work.

- [ ] Implement plan intake (parse plan from GitHub issue or direct input).
- [ ] Implement codebase high-level analysis (structure, modules, deps).
- [ ] Implement task decomposition with minimal-file-change constraint.
- [ ] Implement dependency graph builder.
- [ ] Integrate Nelson for decomposition validation.
- [ ] Output task list to Redis queue.
- [ ] Write tests with sample plans and expected decompositions.
- [ ] Dockerize Julius.

### Phase 4 — Sherlock: Task Enrichment (Week 7)
> Deep codebase understanding for each task.

- [ ] Implement targeted codebase reading (files, functions, patterns).
- [ ] Implement mini execution plan generation.
- [ ] Integrate with `.ai-team.yaml` guidelines.
- [ ] Integrate Nelson for ambiguous approach resolution.
- [ ] Write tests with sample tasks and expected execution plans.
- [ ] Dockerize Sherlock.

### Phase 5 — Leonard: Implementation (Week 8-9)
> The coding agent. Most complex, most risk.

- [ ] Implement code generation from mini execution plan.
- [ ] Implement test execution (run commands from `.ai-team.yaml`).
- [ ] Implement lint/format execution.
- [ ] Implement validation against acceptance criteria.
- [ ] Implement retry logic (max N attempts).
- [ ] Implement code simplification pass.
- [ ] Implement escalation on persistent failure.
- [ ] Integration with Richelieu for worktree lifecycle.
- [ ] Write tests with controlled codebases and expected outputs.
- [ ] Dockerize Leonard (needs code execution capabilities).

### Phase 6 — Katherine: Code Review (Week 10)
> Quality gate with adaptive human escalation.

- [ ] Implement diff analysis.
- [ ] Integrate Nelson for multi-LLM code review consensus.
- [ ] Implement review feedback → Leonard rework loop.
- [ ] Implement human review scoring (novelty, complexity, confidence).
- [ ] Implement scoring via Nelson consensus.
- [ ] Implement adaptive threshold learning.
- [ ] Record human decisions for threshold calibration.
- [ ] Write tests for scoring edge cases.
- [ ] Dockerize Katherine.

### Phase 7 — Pipeline Integration (Week 11-12)
> Wire everything together end-to-end.

- [ ] Implement full pipeline orchestration (Julius → Sherlock → Leonard → Katherine).
- [ ] Implement parallel Leonard dispatch based on dependency graph.
- [ ] Implement Richelieu integration at every stage.
- [ ] End-to-end integration test with a real repo and real LLM calls.
- [ ] Error handling and graceful degradation across the pipeline.
- [ ] GitHub webhook integration for issue intake.

### Phase 8 — Dashboard & Observability (Week 13)
> See what the agents are doing.

- [ ] FastAPI app with basic routes.
- [ ] Pipeline status view (which agents are active, what they're working on).
- [ ] Task list view with dependency graph visualization.
- [ ] Consensus history view (see how LLMs debated).
- [ ] Confidence score trends over time.
- [ ] Log viewer with filtering.
- [ ] Dockerize dashboard.

### Phase 9 — Hardening & Polish (Week 14+)
> Production-readiness.

- [ ] Rate limiting for LLM API calls.
- [ ] Cost tracking per pipeline run.
- [ ] Retry / circuit breaker for external API calls.
- [ ] Graceful shutdown handling for all agents.
- [ ] Documentation (setup guide, architecture overview, config reference).
- [ ] CI pipeline for the ai-team repo itself.
- [ ] Security review (secrets handling, container permissions).

---

## 10. Key Design Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Agent isolation | Docker containers | Safety — Leonard runs arbitrary code. Blast radius control. |
| Communication | Redis queue + direct Nelson calls | Queue decouples pipeline stages; Nelson is a utility, not a pipeline step. |
| State storage | Redis + SQLite | Redis for ephemeral/real-time, SQLite for durable data. No infra overhead. |
| LLM routing | LiteLLM | Single API for 3+ providers. Fallback, retries, streaming built-in. |
| Agent framework | PydanticAI | Typed tool use, structured outputs, minimal magic. Maximum control. |
| Consensus termination | Hybrid (majority → weighted → human) | Balances speed (quick consensus) with accuracy (weighted tiebreak) and safety (human escalation). |
| Task sizing | Minimum file changes | Speeds up Leonard, simplifies Katherine's review, reduces merge conflicts. |
| Parallelism | Julius-managed dependency graph | Smart parallelism — not just "run everything at once" but respecting dependencies. |
| Human review | Adaptive scoring via Nelson | Starts conservative, learns the team's standards over time. |
| Project config | `.ai-team.yaml` per repo | Clean interface between the agent system and any target codebase. |
| Package management | uv workspaces | Fast, modern Python tooling. Workspace support for monorepo. |

---

## 11. Open Questions & Future Considerations

- **Long-term memory**: Should agents build persistent knowledge about a codebase across
  runs? (e.g., "last time we changed this module, tests in X broke")
- **Cost optimization**: Nelson's consensus loop is 3x the LLM cost. When is it worth
  skipping consensus for low-stakes decisions?
- **Streaming**: Should agents stream their progress to the dashboard in real-time?
- **Multi-repo**: Can the system work across multiple repos in a single feature?
- **Kubernetes migration**: Docker Compose → k8s when scaling is needed.
- **Plugin system**: Allow custom agents to be added by users.
- **Notification channels**: Slack/Discord integration for human escalation.
- **Caching**: Cache LLM responses for identical prompts across consensus rounds?
