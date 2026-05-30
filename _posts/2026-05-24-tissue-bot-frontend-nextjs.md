---
title: "tissue-bot Frontend: Next.js UI"
date: 2026-05-24 10:00:00 -0400
categories:
  - tissue-bot
tags:
  - nextjs
  - react
  - typescript
  - frontend
series: tissue-bot
excerpt: "Building a Next.js frontend for tissue-bot — JWT auth, repo/issue tables, search, and hooks into the agent chat flow."
---

With the FastAPI backend running, we built a **Next.js** frontend so we could browse collected data and trigger collection from the browser.

The app lives in `frontend/` and talks to `tissue-api` over HTTP.

## Stack

- **Next.js 16** (App Router)
- **React 19** + TypeScript
- **Tailwind CSS 4**
- JWT stored in `localStorage` (`tissue_token`)

No server-side data fetching for the main views — pages are client components that call the API after login. This kept the frontend simple and aligned with our JWT-in-browser auth model.

## Project structure

```
frontend/src/
├── app/
│   ├── login/ register/ repos/ chat/
│   └── repos/[owner]/[repo]/issues/
├── components/
│   ├── repos-table.tsx
│   ├── issues-table.tsx
│   ├── chat-page.tsx
│   └── ...
├── contexts/auth-context.tsx
└── lib/
    ├── api.ts
    ├── auth-storage.ts
    └── search.ts
```

## Authentication flow

1. User registers or logs in via `/auth/register` and `/auth/jwt/login`
2. JWT is saved to `localStorage`
3. `AuthProvider` wraps the app and exposes login state
4. `ProtectedRoute` redirects unauthenticated users to `/login`
5. `api.ts` attaches `Authorization: Bearer …` on every authenticated request

One early bug: login sent `Content-Type: application/json` with a `URLSearchParams` body, which caused 422 errors. The fix was to omit `Content-Type` and let the browser set the correct form encoding.

## Key pages

### Repositories (`/repos`)

- Table of stored repos (stars, language, topics)
- **Collect** form — enter `owner/repo`, POST to `/repos/{owner}/{repo}/collect`
- Client-side search/filter via `lib/search.ts`

### Issues (`/repos/{owner}/{repo}/issues`)

- Lists stored issues for a repo
- State filter (open / closed / all), defaulting to **open**
- Collect issues form — POST to `/issues/collect`
- Issue modal with full title and body
- **Resolve** and **Chat** buttons linking to the agent

### Agent chat (`/chat`)

- Thread list and message UI
- Sends messages to `/chat/threads/{id}/messages`

## API client

`lib/api.ts` centralizes all backend calls:

```typescript
const API_BASE = process.env.NEXT_PUBLIC_API_URL ?? "http://127.0.0.1:8000";
```

Configured via `frontend/.env.local`:

```
NEXT_PUBLIC_API_URL=http://127.0.0.1:8000
```

The client handles token injection, error parsing (including FastAPI validation errors), and 401 logout.

## UX details we cared about

- **Search** on repos and issues tables (client-side, instant)
- **Author links** to GitHub profiles
- **Compact issue table** with `table-fixed` layout
- Issue title click opens modal (removed duplicate View link)
- State filter defaults to open issues

## Static export for Docker

When we containerized the frontend (covered in the [Docker post](/tissue-bot/tissue-bot-2026-05-29-tissue-bot-docker-compose/)), we switched to **static export**:

```typescript
// next.config.ts
const nextConfig = { output: "export" };
```

nginx serves the built files. Dynamic issue routes use a placeholder page plus URL path parsing — because `useParams()` alone returns `_/_` from the static shell. We added `lib/repo-path.ts` to read owner/repo from `window.location.pathname`.

## Running locally

```bash
# terminal 1
cd backend && uv run tissue-api

# terminal 2
cd frontend
cp .env.local.example .env.local
npm install && npm run dev
# → http://localhost:3000
```

## What came next

Browsing and collecting data was useful, but the real goal was **agent-assisted issue resolution** — covered in the [LangGraph post](/tissue-bot/tissue-bot-2026-05-25-tissue-bot-langgraph-agent/).
