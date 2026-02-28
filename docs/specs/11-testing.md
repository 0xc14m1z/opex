# 11 — Testing Strategy

## Overview

Testing agentic systems is hard because the core behavior depends on non-deterministic
LLM responses. Our strategy: record/replay (VCR-style) for deterministic regression tests,
with the ability to update recordings like snapshot tests. Live integration tests run
separately against real LLM APIs.

---

## Test Pyramid

```
        ╱╲
       ╱  ╲          Live E2E Tests
      ╱ E2E╲         Real LLMs, real repo, full pipeline
     ╱──────╲         (CI: separate stage, manual trigger)
    ╱        ╲
   ╱Integration╲     Integration Tests
  ╱──────────────╲    Redis, SQLite, Docker — mocked LLMs
 ╱                ╲
╱   Unit Tests     ╲  Unit Tests
╱───────────────────╲  Pure logic, no I/O, fast
```

| Layer        | LLM Calls | Redis | SQLite | Docker | Speed    | CI Stage    |
|-------------|-----------|-------|--------|--------|----------|-------------|
| Unit        | Mocked    | No    | No     | No     | < 1s     | Every PR    |
| Integration | Recorded  | Yes   | Yes    | No     | < 30s    | Every PR    |
| E2E         | Real      | Yes   | Yes    | Yes    | Minutes  | Manual/nightly |

## Record/Replay (VCR-Style)

### How It Works

1. **Record mode**: Run tests against real LLM APIs. All requests and responses are
   captured and saved as fixture files.
2. **Replay mode** (default): Tests use recorded fixtures. No API calls made.
   Deterministic and fast.
3. **Update mode**: Like snapshot testing — re-record fixtures when expected behavior
   changes. Developer reviews the diff.

### Implementation

