# 16 — Nelson (Consensus Orchestrator)

> **Migrated from**: `docs/specs/04-consensus.md`, `PLAN.md` section 3

## Overview

Nelson is the multi-LLM consensus engine at the heart of the system. Any agent
can request Nelson to evaluate a prompt across multiple LLM providers, run
cross-review rounds, and return a consensus result with a confidence score.
Nelson is **not** a domain agent -- it is a shared utility invoked by every other
agent when a high-confidence decision is needed.

### Core Principles

1. **No single LLM is trusted alone** for important decisions. The consensus mechanism
   catches individual LLM blind spots, hallucinations, and biases.
2. **Convergence is the goal**, not unanimity. Two out of three agreeing is sufficient
   for most decisions.
3. **Disagreement is signal**. When LLMs disagree, the reasons for disagreement are often
   more valuable than the individual answers.
4. **Speed vs. confidence is configurable**. Low-stakes decisions can skip consensus.
   High-stakes decisions get more rounds.

---

## Responsibilities

- Run the multi-LLM consensus loop (parallel dispatch, cross-review, termination).
- Support structured and unstructured response formats.
- Calculate confidence scores for consensus outcomes.
- Track and apply per-provider accuracy weights (learned from human feedback).
- Return dissenting opinions alongside the consensus result.
- Support direct human queries via TUI/CLI.
- Be invokable by **every** other agent (Julius, Sherlock, Leonard, Katherine, Richelieu).

---

## Lifecycle

- **Spawned when**: A `consensus_request` message appears on a `pipeline:{id}:consensus` stream (see spec 13-controller).
- **Exits when**: Publishes a `consensus_response` to `pipeline:{id}:consensus_resp`.
- **Parallelism**: Up to `max_parallel_nelsons` (default: 5) running simultaneously. Each Nelson handles one consensus request.

---

## Input / Output

### Input

- **Message type**: `consensus_request`
- **Redis stream**: `pipeline:{id}:consensus`
- **Payload** (`ConsensusRequest`):

```python
class ConsensusRequest(BaseModel):
    """Sent by any agent to Nelson via Redis."""
    request_id: str                          # Unique ID (UUID)
    requester: str                           # Agent name (e.g., "katherine")
    task_id: str | None                      # Associated task (for tracking)

    # The prompt to evaluate
    prompt: str                              # The main question/task for LLMs
    system_context: str | None               # System prompt / background context

    # Cross-review configuration
    review_goals: list[str] | None           # What to focus on during cross-review
                                             # e.g., ["correctness", "security",
                                             #        "adherence to style guide"]

    # Response format
    response_schema: dict | None             # JSON schema for structured output
                                             # If None, LLMs respond freely

    # Consensus settings (overridable per request)
    max_rounds: int = 3                      # Max cross-review iterations
    required_agreement: float = 0.67         # Fraction needed (0.67 = 2/3)
    providers: list[str] | None              # Override models from .ai-team.yaml
                                             # e.g., ["anthropic/claude-sonnet-4",
                                             #        "openai/gpt-4o"]
    priority: Literal["low", "normal", "high"] = "normal"
```

### Output

- **Message type**: `consensus_response`
- **Redis stream**: `pipeline:{id}:consensus_resp`
- **Payload** (`ConsensusResponse`):

```python
class ConsensusResponse(BaseModel):
    """Returned by Nelson after consensus (or failure to reach it)."""
    request_id: str
    status: Literal["agreed", "majority", "weighted", "escalated", "error"]

    # The consensus result
    decision: dict | str                     # Structured (if schema) or free text
    confidence: float                        # 0.0 - 1.0
    reasoning: str                           # Merged reasoning from agreeing LLMs

    # Dissent (if any)
    dissenting_opinions: list[DissentRecord] # Which providers disagreed and why

    # Metadata
    rounds_taken: int
    provider_responses: list[ProviderRound]  # Full history of every round
    total_tokens: TokenUsage
    total_cost: float
    elapsed_seconds: float


class DissentRecord(BaseModel):
    provider: str
    position: str
    reasoning: str


class ProviderRound(BaseModel):
    round_number: int
    provider: str
    response: dict | str                     # The raw response
    review_of_others: str | None             # Their cross-review feedback
    changed_position: bool                   # Did they change from previous round?
```

---

## Algorithm / Process

