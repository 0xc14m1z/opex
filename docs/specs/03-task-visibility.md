# 03 — Task Visibility

## Overview

A terminal UI (TUI) built with [Textual](https://textual.textualize.io/) provides
real-time visibility into what the agents are doing. The TUI runs on the host machine
and connects to Redis to stream agent activity.

---

## TUI Screens

### 1. Pipeline Overview (Default Screen)

The main view showing the current state of all active work.

```
┌─ AI-Team Dashboard ──────────────────────────────────── 14:32:05 ─┐
│                                                                    │
│  PIPELINE: feature/add-user-auth ───────────────────────────────── │
│                                                                    │
│  Julius ✓  →  Sherlock ●  →  Leonard ◌ ◌ ◌  →  Katherine ◌       │
│  5 tasks      task-2       pending (3)        waiting              │
│  decomposed   enriching                                            │
│                                                                    │
│  ┌─ Tasks ──────────────────────────────────────────────────────┐  │
│  │  #1  ✓ Add User model + migration       Leonard-1 ✓ merged  │  │
│  │  #2  ● Create auth endpoints             Sherlock  enriching │  │
│  │  #3  ◌ Add JWT middleware                 pending  (needs #2) │  │
│  │  #4  ◌ Write auth integration tests       pending  (needs #3) │  │
│  │  #5  ◌ Update API docs                    pending  (needs #1) │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌─ Active Agents ──────────────────────────────────────────────┐  │
│  │  Nelson     idle        last: 2m ago     calls: 12           │  │
│  │  Julius     idle        last: 8m ago     tasks: 5            │  │
│  │  Sherlock   working     task: #2         elapsed: 1m32s      │  │
│  │  Leonard-1  idle        last: 4m ago     completed: 1        │  │
│  │  Katherine  idle        last: 5m ago     reviews: 1          │  │
│  │  Richelieu  idle        last: 30s ago    branches: 2         │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  Cost: $2.34 (soft limit: $25.00)  │  Tokens: 142,830            │
│                                                                    │
│  [T]asks  [L]ogs  [C]onsensus  [K]ost  [H]elp  [Q]uit           │
└────────────────────────────────────────────────────────────────────┘
```

### 2. Task Detail View

Drill into a specific task to see its full lifecycle.

```
┌─ Task #2: Create auth endpoints ─────────────────────────────────┐
│                                                                    │
│  Status: enriching (Sherlock)                                      │
│  Dependencies: none                                                │
│  Blocks: #3, #4                                                    │
│  Branch: ai-team/add-user-auth/task-2                             │
│                                                                    │
│  ┌─ Timeline ───────────────────────────────────────────────────┐  │
│  │  14:24:05  Julius    created task                            │  │
│  │  14:24:05  Julius    no dependencies, ready immediately      │  │
│  │  14:28:12  Sherlock  started enrichment                      │  │
│  │  14:28:15  Sherlock  reading src/api/routes/ (4 files)       │  │
│  │  14:29:30  Sherlock  reading src/models/user.py              │  │
│  │  14:30:42  Nelson    consensus: approach for token storage   │  │
│  │            ├─ Claude:  use httpOnly cookies ✓ (agreed)       │  │
│  │            ├─ GPT-4o:  use httpOnly cookies ✓ (agreed)       │  │
│  │            └─ Gemini:  use localStorage ✗ (overruled)        │  │
│  │  14:31:05  Sherlock  execution plan ready (12 steps)         │  │
│  │  ...                                                         │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  Cost so far: $0.48  │  Tokens: 31,200                            │
│                                                                    │
│  [B]ack  [E]xecution plan  [C]onsensus history  [L]ogs            │
└────────────────────────────────────────────────────────────────────┘
```

### 3. Log Stream View

Live tail of structured logs, filterable by agent, level, and task.

```
┌─ Logs ─────────── Filter: [agent:all] [level:INFO+] [task:all] ──┐
│                                                                    │
│  14:31:05 INFO  sherlock  task=2  Execution plan generated         │
│  14:31:06 INFO  richelieu task=2  Worktree created at .worktrees/2 │
│  14:31:06 INFO  leonard-1 task=2  Starting implementation          │
│  14:31:08 DEBUG leonard-1 task=2  Reading src/api/routes/auth.py   │
│  14:31:12 DEBUG leonard-1 task=2  LLM call: claude-sonnet tokens=… │
│  14:31:15 INFO  leonard-1 task=2  Created src/api/routes/auth.py   │
│  14:31:18 DEBUG leonard-1 task=2  LLM call: claude-sonnet tokens=… │
│  14:31:22 INFO  leonard-1 task=2  Created tests/test_auth.py       │
│  14:31:25 INFO  leonard-1 task=2  Running: uv run pytest           │
│  14:31:30 INFO  leonard-1 task=2  Tests passed (12/12)             │
│  14:31:31 INFO  leonard-1 task=2  Running: uv run ruff check .     │
│  14:31:32 INFO  leonard-1 task=2  Lint passed                      │
│  14:31:33 INFO  leonard-1 task=2  Implementation complete          │
│  14:31:33 INFO  katherine task=2  Starting review                  │
│  ...                                                               │
│                                                                    │
│  [F]ilter  [P]ause  [B]ack                                        │
└────────────────────────────────────────────────────────────────────┘
```

### 4. Consensus History View

See how LLMs debated and reached (or failed to reach) consensus.

```
┌─ Consensus History ──────────────────────────────────────────────┐
│                                                                    │
│  ┌─ #7 Code review: task-2 implementation ── AGREED (round 1) ─┐  │
│  │  Requester: Katherine    Confidence: 0.92    Cost: $0.18     │  │
│  │                                                               │  │
│  │  Claude:  APPROVE  "Clean implementation, follows patterns"   │  │
│  │  GPT-4o:  APPROVE  "Good test coverage, minor style nit"     │  │
│  │  Gemini:  APPROVE  "Correct approach, well-structured"       │  │
│  │                                                               │  │
│  │  Result: APPROVED (3/3 agree)                                 │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌─ #6 Approach: token storage ── AGREED (round 2) ────────────┐  │
│  │  Requester: Sherlock     Confidence: 0.78    Cost: $0.32     │  │
│  │                                                               │  │
│  │  Round 1: Claude=cookies, GPT-4o=cookies, Gemini=localStorage │  │
│  │  Round 2: Gemini revised → cookies (after reviewing security) │  │
│  │                                                               │  │
│  │  Result: httpOnly cookies (3/3 after round 2)                 │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  [B]ack  [D]etail  [F]ilter by agent/outcome                     │
└────────────────────────────────────────────────────────────────────┘
```

### 5. Cost Dashboard

Token usage and cost breakdown.

```
┌─ Cost Dashboard ─────────────────────────────────────────────────┐
│                                                                    │
│  Today: $8.42 / $100.00 daily limit                               │
│  ████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  8.4%                   │
│                                                                    │
│  ┌─ By Agent ───────────────────────────────────────────────────┐  │
│  │  Nelson      $4.20  (consensus calls are expensive)          │  │
│  │  Leonard-1   $1.80                                           │  │
│  │  Sherlock    $1.22                                           │  │
│  │  Katherine   $0.85                                           │  │
│  │  Julius      $0.35                                           │  │
│  │  Richelieu   $0.00  (no LLM calls)                          │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌─ By Provider ────────────────────────────────────────────────┐  │
│  │  Claude      $3.40   tokens: 89,200   avg latency: 2.1s     │  │
│  │  GPT-4o     $2.80   tokens: 112,400  avg latency: 1.8s     │  │
│  │  Gemini      $2.22   tokens: 98,600   avg latency: 1.5s     │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌─ By Task ────────────────────────────────────────────────────┐  │
│  │  #1  Add User model          $2.10  ✓ completed              │  │
│  │  #2  Create auth endpoints   $3.80  ● in progress            │  │
│  │  #3  Add JWT middleware      $0.00  ◌ pending                │  │
│  │  #4  Write auth tests        $0.00  ◌ pending                │  │
│  │  #5  Update API docs         $2.52  ✓ completed              │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  [B]ack  [H]ourly chart  [W]eekly summary                        │
└────────────────────────────────────────────────────────────────────┘
```

## Human Escalation Points

When agents need human input, the TUI shows a notification banner:

```
┌─ ⚠ ATTENTION NEEDED ──────────────────────────────────────────────┐
│                                                                    │
│  1. Katherine flagged PR #42 for human review (score: 0.82)       │
│  2. Nelson couldn't reach consensus on database schema approach    │
│  3. Leonard failed 3 times on task #7 — needs human guidance     │
│                                                                    │
│  Press [1] [2] [3] to view details, [D]ismiss                    │
└────────────────────────────────────────────────────────────────────┘
```

## GitHub-Side Visibility

In addition to the TUI, the system provides visibility through GitHub:

- **Issue labels**: Applied by agents to show task status (see 02-repo-connection.md).
- **PR descriptions**: Include a summary of what agents did, which tasks are covered,
  consensus decisions made, and the human review score.
- **PR comments**: Katherine posts review findings as PR comments.
- **Status checks**: Leonard reports test/lint results as GitHub status checks.

## Data Flow for TUI

The TUI subscribes to Redis Streams to get real-time updates:

```
Agent → (structured log) → Redis Stream "logs" → TUI renders
Agent → (status update) → Redis Stream "status" → TUI renders
Agent → (cost event) → Redis Stream "costs" → TUI renders
```

The TUI is read-only — it never sends commands to agents through this channel.
Human input (approve, reject, provide guidance) goes through a separate
"human-input" Redis Stream that agents listen to.

## Grafana / Loki (Historical Log Querying)

In addition to the TUI's real-time views, **Grafana + Loki** provide historical
log querying and dashboards. All container logs are shipped to Loki automatically
via Docker's Loki logging driver (see spec 06).

Grafana complements the TUI:
- **TUI**: Real-time streaming view, quick task status, human escalation prompts.
- **Grafana**: Historical querying (LogQL), cross-agent correlation, dashboards,
  alerting, and long-term trend analysis.

Grafana is accessible at `http://localhost:3000` when the stack is running.
