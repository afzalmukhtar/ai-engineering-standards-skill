---
name: project-structure
description: >-
  Python project structure, code hygiene, testing standards, and folder layout
  patterns. Use when scaffolding a new project, organizing modules, writing
  tests, setting up config, or cleaning up code. Covers folder layout, config
  management, test organization, .gitignore, and pre-completion checklists.
---

# Project Structure & Hygiene

## Core Principles

1. **One bootstrap file** — all wiring and dependency injection happens in one place
2. **Every module has a test** — no module ships without a corresponding test file
3. **Config in one place** — model names, retry counts, thresholds live in a settings module
4. **Dead code gets deleted** — unused functions, imports, models, and empty modules are removed immediately

## Folder Layout

### Standard AI/Agent Project

```
project-root/
├── agents/
│   ├── __init__.py
│   ├── base.py               # BaseAgent ABC
│   ├── orchestrator.py        # routing, fan-out, state machine
│   ├── planner.py
│   ├── researcher.py
│   └── writer.py
├── tools/
│   ├── __init__.py
│   └── web_search.py          # one file per tool or tool group
├── config/
│   ├── __init__.py
│   ├── settings.py            # MODEL, TIMEOUT, MAX_RETRIES, MAX_ITERATIONS
│   └── prompts.py             # all system/user prompt templates
├── models/
│   ├── __init__.py
│   └── state.py               # Pydantic models: Task, TaskResult, AgentState, Report
├── tests/
│   ├── __init__.py
│   ├── conftest.py            # shared fixtures, mocks
│   ├── test_planner.py
│   ├── test_researcher.py
│   ├── test_writer.py
│   ├── test_orchestrator.py
│   ├── test_tools.py
│   └── test_pipeline.py       # E2E integration test
├── driver.py                   # bootstrap: wire agents, run pipeline
├── llm.py                      # centralized LLM call wrapper
├── pyproject.toml
├── pytest.ini
├── .gitignore
└── .env.example                # template showing required env vars (never .env itself)
```

### Key Layout Rules

- **agents/** — one file per agent, no cross-imports between agent files
- **tools/** — one file per tool or tool group, sync function + async wrapper
- **config/** — all settings and prompts, importable by any module
- **models/** — all Pydantic models, importable by any module
- **tests/** — mirrors source structure; `conftest.py` for shared fixtures

## Config Management

### settings.py

```python
"""Central configuration — all magic numbers live here."""

MODEL = "azure/gpt-4.1"
TEMPERATURE = 0.1
TIMEOUT_S = 60
MAX_RETRIES = 3
MAX_ITERATIONS = 10
MIN_SUBTASKS = 3
MAX_SUBTASKS = 5
SEARCH_TIMEOUT_S = 30
MAX_CONCURRENT_SEARCHES = 5
```

### Rules

- No hardcoded model names, timeouts, or retry counts in agent files
- Settings importable as `from config.settings import MODEL, MAX_RETRIES`
- Environment-specific overrides via env vars or `.env` file
- `.env.example` committed to repo showing required vars (never `.env` itself)

## .gitignore Essentials

```gitignore
# Python
__pycache__/
*.pyc
*.pyo
*.egg-info/
dist/
build/

# Environment
.env
.venv/
venv/

# IDE
.vscode/
.idea/

# Runtime artifacts
*.db
*.sqlite3
chroma_db/
.chroma/

# OS
.DS_Store
Thumbs.db
```

## Testing Standards

### File Organization

```
tests/
├── conftest.py           # shared fixtures
├── test_planner.py       # unit tests for planner agent
├── test_researcher.py    # unit tests for researcher agent
├── test_writer.py        # unit tests for writer agent
├── test_orchestrator.py  # routing, fan-out, failure handling
├── test_tools.py         # tool functions
└── test_pipeline.py      # E2E integration
```

### Minimum Test Coverage Per Module

| Module Type | Required Tests |
|---|---|
| Tool function | happy path, empty input, exception handling |
| Agent class | E2E with mocked LLM, tool routing, graceful failure |
| Orchestrator | routing logic, fan-out, partial failure, total failure, loop cap |
| Pipeline | full E2E with all LLMs mocked, asserts final output structure |
| Concurrent code | timing assertion (N tasks in ~1x not Nx), partial failure |

### Test Naming Convention

```python
# BAD
def test_search_2():
def test_planner():

# GOOD
def test_search_returns_empty_when_no_results_match():
def test_planner_falls_back_to_single_task_on_invalid_json():
def test_researchers_complete_in_parallel_not_sequentially():
```

### conftest.py Shared Fixtures

```python
import pytest
from unittest.mock import AsyncMock

FAKE_LLM_RESPONSE = '{"summary": "test"}'

@pytest.fixture
def mock_achat():
    return AsyncMock(return_value=FAKE_LLM_RESPONSE)

@pytest.fixture
def sample_tasks():
    return [
        Task(id="T1", description="task 1"),
        Task(id="T2", description="task 2"),
        Task(id="T3", description="task 3"),
    ]
```

### pytest Configuration

```toml
# pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
markers = [
    "integration: tests that hit real external services",
]
```

## Code Hygiene Checklist

### Dead Code

- [ ] No unused imports (run `ruff check --select F401`)
- [ ] No unused functions or classes
- [ ] No Pydantic models that are defined but never instantiated
- [ ] No empty `__init__.py` files with unused imports
- [ ] No commented-out code blocks

### Base Class Cleanliness

- [ ] Every method/attribute on the base class is used by at least one subclass
- [ ] If only one subclass uses it, move it to that subclass

### Consistency

- [ ] All agents use the same public method name (`invoke` or `run`, not a mix)
- [ ] All tool functions follow the same sync + async wrapper pattern
- [ ] All imports use the same style (relative vs absolute, consistent across files)

## Pre-Completion Checklist

Run this before marking any task as done:

```
Pre-Completion Verification:
- [ ] Linter passes: ruff check . (or flake8/pylint equivalent)
- [ ] All tests pass: pytest -v
- [ ] No dead code: unused imports, functions, models
- [ ] Config values in settings.py, not hardcoded
- [ ] Prompts in config/prompts.py, not inline
- [ ] All Pydantic models are used in actual data flow
- [ ] .gitignore covers runtime artifacts
- [ ] Manually trace critical path: user input → planner → researcher → writer → output
```

## Anti-Patterns to Flag

| Pattern | Problem | Fix |
|---|---|---|
| Settings scattered across files | Hard to find, easy to forget | Centralize in `config/settings.py` |
| `TOOLS_REGISTRY` defined but unused | Dead code | Delete or wire into dispatch |
| Tests in one giant file | Hard to maintain, slow to find failures | One test file per source module |
| `test_1`, `test_2` naming | No idea what's tested | Descriptive names: `test_X_when_Y_then_Z` |
| `.env` committed to repo | Secrets leak | `.gitignore` it, commit `.env.example` |
| No conftest.py | Fixtures duplicated | Shared fixtures in `conftest.py` |