### Phase 1: Initial Responses

1. Nelson sends the `prompt` + `system_context` to all providers in **parallel**.
2. If `response_schema` is provided, each provider is instructed to respond with that
   JSON structure. The structured part MUST include a `decision` field and a `reasoning`
   field at minimum.
3. If `review_goals` are provided, they are included in the system prompt:
   _"When forming your response, pay special attention to: {goals}"_.
4. Collect all responses. If a provider fails (timeout, error), mark it as absent and
   proceed with remaining providers. Minimum 2 providers required.

### Phase 2: Cross-Review Rounds

For each round (up to `max_rounds`):

1. Each provider receives:
   - The original prompt.
   - Its own previous response.
   - All other providers' latest responses (anonymized as "Reviewer A", "Reviewer B").
   - The `review_goals` (if any).
   - Instruction: _"Review the other responses. If you find their reasoning compelling,
     you may update your position. If you stand by your original position, explain why
     the alternatives are insufficient. Focus on: {review_goals}"_.
2. Each provider responds with:
   - Updated (or unchanged) decision.
   - Review of each other response (what is good, what is wrong).
   - Whether they changed their position and why.
3. Check termination conditions (see below).

### Phase 3: Termination

After each cross-review round, check in order:

```
1. UNANIMOUS AGREEMENT
   All providers gave the same decision.
   -> Return with status="agreed", confidence=0.95+

2. MAJORITY AGREEMENT (rounds 1-2)
   >= required_agreement fraction (default 2/3) agree.
   -> Return with status="majority", confidence based on strength of agreement.

3. MAX ROUNDS REACHED -> proceed to weighted vote

4. WEIGHTED VOTE
   Each provider's decision is weighted by its learned accuracy weight.
   If weighted vote produces a clear winner (>60% of total weight):
   -> Return with status="weighted", confidence based on weight margin.

5. NO CONVERGENCE
   -> Return with status="escalated", include all positions.
   -> The calling agent decides how to handle (usually escalate to human).
```

### Nelson Loop Diagram

```
+---------------------------------------------------------------+
|                         NELSON LOOP                           |
|                                                               |
|  +---------+    +---------+    +---------+                    |
|  |  Claude  |    |  GPT-4o |    | Gemini  |                   |
|  +----+----+    +----+----+    +----+----+                    |
|       |              |              |                          |
|       v              v              v                          |
|   Response A     Response B     Response C                    |
|       |              |              |                          |
|       v              v              v                          |
|  +-------------------------------------------+               |
|  |         CROSS-REVIEW ROUND                |               |
|  |  A reviews B+C                            |               |
|  |  B reviews A+C                            |               |
|  |  C reviews A+B                            |               |
|  +-------------------------------------------+               |
|       |                                                       |
|       v                                                       |
|  +------------------+     +---------------+                   |
|  | Majority (2/3)?  |--YES-->| Return result |                |
|  +--------+---------+     +---------------+                   |
|           NO                                                  |
|           v                                                   |
|  +------------------+                                         |
|  | Max rounds?      |--NO--> Loop again                       |
|  +--------+---------+                                         |
|           YES                                                 |
|           v                                                   |
|  +------------------+     +---------------+                   |
|  | Weighted vote    |--WIN-->| Return result |                |
|  +--------+---------+     +---------------+                   |
|           TIE                                                 |
|           v                                                   |
|  +-------------------------------------------+               |
|  | ESCALATE TO HUMAN                         |               |
|  | (with all positions + reasoning)          |               |
|  +-------------------------------------------+               |
+---------------------------------------------------------------+
```

### Confidence Calculation

```python
def calculate_confidence(
    responses: list[ProviderResponse],
    weights: dict[str, float],
    status: str,
) -> float:
    if status == "agreed":
        # Unanimous -- high confidence, boosted by round count
        return min(0.95 + (0.01 * len(responses)), 0.99)

    if status == "majority":
        agreeing_weight = sum(weights[r.provider] for r in responses if r.in_majority)
        total_weight = sum(weights.values())
        return 0.6 + 0.3 * (agreeing_weight / total_weight)

    if status == "weighted":
        # Lower confidence -- weighted vote means there was real disagreement
        winning_weight = max(group_weights.values())
        return 0.5 + 0.2 * (winning_weight / sum(group_weights.values()))

    return 0.0  # escalated -- no confidence
```

