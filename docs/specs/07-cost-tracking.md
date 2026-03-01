# 07 — Cost Tracking

> **Migrated from**: `docs/specs/07-cost-tracking.md`

## Overview

Every LLM call in the system has a cost entry linked to the agent, task, consensus round,
and provider. The system supports configurable budget limits per task (soft or hard) and
a hard daily ceiling as a safety net.

---

## Cost Attribution Model

```
Daily Budget ($100)
  └── Pipeline Run (feature/add-user-auth)
       └── Task #2 (Create auth endpoints)
            ├── Sherlock: enrichment calls
            │    ├── LLM call: anthropic/claude-sonnet-4  $0.12
            │    └── LLM call: openai/gpt-4o             $0.08
            ├── Nelson: consensus for approach
            │    ├── Round 1: anthropic/claude-sonnet-4 $0.15 + openai/gpt-4o $0.12 + google/gemini-pro-1.5 $0.09
            │    └── Round 2: anthropic/claude-sonnet-4 $0.18 + openai/gpt-4o $0.14 + google/gemini-pro-1.5 $0.11
            ├── Leonard: implementation
            │    ├── LLM call: anthropic/claude-sonnet-4  $0.45
            │    └── LLM call: anthropic/claude-sonnet-4  $0.22
            └── Katherine: review
                 └── Nelson: consensus for review
                      └── Round 1: anthropic/claude-sonnet-4 $0.15 + openai/gpt-4o $0.12 + google/gemini-pro-1.5 $0.09
```

## Cost Record Schema

Every LLM call generates a cost record stored in PostgreSQL:

```python
class CostRecord(BaseModel):
    """One record per LLM API call."""
    id: str                                  # UUID
    timestamp: datetime
    pipeline_id: str                         # Which pipeline run
    task_id: str | None                      # Which task (null for pre-task calls)
    agent: str                               # Which agent made the call
    consensus_request_id: str | None         # If part of a Nelson consensus
    consensus_round: int | None              # Which round in the consensus

    # LLM details — provider is the OpenRouter model ID
    provider: str                            # "anthropic/claude-sonnet-4", "openai/gpt-4o"
    model: str                               # Full model identifier from OpenRouter
    prompt_tokens: int
    completion_tokens: int
    total_tokens: int

    # Cost — sourced from OpenRouter response metadata via LiteLLM
    cost_usd: float                          # From OpenRouter's response_cost

    # Context
    purpose: str                             # "enrichment", "implementation",
                                             # "review", "consensus", "scoring"
```

## Pricing: OpenRouter Cost Extraction

All LLM calls are routed through OpenRouter, which provides per-call cost data
in API responses. LiteLLM extracts this into a `response_cost` field, so there
is no need to maintain a local pricing table for every model.

Cost is extracted immediately after each LLM call:

```python
# core/cost/pricing.py

def extract_cost(litellm_response) -> float:
    """Extract cost from LiteLLM response (sourced from OpenRouter)."""
    if hasattr(litellm_response, "_hidden_params"):
        cost = litellm_response._hidden_params.get("response_cost")
        if cost is not None:
            return cost
    # Fallback: estimate from token counts
    return estimate_cost_from_tokens(
        model=litellm_response.model,
        prompt_tokens=litellm_response.usage.prompt_tokens,
        completion_tokens=litellm_response.usage.completion_tokens,
    )
```

A fallback pricing table is kept for the rare case where OpenRouter does not
return cost data (e.g., network errors, unsupported models). This table only
needs approximate values since it is a safety net, not the primary source:

```python
# Fallback only — OpenRouter is the source of truth
FALLBACK_PRICING: dict[str, ModelPricing] = {
    "anthropic/claude-sonnet-4": ModelPricing(input_per_1k=0.003, output_per_1k=0.015),
    "openai/gpt-4o": ModelPricing(input_per_1k=0.005, output_per_1k=0.015),
    "google/gemini-pro-1.5": ModelPricing(input_per_1k=0.00125, output_per_1k=0.005),
}

def estimate_cost_from_tokens(model: str, prompt_tokens: int, completion_tokens: int) -> float:
    pricing = FALLBACK_PRICING.get(model)
    if pricing is None:
        log.warning("no_fallback_pricing", model=model)
        return 0.0
    return (
        (prompt_tokens / 1000) * pricing.input_per_1k
        + (completion_tokens / 1000) * pricing.output_per_1k
    )
```

## Budget Limits

### Hierarchy

```
Daily Hard Limit ($100/day) — ALWAYS enforced, cannot be overridden
  └── Per-Task Limit — configurable per task
       ├── Soft Limit ($5 default) — warning, agents continue
       └── Hard Limit ($20 default) — agent stops, escalates
```

### Configuration Sources (Priority Order)

