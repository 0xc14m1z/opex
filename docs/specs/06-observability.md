# 06 — Observability

> **Migrated from**: `docs/specs/06-logging.md`

## Overview

Every operation in the system is logged — every LLM call, file read, git operation,
message sent, and decision made. Logs are structured JSON, shipped to Redis Streams
for real-time consumption by the TUI, and persisted to Loki for historical analysis.

**Philosophy**: Log everything, always. Storage is cheap. Debugging agentic systems
without full logs is nearly impossible.

---

## Stack

- **Library**: [structlog](https://www.structlog.org/) — structured, contextual logging.
- **Format**: JSON lines (one JSON object per log line).
- **Transport**: Dual output — stdout (captured by Docker/Loki driver) + Redis Stream.
- **Aggregation**: Loki (via Docker Loki logging driver) — no Promtail or sidecar needed.
- **Dashboards**: Grafana — LogQL querying, pre-built dashboards, alerting.
- **Retention**: Loki default retention. Log files not written to disk by agents.

## Log Schema

Every log entry contains these base fields:

```json
{
  "timestamp": "2025-03-15T14:32:05.123Z",
  "level": "info",
  "agent": "leonard-1",
  "task_id": "task-42",
  "pipeline_id": "pipe-7",
  "correlation_id": "corr-abc123",
  "event": "llm_call_completed",
  "message": "Generated auth endpoint implementation",
  "data": { }
}
```

| Field            | Type   | Description                                        |
|-----------------|--------|----------------------------------------------------|
| `timestamp`     | string | ISO 8601 with milliseconds, UTC                   |
| `level`         | string | `debug`, `info`, `warning`, `error`, `critical`   |
| `agent`         | string | Agent name + instance (e.g., `leonard-1`)          |
| `task_id`       | string | Current task being worked on (null if none)        |
| `pipeline_id`   | string | The overall pipeline run ID                        |
| `correlation_id`| string | Traces a request across agents (e.g., Nelson call) |
| `event`         | string | Machine-readable event type (see Event Types)      |
| `message`       | string | Human-readable description                         |
| `data`          | object | Event-specific payload                             |

## Correlation IDs

Correlation IDs trace a logical operation across multiple agents:

```
Julius creates task → correlation_id = "decompose-pipe7-task42"
  ↓
Sherlock enriches same task → same correlation_id
  ↓
Leonard implements → same correlation_id
  ↓
Katherine reviews → same correlation_id
```

When an agent calls Nelson, a **child correlation ID** is created:
```
katherine (corr-abc123) → nelson (corr-abc123:consensus-7)
```

This allows the TUI to show the full trace of any operation.

## Event Types

### Agent Lifecycle
| Event                    | Level | Data                                  |
|-------------------------|-------|---------------------------------------|
| `agent_started`         | info  | `{version, config}`                   |
| `agent_stopped`         | info  | `{reason, uptime_seconds}`            |
| `agent_error`           | error | `{error_type, message, traceback}`    |
| `agent_health_check`    | debug | `{status, memory_mb, cpu_percent}`    |

### LLM Calls
| Event                    | Level | Data                                  |
|-------------------------|-------|---------------------------------------|
| `llm_call_started`      | debug | `{provider, model, prompt_tokens_est}`|
| `llm_call_completed`    | info  | `{provider, model, prompt_tokens, completion_tokens, latency_ms, cost_usd}` |
| `llm_call_failed`       | error | `{provider, model, error, retries}`   |
| `llm_call_retried`      | warn  | `{provider, model, attempt, reason}`  |

### Consensus (Nelson)
| Event                    | Level | Data                                  |
|-------------------------|-------|---------------------------------------|
| `consensus_started`     | info  | `{request_id, requester, prompt_preview}` |
| `consensus_round`       | info  | `{round, provider_decisions, agreements}` |
| `consensus_reached`     | info  | `{status, confidence, rounds_taken, cost}` |
| `consensus_escalated`   | warn  | `{positions, reason}`                 |

### Task Pipeline
| Event                    | Level | Data                                  |
|-------------------------|-------|---------------------------------------|
| `task_created`          | info  | `{task_id, title, dependencies}`      |
| `task_enriched`         | info  | `{task_id, plan_steps, files_identified}` |
| `task_started`          | info  | `{task_id, agent, worktree_path}`     |
| `task_completed`        | info  | `{task_id, agent, files_changed, tests_passed}` |
| `task_failed`           | error | `{task_id, agent, reason, attempt}`   |
| `task_escalated`        | warn  | `{task_id, reason, context}`          |

### Code Operations
| Event                    | Level | Data                                  |
|-------------------------|-------|---------------------------------------|
| `file_read`             | debug | `{path, lines}`                       |
| `file_written`          | info  | `{path, lines_changed}`               |
| `test_run`              | info  | `{command, passed, failed, duration}`  |
| `lint_run`              | info  | `{command, issues_found}`              |

### Git Operations
| Event                    | Level | Data                                  |
|-------------------------|-------|---------------------------------------|
| `branch_created`        | info  | `{branch, from_ref}`                  |
| `worktree_created`      | info  | `{path, branch, task_id}`             |
| `worktree_removed`      | info  | `{path, reason}`                      |
| `merge_completed`       | info  | `{source, target, conflicts}`         |
| `merge_conflict`        | warn  | `{source, target, conflicting_files}` |
| `pr_created`            | info  | `{pr_number, url, title}`             |

### Review
| Event                    | Level | Data                                  |
|-------------------------|-------|---------------------------------------|
| `review_started`        | info  | `{task_id, files_in_diff}`            |
| `review_completed`      | info  | `{task_id, decision, score}`          |
| `human_review_flagged`  | info  | `{task_id, score, reasons}`           |
| `human_decision`        | info  | `{task_id, decision, feedback}`       |

### Budget
| Event                    | Level | Data                                  |
|-------------------------|-------|---------------------------------------|
| `budget_warning`        | warn  | `{scope, current, limit, percent}`    |
| `budget_exceeded`       | error | `{scope, current, limit, action}`     |

## structlog Configuration

```python
import structlog

def configure_logging(agent_name: str, log_level: str = "DEBUG") -> None:
    structlog.configure(
        processors=[
            structlog.contextvars.merge_contextvars,
            structlog.processors.add_log_level,
            structlog.processors.TimeStamper(fmt="iso", utc=True),
            structlog.processors.StackInfoRenderer(),
            structlog.processors.format_exc_info,
            # Add agent name to every log entry
            _add_agent_name(agent_name),
            structlog.processors.JSONRenderer(),
        ],
        wrapper_class=structlog.make_filtering_bound_logger(
            getattr(logging, log_level.upper())
        ),
        context_class=dict,
        logger_factory=structlog.PrintLoggerFactory(),
    )
```

## Log Output Destinations

Each agent writes logs to two destinations, each serving a different purpose:

1. **Stdout** (captured by Docker): For `docker compose logs` and standard container
   log management. Automatically forwarded to Loki via the Docker Loki logging driver
   (see spec 05 for Docker Compose configuration).
2. **Redis Stream** (`logs:{agent_name}`): For real-time TUI consumption.

The TUI subscribes to all `logs:*` streams and merges them into a unified view.

### Three-destination architecture

| Destination     | Purpose                                      |
|-----------------|----------------------------------------------|
| Loki / Grafana  | Historical querying, dashboards, alerting    |
| Redis Streams   | Real-time TUI consumption                    |
| Stdout          | Docker native logs (`docker compose logs`)   |

## Log Levels in Practice

| Level    | When to use                                               |
|----------|-----------------------------------------------------------|
| DEBUG    | File reads, individual LLM prompt details, internal state |
| INFO     | Task lifecycle events, LLM call completions, git ops      |
| WARNING  | Budget approaching limit, consensus escalation, retries   |
| ERROR    | Agent crash, LLM call failure, test failure               |
| CRITICAL | System-wide failure, data corruption, security issue      |

## Querying Logs

Logs can be queried through:
- **Grafana**: LogQL queries against Loki — the primary tool for log exploration,
  dashboards, and alerting.
- **TUI**: Filter by agent, level, task, and time range (real-time via Redis Streams).
- **CLI**: `make logs AGENT=leonard-1 LEVEL=error TASK=42`

---

## Grafana / Loki Integration

Loki is the primary log aggregation and querying backend. It ingests logs
automatically via Docker's Loki logging driver — no Promtail or sidecar needed.
Each container's stdout is forwarded directly to Loki by the Docker daemon.

### How the Docker Loki logging driver works

The Loki logging driver replaces Docker's default `json-file` driver. When
configured, the Docker daemon sends container stdout/stderr directly to the Loki
HTTP push endpoint.

**For Docker Compose services** (Redis, PostgreSQL, controller, API server), the
logging driver is configured via the YAML anchor in `docker-compose.yml` (see
spec 05 for the canonical Docker Compose config):

```yaml
x-logging: &default-logging
  driver: loki
  options:
    loki-url: "http://loki:3100/loki/api/v1/push"
    loki-retries: "2"
    loki-batch-size: "400"
    labels: "service={{.Name}}"
```

**For ephemeral agent containers**, the controller's Launcher explicitly configures
the Loki logging driver on each `container.run()` call, since ephemeral containers
do not inherit Docker Compose's logging configuration (see spec 05 for Launcher
implementation):

