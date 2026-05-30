---
title: "tissue-bot Backend: GitHub Integration and SQLite"
date: 2026-05-22 10:00:00 -0400
categories:
  - tissue-bot
tags:
  - python
  - sqlite
  - github
  - sqlalchemy
series: tissue-bot
excerpt: "How we built the tissue-bot data layer — async GitHub REST client, SQLite schema, and a CLI for collecting scientific Python repos."
---

The first goal for [tissue-bot](https://github.com/kurtc3b3/tissue-bot) was straightforward: **pull GitHub repository and issue data into a local store** so we could query it, analyze it, and eventually hand it to agents.

We started with shell scripts and the `gh` CLI, then moved the core logic into a Python package managed with **uv**.

## Project layout

The backend lives under `backend/`:

```
backend/
├── pyproject.toml
├── src/backend/
│   ├── github/      # REST client + Pydantic models
│   ├── db/            # SQLAlchemy models, engine, repository
│   ├── collectors/    # repo + issue ingestion
│   └── cli.py         # `tissue` CLI entry point
└── data/              # SQLite files (gitignored)
```

Shared SQL schema and tracked-repo config sit at the repo root in `scripts/`:

```
scripts/
├── db/schema.sql
└── config/scientific-repos.txt
```

## SQLite schema

We designed the schema for data collected from the GitHub REST API. Core tables:

- **`repos`** — owner, name, stars, language, topics (JSON), timestamps
- **`issues`** — linked to repos, with state, body, author, labels
- **`issue_labels`** — many-to-many labels per issue
- **`sync_log`** — audit trail for collection runs

Later we added tables for the agent layer (`chat_threads`, `chat_messages`, `issue_resolutions`), but the ingestion schema came first.

`scripts/db/schema.sql` is the source of truth. The CLI command `tissue init-db` applies it to create `backend/data/tissue-bot.db`.

## GitHub client

The GitHub integration uses **httpx** for async HTTP. A thin `GitHubClient` wraps common endpoints:

- `GET /repos/{owner}/{repo}`
- `GET /repos/{owner}/{repo}/issues`

Pydantic models (`RepoRecord`, `IssueRecord`) normalize API responses — parsing topics JSON, normalizing issue state to `OPEN`/`CLOSED`, flattening label objects into strings.

Authentication is flexible:

1. `GITHUB_TOKEN` in `backend/.env`, or
2. Fallback to `gh auth token` if the GitHub CLI is logged in

Settings are centralized in `backend/settings.py` using **pydantic-settings**, so the same config works for CLI, API, and Docker.

## Collectors

Two collector modules drive ingestion:

- **`collectors/repos.py`** — fetch repo metadata and upsert into SQLite
- **`collectors/issues.py`** — fetch issues for a repo; auto-collects repo metadata first if missing

The CLI exposes this as:

```bash
tissue init-db
tissue collect-repos numpy/numpy
tissue collect-issues numpy/numpy --state open --limit 50
tissue collect-tracked --issue-limit 50   # uses scientific-repos.txt
```

`collect-tracked` reads `scripts/config/scientific-repos.txt` — a curated list of scientific Python libraries (numpy, pandas, scipy, etc.) — and batch-collects issues from each.

## Database access

We use **SQLAlchemy 2.x** with **aiosqlite** for async sessions in the API layer, and a simpler synchronous `Database` wrapper for the CLI collectors.

The repository module (`db/repository.py`) holds query helpers: list repos ordered by stars, fetch issues with filters, search across stored data.

## Why SQLite?

For a local-first tool that collects public GitHub data, SQLite is a good fit:

- Zero ops — one file in `backend/data/`
- Fast enough for tens of thousands of issues
- Easy to inspect with any SQL client
- Works identically in dev, tests (in-memory), and Docker (volume mount)

## What came next

With ingestion working, the natural next step was exposing the same data over HTTP — which led to the FastAPI layer covered in the [next post](/tissue-bot/tissue-bot-2026-05-23-tissue-bot-fastapi-setup/).

## Try it

```bash
cd backend
cp .env.template .env
uv sync
uv run tissue init-db
uv run tissue collect-repos pandas-dev/pandas
```