1. **Per-task override** (set by Julius when creating tasks, or by human).
2. **`.ai-team.yaml`** project config (`budget.soft_limit_per_task`, `budget.hard_limit_per_task`).
   This is the primary place to configure soft/hard limits per project.
3. **`.env`** — only `DAILY_BUDGET_HARD` as a system-level safety net.
   Per-task defaults come from `.ai-team.yaml`, not `.env`.

### Enforcement Logic

```python
async def check_budget(task_id: str, new_cost: float) -> BudgetAction:
    task_total = await get_task_total_cost(task_id)
    daily_total = await get_daily_total_cost()
    task_limit = await get_task_budget(task_id)

    # Daily hard limit — non-negotiable
    if daily_total + new_cost > DAILY_BUDGET_HARD:
        return BudgetAction.HALT_ALL  # Stop ALL agents

    # Per-task hard limit
    if task_total + new_cost > task_limit.hard:
        return BudgetAction.HALT_TASK  # Stop this task, escalate

    # Per-task soft limit
    if task_total + new_cost > task_limit.soft:
        return BudgetAction.WARN  # Log warning, notify TUI, continue

    return BudgetAction.OK
```

### What Happens on Budget Actions

| Action       | Behavior                                                         |
|-------------|------------------------------------------------------------------|
| `OK`        | Proceed normally.                                                |
| `WARN`      | Log a `budget_warning` event. TUI shows yellow indicator.        |
| `HALT_TASK` | Agent stops work on this task. Posts a message to Redis stream. TUI shows escalation banner. Human decides: increase budget or cancel task. |
| `HALT_ALL`  | ALL agents stop. TUI shows critical alert. System pauses until human increases daily limit or next day. |

## Aggregation Queries

PostgreSQL makes it easy to query costs at any granularity:

```sql
-- Total cost by agent today
SELECT agent, SUM(cost_usd) as total
FROM cost_records
WHERE timestamp::date = CURRENT_DATE
GROUP BY agent;

-- Total cost by provider this week
SELECT provider, SUM(cost_usd) as total,
       SUM(prompt_tokens) as input_tokens,
       SUM(completion_tokens) as output_tokens
FROM cost_records
WHERE timestamp > CURRENT_DATE - INTERVAL '7 days'
GROUP BY provider;

-- Most expensive tasks
SELECT task_id, SUM(cost_usd) as total, COUNT(*) as llm_calls
FROM cost_records
GROUP BY task_id
ORDER BY total DESC
LIMIT 10;

-- Cost breakdown for a specific task
SELECT agent, provider, purpose, SUM(cost_usd) as total
FROM cost_records
WHERE task_id = 'task-42'
GROUP BY agent, provider, purpose;

-- Consensus cost vs. direct call cost
SELECT
    CASE WHEN consensus_request_id IS NOT NULL THEN 'consensus' ELSE 'direct' END as type,
    SUM(cost_usd) as total,
    COUNT(*) as calls
FROM cost_records
GROUP BY type;
```

## TUI Cost Integration

The TUI displays cost data in real-time (see spec 15 for TUI mockups):

- **Pipeline overview**: Running total for current pipeline.
- **Task list**: Cost per task, colored by budget status (green/yellow/red).
- **Cost dashboard**: Breakdown by agent, provider, and task.
- **Alerts**: Banner when approaching soft/hard limits.

## Cost Events (Redis Stream)

Every cost-relevant event is published to the `costs` Redis Stream:

```python
class CostEvent(BaseModel):
    event_type: Literal["cost_recorded", "budget_warning", "budget_halt"]
    cost_record: CostRecord | None          # For cost_recorded
    budget_status: BudgetStatus | None      # For warnings/halts
    timestamp: datetime
```

The TUI subscribes to this stream for real-time cost updates.

## Future: Cost Optimization Insights

Once enough data is collected, the system can suggest optimizations:

- "Nelson consensus on code review costs $0.50 avg. Consider using 2 providers
  instead of 3 for reviews where Katherine's confidence is already high."
- "Sherlock spends 40% of its budget re-reading the same files. Consider caching
  file contents across tasks in the same pipeline."
- "GPT-4o is 30% cheaper than Claude for consensus cross-review with similar
  accuracy. Consider using it as the primary review model."

---

## Cross-References

- **Spec 05** (Infrastructure): Docker Compose config, `.env` for `DAILY_BUDGET_HARD`.
- **Spec 06** (Orchestrator): Budget enforcement halts pipeline if daily limit exceeded.
- **Spec 08** (TUI): Real-time cost display, budget alerts.
- **Spec 15** (Observability): `llm_call_completed`, `budget_warning`, `budget_exceeded` events.
- **Spec 17** (Error Recovery): Budget exceeded is a fatal error that escalates to human.
- **Spec 21** (Repo Connection): `.ai-team.yaml` `budget:` section for per-project limits.