---

## Provider Weight System

### Initial Weights

All providers start with equal weight. The providers used for consensus are
configured in `.ai-team.yaml` under `llm.consensus.models`. For example, with
three providers (`anthropic/claude-sonnet-4`, `openai/gpt-4o`,
`google/gemini-2.0-flash`):
- anthropic/claude-sonnet-4: 1.0
- openai/gpt-4o: 1.0
- google/gemini-2.0-flash: 1.0

Weights always normalize to sum = N (number of providers).

### Weight Updates

When a human makes a decision that overrides or confirms a consensus:

```python
def update_weights(
    consensus: ConsensusResponse,
    human_decision: str,
    learning_rate: float = 0.05,
):
    for response in consensus.provider_responses[-1]:  # Last round
        provider = response.provider
        agreed_with_human = response.decision == human_decision

        if agreed_with_human:
            weights[provider] += learning_rate
        else:
            weights[provider] -= learning_rate

        # Floor: no provider drops below 0.5
        weights[provider] = max(weights[provider], 0.5)

    # Normalize to sum = N
    total = sum(weights.values())
    n = len(weights)
    for provider in weights:
        weights[provider] = weights[provider] * n / total
```

### Weight Persistence

Weights are stored in PostgreSQL per scope:

| Scope          | Example                       | Purpose                         |
|---------------|-------------------------------|---------------------------------|
| Global        | All decisions                 | Overall provider reliability    |
| Per task type | "code_review", "decomposition"| Provider strengths per domain   |
| Per project   | "my-app"                      | Project-specific accuracy       |

---

## Interaction with Other Agents

Nelson is a **service agent** -- it does not initiate work. It only responds to
`consensus_request` messages from other agents:

- **Julius**: Validates task decomposition quality and dependency graph correctness.
- **Sherlock**: Resolves ambiguous implementation approaches when multiple strategies exist.
- **Katherine**: Runs multi-LLM code review consensus (correctness, style, safety, simplicity) and calculates human review scores.
- **Leonard**: May request consensus on approach when stuck (rare).
- **Richelieu**: May request consensus on conflict resolution strategy (rare).
- **Human (via TUI/CLI)**: Direct queries for any question.

---

## Prompt Templates

### Default Cross-Review Prompt

```
You previously answered a question and provided your reasoning.
Other reviewers have also answered the same question.

Your previous response:
{own_response}

Other responses:
{other_responses}

Review goals: {review_goals or "Evaluate correctness, completeness, and reasoning quality."}

Instructions:
1. Review each other response. What do they get right? What do they get wrong?
2. Consider whether their reasoning changes your position.
3. Provide your updated (or unchanged) decision with reasoning.
4. If you changed your position, explain what convinced you.

Respond with:
{response_schema or "Your decision and detailed reasoning."}
```

### Agent-Specific Goal Examples

**Katherine (code review)**:
```yaml
review_goals:
  - "Does the code correctly implement the task requirements?"
  - "Does it follow the codebase conventions found in the style guide?"
  - "Are there security concerns (injection, auth bypass, data exposure)?"
  - "Is the code unnecessarily complex? Could it be simpler?"
```

**Julius (decomposition validation)**:
```yaml
review_goals:
  - "Is each task truly independent where marked independent?"
  - "Are the tasks small enough for one agent to implement?"
  - "Does the dependency graph have any missing edges?"
  - "Are there implicit dependencies not captured?"
```

**Katherine (human review scoring)**:
```yaml
review_goals:
  - "Rate the novelty of these changes (0-1): are new patterns introduced?"
  - "Rate the complexity (0-1): how hard is this to reason about?"
  - "Rate the risk (0-1): what's the blast radius if this is wrong?"
  - "Should a human review this? Why or why not?"
```

---

## Principles Integration

Nelson does not directly apply principles. However, when agents include principles
in their `system_context` or `prompt` fields, Nelson evaluates them as part of the
consensus. For example, when Katherine asks Nelson to review code, the review
principles from `.ai-team/principles/review/` are included in the prompt context.

Nelson also plays a key role in **learning mode** (see spec 03): it analyzes human
comments on PRs and extracts candidate principles via consensus.

---

## Direct Usage

Nelson can be called directly by humans through the TUI or CLI:

