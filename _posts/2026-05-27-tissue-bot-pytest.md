---
title: "tissue-bot Testing: Pytest and Async Fixtures"
date: 2026-05-27 10:00:00 -0400
categories:
  - tissue-bot
tags:
  - pytest
  - testing
  - python
series: tissue-bot
excerpt: "Adding 18 pytest unit tests to tissue-bot — async DB fixtures, API schema tests, agent helpers, and health/auth endpoint coverage."
---

Pre-commit keeps the code clean; **pytest** keeps it correct.

We added a test suite under `backend/tests/` covering models, schemas, DB queries, agent helpers, and API endpoints.

## Configuration

In `backend/pyproject.toml`:

```toml
[dependency-groups]
dev = [
    "pytest>=9.0.3",
    "pytest-asyncio>=1.4.0",
    # ...
]

[tool.pytest.ini_options]
testpaths = ["tests"]
pythonpath = ["src"]
asyncio_mode = "auto"
asyncio_default_fixture_loop_scope = "function"
```

The important detail: **`pythonpath = ["src"]`**. Without it, `import backend` fails when running plain `pytest`. Always use:

```bash
cd backend && uv run pytest
```

`uv run` ensures the venv and path are correct.

## Fixtures (`conftest.py`)

Shared fixtures set up an **in-memory SQLite database** for repository tests:

- Apply schema SQL
- Seed sample repo + issue data
- Provide async `db_session` for query tests
- Clear `get_settings()` cache between tests

This lets DB tests run fast and isolated without touching `backend/data/tissue-bot.db`.

## What's tested

| Module | Coverage |
|--------|----------|
| `test_github_models.py` | `RepoRecord` / `IssueRecord` parsing from API JSON |
| `test_api_schemas.py` | `RepoOut`, `IssueOut`, `ChatMessageOut` serialization |
| `test_agents_graph.py` | Reply extraction, initial message building |
| `test_db_repository.py` | Repo ordering, issue fetch, search |
| `test_api_app.py` | `/health`, auth-required routes |
| `test_api_deps.py` | Helper functions |
| `test_settings.py` | CORS env parsing |

**18 tests** total, all passing in under half a second.

## Example: schema test

API responses need to handle messy SQLite storage — topics as JSON strings, labels as objects:

```python
def test_repo_out_parses_topics_json_and_booleans():
    repo = RepoOut.model_validate({...})
    assert repo.topics == ["python", "science"]
```

These tests caught real bugs before they reached the frontend — like `ChatMessageOut` missing `from_attributes=True` for ORM serialization.

## Running tests

```bash
cd backend
uv sync --group dev
uv run pytest
uv run pytest -v          # verbose
uv run pytest tests/test_api_app.py  # single file
```

Or:

```bash
make test
```

## What came next

Tests and linting were easy to run but easy to forget — so we wrapped common commands in a **Makefile** ([next post](/tissue-bot/tissue-bot-2026-05-28-tissue-bot-makefile/)).
