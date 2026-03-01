# 06 — Logging

## Overview

Every operation in the system is logged — every LLM call, file read, git operation,
message sent, and decision made. Logs are structured JSON, shipped to Redis Streams
for real-time consumption by the TUI, and persisted to files for historical analysis.

**Philosophy**: Log everything, always. Storage is cheap. Debugging agentic systems
without full logs is nearly impossible.

---

## Stack

- **Library**: [structlog](https://www.structlog.org/) — structured, contextual logging.
- **Format**: JSON lines (one JSON object per log line).
- **Transport**: Dual output — local file + Redis Stream.
- **Retention**: Log files rotated daily, kept for 30 days (configurable).

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

Each agent writes logs to three destinations, each serving a different purpose:

1. **Stdout** (captured by Docker): For `docker compose logs` and standard container
   log management. Automatically forwarded to Loki via the Docker logging driver.
2. **Redis Stream** (`logs:{agent_name}`): For real-time TUI consumption.
3. **Loki** (via Docker logging driver): For historical querying, dashboards, and
   alerting through Grafana. This requires no agent code changes — Docker forwards
   stdout to Loki automatically.

The TUI subscribes to all `logs:*` streams and merges them into a unified view.

## Loki + Grafana

Loki is the primary log aggregation and querying backend. It ingests logs
automatically via Docker's Loki logging driver — no Promtail or sidecar needed.
Each container's stdout is forwarded directly to Loki by the Docker daemon.

**Grafana** sits on top of Loki and provides:

- **LogQL querying**: Filter and search logs by any structured field (agent, task,
  pipeline, correlation_id, event type, level, etc.).
- **Dashboards**: Pre-built views for pipeline health, LLM cost tracking, error
  rates, and agent activity.
- **Alerting**: Rules that fire on error spikes, budget warnings, or agent crashes.

The three log destinations serve complementary purposes:

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
- **Direct file access**: JSON files in `/data/logs/`, parseable with `jq`.

### LogQL Examples (Grafana)

```logql
# All errors from leonard in the last hour
{service="leonard-1"} |= "error"

# All LLM calls for task 42
{service=~".*"} | json | task_id="task-42" and event="llm_call_completed"

# Total tokens by provider
{service=~".*"} | json | event="llm_call_completed" | line_format "{{.provider}} {{.data_prompt_tokens}}"
```
