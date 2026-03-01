# Next Steps

> **Purpose**: This file tracks where we left off and what to do next. Update it
> at the end of every session so the next session can pick up seamlessly.
>
> **Last updated**: 2026-03-01 (session 3, in progress)

---

## Current Status

### Completed

- **Spec 00 — Overview**: Done. Mermaid diagrams, spec index, agent roster, tech
  stack, design decisions, implementation phases. Fully reviewed.
- **Spec reorganization**: All 22 spec files created, renumbered, cross-referenced.
  Old files deleted. Foundation (00-05) → Cross-cutting (06-12) → Components (13-21).
- **Spec 01 deep-dive decisions**: 13 original improvements + 5 new major topics
  (learning mode, principles, API server, TUI architecture, branching/PR model).
  All decisions documented and migrated into the appropriate specs.
- **Specs 01/02 swap**: Workflow is now `01-workflow.md`, Data Models is now
  `02-data-models.md`. All cross-references updated.
- **Controller → Orchestrator rename**: File renamed to `13-orchestrator.md`.
  222 occurrences replaced across 23 files. Zero stale references.
- **Project structure overhaul** (in `00-overview.md`):
  - Added `orchestrator/`, `api/`, `tui/` packages. Removed `dashboard/`.
  - All packages under `ai_team.*` namespace (PEP 420 namespace packages).
  - Flat layout (no `src/` directory). Tests colocated with each package.
  - Phases updated: added Phase 8 (API Server), Phase 9 (TUI), renumbered
    E2E Integration to Phase 10, Hardening to Phase 11.
- **Planning Principles**: Created `knowledge_base/planning-principles.md` with
  8 principles (P1–P8) for validating specs. Coverage lattice: missing behavior,
  interaction bugs, vague design, runaway behavior, consistency drift.
- **Workflow Issue 1 — Failure scenarios**: Fully resolved. Decisions:
  - Task-level failure isolation (independent siblings continue).
  - No automatic `FAILED` state (P8 — No Silent Failures). All failures
    escalate to `NEEDS_HUMAN`.
  - `PARTIALLY_FAILED` pipeline state for recovery.
  - Auto-retries from scratch (configurable, default 2, overridable from TUI).
  - Single commit per task (Julius creates small tasks).
  - Two-phase human intervention: diagnostic chat → context distillation.
  - Nelson consensus invokable at any point in diagnostic chat.
  - Human context history as timeline (`task_context_entries` table).
  - Chat-mediated task description editing with diff review.
  - Two abandon modes: task-level takeover (Katherine verifies prerequisites)
    and pipeline-level cancellation (reason recorded, branches preserved).
  - Katherine `verify_prerequisites` mode for human takeover validation.
  - Updated specs: 01 (workflow), 02 (data models), 08 (error recovery),
    14 (API server), 15 (TUI).

---

## Where We Left Off

**Continuing workflow spec (01) deep-dive.** Issue 1 is resolved. 14 issues
remain across three categories.

### Workflow Spec Issues — Ready for Discussion

#### Category 1: Missing Flows (4 remaining)

**Issue 2 — Pipeline cancellation flow is undefined.**
`CANCELLED` is a lifecycle state but there's no description of: how a human triggers
it, what happens to running containers, what happens to open PRs/branches, or whether
a cancelled pipeline can be resumed.
*Note: Issue 1 partially addressed this (Option 3: Cancel pipeline), but the
detailed container shutdown sequence and PR cleanup are not yet specified.*

**Issue 3 — NEEDS_HUMAN recovery path is incomplete.**
*Mostly resolved by Issue 1* (diagnostic chat, context distillation, takeover,
cancel). Remaining question: is there a notification mechanism beyond the TUI
banner (e.g., email, Slack webhook) for when NEEDS_HUMAN is triggered?

**Issue 4 — Post-merge conflict resolution trigger is missing.**
Between steps 14 and 15, after a task PR merges, the spec describes dependency
dispatch but doesn't mention rebasing sibling PRs. The conflict resolution section
exists but isn't integrated into the 20-step walkthrough.