```python
# In DockerLauncher.launch()
log_config = docker.types.LogConfig(
    type="loki",
    config={
        "loki-url": "http://loki:3100/loki/api/v1/push",
        "loki-retries": "2",
        "loki-batch-size": "400",
        "labels": f"service={agent},pipeline={pipeline_id},task={task_id}",
    },
)
```

**Label strategy**: Every container's logs are tagged with:
- `service` — the agent name (e.g., `leonard`, `nelson`)
- `pipeline` — the pipeline ID
- `task` — the task ID (empty for pipeline-level agents like Julius)

**Services that do NOT use the Loki driver**:
- **Loki**: Uses `json-file` (cannot log to itself — circular dependency).
- **Grafana**: Uses `json-file` (infrastructure service).
- **Docker Socket Proxy**: Uses `json-file` (infrastructure service, minimal output).

These services' logs are accessible via `docker logs <service>`.

### LogQL Query Examples

#### Agent errors

```logql
# All errors from leonard in the last hour
{service="leonard-1"} |= "error"

# All errors across all agents
{service=~".*"} | json | level="error"

# Errors for a specific pipeline
{pipeline="pipe-abc123"} | json | level="error"

# Agent crash events
{service=~".*"} | json | event="agent_error"
```

#### Pipeline progress

```logql
# All task lifecycle events for a pipeline
{pipeline="pipe-abc123"} | json | event=~"task_.*"

# Task completions across all pipelines
{service=~".*"} | json | event="task_completed"

# Consensus decisions
{service=~"nelson.*"} | json | event="consensus_reached"

# Follow a single task through all agents via correlation ID
{service=~".*"} | json | correlation_id="decompose-pipe7-task42"
```

