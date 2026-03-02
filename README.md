# Opex

**Autonomous multi-agent system that turns high-level plans into pull requests.**

Opex decomposes a feature plan into parallelizable tasks, implements them through
ephemeral AI agents, reviews every output via multi-LLM consensus, and learns
from human feedback to decide when to escalate. Multiple LLMs cross-review
each other until they converge — no single model is trusted alone.

---

## How It Works

```
Plan → Decompose → Enrich → Implement → Review → PR
```

1. **You write a plan** — a plain-language description of the feature you want.
2. **Julius** decomposes it into minimal, independent tasks with a dependency graph.
3. **Richelieu** creates a feature branch and per-task worktrees.
4. **Sherlock** inspects the codebase for each task and produces a file-level execution plan.
5. **Leonard** implements, tests, and validates — multiple instances run in parallel.
6. **Katherine** reviews via Nelson consensus (multi-LLM cross-review). If rework is needed, Leonard iterates.
7. **Richelieu** merges task branches and opens a feature PR targeting `main`.

Every failure escalates to a human with full context — the system never silently gives up.

## Agent Roster

| Agent | Role | Description |
|---|---|---|
| **Orchestrator** | Pipeline driver | Deterministic event router. Spawns and monitors all agents. No LLM. |
| **Nelson** | Consensus | Runs multi-LLM consensus loops. Multiple models cross-review until convergence. |
| **Julius** | Decomposer | Breaks plans into minimal tasks with dependency graphs. |
| **Sherlock** | Enricher | Deep codebase inspection → file-level execution plans. |
| **Leonard** | Implementer | Writes code, runs tests, validates against acceptance criteria. |
| **Katherine** | Reviewer | Reviews implementations via consensus. Adaptive scoring learns when to escalate. |
| **Richelieu** | Git manager | Branches, worktrees, merges, conflict resolution, PR creation. No LLM. |

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│  Docker Compose Network                                      │
│                                                              │
│  ┌───────────┐  ┌────────────┐  ┌────────┐  ┌────────────┐   │
│  │   Redis   │  │ PostgreSQL │  │  Loki  │  │  Grafana   │   │
│  │ (streams) │  │  (state)   │  │ (logs) │  │  (:3000)   │   │
│  └─────┬─────┘  └──────┬─────┘  └───┬────┘  └────────────┘   │
│        │               │            │                        │
│  ┌─────┴───────────────┴────────────┴─────┐                  │
│  │            Orchestrator                │                  │
│  │      (event router + launcher)         │                  │
│  └─────────────────┬──────────────────────┘                  │
│                    │ spawns ephemeral containers             │
│  ┌─────────────────┼──────────────────────────────┐          │
│  │  Nelson   Julius   Sherlock   Leonard   ...    │          │
│  │  Katherine   Richelieu                         │          │
│  └────────────────────────────────────────────────┘          │
│                                                              │
│  ┌────────────────┐                                          │
│  │   API Server   │ ◄── REST + SSE (:8080)                   │
│  └────────────────┘                                          │
└──────────────────────────────────────────────────────────────┘
         ▲
         │
    ┌────┴─────┐
    │   TUI    │  (pip-installable Textual client)
    └──────────┘
```

- **Redis Streams** for all inter-agent communication (durable, replayable)
- **PostgreSQL** for persistent state (asyncpg, concurrent access)
- **Loki + Grafana** for structured log aggregation and inspection
- **Docker** isolates every agent — Leonard runs arbitrary code safely
- **API Server** is the only externally exposed service

## Tech Stack

| Layer | Technology |
|---|---|
| Agent framework | PydanticAI |
| LLM routing | LiteLLM + OpenRouter |
| Communication | Redis Streams |
| State | PostgreSQL (asyncpg) |
| Containerization | Docker Compose |
| Logging | structlog → Loki |
| Dashboard | Textual TUI + Grafana |
| Testing | pytest + testcontainers + VCR |
| Package management | uv (workspaces) |
| Dev standards | ruff + mypy strict + 90% coverage |

## Configuration

Drop an `.opex.yaml` in your target repository. This is the **repo-owner's** configuration
— it controls how Opex behaves for this specific project. System-level secrets (API keys,
credentials) live in `.env` instead.

### Full example

```yaml
version: "1"

# ── Project metadata ──────────────────────────────────────────────
project:
  name: "my-app"
  description: "E-commerce API backend"
  language: "python"
  framework: "fastapi"          # optional
  python_version: "3.12"        # optional

# ── Codebase knowledge ────────────────────────────────────────────
knowledge:
  architecture_docs: "docs/architecture.md"
  api_docs: "docs/api/"
  style_guide: "CONTRIBUTING.md"
  adr_directory: "docs/adr/"
  additional:
    - "docs/patterns.md"
    - "docs/conventions.md"

# ── Commands agents execute ───────────────────────────────────────
commands:
  install: "uv sync"
  test: "uv run pytest"
  lint: "uv run ruff check ."         # optional
  format: "uv run ruff format ."      # optional
  type_check: "uv run mypy src/"      # optional
  build: "uv run python -m build"     # optional
  custom:                              # optional, arbitrary commands
    migrate: "alembic upgrade head"