**Issue 5 — Parallel pipelines interaction is undefined.**
Branching model shows Feature A and Feature B in parallel, but: do they share
parallelism limits? Can they conflict on `main`? What about simultaneous feature PRs?

#### Category 2: Ambiguities (7 issues)

**Issue 6 — Step 1: TUI → API → pipeline, not TUI → Redis.**
Step 1 says "TUI publishes pipeline_created to system:pipelines" but TUI connects
to the API server (spec 14), which writes to Redis. TUI should never touch Redis.

**Issue 7 — Step 4: What does Julius's consensus request validate?**
Julius publishes a consensus_request and Nelson validates — but validates what?
Decomposition quality? Task graph? Scope?

**Issue 8 — Step 12: How is feedback "attached" to respawned Leonard?**
"Respawns Leonard with feedback attached" — via env var? PostgreSQL record? Redis?

**Issue 9 — Step 13: GitHub PR merge vs. local git merge.**
Does Richelieu merge via GitHub API (PR metadata, status checks) or local
`git merge` + push (faster but bypasses checks)?

**Issue 10 — Review score semantics.**
"Human review score" — what does it represent? Confidence? Risk? What's the scale?
Higher = more human review needed, but this should be explicit.

**Issue 11 — Max review cycles.**
Leonard implements → Katherine requests changes → loop. No limit specified. Should
there be a max (e.g., 3 cycles) before escalation?

**Issue 12 — "Auto-resolution" in conflict resolution.**
What does "Richelieu attempts auto-resolution" mean? Git's built-in merge? LLM-assisted?

#### Category 3: Consistency Checks (3 issues)

**Issue 13 — Worktree path vs. volume mount.**
Spec says `/workspace/` is the main clone. Is this the Docker volume mount? Cross-check
with spec 05 (infrastructure).

**Issue 14 — Branch naming origin.**
`ai-team/add-auth/task-1` — where does `add-auth` come from? Pipeline name? Human input?
Julius?

**Issue 15 — Parallelism limits scope.**
Are `max_parallel_leonards: 3` etc. per-pipeline or global across all pipelines?

---

## Spec Review Progress

| Spec | Status | Notes |
|------|--------|-------|
| 00 Overview | **Reviewed** | Done. Updated with orchestrator rename, new project structure, ai_team.* namespace, revised phases. |
| 01 Workflow | **In review** | 1/15 issues resolved (Issue 1: failure handling). 14 remaining. |
| 02 Data Models | **Updated** | New: TaskStatus states, PipelineStatus states, RetryConfig, task_attempts, task_context_entries, diagnostic_chat tables. Needs full review after spec 01 completes. |
| 03 Learning & Principles | Needs review | Verify Extract→Replay→Verify loop is fully specified. |
| 04 Communication | Needs review | Check message types match orchestrator routing table. |
| 05 Infrastructure | Needs review | Docker Compose is canonical. Check for consistency. |
| 06 Observability | Needs review | Logging, Grafana/Loki, dashboards. |
| 07 Cost Tracking | Needs review | Budget enforcement, attribution. |
| 08 Error Recovery | **Updated** | Added P8 note, two-level recovery (agent vs task), updated partial failure section. Needs full review. |
| 09 Security | Needs review | Secret scoping, socket proxy, egress. |
| 10 Testing | Needs review | VCR, testcontainers. |
| 11 Dev Standards | Needs review | ruff, mypy, coverage. |
| 12 Repo Connection | Needs review | GitHub auth, .ai-team.yaml full schema. |
| 13 Orchestrator | Needs review | Event router, launcher, watchdog. Largest component spec. |
| 14 API Server | **Updated** | Added human intervention endpoints, updated capabilities table. Still has TODOs for schemas/error format. |
| 15 TUI | **Updated** | Added diagnostic chat mockups, human intervention actions, diff view capability. Still has TODOs for keybindings/principle browser. |
| 16 Nelson | Mostly complete | Full consensus algorithm. Needs prompt/tool review. |
| 17 Julius | **Has TODOs** | System prompt, PydanticAI tools, decomposition examples. |
| 18 Richelieu | Mostly complete | Git operations well-defined. Needs prompt/tool review. |
| 19 Sherlock | **Has TODOs** | System prompt, PydanticAI tools, enrichment examples. |
| 20 Leonard | **Has TODOs** | System prompt, PydanticAI tools, implementation examples. |
| 21 Katherine | **Has TODOs** | System prompt, PydanticAI tools, review criteria detail. Add `verify_prerequisites` mode. |

