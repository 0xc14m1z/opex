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
│  ┌───────────┐  ┌────────────┐  ┌────────┐  ┌────────────┐  │
│  │   Redis   │  │ PostgreSQL │  │  Loki  │  │  Grafana   │  │
│  │ (streams) │  │  (state)   │  │ (logs) │  │  (:3000)   │  │
│  └─────┬─────┘  └──────┬─────┘  └───┬────┘  └────────────┘  │
│        │               │            │                        │
│  ┌─────┴───────────────┴────────────┴─────┐                  │
│  │            Orchestrator                │                  │
│  │      (event router + launcher)         │                  │
│  └─────────────────┬──────────────────────┘                  │
│                    │ spawns ephemeral containers              │
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

Drop an `.opex.yaml` in your target repository:

```yaml
version: "1"

project:
  name: "my-app"
  language: "python"
  framework: "fastapi"

commands:
  install: "uv sync"
  test: "uv run pytest"
  lint: "uv run ruff check ."

llm:
  default_model: provider/model-name
  consensus:
    models:
      - provider-a/model-1
      - provider-b/model-2
      - provider-c/model-3

budget:
  soft_limit: 5.00
  hard_limit: 20.00

review:
  human_review_threshold: 0.7
```

## Design Principles

The system is built on eight validated planning principles:

1. **Happy Path Is Not a Plan** — all failure modes are designed, not just the success case
2. **Every State Needs Entry & Exit** — no dead-end states
3. **Boundaries Must Be Explicit** — component interactions name specific source and target
4. **Parallel Means Conflicting** — parallelism includes conflict detection and resolution
5. **Name the Mechanism** — specs describe *how*, not just *that*
6. **Limits Prevent Runaway** — every loop, retry, and queue is bounded
7. **Cross-Reference or Contradict** — one spec owns each concept
8. **No Silent Failures** — every failure escalates to a human with full context

## Project Status

Opex is in the **specification phase**. All 22 design specs are written and cross-referenced.
Implementation begins with Phase 0 (foundation: uv workspace, core library, Docker Compose, test infrastructure).

See [`docs/specs/00-overview.md`](docs/specs/00-overview.md) for the full system design.

## License

All rights reserved.
