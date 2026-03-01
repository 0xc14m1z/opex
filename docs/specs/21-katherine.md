# 21 — Katherine (Code Reviewer)

> **Migrated from**: `PLAN.md` sections 2 (Katherine description) and 4 (Confidence Scoring & Adaptive Learning)

## Overview

Katherine is the quality gate of the system. She reviews Leonard's implementation,
decides whether the code meets standards, and determines whether a human needs to
review the PR. Katherine uses Nelson (spec 16) for multi-LLM consensus on every
review dimension, calculates a human review score, and adapts her threshold over
time based on human feedback.

Katherine's decisions directly affect the system's autonomy: as she learns the
team's standards, she flags fewer PRs for human review, allowing the system to
operate more independently.

---

## Responsibilities

- Read the diff, the original task definition, and Sherlock's execution plan.
- Use Nelson (spec 16) to get multi-LLM consensus on code quality across four dimensions: correctness, style, safety, simplicity.
- If changes are needed: send structured feedback to Leonard for rework.
- If approved: calculate a **human review score** via Nelson consensus.
- Apply the adaptive threshold to decide whether human review is required.
- Post structured review comments on the task PR (via Richelieu).
- Enforce review principles from `.opex/principles/review/`.
- Learn from human review decisions to calibrate scoring weights over time.

---

## Lifecycle

- **Spawned when**: Leonard publishes `implementation_complete` for a task. The orchestrator (spec 13-orchestrator) spawns Katherine.
- **Exits when**: Publishes `review_result` (approved, changes_requested, or escalated) to `pipeline:{id}:reviews`.
- **Parallelism**: Up to `max_parallel_katherines` (default: 3) running simultaneously.

---

## Input / Output

### Input

- **Message type**: `implementation_complete`
- **Redis stream**: `pipeline:{id}:reviews`
- **Environment variables**: `PIPELINE_ID`, `TASK_ID`
- **Data available**:
  - The diff (changes made by Leonard on the task branch).
  - The original task definition from Julius.
  - Sherlock's mini execution plan (acceptance criteria, patterns to follow).
  - `.opex.yaml` configuration (guidelines, review settings).
  - Review principles from `.opex/principles/review/`.
  - Historical review data from PostgreSQL (for adaptive threshold).

### Output

- **Message type**: `review_result`
- **Redis stream**: `pipeline:{id}:reviews`
- **Payload variants**:
  - `review_result:approved` -- code passes review, includes human review score.
  - `review_result:changes_requested` -- structured feedback for Leonard.
  - `review_result:escalated` -- review could not be completed, human needed.

---

## Algorithm / Process

### Step 1: Read Context