---

## Critical Open Topics

These block implementation. From `docs/specs/TODO-open-topics.md`:

### 1. Agent Prompts & Tool Definitions (specs 16-21)
The #1 blocker. Each agent needs:
- System prompt template (what instructions does the LLM receive?)
- PydanticAI tool definitions (what tools can it call? file read, shell exec, git, search?)
- Input/output schemas (what structured data does it receive/produce?)
- Decision examples (sample inputs → expected behavior)

**Suggested order** (follows data dependency chain):
1. Nelson (16) — everything depends on consensus
2. Julius (17) — first pipeline step (decomposition)
3. Richelieu (18) — git/workspace (needed before Leonard)
4. Sherlock (19) — enrichment (feeds Leonard)
5. Leonard (20) — implementation
6. Katherine (21) — review (last in chain)

### 2. API Server Detail (spec 14)
- Request/response Pydantic schemas for each endpoint
- Error response format
- Rate limiting for TUI requests
- Token management
- Diagnostic chat endpoint schemas (new from Issue 1)

### 3. TUI Detail (spec 15)
- Textual widget hierarchy
- Keybindings
- Learning mode chat interface UX
- Diagnostic chat UX (new from Issue 1)
- Diff view component (new from Issue 1)
- Multi-team switching UX

### 4. Rate Limiting (system-wide)
How to handle OpenRouter rate limits with parallel Nelson calls + multiple Leonards.

### 5. Caching Strategy
File read caching across Sherlock/Leonard. Prompt caching for Nelson.

---

## Knowledge Base

Planning principles and other reusable knowledge are stored in the
`knowledge_base/` directory at the project root.

| File | Content |
|------|---------|
| `planning-principles.md` | 8 principles (P1–P8) for validating specs. Coverage lattice, validation checklists, anti-patterns, issue mapping. |

---

## Suggested Next Session Plan

### Step 1: Continue the 14 remaining Workflow issues (one by one)
Resume with Issue 2 (pipeline cancellation). Use planning principles to
validate decisions. Update specs after each resolution.

### Step 2: Deep-dive Data Models (spec 02)
Verify every PostgreSQL table and Pydantic model matches the decisions from all specs.

### Step 3: Deep-dive remaining specs (03-12)
Continue the one-by-one review through foundation + cross-cutting specs.

### Step 4: Deep-dive component specs (13-21)
Review orchestrator, API server, TUI, and all agent specs.

### Step 5: Fill agent prompt/tool TODOs
Define system prompts and PydanticAI tools for each agent (16-21).

### Step 6: Build plan
Create a detailed Phase 0 implementation plan.

---

## Session Log

| Date | What was done |
|------|---------------|
| 2026-03-01 (s1) | Deep-dive spec 01 (deployment). 13 improvements + 5 new topics (learning mode, principles, API server, TUI, branching). Full rewrite. Reorganized all specs from 13→22 files. Renumbered: foundation (00-05), cross-cutting (06-12), components (13-21). Converted diagrams to Mermaid. |
| 2026-03-01 (s2) | Swapped specs 01/02 (Workflow before Data Models). Renamed Controller → Orchestrator (file + 222 references). Overhauled project structure: ai_team.* namespace packages, flat layout, colocated tests, added orchestrator/api/tui packages, removed dashboard. Updated phases (added API Server + TUI, renumbered). Analyzed workflow spec — identified 15 issues for deep-dive discussion. |
| 2026-03-01 (s3) | Created `knowledge_base/planning-principles.md` (8 principles: P1–P8). Resolved Workflow Issue 1 (failure scenarios) with 14 sub-decisions. Key design: no auto-FAILED state (P8), task-level failure isolation, PARTIALLY_FAILED pipeline state, diagnostic chat for human intervention, two abandon modes, Katherine verify_prerequisites mode. Updated specs 01, 02, 08, 14, 15. |
