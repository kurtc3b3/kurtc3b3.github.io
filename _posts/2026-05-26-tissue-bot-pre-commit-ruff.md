---
title: "tissue-bot Dev Workflow: Pre-commit and Ruff"
date: 2026-05-26 10:00:00 -0400
categories:
  - tissue-bot
tags:
  - pre-commit
  - ruff
  - python
  - linting
series: tissue-bot
excerpt: "Adding pre-commit hooks and Ruff linting to the tissue-bot Python backend for consistent formatting and fast feedback on every commit."
---

As tissue-bot grew — collectors, API routes, agents, tests — we wanted **automated linting on every commit** instead of relying on manual `ruff check` runs.

We added **pre-commit** at the repo root and **Ruff** as the Python linter/formatter.

## Pre-commit config

`.pre-commit-config.yaml` at the repo root:

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v6.0.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files

  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.15.15
    hooks:
      - id: ruff-check
        name: ruff check (backend)
        args: [--fix]
        files: ^backend/
      - id: ruff-format
        name: ruff format (backend)
        files: ^backend/
```

Generic file hygiene hooks run on the whole repo. Ruff hooks are scoped to `backend/` only — the frontend uses ESLint separately.

## Ruff configuration

In `backend/pyproject.toml`:

```toml
[tool.ruff]
target-version = "py313"
line-length = 100
src = ["src", "tests"]

[tool.ruff.lint]
select = ["E", "W", "F", "I", "UP", "B", "SIM"]
ignore = ["B008"]  # Allow Depends() in FastAPI route defaults
```

We enable pycodestyle, pyflakes, isort, pyupgrade, bugbear, and simplify rules. `B008` is ignored because FastAPI's `Depends()` in route signatures triggers it falsely.

## Setup

```bash
cd backend
uv sync --group dev
cd .. && pre-commit install
```

After that, every `git commit` runs the hooks automatically. To lint everything manually:

```bash
pre-commit run --all-files
```

Or from `backend/`:

```bash
make lint
```

## What we fixed along the way

The initial Ruff pass caught real issues — unused imports, import ordering, simplifiable conditionals, and formatting inconsistencies across `backend/src/`.

Having `--fix` on the ruff-check hook means many issues are auto-corrected before the commit completes. If a hook modifies files, you re-stage and commit again.

## Why Ruff over black + flake8 + isort?

One tool, fast, and already in our dev dependency group. For a single Python package it keeps config in one place (`pyproject.toml`) with no extra config files.

## What came next

Linting catches style issues; **tests** catch behavior regressions — covered in the [pytest post](/tissue-bot/tissue-bot-2026-05-27-tissue-bot-pytest/).
