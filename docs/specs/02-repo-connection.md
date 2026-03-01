# 02 — Repo Connection

## Overview

The ai-team system connects to a target repository to work on it. The connection involves
cloning the repo, reading its `.ai-team.yaml` configuration, and setting up webhooks
for automated task intake from GitHub Issues.

---

## Connection Flow

```
User runs: make connect REPO=https://github.com/org/my-app

    │
    ▼
┌─────────────────────────────────────────┐
│ 1. Authenticate with GitHub             │
│    - Try GitHub App installation first  │
│    - Fall back to PAT if no App         │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│ 2. Clone repo into /workspace volume    │
│    - Shallow clone (--depth=1) for speed│
│    - Full fetch if history needed later │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│ 3. Read .ai-team.yaml                  │
│    - Validate against Pydantic schema   │
│    - Error if missing or invalid        │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│ 4. Register webhook (if GitHub App)     │
│    - Listen for: issues.opened,         │
│      issues.labeled, issue_comment,     │
│      pull_request.review_submitted      │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│ 5. Store connection in PostgreSQL        │
│    - Repo URL, auth method, branch,     │
│      config hash, connection timestamp  │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│ 6. Validate environment                 │
│    - Run commands.install from config   │
│    - Run commands.test to verify setup  │
│    - Report any failures               │
└─────────────────────────────────────────┘
```

## Authentication

### GitHub App (Primary)

A GitHub App is the preferred authentication method for organizations:

- **Installation**: The org admin installs the ai-team GitHub App on specific repos.
- **Permissions required**:
  - `contents: write` — clone, push branches.
  - `pull_requests: write` — create/update PRs.
  - `issues: read` — read issues for task intake.
  - `checks: write` — report status checks.
  - `webhooks: write` — register event listeners.
- **Token refresh**: GitHub App tokens expire after 1 hour. The system refreshes
  automatically using the App's private key.
- **Per-repo scoping**: Tokens are scoped to the specific repo, not the whole org.

### Personal Access Token (Fallback)

For personal repos or quick setup:

- **Token type**: Fine-grained PAT (not classic) with minimal permissions.
- **Permissions**: Same as GitHub App but tied to a user account.
- **Limitation**: No webhook support — must poll for new issues or use manual trigger.
- **Storage**: In `.env` file, never committed.

### Auth Selection Logic

```python
def get_auth(repo_url: str) -> GitHubAuth:
    if github_app_configured():
        installation = find_installation(repo_url)
        if installation:
            return GitHubAppAuth(installation)
    if pat_configured():
        return PATAuth(os.environ["GITHUB_PAT"])
    raise AuthError("No GitHub credentials configured")
```

## `.ai-team.yaml` — Full Specification

```yaml
version: "1"

project:
  name: "my-app"                         # Required: project identifier
  description: "E-commerce API"          # Required: what this project does
  language: "python"                     # Required: primary language
  framework: "fastapi"                   # Optional: primary framework
  python_version: "3.12"                 # Optional: language version

# Codebase knowledge — paths relative to repo root
knowledge:
  architecture_docs: "docs/architecture.md"     # Optional
  api_docs: "docs/api/"                         # Optional: directory or file
  style_guide: "docs/STYLE_GUIDE.md"           # Optional
  adr_directory: "docs/adr/"                    # Optional: Architecture Decision Records
  additional:                                    # Optional: extra context files
    - "CONTRIBUTING.md"
    - "docs/patterns.md"
    - "docs/database-schema.md"

# Commands the agents execute
commands:
  install: "uv sync"                             # Required: install dependencies
  test: "uv run pytest"                          # Required: run tests
  lint: "uv run ruff check ."                    # Optional: lint check
  format: "uv run ruff format ."                 # Optional: auto-format
  type_check: "uv run mypy src/"                 # Optional: type checking
  build: "uv run python -m build"                # Optional: build step
  custom:                                         # Optional: extra commands
    migrate: "uv run alembic upgrade head"
    seed: "uv run python scripts/seed.py"

# Rules the agents must follow
guidelines:
  - "All functions must have type annotations"
  - "Test coverage must not decrease"
  - "No direct database queries outside the repository layer"
  - "Use structured logging, never print()"
  - "Follow conventional commits for commit messages"
  - "No new dependencies without documenting the reason"

# Directories/files the agents must NEVER modify
protected_paths:
  - "migrations/"                 # Only modify via migration commands
  - "infrastructure/"
  - ".github/"
  - "scripts/deploy.sh"

# Git configuration
git:
  default_branch: "main"
  branch_prefix: "ai-team/"              # All AI branches: ai-team/feature-name
  require_pr: true                        # Always create PRs, never push to default
  auto_merge: false                       # Even if approved, wait for human merge
  conventional_commits: true              # Enforce conventional commit format

# Human review configuration
review:
  human_review_threshold: 0.7             # Nelson-calculated score above this → human
  always_human_review:                    # Paths that always need human eyes
    - "migrations/"
    - "infrastructure/"
    - "*.sql"
    - "docker-compose*.yml"
  never_auto_approve:                     # Even if score is low, flag these
    - "security-related changes"
    - "dependency updates"

# Task intake — how issues become tasks
intake:
  labels:
    trigger: "ai-team"                    # Issues with this label get picked up
    in_progress: "ai-team:working"        # Applied when agents start working
    done: "ai-team:done"                  # Applied when PR is created
    needs_human: "ai-team:needs-human"    # Applied when agents need help
  issue_template: |                       # Optional: expected format for issues
    ## Description
    [What should be built or fixed]

    ## Acceptance Criteria
    - [ ] Criterion 1
    - [ ] Criterion 2

# Budget overrides (defaults from system config)
budget:
  soft_limit_per_task: 5.00               # USD
  hard_limit_per_task: 20.00              # USD

# LLM configuration
llm:
  default_model: anthropic/claude-sonnet-4       # Model for direct agent calls
  consensus:
    models:                                       # Models used by Nelson
      - anthropic/claude-sonnet-4
      - openai/gpt-4o
      - google/gemini-2.0-flash
    max_rounds: 3                                 # Max cross-review iterations
  overrides:                                      # Optional: per-agent model override
    leonard: anthropic/claude-sonnet-4
    sherlock: google/gemini-2.0-flash
```

## Repo Sync

The local clone needs to stay in sync with the remote:

- **Before starting any new work**: `git fetch origin && git pull --rebase`.
- **Periodic sync**: Every 5 minutes, Richelieu fetches the latest from remote.
- **On webhook event**: Immediate fetch when a relevant event (push, PR merge) is received.
- **Conflict detection**: If upstream changes conflict with in-progress work, Richelieu
  attempts rebase. If it fails, escalate to human.

## Multi-Repo Support (Future)

The system is designed for single-repo connection initially. Multi-repo support can be
added by:
- Allowing multiple `make connect` calls.
- Each connection gets its own workspace volume.
- Tasks are scoped to a specific repo.
- Cross-repo features would need an orchestrator above Julius.

## Disconnection

```bash
make disconnect REPO=https://github.com/org/my-app
```

- Removes the workspace volume.
- Deregisters webhooks.
- Cleans up PostgreSQL connection records.
- Stops any in-progress work on that repo (with confirmation).
