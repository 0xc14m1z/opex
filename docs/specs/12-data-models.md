# 12 — Data Models

## Overview

All data flowing through the system is defined as Pydantic models. No raw dicts.
This document catalogs every model, its purpose, and where it's used.

---

## Model Organization

```
core/src/ai_team_core/models/
├── __init__.py          # Re-exports all models
├── config.py            # .ai-team.yaml schema
├── task.py              # Tasks, execution plans, dependency graphs
├── consensus.py         # Consensus requests, responses, weights
├── review.py            # Code review, human review scoring
├── git.py               # Git operations, branches, worktrees
├── cost.py              # Cost records, budget limits
├── messages.py          # Redis message envelopes and payloads
├── checkpoint.py        # Agent checkpoints
└── common.py            # Shared enums, base classes
```

## Common Types

```python
# models/common.py

from enum import StrEnum

class AgentName(StrEnum):
    NELSON = "nelson"
    JULIUS = "julius"
    SHERLOCK = "sherlock"
    LEONARD = "leonard"
    KATHERINE = "katherine"
    RICHELIEU = "richelieu"

class TaskStatus(StrEnum):
    PENDING = "pending"              # Created, waiting for dependencies
    READY = "ready"                  # Dependencies met, can be picked up
    ENRICHING = "enriching"          # Sherlock is analyzing
    IN_PROGRESS = "in_progress"      # Leonard is implementing
    IN_REVIEW = "in_review"          # Katherine is reviewing
    REWORK = "rework"                # Leonard is fixing review feedback
    COMPLETED = "completed"          # Merged into feature branch
    FAILED = "failed"                # Max retries exceeded
    CANCELLED = "cancelled"          # Cancelled by human
    NEEDS_HUMAN = "needs_human"      # Escalated, waiting for human

class Priority(StrEnum):
    LOW = "low"
    NORMAL = "normal"
    HIGH = "high"

class ConsensusStatus(StrEnum):
    AGREED = "agreed"                # Unanimous
    MAJORITY = "majority"            # 2/3 agree
    WEIGHTED = "weighted"            # Weighted vote winner
    ESCALATED = "escalated"          # No convergence, needs human
    ERROR = "error"                  # Something went wrong
```

## Configuration Models

```python
# models/config.py

class ProjectConfig(BaseModel):
    name: str
    description: str
    language: str
    framework: str | None = None
    python_version: str | None = None

class KnowledgeConfig(BaseModel):
    architecture_docs: str | None = None
    api_docs: str | None = None
    style_guide: str | None = None
    adr_directory: str | None = None
    additional: list[str] = []

class CommandsConfig(BaseModel):
    install: str
    test: str
    lint: str | None = None
    format: str | None = None
    type_check: str | None = None
    build: str | None = None
    custom: dict[str, str] = {}

class GitConfig(BaseModel):
    default_branch: str = "main"
    branch_prefix: str = "ai-team/"
    require_pr: bool = True
    auto_merge: bool = False
    conventional_commits: bool = True

class ReviewConfig(BaseModel):
    human_review_threshold: float = 0.7
    always_human_review: list[str] = []
    never_auto_approve: list[str] = []

class IntakeConfig(BaseModel):
    class Labels(BaseModel):
        trigger: str = "ai-team"
        in_progress: str = "ai-team:working"
        done: str = "ai-team:done"
        needs_human: str = "ai-team:needs-human"
    labels: Labels = Labels()
    issue_template: str | None = None

class BudgetConfig(BaseModel):
    soft_limit_per_task: float = 5.0
    hard_limit_per_task: float = 20.0

class AiTeamConfig(BaseModel):
    """Root model for .ai-team.yaml."""
    version: str
    project: ProjectConfig
    knowledge: KnowledgeConfig = KnowledgeConfig()
    commands: CommandsConfig
    guidelines: list[str] = []
    protected_paths: list[str] = []
    git: GitConfig = GitConfig()
    review: ReviewConfig = ReviewConfig()
    intake: IntakeConfig = IntakeConfig()
    budget: BudgetConfig = BudgetConfig()
```

## Task Models

