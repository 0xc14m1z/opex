# 10 — Testing

> **Migrated from**: `docs/specs/11-testing.md`

## Overview

Testing agentic systems is hard because the core behavior depends on non-deterministic
LLM responses. Our strategy: no mocks. Real infrastructure via testcontainers, real LLM
responses via VCR cassettes (recorded once, replayed deterministically), and full
`docker compose` stacks for integration tests. Every test talks to real services.

---

## Test Pyramid

```mermaid
%%{init: {"theme": "base"}}%%
block-beta
    columns 1
    block:e2e["E2E Tests\nAll agents as containers, real repo, VCR cassettes\n(CI: nightly or manual)"]
    end
    block:integration["Integration Tests\nFull docker compose up -- Redis, PostgreSQL, Loki, Grafana"]
    end
    block:unit["Unit Tests\ntestcontainers (Redis + PostgreSQL), VCR cassettes for LLM"]
    end

    style e2e fill:#f9d,stroke:#333
    style integration fill:#fd9,stroke:#333
    style unit fill:#9df,stroke:#333
```

| Layer        | LLM Calls      | Redis              | PostgreSQL          | Docker   | Speed    | CI Stage           |
|-------------|----------------|--------------------|---------------------|----------|----------|--------------------|
| Unit        | VCR cassettes  | testcontainers     | testcontainers      | No       | < 10s    | Every PR           |
| Integration | VCR cassettes  | docker compose     | docker compose      | Yes      | < 60s    | Every PR           |
| E2E         | VCR cassettes  | docker compose     | docker compose      | Yes      | Minutes  | Nightly/manual     |

No mocks at any layer. Every test exercises real service implementations.

## Record/Replay (VCR Cassettes)

### How It Works

1. **Record mode** (`--vcr-record=all`): Run tests against real LLM APIs. All HTTP
   requests and responses are captured and saved as cassette files.
2. **Replay mode** (`--vcr-record=none`, default): Tests use recorded cassettes. No API
   calls made. Deterministic and fast.
3. **Update mode** (`--vcr-record=new_episodes`): Re-record only new interactions not
   already in the cassette. Existing recordings are preserved.

This is the same mechanism described elsewhere as "record/replay with snapshot update."
VCR IS the record/replay mechanism. The cassettes contain real responses from real
models — they are not hand-crafted fakes.

### Implementation

