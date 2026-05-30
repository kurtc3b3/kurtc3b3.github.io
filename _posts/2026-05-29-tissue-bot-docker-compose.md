---
title: "tissue-bot Docker: Containers, Compose, and Deployment Lessons"
date: 2026-05-29 10:00:00 -0400
categories:
  - tissue-bot
tags:
  - docker
  - nginx
  - devops
  - deployment
series: tissue-bot
excerpt: "Containerizing tissue-bot with non-root Docker images, docker-compose, nginx static export, and the CORS and routing bugs we hit along the way."
---

The last piece of the tissue-bot dev stack was **Docker** — run the API and frontend the same way on any machine, without manual uv/npm setup.

## Images

Two Dockerfiles, build context at the repo root:

### Backend (`backend/Dockerfile`)

- Python 3.13 slim + **uv** for dependency install
- Runs as non-root **`app`** user
- Copies `backend/` and `scripts/` (schema + tracked repos config)
- Volume at `/app/backend/data` for SQLite persistence
- Starts with `tissue init-db && tissue-api` on `0.0.0.0:8000`

### Frontend (`frontend/Dockerfile`)

Multi-stage build:

1. **Node 22** — `npm ci && npm run build` (static export)
2. **nginx Alpine** — serves `out/` as non-root **`app`** user on port **3000**

nginx config includes a fallback for dynamic issue routes:

```nginx
location ~ ^/repos/[^/]+/[^/]+/issues/?$ {
    try_files $uri $uri/ /repos/_/_/issues.html;
}
```

## docker-compose.yml

```bash
docker compose up --build
```

| Service | Port | Notes |
|---------|------|-------|
| backend | 8000 | SQLite volume, env vars for secrets |
| frontend | 3000 | Static nginx, `NEXT_PUBLIC_API_URL=http://localhost:8000` |

Optional env:

```bash
GITHUB_TOKEN=ghp_... SECRET_KEY=your-secret docker compose up --build
```

Ollama runs on the host by default (`host.docker.internal:11434`) so we don't bundle a GPU-heavy LLM image.

## Static export change

Docker drove a frontend architecture change. nginx serves static files, so we enabled Next.js export:

```typescript
// next.config.ts
output: "export"
```

Dynamic issue pages use a placeholder route (`/_/_/issues`) at build time. At runtime, **`lib/repo-path.ts`** parses owner/repo from the browser URL — because `useParams()` alone returned `_/_` from the static shell.

This was the root cause of a confusing bug: API calls went to `/repos/_/_/issues/collect` instead of the real repo.

## CORS lessons

Two separate CORS issues:

### 1. Env var parsing

`CORS_ORIGINS=http://localhost:3000,...` failed inside Docker because pydantic-settings JSON-parses list types. Fix: store as a plain string, split in a computed property.

### 2. Misleading browser errors

Unhandled 500s (e.g. GitHub 404 for `repos/_/_`) sometimes showed as **CORS errors** in the browser because the response lacked CORS headers. Fixing the `_/_` routing bug and mapping GitHub errors to proper HTTP responses resolved this.

Final compose config:

```yaml
CORS_ORIGINS: "http://localhost:3000,http://127.0.0.1:3000,http://localhost:8000,http://127.0.0.1:8000"
```

## Security defaults

- Both containers run as **non-root** `app` users
- nginx listens on port 3000 (non-privileged)
- `.dockerignore` excludes `.env`, `node_modules`, `.venv`, local DB files
- `SECRET_KEY` must be set for real deployments

## Full stack URLs

After `docker compose up --build`:

- Frontend: http://localhost:3000
- API docs: http://localhost:8000/docs
- Health: http://localhost:8000/health

## Series recap

This completes the tissue-bot build log:

1. [Backend: GitHub + SQLite](/tissue-bot/tissue-bot-2026-05-22-tissue-bot-backend-github-sqlite/)
2. [FastAPI setup](/tissue-bot/tissue-bot-2026-05-23-tissue-bot-fastapi-setup/)
3. [Next.js frontend](/tissue-bot/tissue-bot-2026-05-24-tissue-bot-frontend-nextjs/)
4. [LangGraph agent](/tissue-bot/tissue-bot-2026-05-25-tissue-bot-langgraph-agent/)
5. [Pre-commit + Ruff](/tissue-bot/tissue-bot-2026-05-26-tissue-bot-pre-commit-ruff/)
6. [Pytest](/tissue-bot/tissue-bot-2026-05-27-tissue-bot-pytest/)
7. [Makefile](/tissue-bot/tissue-bot-2026-05-28-tissue-bot-makefile/)
8. Docker (this post)

What's next: GitHub PR automation for saved resolutions, and an analysis agent with visualizations.