```python
# models/task.py

class Task(BaseModel):
    """A single unit of work in a pipeline."""
    id: str                                  # "task-{uuid}"
    pipeline_id: str
    title: str
    description: str
    status: TaskStatus = TaskStatus.PENDING
    priority: Priority = Priority.NORMAL

    # Dependencies
    depends_on: list[str] = []               # Task IDs
    blocks: list[str] = []                   # Task IDs (computed)

    # Metadata from Julius
    estimated_complexity: Literal["low", "medium", "high"]
    files_likely_affected: list[str]
    acceptance_criteria: list[str]

    # Assigned agent
    assigned_to: str | None = None           # "leonard-1"
    branch: str | None = None                # "ai-team/feature/task-id"
    worktree_path: str | None = None

    # Execution tracking
    attempt_count: int = 0
    max_attempts: int = 3

    # Cost
    budget_soft: float
    budget_hard: float
    cost_so_far: float = 0.0

    # Timestamps
    created_at: datetime
    started_at: datetime | None = None
    completed_at: datetime | None = None


class DependencyGraph(BaseModel):
    """Represents task dependencies and parallelization groups."""
    tasks: list[Task]
    edges: list[tuple[str, str]]             # (from_task_id, to_task_id)
    parallel_groups: list[list[str]]         # Groups of tasks that can run together

    def get_ready_tasks(self) -> list[Task]:
        """Returns tasks whose dependencies are all completed."""
        completed = {t.id for t in self.tasks if t.status == TaskStatus.COMPLETED}
        return [
            t for t in self.tasks
            if t.status == TaskStatus.PENDING
            and all(dep in completed for dep in t.depends_on)
        ]


class ExecutionPlan(BaseModel):
    """Sherlock's output: a detailed plan for Leonard."""
    task_id: str
    steps: list[ExecutionStep]
    files_to_modify: list[FileModification]
    files_to_create: list[FileCreation]
    patterns_to_follow: list[CodePattern]
    test_strategy: TestStrategy
    guidelines: list[str]


class ExecutionStep(BaseModel):
    order: int
    description: str
    file_path: str | None
    action: Literal["create", "modify", "delete", "run_command"]
    details: str                             # Specific instructions


class FileModification(BaseModel):
    path: str
    what_to_change: str                      # Description of changes
    relevant_lines: tuple[int, int] | None   # Line range hint
    example_pattern: str | None              # Code pattern to follow


class FileCreation(BaseModel):
    path: str
    purpose: str
    template: str | None                     # Boilerplate to start from


class CodePattern(BaseModel):
    description: str
    example_file: str
    example_lines: tuple[int, int] | None
    snippet: str                             # The actual code pattern


class TestStrategy(BaseModel):
    test_files_to_modify: list[str]
    test_files_to_create: list[str]
    test_command: str                        # From .ai-team.yaml
    coverage_expectation: str                # "maintain or increase"
```

## Consensus Models

```python
# models/consensus.py

class ConsensusRequest(BaseModel):
    request_id: str
    requester: AgentName | str               # Agent name or "human"
    task_id: str | None = None
    prompt: str
    system_context: str | None = None
    review_goals: list[str] | None = None
    response_schema: dict | None = None
    max_rounds: int = 3
    required_agreement: float = 0.67
    providers: list[str] | None = None
    priority: Priority = Priority.NORMAL


class ProviderResponse(BaseModel):
    provider: str
    model: str
    round_number: int
    decision: dict | str                     # Structured or free text
    reasoning: str
    review_of_others: str | None = None
    changed_position: bool = False
    tokens: TokenUsage
    cost_usd: float
    latency_ms: int


class DissentRecord(BaseModel):
    provider: str
    position: str
    reasoning: str


class ConsensusResponse(BaseModel):
    request_id: str
    status: ConsensusStatus
    decision: dict | str
    confidence: float
    reasoning: str
    dissenting_opinions: list[DissentRecord] = []
    rounds_taken: int
    provider_responses: list[list[ProviderResponse]]  # Grouped by round
    total_tokens: TokenUsage
    total_cost: float
    elapsed_seconds: float


class ProviderWeight(BaseModel):
    provider: str
    scope: str                               # "global", "code_review", "project:my-app"
    weight: float
    updated_at: datetime
    sample_count: int                        # How many decisions this is based on
```

## Review Models

```python
# models/review.py

class ReviewComment(BaseModel):
    file_path: str
    line_start: int
    line_end: int | None = None
    severity: Literal["critical", "major", "minor", "suggestion"]
    comment: str
    suggested_fix: str | None = None


class HumanReviewScore(BaseModel):
    """Nelson-calculated score determining if human review is needed."""
    task_id: str
    novelty: float                           # 0.0 – 1.0
    complexity: float                        # 0.0 – 1.0
    ai_confidence: float                     # Nelson's consensus confidence
    risk: float                              # 0.0 – 1.0
    composite_score: float                   # Weighted combination
    needs_human: bool                        # score > threshold
    reasoning: str                           # Why this score


class ReviewResult(BaseModel):
    task_id: str
    decision: Literal["approved", "changes_requested", "escalated"]
    comments: list[ReviewComment]
    human_review_score: HumanReviewScore
    consensus_id: str                        # Which Nelson consensus backed this
    reviewed_at: datetime


class HumanDecision(BaseModel):
    """Recorded when a human approves or requests changes."""
    task_id: str
    pipeline_id: str
    decision: Literal["approved", "changes_requested"]
    was_flagged_for_review: bool             # Did Katherine flag this?
    katherine_decision: str                  # What Katherine decided
    feedback: str | None                     # Human's comment
    decided_at: datetime
```

## Git Models

```python
# models/git.py

class WorktreeInfo(BaseModel):
    path: str
    branch: str
    task_id: str
    created_at: datetime
    status: Literal["active", "merged", "abandoned"]


class BranchInfo(BaseModel):
    name: str
    base_branch: str
    pipeline_id: str
    task_id: str | None                      # None for feature branches
    created_at: datetime
    merged_at: datetime | None = None


class MergeResult(BaseModel):
    success: bool
    source_branch: str
    target_branch: str
    commit_sha: str | None                   # None if merge failed
    conflicts: list[str] | None              # Conflicting files if any
    auto_resolved: bool = False


class PRInfo(BaseModel):
    number: int
    url: str
    title: str
    body: str
    branch: str
    base_branch: str
    pipeline_id: str
    tasks_included: list[str]                # Task IDs
    human_review_needed: bool
    created_at: datetime
```