```bash
# Ask Nelson a question directly
ai-team ask "What's the best approach to implement caching for this API?"

# Ask with specific review goals
ai-team ask "Review this SQL migration" --goals "data safety" "backwards compatibility"
```

---

## System Prompt

<!-- TODO: Define the system prompt template for Nelson -->
<!-- Nelson's system prompt should establish its role as an impartial evaluator,
     emphasize that it must weigh all perspectives fairly, and instruct it to
     surface disagreements rather than force artificial consensus. -->

---

## PydanticAI Tools

<!-- TODO: Define the tools available to Nelson -->

Nelson's tool requirements are minimal since it primarily orchestrates LLM calls:

- **LLM dispatch**: Send prompts to multiple providers in parallel via LiteLLM/OpenRouter.
- **Weight lookup**: Read provider weights from PostgreSQL.
- **Weight update**: Write updated weights to PostgreSQL after human feedback.
- **Cache lookup**: Check if an identical prompt was recently evaluated (optional optimization).
- **Redis publish**: Publish `consensus_response` to the response stream.

---

## Cost Optimization

Nelson is the most expensive component (3x LLM calls per round, multiple rounds).
Optimization strategies:

1. **Skip consensus for low-stakes decisions**: If a single LLM can handle it
   (e.g., formatting, simple code generation), do not involve Nelson.
2. **Use cheaper models for cross-review**: Initial response from top-tier models,
   cross-review from cheaper tiers. Configure via `llm.consensus.models` in
   `.ai-team.yaml` -- use cheaper models in the list to reduce cost.
3. **Cache identical prompts**: If the same exact prompt was recently evaluated,
   return the cached result.
4. **Early termination**: If all providers agree in round 1, skip remaining rounds.
5. **2-provider mode**: For medium-stakes, configure only 2 models in
   `llm.consensus.models` instead of 3.
6. **OpenRouter consolidation**: All calls go through OpenRouter, which handles
   billing consolidation. Monitor costs via the Grafana dashboard.

---

## Error Handling

- **Provider failure**: If one provider fails (timeout, rate limit, error), proceed
  with remaining providers. Minimum 2 providers required for consensus. If fewer
  than 2 are available, return `status="error"`.
- **All providers fail**: Return `consensus_response` with `status="error"`. The
  requesting agent handles the error (typically escalates to human).
- **Container crash**: The controller's watchdog detects the crash and respawns
  Nelson with the same `consensus_request`. Pending requests queue in Redis (durable).
  Up to 3 retries before escalating (see spec 13-controller).
- **Timeout**: Each provider call has a configurable timeout. If a provider times out,
  it is treated as a failure for that round.

---

## Resource Profile

| Metric             | Value  | Rationale                                |
|--------------------|--------|------------------------------------------|
| CPU limit          | 1.0    | Parallel LLM API calls (I/O bound)      |
| Memory limit       | 1G     | Response parsing and cross-review state  |
| CPU reservation    | 0.25   |                                          |
| Memory reservation | 256M   |                                          |

Overridable via `.ai-team.yaml` `resources.nelson` section (see spec 12).

---

## Configuration

Nelson reads its configuration from multiple sources:

- **`.ai-team.yaml`** (`llm.consensus` section):
  - `models`: List of LLM providers to use (e.g., `["anthropic/claude-sonnet-4", "openai/gpt-4o", "google/gemini-2.0-flash"]`).
  - `max_rounds`: Default maximum cross-review rounds (default: 3).
- **Per-request overrides**: Each `ConsensusRequest` can override `max_rounds`, `required_agreement`, and `providers`.
- **Environment variables**: `OPENROUTER_API_KEY` for LLM access, `DATABASE_URL` for weight storage, `REDIS_URL` for messaging.
- **PostgreSQL**: Provider weights (read on startup, updated on human feedback).

---

## Testing Strategy

- **Unit tests**: Mock LLM responses to test the consensus algorithm, cross-review
  logic, termination conditions, confidence calculation, and weight updates.
- **Integration tests**: Use VCR cassettes (see spec 10) to record and replay real
  LLM provider calls. Test the full consensus loop end-to-end with recorded responses.
- **Weight learning tests**: Simulate human overrides and verify weight convergence
  over multiple iterations.
- **Edge cases**: Test with 2 providers (one fails), all providers disagree,
  identical responses in round 1 (early termination), minimum weight floor behavior.