#### Cost tracking

```logql
# All LLM calls with cost data
{service=~".*"} | json | event="llm_call_completed"
  | line_format "{{.agent}} {{.provider}} ${{.cost_usd}}"

# Total tokens by provider
{service=~".*"} | json | event="llm_call_completed"
  | line_format "{{.provider}} {{.data_prompt_tokens}}"

# Budget warnings
{service=~".*"} | json | event="budget_warning"

# Budget exceeded events
{service=~".*"} | json | event="budget_exceeded"
```

#### Review and learning

```logql
# All review decisions
{service=~"katherine.*"} | json | event="review_completed"

# Human review flags
{service=~".*"} | json | event="human_review_flagged"

# Consensus escalations (could not reach agreement)
{service=~"nelson.*"} | json | event="consensus_escalated"
```

### Dashboard Suggestions

Pre-built Grafana dashboards for common monitoring scenarios:

#### 1. Pipeline Status Dashboard

- **Active pipelines**: Count of pipelines in each status (IN_PROGRESS, LEARNING, COMPLETED, FAILED).
- **Task progress**: Bar chart showing completed/in-progress/blocked/failed tasks per pipeline.
- **Pipeline timeline**: Gantt-style view of task durations within each pipeline.
- **Recent events**: Live log stream filtered to `task_*` and `pipeline_*` events.

