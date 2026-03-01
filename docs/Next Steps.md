# Next Steps

> **Purpose**: This file tracks where we left off and what to do next. Update it
> at the end of every session so the next session can pick up seamlessly.
>
> **Last updated**: 2026-03-01

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

### ~~Pending numbering fix~~ — DONE

Specs 01 and 02 swapped. Workflow is now `01-workflow.md`, Data Models is now
`02-data-models.md`. All cross-references updated across all 22 specs.

---

## Spec Review Progress

Each spec needs a deep-dive review (like we did for deployment → spec 05) to
identify gaps, clarify details, and resolve open questions.

| Spec | Status | Notes |
|------|--------|-------|
| 00 Overview | **Reviewed** | Done. Mermaid diagrams, final spec index. |
| 01 Workflow | **Next up** | Review the pipeline flow, branching model, PR flow. Verify consistency with controller (13) and Richelieu (18). |
| 02 Data Models | Needs review | Verify all tables match decisions. Add any missing schemas from workflow/learning discussions. |
| 03 Learning & Principles | Needs review | Rich content from our discussion. Verify Extract→Replay→Verify loop is fully specified. |
| 04 Communication | Needs review | Migrated from old spec 10. Check message types match controller routing table. |
| 05 Infrastructure | Needs review | Migrated from old spec 01. Docker Compose is canonical. Check for consistency. |
| 06 Observability | Needs review | Logging, Grafana/Loki, dashboards. |
| 07 Cost Tracking | Needs review | Budget enforcement, attribution. |
| 08 Error Recovery | Needs review | Checkpoints, retries, idempotency. |
| 09 Security | Needs review | Secret scoping, socket proxy, egress. |
| 10 Testing | Needs review | VCR, testcontainers. |
| 11 Dev Standards | Needs review | ruff, mypy, coverage. |
| 12 Repo Connection | Needs review | GitHub auth, .ai-team.yaml full schema. |
| 13 Controller | Needs review | Event router, launcher, watchdog. Largest component spec. |
| 14 API Server | **Has TODOs** | Request/response schemas, error format, rate limiting. |
| 15 TUI | **Has TODOs** | Learning chat UX, keybindings, principle browser. |
| 16 Nelson | Mostly complete | Full consensus algorithm migrated from old spec 04. Needs prompt/tool review. |
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

### Step 1: Swap specs 01/02
Rename workflow to 01, data models to 02. Update cross-references.

### Step 2: Deep-dive the Workflow spec (new 01)
Review the pipeline lifecycle, branching model, PR flow, 20-step event walkthrough.
Questions to resolve:
- Is the dependency dispatch logic clear enough to implement?
- Are all edge cases covered (partial failures, human intervention mid-pipeline)?
- Does the conflict resolution flow handle all scenarios?
- How does learning mode interact with the workflow (replay = re-enter the loop)?

### Step 3: Deep-dive Data Models (new 02)
Verify every PostgreSQL table and Pydantic model matches the decisions from all specs.
Ensure the learning/principles tables are complete.

### Step 4: Deep-dive remaining specs
Continue the one-by-one review through specs 03-12 (foundation + cross-cutting),
then 13-21 (components). Each review follows the same pattern:
- Read the spec
- Identify gaps and inconsistencies
- Ask clarifying questions one by one
- Update the spec with decisions

### Step 5: Fill agent prompt/tool TODOs
The big one. Define system prompts and PydanticAI tools for each agent (16-21).
This is creative design work — needs careful discussion per agent.

### Step 6: Build plan
Once all specs are reviewed and complete, create a detailed Phase 0 implementation
plan: what to build first, what depends on what, what can be parallelized.

---

## Session Log

| Date | What was done |
|------|---------------|
| 2026-03-01 | Deep-dive spec 01 (deployment). 13 improvements + 5 new topics (learning mode, principles, API server, TUI, branching). Full rewrite. Reorganized all specs from 13→22 files. Renumbered: foundation (00-05), cross-cutting (06-12), components (13-21). Converted diagrams to Mermaid. |
