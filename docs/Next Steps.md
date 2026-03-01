# Next Steps

> **Purpose**: This file tracks where we left off and what to do next. Update it
> at the end of every session so the next session can pick up seamlessly.
>
> **Last updated**: 2026-03-01 (session 2)

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

---

## Where We Left Off

**Workflow spec (01) deep-dive is next.** The spec has been read and analyzed.
15 issues were identified across three categories. None have been discussed yet.

### Workflow Spec Issues — Ready for Discussion

#### Category 1: Missing Flows (5 issues)

**Issue 1 — Failure scenarios are absent.**
The 20-step walkthrough only covers the happy path. No description of what happens
when Julius/Sherlock/Leonard/Katherine fails, container OOM-kills, or GitHub API is
down. The `FAILED` pipeline state exists but nothing says how we get there or what
cleanup occurs. Key question: does a single task failure fail the whole pipeline or
just that task?

**Issue 2 — Pipeline cancellation flow is undefined.**
`CANCELLED` is a lifecycle state but there's no description of: how a human triggers
it, what happens to running containers, what happens to open PRs/branches, or whether
a cancelled pipeline can be resumed.

**Issue 3 — NEEDS_HUMAN recovery path is incomplete.**
Step 12 shows "escalated" → `NEEDS_HUMAN` but the story ends there. What happens
after the human reviews? What event resumes the pipeline? How long can it sit idle?

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
| 01 Workflow | **In review** | Read and analyzed. 15 issues identified (see above). Discussion not yet started. |
| 02 Data Models | Needs review | Verify all tables match decisions. Add any missing schemas. |
| 03 Learning & Principles | Needs review | Verify Extract→Replay→Verify loop is fully specified. |
| 04 Communication | Needs review | Check message types match orchestrator routing table. |
| 05 Infrastructure | Needs review | Docker Compose is canonical. Check for consistency. |
| 06 Observability | Needs review | Logging, Grafana/Loki, dashboards. |
| 07 Cost Tracking | Needs review | Budget enforcement, attribution. |
| 08 Error Recovery | Needs review | Checkpoints, retries, idempotency. |
| 09 Security | Needs review | Secret scoping, socket proxy, egress. |
| 10 Testing | Needs review | VCR, testcontainers. |
| 11 Dev Standards | Needs review | ruff, mypy, coverage. |
| 12 Repo Connection | Needs review | GitHub auth, .ai-team.yaml full schema. |
| 13 Orchestrator | Needs review | Event router, launcher, watchdog. Largest component spec. |
| 14 API Server | **Has TODOs** | Request/response schemas, error format, rate limiting. |
| 15 TUI | **Has TODOs** | Learning chat UX, keybindings, principle browser. Rich interactive app. |
| 16 Nelson | Mostly complete | Full consensus algorithm. Needs prompt/tool review. |
| 17 Julius | **Has TODOs** | System prompt, PydanticAI tools, decomposition examples. |
| 18 Richelieu | Mostly complete | Git operations well-defined. Needs prompt/tool review. |
| 19 Sherlock | **Has TODOs** | System prompt, PydanticAI tools, enrichment examples. |
| 20 Leonard | **Has TODOs** | System prompt, PydanticAI tools, implementation examples. |
| 21 Katherine | **Has TODOs** | System prompt, PydanticAI tools, review criteria detail. |

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

### 3. TUI Detail (spec 15)
- Textual widget hierarchy
- Keybindings
- Learning mode chat interface UX
- Multi-team switching UX

### 4. Rate Limiting (system-wide)
How to handle OpenRouter rate limits with parallel Nelson calls + multiple Leonards.

### 5. Caching Strategy
File read caching across Sherlock/Leonard. Prompt caching for Nelson.

---

## Suggested Next Session Plan

### Step 1: Discuss the 15 Workflow issues (one by one)
Go through each issue, make a decision, and update `01-workflow.md` accordingly.
Start with Category 1 (missing flows) since those are the biggest gaps.

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
