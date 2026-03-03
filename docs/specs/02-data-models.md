# 02 — Data Models

> **Migrated from**: `docs/specs/12-data-models.md` (all content) with additions from `docs/specs/01-deployment.md` (principles, learning_conversations, review_threshold_history tables, and pipelines table extensions)

---

## Overview

All data flowing through the system is defined as Pydantic models. No raw dicts.
This document catalogs every model, its purpose, and where it's used.

---

## Model Organization

```
core/src/opex_core/models/
├── __init__.py          # Re-exports all models
├── config.py            # .opex.yaml schema
├── task.py              # Tasks, execution plans, dependency graphs
├── consensus.py         # Consensus requests, responses, weights
├── review.py            # Code review, confidence scoring
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
    PENDING = "pending"                          # Created, waiting for dependencies
    READY = "ready"                              # Dependencies met, can be picked up
    ENRICHING = "enriching"                      # Sherlock is analyzing
    IN_PROGRESS = "in_progress"                  # Leonard is implementing
    IN_REVIEW = "in_review"                      # Katherine is reviewing
    REWORK = "rework"                            # Leonard is fixing review feedback
    MERGING = "merging"                          # Richelieu is merging the task PR
    RETRYING = "retrying"                        # New attempt starting (after human context)
    NEEDS_HUMAN = "needs_human"                  # Escalated, waiting for human decision
    ABANDONED_HUMAN_TAKEOVER = "abandoned_human_takeover"  # Human took over externally
    COMPLETED = "completed"                      # Merged into feature branch
    COMPLETED_EXTERNALLY = "completed_externally"  # Human finished, Katherine verified
    CANCELLED = "cancelled"                      # Cancelled by human (pipeline-level)
    # NOTE: There is no FAILED state. The system never gives up autonomously.
    # All failures escalate to NEEDS_HUMAN. See spec 01 (P8 — No Silent Failures).

class PipelineStatus(StrEnum):
    CREATED = "created"              # Plan received, not yet started
    DECOMPOSING = "decomposing"      # Julius is running
    IN_PROGRESS = "in_progress"      # Tasks are being processed
    COMPLETING = "completing"        # All tasks done, opening feature PR
    MONITORING = "monitoring"        # Feature PR open; system actively maintains branch
                                     # (rebases, CI fixes, Katherine re-reviews)
    MERGED = "merged"                # Human merged the feature PR. Pipeline truly done.
    PARTIALLY_FAILED = "partially_failed"  # Some tasks need human intervention
    CANCELLING = "cancelling"        # Human requested cancel, waiting for containers to exit
    CANCELLED = "cancelled"          # All containers exited, cancellation finalized
    # NOTE: There is no FAILED state. See spec 01 (P8 — No Silent Failures).
    # PARTIALLY_FAILED can recover via retry or human takeover → MONITORING,
    # or be explicitly cancelled by the human → CANCELLED.

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
    branch_prefix: str = "opex/"
    require_pr: bool = True
    auto_merge: bool = False
    conventional_commits: bool = True

class ReviewConfig(BaseModel):
    confidence_threshold: float = 0.7
    always_human_review: list[str] = []
    never_auto_approve: list[str] = []

class IntakeConfig(BaseModel):
    class Labels(BaseModel):
        trigger: str = "opex"
        in_progress: str = "opex:working"
        done: str = "opex:done"
        needs_human: str = "opex:needs-human"
    labels: Labels = Labels()
    issue_template: str | None = None

class BudgetConfig(BaseModel):
    soft_limit_per_task: float = 5.0
    hard_limit_per_task: float = 20.0

class RetryConfig(BaseModel):
    max_task_retries: int = 2              # 2 retries = 3 total attempts
    cleanup_ttl_hours: int = 48            # Auto-cleanup failed branches after N hours

class LLMConsensusConfig(BaseModel):
    models: list[str]                          # At least 2 for meaningful consensus
    max_rounds: int = 3

class LLMConfig(BaseModel):
    default_model: str = "anthropic/claude-sonnet-4"
    consensus: LLMConsensusConfig
    overrides: dict[AgentName, str] = {}       # agent → model ID

class OpexConfig(BaseModel):
    """Root model for .opex.yaml."""
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
    retries: RetryConfig = RetryConfig()       # Task retry configuration (spec 01)
    llm: LLMConfig                             # Model selection config
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
    branch: str | None = None                # "opex/feature/task-id"
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
    test_command: str                        # From .opex.yaml
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


class ConfidenceScore(BaseModel):
    """Nelson-calculated confidence score. Low confidence triggers human review."""
    task_id: str
    novelty: float                           # 0.0 – 1.0 (higher = more novel = less confident)
    complexity: float                        # 0.0 – 1.0 (higher = more complex = less confident)
    ai_confidence: float                     # Nelson's consensus confidence
    risk: float                              # 0.0 – 1.0 (higher = riskier = less confident)
    composite_score: float                   # Weighted combination (0.0 – 1.0, higher = more confident)
    needs_human: bool                        # score < threshold
    reasoning: str                           # Why this score


class ReviewResult(BaseModel):
    task_id: str
    decision: Literal["approved", "changes_requested", "escalated"]
    comments: list[ReviewComment]
    confidence_score: ConfidenceScore
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

## Task Attempt & Human Intervention Models

These models support the failure handling and human intervention flows
defined in spec 01. See spec 01 for the full task lifecycle and P8
(No Silent Failures) design principle.

```python
# models/task.py (additional models)

