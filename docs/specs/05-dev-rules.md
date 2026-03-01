# 05 — Development Rules

## Overview

Standards and practices for developing the ai-team system itself. This is our own
codebase's rulebook — separate from the rules agents enforce on target repos.

---

## Language & Runtime

- **Python**: 3.12+
- **Package manager**: uv
- **Workspace layout**: uv workspaces (monorepo)

## Code Quality

### Linting & Formatting

- **Ruff**: Linter and formatter. Single tool for both.
- **Configuration**: `pyproject.toml` at the root.

```toml
[tool.ruff]
target-version = "py312"
line-length = 99

[tool.ruff.lint]
select = [
    "E",    # pycodestyle errors
    "W",    # pycodestyle warnings
    "F",    # pyflakes
    "I",    # isort
    "N",    # pep8-naming
    "UP",   # pyupgrade
    "B",    # flake8-bugbear
    "A",    # flake8-builtins
    "SIM",  # flake8-simplify
    "TCH",  # flake8-type-checking
    "RUF",  # ruff-specific rules
]

[tool.ruff.lint.isort]
known-first-party = ["ai_team_core", "nelson", "julius", "sherlock", "leonard", "katherine", "richelieu"]
```

### Type Checking

- **mypy**: Strict mode.
- Every function must have full type annotations (parameters + return type).
- No `Any` types without a justifying comment.

```toml
[tool.mypy]
python_version = "3.12"
strict = true
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true
disallow_incomplete_defs = true
```

### Test Coverage

- **Target**: 90%+ line coverage.
- **Tool**: pytest + pytest-cov.
- **Enforcement**: CI fails if coverage drops below threshold.
- **Exclusions**: Only `# pragma: no cover` for genuinely untestable code
  (e.g., Docker SDK calls, main entry points).

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "--cov=core --cov=agents --cov-report=term-missing --cov-fail-under=90"
```

### Test Infrastructure

- **No mocks**: Tests use real services, never fakes or mocks.
- **testcontainers**: Real Redis (`testcontainers-redis`) and real PostgreSQL
  (`testcontainers-postgres`) instances per test session.
- **VCR cassettes**: LLM API calls are recorded on first run and replayed
  deterministically on subsequent runs via `vcrpy` / `pytest-recording`.
- **CI**: Runs with `--vcr-record=none` (replay only, no real API calls).

## Coding Conventions

### Project-Wide

- **Pydantic for all data models**. No raw dicts flowing between components.
- **Structured logging everywhere** (structlog). Never `print()`.
- **Async by default**. Agents are async (asyncio). Sync wrappers only at boundaries.
- **Dependency injection** for LLM clients, Redis connections, and state stores.
  Makes testing straightforward — inject mocks in tests.
- **No global mutable state**. Configuration is loaded once and passed through.

### Naming

- **Files**: `snake_case.py`
- **Classes**: `PascalCase`
- **Functions/methods**: `snake_case`
- **Constants**: `UPPER_SNAKE_CASE`
- **Pydantic models**: Named descriptively, suffixed by role:
  - `ConsensusRequest` (not `Request`)
  - `TaskDecomposition` (not `Result`)
- **Agent packages**: Named after the agent: `nelson/`, `julius/`, etc.

### Imports

- Standard library → third-party → first-party (`ai_team_core`) → local.
- Enforced by ruff's isort integration.
- Prefer explicit imports over star imports.

### Error Handling

- **Custom exception hierarchy** rooted in `AiTeamError`.
- Each agent defines its own exception subtypes.
- Always include context in exceptions (task ID, agent name, what was being attempted).
- Never catch bare `Exception` unless re-raising or logging + escalating.

```python
class AiTeamError(Exception):
    """Base exception for all ai-team errors."""

class ConsensusError(AiTeamError):
    """Nelson-specific errors."""

class ConsensusTimeoutError(ConsensusError):
    """All providers failed to respond within the timeout."""

class DecompositionError(AiTeamError):
    """Julius-specific errors."""
```

### Commits

- **Conventional commits**: `type(scope): description`
- Types: `feat`, `fix`, `refactor`, `test`, `docs`, `chore`, `ci`
- Scopes: `core`, `nelson`, `julius`, `sherlock`, `leonard`, `katherine`,
  `richelieu`, `dashboard`, `infra`
- Examples:
  - `feat(nelson): add weighted voting to consensus loop`
  - `fix(leonard): retry test execution on transient failures`
  - `test(core): add unit tests for Redis queue abstraction`

## Branch Strategy

- **Main branch**: `main` — always passing CI.
- **Feature branches**: `feat/<description>` or `<name>/<description>`.
- **PR required**: No direct pushes to `main`.
- **Squash merge**: PRs are squash-merged to keep history clean.
- **Delete branches after merge**.

## CI Pipeline (GitHub Actions)

```yaml
# Runs on every PR and push to main
jobs:
  lint:
    - ruff check .
    - ruff format --check .

  typecheck:
    - mypy .

  test-unit:
    # testcontainers auto-starts Redis + PostgreSQL
    - pytest tests/unit/ --cov --cov-fail-under=90 --vcr-record=none

  test-integration:
    # Only on main or when manually triggered
    - docker compose up -d redis postgres
    - pytest tests/integration/ --vcr-record=none
    - docker compose down

  build:
    # Build all Docker images to verify Dockerfiles work
    - docker compose build
```

## Dependency Management

- **Minimal dependencies**: Every new dependency must be justified.
- **Pin versions**: In `uv.lock` (auto-managed by uv).
- **Security scanning**: `uv audit` in CI to check for known vulnerabilities.
- **Shared deps in `core/`**: Common dependencies (pydantic, structlog, litellm, redis)
  live in the core package. Agent packages depend on core.

## Documentation

- **Code docstrings**: Required for public functions and classes.
  Google-style docstrings.
- **Spec files**: This `docs/specs/` directory. Updated when designs change.
- **PLAN.md**: High-level overview, updated as phases complete.
- **No auto-generated docs** unless they serve a real purpose.
- **ADRs (Architecture Decision Records)**: For significant design decisions
  that have alternatives worth documenting.

## Review Process (For Our Own Code)

- All PRs require at least one review.
- CI must pass before merge.
- Dog-fooding (Phase 8+): ai-team reviews its own PRs through the standard
  agent pipeline. Human still has final say.