Use [pytest-recording](https://github.com/kiwicom/pytest-recording) (built on VCR.py),
configured to intercept LiteLLM's HTTP calls to provider APIs.

```python
# pyproject.toml
[tool.pytest.ini_options]
markers = [
    "vcr: marks tests that use VCR cassettes",
]

[tool.pytest-recording]
vcr_cassette_dir = "tests/cassettes"
```

### Cassette Format

Each recorded interaction is stored as a YAML file by pytest-recording:

```yaml
# tests/cassettes/test_nelson/test_consensus_majority.yaml
interactions:
  - request:
      uri: "https://api.anthropic.com/v1/messages"
      method: POST
      headers:
        x-api-key: "REDACTED"
        content-type: "application/json"
      body:
        model: "claude-3-5-sonnet-20241022"
        messages:
          - role: "user"
            content: "Review the following code..."
        temperature: 0.0
    response:
      status:
        code: 200
      headers:
        content-type: "application/json"
      body:
        content: '{"decision": "approve", "reasoning": "..."}'
        usage:
          input_tokens: 1200
          output_tokens: 450

  - request:
      uri: "https://api.openai.com/v1/chat/completions"
      # ...
    response:
      # ...

  - request:
      uri: "https://generativelanguage.googleapis.com/v1beta/models/gemini-pro:generateContent"
      # ...
    response:
      # ...
```

### Recording and Updating Cassettes

```bash
# Record all cassettes (hits real APIs, costs real money)
uv run pytest tests/ --vcr-record=all

# Record cassettes for a specific test
uv run pytest tests/unit/test_nelson.py::test_consensus_majority --vcr-record=all

# Add new interactions without re-recording existing ones
uv run pytest tests/ --vcr-record=new_episodes

# Replay mode (default, what CI uses)
uv run pytest tests/ --vcr-record=none
```

### When to Update Recordings

- **Prompt changes**: If you modify an agent's prompt template, re-record affected tests.
- **Model upgrades**: When switching to a newer model version, re-record all.
- **Behavior changes**: If the expected output changes, update and review the diff.
- **Periodic freshness**: Monthly re-record to catch model drift (automated via CI).

Cassettes are committed to the repo under `tests/cassettes/`. Reviewing cassette diffs
in PRs is part of the normal review process.

## Infrastructure Fixtures (conftest.py)

All infrastructure is real. No fakes, no in-memory substitutes.

```python
# tests/conftest.py

import pytest
from testcontainers.redis import RedisContainer
from testcontainers.postgres import PostgresContainer

@pytest.fixture(scope="session")
def redis_container():
    """Real Redis instance for the entire test session."""
    with RedisContainer("redis:7-alpine") as redis:
        yield redis

@pytest.fixture(scope="session")
def postgres_container():
    """Real PostgreSQL instance for the entire test session."""
    with PostgresContainer("postgres:16-alpine") as pg:
        yield pg

@pytest.fixture
def redis_url(redis_container):
    """Connection URL for the session-scoped Redis container."""
    return redis_container.get_connection_url()

@pytest.fixture
def database_url(postgres_container):
    """Connection URL for the session-scoped PostgreSQL container."""
    return postgres_container.get_connection_url()

@pytest.fixture
async def redis_client(redis_url):
    """Async Redis client connected to the testcontainer, flushed between tests."""
    import redis.asyncio as aioredis
    client = aioredis.from_url(redis_url)
    await client.flushdb()
    yield client
    await client.flushdb()
    await client.aclose()

@pytest.fixture
def state_store(database_url):
    """StateStore backed by real PostgreSQL, with schema applied."""
    store = StateStore(database_url)
    store.run_migrations()
    yield store

@pytest.fixture
def test_repo(tmp_path):
    """Creates a temporary git repo with a .ai-team.yaml."""
    repo_path = tmp_path / "test-repo"
    repo_path.mkdir()
    subprocess.run(["git", "init"], cwd=repo_path, check=True)
    (repo_path / ".ai-team.yaml").write_text(TEST_CONFIG_YAML)
    (repo_path / "src").mkdir()
    (repo_path / "src" / "main.py").write_text("def hello(): return 'world'")
    subprocess.run(["git", "add", "."], cwd=repo_path, check=True)
    subprocess.run(["git", "commit", "-m", "initial"], cwd=repo_path, check=True)
    yield repo_path
```

The `scope="session"` on the containers means one Redis and one PostgreSQL instance per
test run. Each test gets a clean state via `flushdb()` and migrations. This keeps startup
cost amortized while maintaining isolation.

## Unit Tests

### What to Unit Test

| Component | Test Focus |
|---|---|
| Consensus algorithm | Majority detection, weighted voting, termination logic |
| Weight learning | Weight updates, normalization, floor enforcement |
| Cost calculation | Token-to-cost math, budget limit logic |
| Dependency graph | Task ordering, parallel group detection, cycle detection |
| Message serialization | Pydantic model -> Redis -> Pydantic round-trip (real Redis) |
| Checkpoint logic | Save, load, resume-from-checkpoint (real PostgreSQL) |
| Config parsing | `.ai-team.yaml` validation, defaults, error messages |
| Retry logic | Counter, max retries, escalation decision |

Even "unit" tests use real Redis and real PostgreSQL via testcontainers. There is no
mock boundary — the only thing that isn't live is the LLM, which replays from VCR
cassettes.

### Unit Test Style

```python
# tests/unit/test_consensus_algorithm.py

@pytest.mark.vcr  # Replays from tests/cassettes/test_consensus_algorithm/...
class TestMajorityDetection:
    def test_unanimous_agreement(self):
        responses = [
            ProviderResponse(provider="claude", decision="approve"),
            ProviderResponse(provider="gpt4o", decision="approve"),
            ProviderResponse(provider="gemini", decision="approve"),
        ]
        result = detect_majority(responses)
        assert result.status == "agreed"
        assert result.confidence > 0.95

    def test_two_of_three_majority(self):
        responses = [
            ProviderResponse(provider="claude", decision="approve"),
            ProviderResponse(provider="gpt4o", decision="approve"),
            ProviderResponse(provider="gemini", decision="reject"),
        ]
        result = detect_majority(responses)
        assert result.status == "majority"
        assert result.decision == "approve"

    def test_no_majority_three_different(self):
        responses = [
            ProviderResponse(provider="claude", decision="approve"),
            ProviderResponse(provider="gpt4o", decision="reject"),
            ProviderResponse(provider="gemini", decision="revise"),
        ]
        result = detect_majority(responses)
        assert result is None  # No majority


# tests/unit/test_message_serialization.py

class TestMessageRoundTrip:
    async def test_pydantic_to_redis_and_back(self, redis_client):
        """Real Redis via testcontainers. No fakeredis."""
        msg = TaskMessage(task_id="t-1", agent="julius", payload={"foo": "bar"})
        await redis_client.xadd("stream:tasks", msg.to_redis_dict())
        entries = await redis_client.xrange("stream:tasks")
        restored = TaskMessage.from_redis_dict(entries[0][1])
        assert restored == msg


# tests/unit/test_checkpoint.py

class TestCheckpoint:
    def test_save_and_resume(self, state_store):
        """Real PostgreSQL via testcontainers. No SQLite-in-memory."""
        state_store.save_checkpoint("task-1", "decomposed", {"subtasks": 3})
        checkpoint = state_store.get_latest_checkpoint("task-1")
        assert checkpoint.name == "decomposed"
        assert checkpoint.data == {"subtasks": 3}
```

## Integration Tests

### What to Integration Test

| Scenario | Components Involved |
|---|---|
| Task flows through pipeline | Redis streams, message serialization, consumer groups |
| Nelson consensus with recorded LLM | Nelson + Redis + VCR cassettes |
| Checkpoint save/resume | Agent + PostgreSQL + Redis |
| Cost tracking accumulation | LLM calls (VCR) + PostgreSQL + budget checking |
| Git operations | Richelieu + test git repo |
| Config loading | `.ai-team.yaml` parsing from a test repo |
| Orchestrator launches agents | Orchestrator + Docker SDK (Docker-in-Docker) |

### Integration Test Infrastructure

Integration tests run against a full `docker compose up` of the infrastructure stack.
This goes beyond what testcontainers provides — it exercises the real compose networking,
service discovery, and configuration.

```python
# tests/integration/conftest.py

import subprocess
import pytest

@pytest.fixture(scope="session", autouse=True)
def infra_stack():
    """Bring up the full infrastructure stack for integration tests."""
    subprocess.run(
        ["docker", "compose", "-f", "docker-compose.test.yml", "up", "-d",
         "redis", "postgres", "loki", "grafana"],
        check=True,
    )
    # Wait for services to be healthy
    subprocess.run(
        ["docker", "compose", "-f", "docker-compose.test.yml", "up",
         "--wait", "redis", "postgres"],
        check=True,
    )
    yield
    subprocess.run(
        ["docker", "compose", "-f", "docker-compose.test.yml", "down", "-v"],
        check=True,
    )
```

For orchestrator integration tests (where the orchestrator uses the Docker SDK to launch
agent containers), Docker-in-Docker (DinD) is used so the test environment has access
to a Docker daemon.

## E2E Tests

### What to E2E Test

Full pipeline runs against a test repository. LLM calls are replayed from VCR cassettes
(pre-recorded from real interactions) so E2E tests are deterministic, fast, and free
to run repeatedly.

1. **Happy path**: Issue -> decomposition -> enrichment -> implementation -> review -> PR.
2. **Conflict resolution**: Two Leonards modify overlapping files.
3. **Review rework loop**: Katherine requests changes, Leonard fixes them.
4. **Budget halt**: Task exceeds hard limit mid-implementation.
5. **Agent crash recovery**: Kill a Leonard mid-task, verify checkpoint resume.
6. **Human escalation**: Nelson can't reach consensus, verify TUI notification.

### E2E Test Repo

A dedicated test repository with:
- Simple Python project (FastAPI app).
- Known `.ai-team.yaml` configuration.
- Pre-written issues that exercise different agent capabilities.
- Expected outcomes documented for each test case.

```
tests/
  e2e/
    test_repo/                  # The test target repo
      .ai-team.yaml
      src/
      tests/
    scenarios/
      test_happy_path.py
      test_conflict_resolution.py
      test_budget_enforcement.py
      test_crash_recovery.py
    conftest.py                 # E2E fixtures (docker compose up, etc.)
  cassettes/
    e2e/                        # VCR cassettes for E2E scenarios
      test_happy_path/
      test_conflict_resolution/
      ...
```

### Running E2E Tests

```bash
# Start the stack
make up

# Run E2E tests with VCR replay (default — no real API calls)
uv run pytest tests/e2e/ --e2e --timeout=300

# Run a specific scenario
uv run pytest tests/e2e/test_happy_path.py --e2e

# Re-record E2E cassettes (hits real APIs, costs real money)
uv run pytest tests/e2e/ --e2e --vcr-record=all --timeout=600
```

## Test Configuration in CI

```yaml
# .github/workflows/ci.yml
jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v4
      - run: uv sync
      # testcontainers needs Docker — ubuntu-latest has it pre-installed
      - run: uv run pytest tests/unit/ --vcr-record=none --cov --cov-fail-under=90

  integration-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v4
      - run: uv sync
      - run: docker compose -f docker-compose.test.yml up -d --wait
      - run: uv run pytest tests/integration/ --vcr-record=none
      - run: docker compose -f docker-compose.test.yml down -v

  e2e-tests:
    # Only on main or manual trigger
    if: github.ref == 'refs/heads/main' || github.event_name == 'workflow_dispatch'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v4
      - run: uv sync
      - run: docker compose build
      - run: docker compose up -d --wait
      - run: uv run pytest tests/e2e/ --e2e --vcr-record=none --timeout=300
      - run: docker compose down -v
```

All CI jobs run with `--vcr-record=none`. No real API calls are made in CI. No API keys
are needed for unit or integration test jobs. The cassettes committed to the repo are the
single source of truth for expected LLM behavior.

## Test Utilities

### Assertion Helpers

```python
def assert_task_completed(store: StateStore, task_id: str) -> None:
    """Assert a task reached completion with all checkpoints."""
    task = store.get_task(task_id)
    assert task.status == "completed"
    checkpoints = store.get_checkpoints(task_id)
    assert "committed" in [c.checkpoint_name for c in checkpoints]

def assert_consensus_reached(response: ConsensusResponse) -> None:
    """Assert consensus was reached (not escalated)."""
    assert response.status in ("agreed", "majority", "weighted")
    assert response.confidence > 0.5

async def assert_stream_has_messages(
    redis_client, stream: str, expected_count: int
) -> None:
    """Assert a Redis stream has the expected number of messages."""
    length = await redis_client.xlen(stream)
    assert length == expected_count
```

### VCR Helpers

```python
def refresh_cassettes(test_path: str, markers: list[str] | None = None) -> None:
    """Re-record cassettes for a given test path. Used in monthly refresh CI job."""
    cmd = ["uv", "run", "pytest", test_path, "--vcr-record=all"]
    if markers:
        cmd.extend(["-m", " or ".join(markers)])
    subprocess.run(cmd, check=True)
```

## What We Do Not Mock

For clarity, here is the explicit list of things we refuse to mock:

| Component | Instead of mocking... | We use... |
|---|---|---|
| Redis | fakeredis | testcontainers-redis (real Redis 7) |
| PostgreSQL | SQLite-in-memory | testcontainers-postgres (real PostgreSQL 16) |
| LLM APIs | Hand-crafted fake responses | VCR cassettes (recordings of real API responses) |
| Docker daemon | Mock Docker SDK | Docker-in-Docker |
| Git | Mock subprocess calls | Real git repos in `tmp_path` |
| File system | Mock file I/O | Real files in `tmp_path` |

The only abstraction layer is VCR, and even that is just HTTP-level record/replay of
real interactions — not a mock.

---

## Cross-References

- **Spec 05** (Infrastructure): Docker Compose configuration for test infrastructure.
- **Spec 15** (Observability): Log events used in assertions and test verification.
- **Spec 16** (Cost Tracking): Cost calculation tests, budget enforcement tests.
- **Spec 17** (Error Recovery): Checkpoint save/resume tests, retry logic tests.
- **Spec 20** (Development Standards): Coverage targets (90%+), CI pipeline configuration.
- **Spec 21** (Repo Connection): `.ai-team.yaml` parsing tests, test repo fixture.