class TaskAttempt(BaseModel):
    """A single attempt at completing a task. Never overwritten."""
    id: str                                  # UUID
    task_id: str
    attempt_number: int                      # 1-indexed
    status: Literal["running", "succeeded", "failed", "abandoned"]
    started_at: datetime
    finished_at: datetime | None = None
    failure_reason: str | None = None        # Human-readable
    exit_code: int | None = None             # Agent container exit code
    agent_logs_ref: str | None = None        # Reference to logs in Loki
    diff_snapshot: str | None = None         # Git diff at time of failure


class TaskContextEntry(BaseModel):
    """An entry in the context chain for a task. Supports timeline reconstruction."""
    id: str                                  # UUID
    task_id: str
    entry_type: Literal["original", "human_addition", "task_edit"]
    content: str                             # The actual context text
    triggered_by: str | None = None          # FK to TaskAttempt.id (which attempt failed)
    chat_session_id: str | None = None       # FK to DiagnosticChatSession.id
    created_at: datetime
    created_by: str                          # "julius" | "human:{user_id}"


class DiagnosticChatSession(BaseModel):
    """A diagnostic chat session between a human and an LLM about a failed task."""
    id: str                                  # UUID
    task_id: str
    pipeline_id: str
    started_at: datetime
    ended_at: datetime | None = None
    outcome: Literal["context_added", "task_edited", "abandoned", "cancelled"] | None = None


class DiagnosticChatMessage(BaseModel):
    """A single message in a diagnostic chat session."""
    id: str                                  # UUID
    session_id: str                          # FK to DiagnosticChatSession.id
    role: Literal["human", "assistant", "nelson_consensus"]
    content: str
    timestamp: datetime
```

---

## Orchestrator Models

These payload models are used by the pipeline orchestrator (spec 13) and are the
canonical definitions for orchestrator lifecycle events. All use the standard
`MessageEnvelope` from spec 04.

```python
# models/messages.py (orchestrator payloads)

class PipelineCreatedPayload(BaseModel):
    """Published by the TUI/CLI when a human starts a new pipeline."""
    pipeline_id: str
    plan: str                                # The human's plan text
    repo_url: str
    target_branch: str
    config: OpexConfig                     # Parsed .opex.yaml


class DecompositionCompletePayload(BaseModel):
    """Published by Julius when it finishes decomposing the plan into tasks."""
    pipeline_id: str
    task_count: int
    dependency_graph: DependencyGraph
    ready_task_ids: list[str]                # Tasks with no dependencies (can start now)


class BatchDispatchedPayload(BaseModel):
    """Published by the orchestrator when it launches a batch of Sherlocks."""
    pipeline_id: str
    task_ids: list[str]                      # Tasks being dispatched in this batch
    batch_number: int                        # 1-indexed
    total_tasks: int
    remaining_tasks: int


