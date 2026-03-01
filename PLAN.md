# AI-Team: Autonomous Agentic Development System

The detailed plan has been absorbed into the spec documents in `docs/specs/`.

See [00-overview.md](docs/specs/00-overview.md) for the full vision, agent roster,
architecture diagram, tech stack, project structure, and implementation phases.

## Spec Index

| Spec | File | Summary |
|------|------|---------|
| 00 | [00-overview.md](docs/specs/00-overview.md) | Vision, agent roster, architecture, tech stack, design decisions, phases |
| 01 | [01-workflow.md](docs/specs/01-workflow.md) | Pipeline lifecycle, branching model, 20-step event flow, dependency dispatch |
| 02 | [02-data-models.md](docs/specs/02-data-models.md) | All Pydantic models, PostgreSQL schema, complete type catalog |
| 03 | [03-learning-and-principles.md](docs/specs/03-learning-and-principles.md) | Learning mode, principle system, adaptive review threshold |
| 04 | [04-communication.md](docs/specs/04-communication.md) | Redis Streams, consumer groups, message envelopes, dead letter queue |
| 05 | [05-infrastructure.md](docs/specs/05-infrastructure.md) | Docker Compose, networking, volumes, Launcher Protocol, image builds, resource limits |
| 06 | [06-observability.md](docs/specs/06-observability.md) | structlog, JSON lines, correlation IDs, event types catalog, audit trail |
| 07 | [07-cost-tracking.md](docs/specs/07-cost-tracking.md) | Per-LLM-call attribution, budget limits (soft/hard), daily ceiling |
| 08 | [08-error-recovery.md](docs/specs/08-error-recovery.md) | Checkpoints, 3 retries, escalation, idempotency |
| 09 | [09-security.md](docs/specs/09-security.md) | Secrets, container hardening, filesystem permissions, prompt injection defense |
| 10 | [10-testing.md](docs/specs/10-testing.md) | Record/replay (VCR), unit/integration/E2E pyramid, CI stages |
| 11 | [11-dev-standards.md](docs/specs/11-dev-standards.md) | ruff + mypy strict + 90% coverage, coding conventions, CI pipeline |
| 12 | [12-repo-connection.md](docs/specs/12-repo-connection.md) | GitHub App + PAT auth, cloning, webhooks, `.ai-team.yaml` full spec |
| 13 | [13-controller.md](docs/specs/13-controller.md) | Pipeline controller: event router, container launcher, watchdog, dependency dispatch |
| 14 | [14-api-server.md](docs/specs/14-api-server.md) | REST + SSE API for TUI client, authentication, endpoints |
| 15 | [15-tui.md](docs/specs/15-tui.md) | TUI (Textual) with pipeline, task, log, consensus, and cost views |
| 16 | [16-nelson.md](docs/specs/16-nelson.md) | Nelson: consensus algorithm, weight learning, prompt templates, cost optimization |
| 17 | [17-julius.md](docs/specs/17-julius.md) | Julius: plan intake, codebase analysis, task decomposition, dependency graph |
| 18 | [18-richelieu.md](docs/specs/18-richelieu.md) | Richelieu: git operations, worktree management, PR creation, conflict resolution |
| 19 | [19-sherlock.md](docs/specs/19-sherlock.md) | Sherlock: codebase inspection, execution plan generation |
| 20 | [20-leonard.md](docs/specs/20-leonard.md) | Leonard: code implementation, test execution, validation, simplification |
| 21 | [21-katherine.md](docs/specs/21-katherine.md) | Katherine: code review, human review scoring, adaptive threshold |