Use [pytest-recording](https://github.com/kiwicom/pytest-recording) (built on VCR.py)
or a custom implementation tuned for LLM APIs.

```python
# conftest.py
import pytest
from ai_team_core.testing import LLMRecorder

@pytest.fixture
def llm_recorder(request):
    """VCR-style fixture for LLM calls."""
    cassette_path = (
        Path("tests/cassettes")
        / request.node.module.__name__
        / f"{request.node.name}.yaml"
    )
    recorder = LLMRecorder(cassette_path)

    if request.config.getoption("--record-llm"):
        recorder.mode = "record"
    elif request.config.getoption("--update-llm"):
        recorder.mode = "update"  # Re-record and show diff
    else:
        recorder.mode = "replay"

    with recorder:
        yield recorder
```

### Cassette Format

Each recorded interaction is stored as a YAML file:

```yaml
# tests/cassettes/test_nelson/test_consensus_majority.yaml
interactions:
  - request:
      provider: "claude"
      model: "claude-3-5-sonnet-20241022"
      messages:
        - role: "system"
          content: "You are evaluating..."
        - role: "user"
          content: "Review the following code..."
      temperature: 0.0
    response:
      content: '{"decision": "approve", "reasoning": "..."}'
      prompt_tokens: 1200
      completion_tokens: 450
      model: "claude-3-5-sonnet-20241022"

  - request:
      provider: "gpt4o"
      # ...
    response:
      # ...

  - request:
      provider: "gemini"
      # ...
    response:
      # ...
```

### Updating Recordings

```bash
# Re-record all cassettes (hits real APIs)
uv run pytest tests/ --record-llm

# Re-record specific test
uv run pytest tests/unit/test_nelson.py::test_consensus_majority --record-llm

# Update mode: re-record and show diff for review
uv run pytest tests/ --update-llm
# This outputs a diff of changed cassettes, similar to:
# - old response: "approve with minor nits"
# + new response: "approve, clean implementation"
```

### When to Update Recordings

- **Prompt changes**: If you modify an agent's prompt template, re-record affected tests.
- **Model upgrades**: When switching to a newer model version, re-record all.
- **Behavior changes**: If the expected output changes, update and review the diff.
- **Periodic freshness**: Monthly re-record to catch model drift (automated via CI).

## Unit Tests

### What to Unit Test

| Component | Test Focus |
|---|---|
| Consensus algorithm | Majority detection, weighted voting, termination logic |
| Weight learning | Weight updates, normalization, floor enforcement |
| Cost calculation | Token→cost math, budget limit logic |
| Dependency graph | Task ordering, parallel group detection, cycle detection |
| Message serialization | Pydantic model → Redis → Pydantic round-trip |
| Checkpoint logic | Save, load, resume-from-checkpoint |
| Config parsing | `.ai-team.yaml` validation, defaults, error messages |
| Retry logic | Counter, max retries, escalation decision |

### Unit Test Style

```python
# tests/unit/test_consensus_algorithm.py

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
```

## Integration Tests

### What to Integration Test

| Scenario | Components Involved |
|---|---|
| Task flows through pipeline | Redis streams, message serialization, consumer groups |
| Nelson consensus with recorded LLM | Nelson + Redis + LLM recorder |
| Checkpoint save/resume | Agent + SQLite + Redis |
| Cost tracking accumulation | LLM calls + SQLite + budget checking |
| Git operations | Richelieu + test git repo |
| Config loading | `.ai-team.yaml` parsing from a test repo |

### Integration Test Infrastructure

```python
# tests/conftest.py

@pytest.fixture
async def redis():
    """Provides a clean Redis instance (uses a test-specific DB)."""
    client = redis.asyncio.Redis(db=15)  # Use DB 15 for tests
    await client.flushdb()
    yield client
    await client.flushdb()
    await client.close()

@pytest.fixture
def sqlite_db(tmp_path):
    """Provides a temporary SQLite database."""
    db_path = tmp_path / "test.db"
    store = StateStore(db_path)
    store.initialize()
    yield store

@pytest.fixture
def test_repo(tmp_path):
    """Creates a temporary git repo with a .ai-team.yaml."""
    repo_path = tmp_path / "test-repo"
    repo_path.mkdir()
    # Initialize git repo
    subprocess.run(["git", "init"], cwd=repo_path)
    # Write .ai-team.yaml
    (repo_path / ".ai-team.yaml").write_text(TEST_CONFIG_YAML)
    # Write some source files
    (repo_path / "src").mkdir()
    (repo_path / "src" / "main.py").write_text("def hello(): return 'world'")
    # Initial commit
    subprocess.run(["git", "add", "."], cwd=repo_path)
    subprocess.run(["git", "commit", "-m", "initial"], cwd=repo_path)
    yield repo_path
```

## E2E Tests

### What to E2E Test

Full pipeline runs against a test repository with real LLM calls:

1. **Happy path**: Issue → decomposition → enrichment → implementation → review → PR.
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
```

### Running E2E Tests

```bash
# Start the stack
make up

# Run E2E tests (separate CI stage, costs real money)
uv run pytest tests/e2e/ --e2e --timeout=300

# Run a specific scenario
uv run pytest tests/e2e/test_happy_path.py --e2e
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
      - run: uv run pytest tests/unit/ --cov --cov-fail-under=90

  integration-tests:
    runs-on: ubuntu-latest
    services:
      redis:
        image: redis:7-alpine
        ports: ["6379:6379"]
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v4
      - run: uv sync
      - run: uv run pytest tests/integration/

  e2e-tests:
    # Only on main or manual trigger
    if: github.ref == 'refs/heads/main' || github.event_name == 'workflow_dispatch'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v4
      - run: docker compose build
      - run: docker compose up -d
      - run: uv run pytest tests/e2e/ --e2e --timeout=300
      - run: docker compose down
    env:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
      OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
      GOOGLE_API_KEY: ${{ secrets.GOOGLE_API_KEY }}
```

## Test Utilities

### Mock LLM Client

For unit tests that don't use cassettes:

```python
class MockLLMClient:
    """Deterministic LLM client for unit tests."""

    def __init__(self, responses: dict[str, str]):
        self.responses = responses  # prompt_substring → response
        self.calls: list[LLMCall] = []

    async def complete(self, prompt: str, **kwargs) -> LLMResponse:
        for key, response in self.responses.items():
            if key in prompt:
                self.calls.append(LLMCall(prompt=prompt, response=response))
                return LLMResponse(content=response, tokens=TokenUsage(...))
        raise ValueError(f"No mock response for prompt containing: {prompt[:100]}")
```

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
```