class AllTasksCompletePayload(BaseModel):
    """Published by the orchestrator when every task in the pipeline reaches completed."""
    pipeline_id: str
    task_count: int
    total_cost: float
    elapsed_seconds: float


class PipelineMonitoringPayload(BaseModel):
    """Published by the orchestrator when the feature PR is opened and the
    pipeline enters MONITORING state. The system will now actively maintain
    the feature branch (rebases, CI fixes) until the human merges the PR."""
    pipeline_id: str
    pr_url: str
    pr_number: int
    tasks_completed: int
    total_cost: float
    elapsed_seconds: float
    human_review_needed: bool


class PipelineMergedPayload(BaseModel):
    """Published by the orchestrator when it detects the human has merged
    the feature PR. This is the true terminal state for a successful pipeline."""
    pipeline_id: str
    pr_url: str
    pr_number: int
    merged_at: datetime
    total_cost: float
    elapsed_seconds: float
    maintenance_actions: int          # Number of rebases/CI fixes during MONITORING


class PipelinePartiallyFailedPayload(BaseModel):
    """Published by the orchestrator when some tasks need human intervention."""
    pipeline_id: str
    tasks_completed: int
    tasks_needing_human: list[str]           # Task IDs in NEEDS_HUMAN state
    tasks_pending: int                       # Tasks still waiting (blocked by failed deps)
    total_cost: float

class PipelineCancelledPayload(BaseModel):
    """Published when a human explicitly cancels a pipeline."""
    pipeline_id: str
    reason: str                              # Human-provided cancellation reason
    tasks_completed: int
    tasks_cancelled: int
    branches_preserved: list[str]            # Branches left for salvage
    total_cost: float


class PipelineCancelRequestedPayload(BaseModel):
    """Published by the API server when a human requests pipeline cancellation."""
    pipeline_id: str
    reason: str                              # Human-provided cancellation reason
    requested_by: str                        # "human:{user_id}"


class PipelineCleanupRequestedPayload(BaseModel):
    """Published by the API server when a human requests cleanup of a cancelled pipeline."""
    pipeline_id: str
    requested_by: str                        # "human:{user_id}"
