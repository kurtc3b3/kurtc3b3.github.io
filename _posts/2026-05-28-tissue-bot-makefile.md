---
title: "tissue-bot Makefile: One Command for Common Tasks"
date: 2026-05-28 10:00:00 -0400
categories:
  - tissue-bot
tags:
  - makefile
  - developer-experience
  - python
series: tissue-bot
excerpt: "A backend Makefile wrapping setup, API server, lint, tests, and Ollama model pull into simple make targets."
---

After repeating the same commands — `uv sync`, `tissue init-db`, `pre-commit run`, `pytest`, `tissue-api` — we added a **Makefile** to `backend/`.

## Targets

```makefile
make help          # list all commands
make setup         # deps, .env, init-db, pre-commit install
make start-api     # run FastAPI server
make lint          # pre-commit on whole repo
make test          # pytest
make ollama-setup  # ollama pull llama3.2
```

Run everything from `backend/`:

```bash
cd backend
make setup
make start-api
```

## What `setup` does

```makefile
setup:
	@test -f .env || cp .env.template .env
	uv sync --group dev
	uv run tissue init-db
	cd $(ROOT_DIR) && pre-commit install
```

One command gets a new developer from clone to ready-to-work:

1. Copy env template if missing
2. Install Python deps (including dev group)
3. Initialize SQLite
4. Install git hooks from repo root

## Design choices

- **Backend only** — frontend stays `npm install && npm run dev`. No need to couple Node into Make.
- **Pre-commit from repo root** — `.pre-commit-config.yaml` lives at the top level, so `lint` and `setup` cd to `$(ROOT_DIR)`.
- **`OLLAMA_MODEL` override** — `make ollama-setup OLLAMA_MODEL=mistral` if you want a different model.

## Help target

```makefile
help:
	@grep -E '^[a-zA-Z0-9_-]+:.*##' $(MAKEFILE_LIST) | awk ...
```

Every target has a `## description` comment. Bare `make` shows the menu.

## What came next

Local dev was smooth; **Docker** let us ship the same stack in containers — the [final post in this series](/tissue-bot/tissue-bot-2026-05-29-tissue-bot-docker-compose/).