#### 2. Agent Activity Dashboard

- **Agent containers**: Table of running agent containers with uptime, status, current task.
- **Agent spawn rate**: Time series of container launches per agent type.
- **Agent errors**: Error count by agent type over time.
- **Agent duration**: P50/P95/P99 of agent container lifetimes by type.
- **Heartbeat health**: Missing heartbeat alerts.

#### 3. Cost Per Pipeline Dashboard

- **Daily spend**: Time series of cumulative cost per day.
- **Cost by pipeline**: Bar chart of total cost per pipeline.
- **Cost by agent**: Pie chart of cost breakdown across agent types.
- **Cost by provider**: Bar chart comparing spending across LLM providers.
- **Budget utilization**: Gauge charts showing current spend vs. daily hard limit and per-task limits.
- **Most expensive tasks**: Table of top-N tasks by total cost.

#### 4. Error Rates Dashboard

- **Error rate**: Time series of errors per minute across all agents.
- **Error breakdown**: Stacked bar chart by error type (LLM failures, test failures, git conflicts, etc.).
- **Retry rate**: Percentage of tasks requiring retries over time.
- **Escalation rate**: Tasks escalated to human over time.
- **Mean time to recovery**: Average time from error to successful retry.

---

## Audit Logging

> **Migrated from**: `docs/specs/09-security.md` (Audit Trail section)

Every security-relevant event is logged to both the standard structlog output
(queryable in Grafana/Loki) and to a dedicated PostgreSQL table for durable,
queryable audit records.

### Security-relevant events

- Secret access attempts.
- Container restarts.
- Filesystem write operations.
- GitHub API calls (with response codes).
- Budget limit triggers.
- Failed authentication attempts.
- Network egress connections (audit only, no restrictions — see spec 09).

### Audit log table

```sql
CREATE TABLE audit_log (
    id TEXT PRIMARY KEY,
    timestamp TEXT NOT NULL,
    agent TEXT NOT NULL,
    event_type TEXT NOT NULL,    -- "secret_access", "file_write", "github_api", etc.
    details TEXT NOT NULL,       -- JSON
    severity TEXT NOT NULL       -- "info", "warning", "critical"
);
```

The audit trail is stored in PostgreSQL (not just log files) for queryability.
This enables structured queries that go beyond what LogQL provides — for example,
joining audit events with pipeline or task records.

### Audit LogQL queries

```logql
# All file write operations
{service=~".*"} | json | event="file_written"

# GitHub API calls from Richelieu
{service=~"richelieu.*"} | json | event_type="github_api"

# Secret access attempts (should be rare — investigate any)
{service=~".*"} | json | event_type="secret_access"

# Container restart events
{service=~".*"} | json | event="agent_started" | line_format "RESTART: {{.agent}}"
```

---

## Cross-References

- **Spec 05** (Infrastructure): Docker Compose config with Loki logging driver, ephemeral agent Loki config in Launcher.
- **Spec 06** (Controller): Watchdog monitors agent heartbeats; controller logs its own lifecycle events.
- **Spec 08** (TUI): Subscribes to Redis `logs:*` streams for real-time log display.
- **Spec 16** (Cost Tracking): `llm_call_completed` events carry cost data; budget events.
- **Spec 17** (Error Recovery): `agent_error`, `task_failed`, checkpoint and retry events.
- **Spec 18** (Security): Audit logging, log redaction of secrets, egress audit.
