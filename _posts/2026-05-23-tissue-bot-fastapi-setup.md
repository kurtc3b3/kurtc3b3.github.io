---
title: "tissue-bot API: FastAPI, Auth, and Routes"
date: 2026-05-23 10:00:00 -0400
categories:
  - tissue-bot
tags:
  - fastapi
  - jwt
  - python
  - api
series: tissue-bot
excerpt: "Exposing collected GitHub data over HTTP with FastAPI, JWT auth via fastapi-users, and routes for repos, issues, chat, and resolutions."
---

Once the CLI could collect GitHub data into SQLite, we needed a **proper HTTP API** so a frontend (and agents) could access it without shelling out to `tissue`.

We built the API with **FastAPI** and packaged it as the `tissue-api` console script.

## Application structure

```
backend/src/backend/api/
├── app.py          # FastAPI factory + CORS + lifespan
├── main.py         # uvicorn entry point
├── deps.py         # shared dependencies
├── schemas.py      # Pydantic response models
└── routers/
    ├── repos.py
    ├── issues.py
    ├── chat.py
    └── resolutions.py
```

`create_app()` in `app.py` wires everything together:

- **CORS middleware** for the Next.js frontend
- **Lifespan handler** — creates SQLAlchemy tables on startup, initializes the LangGraph agent service
- **Routers** for each resource area

Run locally with:

```bash
uv run tissue-api
# → http://127.0.0.1:8000/docs
```

## Authentication

All data routes require a logged-in user. We use **fastapi-users** with JWT:

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/auth/register` | Create account |
| POST | `/auth/jwt/login` | Login (form: username=email, password) |
| GET | `/users/me` | Current user profile |

Clients send `Authorization: Bearer <token>` on protected routes.

`SECRET_KEY` in `.env` signs tokens. For production this must be a real secret — the template warns about this.

## Repository routes

```
POST /repos/{owner}/{repo}/collect   # fetch from GitHub + store
GET  /repos                          # list stored repos
GET  /repos/{owner}/{repo}           # single repo
```

Collect endpoints reuse the same collector functions as the CLI, so CLI and API always stay in sync.

## Issue routes

```
POST /repos/{owner}/{repo}/issues/collect   # ?state=open&limit=100
GET  /repos/{owner}/{repo}/issues           # list with filters
GET  /repos/{owner}/{repo}/issues/{number}  # single issue
POST /repos/{owner}/{repo}/issues/{number}/resolve  # agent (later post)
```

Response models like `RepoOut` and `IssueOut` handle the messy parts — parsing topics JSON from SQLite, normalizing booleans, serializing ORM objects with `from_attributes=True`.

## Settings and CORS

Configuration lives in `settings.py`. Important API-related env vars:

| Variable | Default | Notes |
|----------|---------|-------|
| `API_HOST` | `127.0.0.1` | Set to `0.0.0.0` in Docker |
| `API_PORT` | `8000` | |
| `CORS_ORIGINS` | comma-separated list | Must include frontend origin |
| `SECRET_KEY` | change-me | Required for JWT |

We hit a subtle bug in Docker: pydantic-settings tries to JSON-parse list-typed env vars. `CORS_ORIGINS=http://localhost:3000,...` failed until we stored it as a plain string and split it in a computed property. More on that in the [Docker post](/tissue-bot/tissue-bot-2026-05-29-tissue-bot-docker-compose/).

## Error handling

GitHub API failures (404 repo, auth errors) used to bubble up as unhandled 500s — which browsers reported as CORS errors. We added `api/errors.py` to map `httpx.HTTPStatusError` to proper HTTP responses with JSON bodies.

## Health check

```
GET /health → {"status": "ok"}
```

Simple, but useful for Docker and compose health checks.

## What came next

The API was ready for a UI. The [frontend post](/tissue-bot/tissue-bot-2026-05-24-tissue-bot-frontend-nextjs/) covers the Next.js app that consumes these endpoints.

## Quick test

```bash
# register
curl -X POST http://127.0.0.1:8000/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"you@example.com","password":"secure-password"}'

# login + collect
TOKEN=$(curl -s -X POST http://127.0.0.1:8000/auth/jwt/login \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=you@example.com&password=secure-password" | jq -r .access_token)

curl -X POST http://127.0.0.1:8000/repos/numpy/numpy/collect \
  -H "Authorization: Bearer $TOKEN"
```