## Cost Models

```python
# models/cost.py

class TokenUsage(BaseModel):
    prompt_tokens: int
    completion_tokens: int
    total_tokens: int


class CostRecord(BaseModel):
    id: str
    timestamp: datetime
    pipeline_id: str
    task_id: str | None
    agent: str
    consensus_request_id: str | None = None
    consensus_round: int | None = None
    provider: str
    model: str
    prompt_tokens: int
    completion_tokens: int
    total_tokens: int
    cost_usd: float
    purpose: str                             # "enrichment", "implementation", etc.


class BudgetStatus(BaseModel):
    scope: Literal["task", "daily"]
    identifier: str                          # Task ID or "daily"
    current_cost: float
    soft_limit: float
    hard_limit: float
    percent_used: float
    status: Literal["ok", "warning", "exceeded"]
```

## Checkpoint Models

```python
# models/checkpoint.py

class Checkpoint(BaseModel):
    id: str
    agent: str
    task_id: str
    pipeline_id: str
    checkpoint_name: str
    state: dict                              # Agent-specific serialized state
    created_at: datetime
    expires_at: datetime
```

## SQLite Schema Summary

```sql
-- Tasks
CREATE TABLE tasks (
    id TEXT PRIMARY KEY,
    pipeline_id TEXT NOT NULL,
    data TEXT NOT NULL,                -- JSON (Task model)
    status TEXT NOT NULL,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL
);

-- Consensus history
CREATE TABLE consensus_history (
    request_id TEXT PRIMARY KEY,
    data TEXT NOT NULL,                -- JSON (ConsensusResponse model)
    created_at TEXT NOT NULL
);

-- Provider weights
CREATE TABLE provider_weights (
    provider TEXT NOT NULL,
    scope TEXT NOT NULL,
    weight REAL NOT NULL,
    sample_count INTEGER NOT NULL,
    updated_at TEXT NOT NULL,
    PRIMARY KEY (provider, scope)
);

-- Cost records
CREATE TABLE cost_records (
    id TEXT PRIMARY KEY,
    pipeline_id TEXT NOT NULL,
    task_id TEXT,
    agent TEXT NOT NULL,
    provider TEXT NOT NULL,
    model TEXT NOT NULL,
    prompt_tokens INTEGER NOT NULL,
    completion_tokens INTEGER NOT NULL,
    cost_usd REAL NOT NULL,
    purpose TEXT NOT NULL,
    consensus_request_id TEXT,
    consensus_round INTEGER,
    timestamp TEXT NOT NULL
);

-- Checkpoints
CREATE TABLE checkpoints (
    id TEXT PRIMARY KEY,
    agent TEXT NOT NULL,
    task_id TEXT NOT NULL,
    pipeline_id TEXT NOT NULL,
    checkpoint_name TEXT NOT NULL,
    state TEXT NOT NULL,              -- JSON
    created_at TEXT NOT NULL,
    expires_at TEXT NOT NULL
);

-- Human decisions (for learning)
CREATE TABLE human_decisions (
    id TEXT PRIMARY KEY,
    task_id TEXT NOT NULL,
    pipeline_id TEXT NOT NULL,
    decision TEXT NOT NULL,
    was_flagged BOOLEAN NOT NULL,
    katherine_decision TEXT NOT NULL,
    feedback TEXT,
    decided_at TEXT NOT NULL
);

-- Audit log
CREATE TABLE audit_log (
    id TEXT PRIMARY KEY,
    timestamp TEXT NOT NULL,
    agent TEXT NOT NULL,
    event_type TEXT NOT NULL,
    details TEXT NOT NULL,            -- JSON
    severity TEXT NOT NULL
);

-- Repo connections
CREATE TABLE repo_connections (
    id TEXT PRIMARY KEY,
    repo_url TEXT NOT NULL UNIQUE,
    auth_method TEXT NOT NULL,
    default_branch TEXT NOT NULL,
    config_hash TEXT NOT NULL,
    connected_at TEXT NOT NULL,
    last_synced_at TEXT NOT NULL
);

-- Indexes
CREATE INDEX idx_tasks_pipeline ON tasks(pipeline_id);
CREATE INDEX idx_tasks_status ON tasks(status);
CREATE INDEX idx_costs_pipeline ON cost_records(pipeline_id);
CREATE INDEX idx_costs_task ON cost_records(task_id);
CREATE INDEX idx_costs_daily ON cost_records(timestamp);
CREATE INDEX idx_checkpoints_lookup ON checkpoints(agent, task_id, pipeline_id);
CREATE INDEX idx_human_decisions_task ON human_decisions(task_id);
CREATE INDEX idx_audit_timestamp ON audit_log(timestamp);
```
