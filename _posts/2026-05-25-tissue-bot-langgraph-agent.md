---
title: "tissue-bot LangGraph Agent: Ollama and Issue Resolution"
date: 2026-05-25 10:00:00 -0400
categories:
  - tissue-bot
tags:
  - langgraph
  - ollama
  - agents
  - llm
series: tissue-bot
excerpt: "Adding a LangGraph ReAct agent with Ollama, SQLite checkpointing, tools over stored GitHub data, and a chat UI for issue resolution."
---

Collecting GitHub data was step one. Step two: **use an agent to analyze issues and propose resolutions** using the local store as context.

We integrated **LangGraph** with **Ollama** (local LLM) and built chat + resolution APIs and UI on top.

## Architecture

```
backend/src/backend/agents/
├── graph.py      # ReAct agent factory + message helpers
├── tools.py      # search repos/issues, save resolutions
└── service.py    # thread lifecycle, checkpointing, persistence
```

The agent is a **ReAct loop** — the model decides which tools to call, observes results, and continues until it produces a final answer.

## Model: Ollama + llama3.2

We run the LLM locally via [Ollama](https://ollama.com/):

```bash
ollama pull llama3.2
ollama serve
```

Configured in `.env`:

```
OLLAMA_BASE_URL=http://127.0.0.1:11434
OLLAMA_MODEL=llama3.2
```

`ChatOllama` from `langchain-ollama` connects to the Ollama HTTP API. Temperature is set low (0.2) for more deterministic analysis.

## Tools

The agent gets tools built per user/thread in `tools.py`:

| Tool | Purpose |
|------|---------|
| Search repos | Query stored repositories |
| Search issues | Find issues by text/state |
| Get stored issue | Full issue + repo context |
| Save issue resolution | Persist proposed fix locally |
| List resolutions | Browse past resolutions |

Tools read from SQLite via SQLAlchemy — the same data the CLI and API collected. We use `selectinload` to avoid detached-instance errors when serializing related repo data.

**Important constraint:** resolutions are saved **locally only**. The system prompt explicitly tells the model not to claim it created GitHub commits or PRs. Opening PRs is on the roadmap.

## Checkpointing

Conversation state is checkpointed with **AsyncSqliteSaver** from `langgraph-checkpoint-sqlite`:

```
backend/data/langgraph-checkpoints.db
```

This lets threads survive server restarts and supports multi-turn conversations. The agent service manages checkpointer lifecycle in an async context manager wired into FastAPI's app lifespan.

## Chat persistence

In addition to LangGraph checkpoints, we persist human-readable history in SQLite:

- **`chat_threads`** — title, optional linked issue, user ID
- **`chat_messages`** — role + content per thread
- **`issue_resolutions`** — saved agent proposals

API routes:

```
POST /chat/threads
GET  /chat/threads
GET  /chat/threads/{id}/messages
POST /chat/threads/{id}/messages
GET  /resolutions
POST /repos/{owner}/{repo}/issues/{number}/resolve
```

The **resolve** endpoint creates a thread scoped to an issue, sends an initial prompt, and returns the agent's reply.

## System prompt

The agent is instructed to:

1. Use `get_stored_issue` before analyzing
2. Propose concrete fixes (steps, snippets, config changes)
3. Save resolutions with `save_issue_resolution`
4. Never guess — look up data with tools

## Frontend integration

- **Navbar** link to `/chat`
- **Resolve** button on issue rows → triggers `/resolve` endpoint
- **Chat** button → opens thread with issue context

The chat page lists threads, shows message history, and sends new messages to the agent.

## Running the full stack

```bash
# Ollama
ollama pull llama3.2 && ollama serve

# API
cd backend && uv run tissue-api

# UI
cd frontend && npm run dev
```

Or `make ollama-setup` from `backend/` (covered in the [Makefile post](/tissue-bot/tissue-bot-2026-05-28-tissue-bot-makefile/)).

## What came next

As the codebase grew, we added **pre-commit + Ruff** and **pytest** to keep quality high — covered in the next posts.