```

## PostgreSQL Schema

```sql
-- Schema migrations tracking
CREATE TABLE schema_migrations (
    version TEXT PRIMARY KEY,
    applied_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Pipelines
CREATE TABLE pipelines (
    id TEXT PRIMARY KEY,                     -- "pipe-{uuid}"
    plan TEXT NOT NULL,                      -- Original plan text
    repo_url TEXT NOT NULL,
    target_branch TEXT NOT NULL,
    status TEXT NOT NULL,                    -- PipelineStatus enum
    task_count INTEGER,                      -- Set after decomposition
    tasks_completed INTEGER DEFAULT 0,
    pr_url TEXT,                             -- Set when feature PR is opened
    pr_number INTEGER,                       -- GitHub PR number for feature PR
    total_cost NUMERIC(10,4) DEFAULT 0.0,
    learning_mode BOOLEAN DEFAULT true,      -- Whether learning mode is active
    learning_disabled_at_task TEXT,           -- Task ID when learning was turned off
    feature_branch TEXT,                     -- Feature branch name (e.g., feature/add-auth)
    target_branch_head TEXT,                 -- Last known HEAD of target branch (for polling)
    maintenance_actions INTEGER DEFAULT 0,   -- Count of rebases/CI fixes during MONITORING
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    started_at TIMESTAMPTZ,                  -- When Julius was launched
    monitoring_at TIMESTAMPTZ,               -- When feature PR opened, entered MONITORING
    merged_at TIMESTAMPTZ,                   -- When human merged the feature PR
    error TEXT                               -- Failure reason if failed
);

CREATE INDEX idx_pipelines_status ON pipelines(status);

-- Tasks
CREATE TABLE tasks (
    id TEXT PRIMARY KEY,
    pipeline_id TEXT NOT NULL REFERENCES pipelines(id),
    data JSONB NOT NULL,                     -- JSON (Task model)
    status TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Consensus history
CREATE TABLE consensus_history (
    request_id TEXT PRIMARY KEY,
    data JSONB NOT NULL,                     -- JSON (ConsensusResponse model)
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Provider weights
CREATE TABLE provider_weights (
    provider TEXT NOT NULL,
    scope TEXT NOT NULL,
    weight DOUBLE PRECISION NOT NULL,
    sample_count INTEGER NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (provider, scope)
);

-- Cost records
CREATE TABLE cost_records (
    id TEXT PRIMARY KEY,
    pipeline_id TEXT NOT NULL REFERENCES pipelines(id),
    task_id TEXT,
    agent TEXT NOT NULL,
    provider TEXT NOT NULL,
    model TEXT NOT NULL,
    prompt_tokens INTEGER NOT NULL,
    completion_tokens INTEGER NOT NULL,
    cost_usd NUMERIC(10,4) NOT NULL,
    purpose TEXT NOT NULL,
    consensus_request_id TEXT,
    consensus_round INTEGER,
    timestamp TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Checkpoints
CREATE TABLE checkpoints (
    id TEXT PRIMARY KEY,
    agent TEXT NOT NULL,
    task_id TEXT NOT NULL,
    pipeline_id TEXT NOT NULL REFERENCES pipelines(id),
    checkpoint_name TEXT NOT NULL,
    state JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at TIMESTAMPTZ NOT NULL
);

-- Human decisions (for learning)
CREATE TABLE human_decisions (
    id TEXT PRIMARY KEY,
    task_id TEXT NOT NULL,
    pipeline_id TEXT NOT NULL REFERENCES pipelines(id),
    decision TEXT NOT NULL,
    was_flagged BOOLEAN NOT NULL,
    katherine_decision TEXT NOT NULL,
    feedback TEXT,
    decided_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Audit log
CREATE TABLE audit_log (
    id TEXT PRIMARY KEY,
    timestamp TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    agent TEXT NOT NULL,
    event_type TEXT NOT NULL,
    details JSONB NOT NULL,
    severity TEXT NOT NULL
);

-- Repo connections
CREATE TABLE repo_connections (
    id TEXT PRIMARY KEY,
    repo_url TEXT NOT NULL UNIQUE,
    auth_method TEXT NOT NULL,
    default_branch TEXT NOT NULL,
    config_hash TEXT NOT NULL,
    connected_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_synced_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Active containers (managed by the orchestrator)
CREATE TABLE active_containers (
    container_id TEXT PRIMARY KEY,
    agent TEXT NOT NULL,                      -- AgentName
    pipeline_id TEXT NOT NULL REFERENCES pipelines(id),
    task_id TEXT,                             -- NULL for Julius (pipeline-level)
    status TEXT NOT NULL,                     -- "running", "exited", "dead"
    exit_code INTEGER,                        -- Set on exit
    retry_count INTEGER DEFAULT 0,
    launched_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_heartbeat TIMESTAMPTZ,
    exited_at TIMESTAMPTZ
);

-- Principles (distilled learnings from human feedback, see spec 03)
CREATE TABLE principles (
    id TEXT PRIMARY KEY,                     -- "impl-001", "rev-003"
    type TEXT NOT NULL,                      -- "implementation" or "review"
    title TEXT NOT NULL,
    content TEXT NOT NULL,                   -- Full markdown content
    agent TEXT NOT NULL,                     -- Target agent
    learned_from_pr TEXT,                    -- GitHub PR URL
    pipeline_id TEXT,
    task_id TEXT,
    confidence REAL DEFAULT 0.5,             -- 0.0 to 1.0
    times_applied INTEGER DEFAULT 0,         -- How many times used in subsequent tasks
    times_violated INTEGER DEFAULT 0,        -- How many times a subsequent task broke this
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Learning conversations (discussion-based learning records, see spec 03)
CREATE TABLE learning_conversations (
    id TEXT PRIMARY KEY,
    pipeline_id TEXT NOT NULL REFERENCES pipelines(id),
    task_id TEXT NOT NULL,
    replay_number INTEGER NOT NULL,          -- Which replay iteration
    messages JSONB NOT NULL,                 -- Full conversation thread
    principles_extracted TEXT[],             -- IDs of principles created/updated
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Review threshold history (adaptive learning audit trail, see spec 03)
CREATE TABLE review_threshold_history (
    id TEXT PRIMARY KEY,
    timestamp TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    previous_threshold REAL NOT NULL,
    new_threshold REAL NOT NULL,
    reason TEXT NOT NULL,                    -- Human-readable explanation
    principles_learned TEXT[],               -- Principle IDs that drove this change
    human_approvals INTEGER NOT NULL,        -- Count since last change
    human_rejections INTEGER NOT NULL,       -- Count since last change
    details JSONB NOT NULL                   -- Full reasoning chain:
                                             --   - What comments were made
                                             --   - What principles were extracted
                                             --   - What insights emerged
);

-- Task attempts (failure history, see spec 01 — Failure Handling)
CREATE TABLE task_attempts (
    id TEXT PRIMARY KEY,
    task_id TEXT NOT NULL REFERENCES tasks(id),
    attempt_number INTEGER NOT NULL,
    status TEXT NOT NULL,                     -- 'running' | 'succeeded' | 'failed' | 'abandoned'
    started_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    finished_at TIMESTAMPTZ,
    failure_reason TEXT,
    exit_code INTEGER,
    agent_logs_ref TEXT,                      -- Loki log reference
    diff_snapshot TEXT,                       -- Git diff at failure
    UNIQUE (task_id, attempt_number)
);

-- Task context entries (human intervention history, see spec 01 — Human Intervention Flow)
CREATE TABLE task_context_entries (
    id TEXT PRIMARY KEY,
    task_id TEXT NOT NULL REFERENCES tasks(id),
    entry_type TEXT NOT NULL,                 -- 'original' | 'human_addition' | 'task_edit'
    content TEXT NOT NULL,
    triggered_by TEXT REFERENCES task_attempts(id),  -- Which attempt failed
    chat_session_id TEXT,                     -- FK to diagnostic_chat_sessions
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by TEXT NOT NULL                  -- 'julius' | 'human:{user_id}'
);

-- Diagnostic chat sessions (see spec 01 — Human Intervention Flow)
CREATE TABLE diagnostic_chat_sessions (
    id TEXT PRIMARY KEY,
    task_id TEXT NOT NULL REFERENCES tasks(id),
    pipeline_id TEXT NOT NULL REFERENCES pipelines(id),
    started_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    ended_at TIMESTAMPTZ,
    outcome TEXT                              -- 'context_added' | 'task_edited' | 'abandoned' | 'cancelled'
);

-- Diagnostic chat messages
CREATE TABLE diagnostic_chat_messages (
    id TEXT PRIMARY KEY,
    session_id TEXT NOT NULL REFERENCES diagnostic_chat_sessions(id),
    role TEXT NOT NULL,                       -- 'human' | 'assistant' | 'nelson_consensus'
    content TEXT NOT NULL,
    timestamp TIMESTAMPTZ NOT NULL DEFAULT NOW()
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
CREATE INDEX idx_containers_pipeline ON active_containers(pipeline_id);
CREATE INDEX idx_containers_status ON active_containers(status);
CREATE INDEX idx_principles_type ON principles(type);
CREATE INDEX idx_principles_agent ON principles(agent);
CREATE INDEX idx_learning_conv_pipeline ON learning_conversations(pipeline_id);
CREATE INDEX idx_learning_conv_task ON learning_conversations(task_id);
CREATE INDEX idx_threshold_history_timestamp ON review_threshold_history(timestamp);
CREATE INDEX idx_task_attempts_task ON task_attempts(task_id);
CREATE INDEX idx_context_entries_task ON task_context_entries(task_id);
CREATE INDEX idx_chat_sessions_task ON diagnostic_chat_sessions(task_id);
CREATE INDEX idx_chat_sessions_pipeline ON diagnostic_chat_sessions(pipeline_id);
CREATE INDEX idx_chat_messages_session ON diagnostic_chat_messages(session_id);
```