# ── Coding guidelines ─────────────────────────────────────────────
guidelines:
  - "All functions must have type annotations"
  - "Use dependency injection, no global state"
  - "Every endpoint must have an integration test"

# ── Protected paths (agents cannot modify) ────────────────────────
protected_paths:
  - "migrations/"
  - "infrastructure/"
  - ".github/"

# ── Git behavior ──────────────────────────────────────────────────
git:
  default_branch: "main"
  branch_prefix: "opex/"              # prefix for AI-created branches
  require_pr: true                    # never push directly to default branch
  auto_merge: false                   # auto-merge approved PRs
  conventional_commits: true          # enforce feat:, fix:, etc.

# ── Review and confidence scoring ─────────────────────────────────
review:
  confidence_threshold: 0.7           # below this → human review required
  always_human_review:                # paths that always need human eyes
    - "migrations/"
    - "*.sql"
  never_auto_approve:                 # high-confidence changes still flagged
    - "security-related changes"

# ── GitHub issue intake ───────────────────────────────────────────
intake:
  labels:
    trigger: "opex"                   # label that triggers task creation
    in_progress: "opex:working"
    done: "opex:done"
    needs_human: "opex:needs-human"
  issue_template: null                # optional markdown template

# ── Budget (per-task, in USD) ─────────────────────────────────────
budget:
  soft_limit_per_task: 5.00           # warning threshold
  hard_limit_per_task: 20.00          # hard stop
  # Daily hard limit is set via DAILY_BUDGET_HARD in .env (system-level)

# ── Task retries ──────────────────────────────────────────────────
retries:
  max_task_retries: 2                 # 2 retries = 3 total attempts
  cleanup_ttl_hours: 48               # auto-cleanup failed branches

# ── LLM models ────────────────────────────────────────────────────
llm:
  default_model: "anthropic/claude-sonnet-4"
  consensus:
    models:                           # min 2 for meaningful consensus
      - "anthropic/claude-sonnet-4"
      - "openai/gpt-4o"
      - "google/gemini-2.0-flash"
    max_rounds: 3                     # max cross-review iterations
  overrides:                          # per-agent model overrides
    leonard: "anthropic/claude-sonnet-4"

# ── Resource limits (per-agent overrides) ─────────────────────────
resources:
  leonard:
    cpu_limit: 2.0
    memory_limit: "4G"
  sherlock:
    cpu_limit: 1.0
    memory_limit: "2G"

# ── Secret allowlist extensions ───────────────────────────────────
secrets:
  leonard:
    additional: ["CUSTOM_API_KEY"]    # extra env vars for this agent

# ── Learning mode ─────────────────────────────────────────────────
learning:
  enabled: true                       # enable for new pipelines
  auto_disable_after: null            # auto-disable after N tasks
```

### Section reference

| Section | Purpose | Required |
|---|---|---|
| `project` | Project metadata — name, language, framework | Yes |
| `knowledge` | Paths to architecture docs, style guides, ADRs for agent context | No |
| `commands` | Shell commands agents run (install, test, lint, format, etc.) | Yes (`install`, `test`) |
| `guidelines` | Coding rules agents must follow | No |
| `protected_paths` | Files/directories agents cannot modify | No |
| `git` | Branch prefix, default branch, PR and commit conventions | No |
| `review` | Confidence threshold, paths requiring human review | No |
| `intake` | GitHub issue labels that trigger/track pipelines | No |
| `budget` | Per-task soft/hard cost limits in USD | No |
| `retries` | Max task retries and failed-branch cleanup TTL | No |
| `llm` | Default model, consensus model list, per-agent overrides | No |
| `resources` | Per-agent CPU/memory limits (overrides infrastructure defaults) | No |
| `secrets` | Per-agent additional environment variable allowlists | No |
| `learning` | Learning mode toggle and auto-disable threshold | No |

All model IDs use OpenRouter format: `{provider}/{model-name}`.

See [`docs/specs/12-repo-connection.md`](docs/specs/12-repo-connection.md) for the full specification.

## Design Principles

The system is built on nine validated planning principles:

1. **Happy Path Is Not a Plan** — all failure modes are designed, not just the success case
2. **Every State Needs Entry & Exit** — no dead-end states
3. **Boundaries Must Be Explicit** — component interactions name specific source and target
4. **Parallel Means Conflicting** — parallelism includes conflict detection and resolution
5. **Name the Mechanism** — specs describe *how*, not just *that*
6. **Limits Prevent Runaway** — every loop, retry, and queue is bounded
7. **Cross-Reference or Contradict** — one spec owns each concept
8. **No Silent Failures** — every failure escalates to a human with full context
9. **Stability Over Wasted Work** — prefer clean shutdown over fast shutdown; never kill mid-operation

## Project Status

Opex is in the **specification phase**. All 22 design specs are written and cross-referenced.
Implementation begins with Phase 0 (foundation: uv workspace, core library, Docker Compose, test infrastructure).

See [`docs/specs/00-overview.md`](docs/specs/00-overview.md) for the full system design.

## License

All rights reserved.
