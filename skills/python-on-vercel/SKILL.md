---
name: Python on Vercel
slug: python-on-vercel
version: 1
course_url: https://vercel.com/academy/python-on-vercel
commands:
  - /python-on-vercel learn
  - /python-on-vercel new
  - /python-on-vercel submit
---

# Python on Vercel

Companion skill for the [Python on Vercel](https://vercel.com/academy/python-on-vercel) course. Ship a FastAPI backend and a Next.js 16 frontend as a single Vercel project, on the hobby plan, under one domain.

## Commands

### `/python-on-vercel learn`

Start the guided learning loop. Six lessons across three sections: setup, connecting the apps, and deploying to Vercel.

### `/python-on-vercel new`

Scaffold a Python + Next.js project on Vercel:

1. Clone the starter repo (`vercel-labs/python-on-vercel-academy`)
2. Install Node deps with `npm install` and Python deps via `pyproject.toml`
3. Install and authenticate the Vercel CLI (`vercel login`)
4. Run both apps together with `vercel dev`
5. Verify `/` renders mock data and `/api/items` returns JSON

### `/python-on-vercel submit`

Evaluate the current implementation against the active lesson's outcomes.

## Content source

```
https://vercel.com/academy/python-on-vercel.md           → course overview
https://vercel.com/academy/python-on-vercel/<lesson>.md   → lesson content
```

## Core concepts

### One project, two runtimes

- Next.js 16 at the project root (`app/`, `package.json`, `next.config.ts`)
- FastAPI in `api/index.py` with `app = FastAPI()` at module level
- Python deps in `pyproject.toml` at the root (no `requirements.txt`)
- No `vercel.json` — Vercel auto-detects Next.js at the root and Python in `api/`

### Local development with `vercel dev`

- One command serves both apps under `http://localhost:3000`
- Same-origin requests mean no CORS configuration
- Mirrors the production routing exactly

### Routing into FastAPI

- Vercel routes `/api/*` to `api/index.py`
- FastAPI route patterns must include the `/api` prefix (e.g., `@app.get("/api/items")`)
- Routes at just `/` or `/items` will not match

### Server-side fetch from Next.js

- Server components need an absolute URL — `fetch("/api/items")` only works in the browser
- Build the base URL from `process.env.VERCEL_URL` (auto-injected on every deployment)
- `VERCEL_URL` is hostname-only — prepend `https://` and fall back to `http://localhost:3000` for local dev

### Deploy

- `vercel deploy --prod` ships both halves under one domain
- Hobby plan is sufficient for the entire course

## Progress detection

| Signal | Lesson area |
|---|---|
| Vercel CLI not installed or not authenticated | Setup (1.1) |
| `api/index.py` runs locally with `fastapi dev` | FastAPI tour (1.2) |
| `app/page.tsx` shows mock furniture data in the browser | Next.js tour (1.3) |
| `vercel dev` serves both apps on `localhost:3000` | Connect (2.1) |
| `app/page.tsx` is `async function Home()` and calls `/api/items` | Wired (2.2) |
| Asks about deploy verification or production URLs | Deploy (3.1) |

## Common fixes

- **404 on `/api/items`** — route in `api/index.py` is missing the `/api` prefix. Check for `@app.get("/api/items")`, not `@app.get("/items")`.
- **Server fetch fails in Next.js** — relative URL from a server component. Build an absolute URL from `process.env.VERCEL_URL` with a localhost fallback.
- **Deploy fails with `app` not found** — the FastAPI variable must be named `app` at module level. Vercel looks for `app` at supported entrypoints (`app.py`, `index.py`, `server.py`, plus `src/` and `app/` variants). Renaming it to `api`, `server`, or `application` breaks deploy detection.
- **`fastapi` import error** — Python deps missing from `pyproject.toml`, or wrong Python version. Pin `requires-python = ">=3.12"`.
- **`vercel dev` does not expose `/api/*`** — confirm the command is run from the project root and Python files live in `api/`.

## Teaching guidelines

- Six lessons total, target a 60-minute end-to-end run
- Course is hobby-plan compatible — do not introduce Services or other Pro/Enterprise features into the core flow
- Do not require `vercel.json` — the project structure is the config
- Keep examples tied to the Hazel Home furniture theme
- Flask and Django are mentioned only as a one-paragraph aside in the deploy lesson