1. Read the diff (Leonard's changes).
2. Read the original task definition.
3. Read Sherlock's execution plan (acceptance criteria, patterns, conventions).
4. Load review principles from `.opex/principles/review/`.
5. Load `.opex.yaml` guidelines and review configuration.

### Step 2: Multi-LLM Code Review via Nelson

Submit a `consensus_request` to Nelson (spec 16) with four review dimensions:

**Correctness**:
- Does the code correctly implement the task requirements?
- Are all acceptance criteria from Sherlock's plan satisfied?
- Are edge cases handled?

**Style**:
- Does the code follow codebase conventions (naming, structure, imports)?
- Does it match patterns identified by Sherlock?
- Does it comply with guidelines in `.opex.yaml`?

**Safety**:
- Are there security concerns (injection, auth bypass, data exposure)?
- Are error cases handled properly?
- Are there potential data loss or corruption scenarios?

**Simplicity**:
- Is the code unnecessarily complex?
- Could it be simpler while maintaining correctness?
- Is there dead code, unnecessary abstraction, or over-engineering?

Nelson review goals for code review:
```yaml
review_goals:
  - "Does the code correctly implement the task requirements?"
  - "Does it follow the codebase conventions found in the style guide?"
  - "Are there security concerns (injection, auth bypass, data exposure)?"
  - "Is the code unnecessarily complex? Could it be simpler?"
```

### Step 3: Decision

Based on Nelson's consensus:

- **All dimensions pass**: Proceed to human review scoring (Step 4).
- **Any dimension fails**: Generate structured feedback and publish `review_result:changes_requested`. Leonard receives the feedback and iterates.
- **Nelson escalates (no consensus)**: Publish `review_result:escalated`.

### Step 4: Human Review Scoring

If the code passes review, Katherine calculates a **human review score** using
Nelson consensus. This score determines whether the PR needs human review.

#### Score Components

| Signal              | Source          | Description                                        |
|---------------------|----------------|----------------------------------------------------|
| Novelty             | Nelson          | New patterns, deps, or architectural changes       |
| Complexity          | Heuristics      | Files changed, cyclomatic complexity delta          |
| AI Confidence       | Nelson          | Consensus confidence from the review loop           |
| Historical Accuracy | PostgreSQL      | How often similar PRs needed human intervention    |

Nelson review goals for human review scoring:
```yaml
review_goals:
  - "Rate the novelty of these changes (0-1): are new patterns introduced?"
  - "Rate the complexity (0-1): how hard is this to reason about?"
  - "Rate the risk (0-1): what's the blast radius if this is wrong?"
  - "Should a human review this? Why or why not?"
```

### Step 5: Threshold Decision

Compare the human review score against the adaptive threshold:

- **Score >= threshold**: Flag the PR for human review. Apply `needs-human-review` label.
- **Score < threshold**: Auto-approve. Apply `autonomous` label.

The threshold is configured in `.opex.yaml` (`review.human_review_threshold`,
default: 0.7) and evolves via adaptive learning (see below).

#### Always-flag paths

Certain file paths always trigger human review regardless of score, configured in
`.opex.yaml`:

```yaml
review:
  always_human_review:
    - "migrations/"
    - "infrastructure/"
    - "*.sql"
```

### Step 6: Publish Result

- Publish `review_result:approved` with the human review score and threshold decision.
- The orchestrator then tells Richelieu to merge (if auto-approved) or waits for human review.

---

## Adaptive Threshold

> For the full adaptive learning mechanism, see spec 03 (Learning and Principles).

### Initial State

The threshold starts conservatively high (default: 0.7), meaning most PRs are
flagged for human review.

### Learning Process

Each human decision on a PR (approve or request changes) is recorded:

1. If human **approves** a PR that Katherine flagged -> the threshold was too conservative for this type of change.
2. If human **requests changes** on a PR that Katherine auto-approved -> the threshold was too permissive for this type of change.
3. If human **approves** a PR that Katherine auto-approved -> correct decision, reinforce.
4. If human **requests changes** on a PR that Katherine flagged -> correct decision, reinforce.

A lightweight model (logistic regression or similar) is periodically retrained on
the decision history to recalibrate the threshold. The system trends toward
flagging fewer PRs as it learns the team's standards.

### Weight Adjustments

When a human reviews Katherine's decisions, the scoring component weights are
adjusted. Over time, Katherine calibrates what truly needs human attention
based on the team's specific patterns and risk tolerance.

---

## Interaction with Other Agents

- **Leonard (spec 20-leonard)**: Katherine reviews Leonard's code. If changes are needed, Katherine publishes structured feedback. Leonard receives the feedback and iterates. This loop continues until Katherine approves or escalates.
- **Nelson (spec 16)**: Katherine uses Nelson for both the code review consensus and the human review scoring consensus. Nelson is Katherine's primary decision-making tool.
- **Richelieu (spec 18)**: Katherine posts review comments on the task PR via the orchestrator (which dispatches to Richelieu). After approval, the orchestrator tells Richelieu to merge.
- **Orchestrator (spec 13-orchestrator)**: The orchestrator spawns Katherine, consumes review results, and orchestrates the rework loop or merge.
- **Sherlock (spec 19)**: Katherine reads Sherlock's execution plan to understand what was expected and verify the implementation matches.

---

## Principles Integration

Katherine enforces **review principles** from `.opex/principles/review/`:

1. Loads all active review principles at the start of each review.
2. Includes principles in the Nelson consensus request context.
3. Each principle acts as an additional review criterion beyond the four standard dimensions.
4. When a principle is violated, Katherine includes it in the feedback to Leonard with a reference to the principle.

### Principle Confidence Updates

When Katherine's reviews align with human reviews:
- Principles that Katherine correctly enforced get a confidence boost.
- Principles that Katherine missed (human caught a violation) get flagged for review.
- This feedback loop improves principle coverage over time.

---

## System Prompt

<!-- TODO: Define the system prompt template for Katherine -->
<!-- Katherine's system prompt should establish her role as a thorough
     code reviewer, emphasize the four review dimensions (correctness,
     style, safety, simplicity), instruct her to provide actionable
     feedback when requesting changes, and stress that the human review
     score must be calculated honestly (not biased toward auto-approve). -->

---

## PydanticAI Tools

<!-- TODO: Define the tools available to Katherine -->

Expected tools:

- **Diff reader**: Read the git diff for the task branch.
- **File reader**: Read source files in the repo and worktree for context.
- **Complexity analysis**: Calculate cyclomatic complexity delta, count files changed.
- **Nelson request**: Submit consensus requests for code review and human review scoring.
- **Principles reader**: Read and filter review principles from `.opex/principles/review/`.
- **Historical data query**: Query PostgreSQL for historical review accuracy data.
- **Redis publish**: Publish `review_result` messages.

---

## Error Handling

- **Nelson failure during review**: If Nelson cannot reach consensus on any review
  dimension, Katherine escalates the review to human (`review_result:escalated`).
- **Container crash**: The orchestrator respawns Katherine with the same task context.
  Since Katherine is stateless (reads diff and context from git/Redis), a respawn
  starts the review from scratch.
- **Rework loop limit**: If Leonard fails to address feedback after N rework
  iterations (tracked by the orchestrator), Katherine escalates to human.
- **Diff too large**: If the diff exceeds a size threshold, Katherine flags for
  human review regardless of score (large diffs are inherently risky).

---

## Resource Profile

| Metric             | Value  | Rationale                                |
|--------------------|--------|------------------------------------------|
| CPU limit          | 1.0    | Diff analysis, LLM calls                |
| Memory limit       | 1G     | Holding diff and context in memory       |
| CPU reservation    | 0.25   |                                          |
| Memory reservation | 256M   |                                          |

Overridable via `.opex.yaml` `resources.katherine` section (see spec 12).

---

## Configuration

- **`.opex.yaml`** (`review:` section):
  - `human_review_threshold`: Score above which human review is required (default: 0.7).
  - `always_human_review`: File path patterns that always trigger human review.
- **`.opex.yaml`** (`llm:` section): Model configuration for Nelson consensus calls.
- **Environment variables**: `PIPELINE_ID`, `TASK_ID`, `OPENROUTER_API_KEY`, `DATABASE_URL`, `REDIS_URL`.
- **LLM model**: Uses `llm.default_model` from `.opex.yaml` (or per-agent override `llm.overrides.katherine`).
- **PostgreSQL**: Historical review decisions for adaptive threshold learning.

---

## Testing Strategy

- **Unit tests**: Test review logic with mock diffs and mock Nelson responses.
  Verify correct categorization (approve, changes_requested, escalated) for
  various review scenarios.
- **Integration tests**: Use VCR cassettes (see spec 10) to record real LLM
  review calls. Test the full flow from diff input to `review_result` output.
- **Scoring tests**: Test human review score calculation with known inputs.
  Verify threshold behavior at boundaries (score exactly at threshold, just above,
  just below).
- **Adaptive threshold tests**: Simulate sequences of human decisions and verify
  that the threshold adjusts correctly over time. Test convergence behavior.
- **Principle enforcement tests**: Verify that review principles are included in
  Nelson requests and that violations are correctly identified in feedback.
- **Edge cases**: Test with empty diffs, very large diffs, diffs touching
  always-flag paths, and reviews where Nelson escalates.
