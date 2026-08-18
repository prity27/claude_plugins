# <project> — frontend

One paragraph: what this application is, who uses it, and which backend it talks to.

## Stack

Framework and version · build tool · language · state management · UI and styling system · data
fetching · routing. Name the version of anything with a recent breaking major.

## Prerequisites

- Runtime `>= x.y`
- A running backend, or the mock — say which and how to point at it

## First run

```bash
npm ci
cp .env.example .env      # VITE_API_BASE_URL, etc.
npm run dev
```

The URL it serves on, and what to log in with locally.

## Scripts

| Command | What it does |
| --- | --- |
| `npm run dev` | dev server with HMR |
| `npm run build` | production build to `dist/` |
| `npm run preview` | serve the built output |
| `npm run lint` | the gate |
| `npm test` | **state the truth** |

If a `server.js` or similar exists for staging, document what it is for — an undocumented server
file gets deleted by someone eventually.

## Environment

Build-time variables and their prefix rule.

| Variable | Required | Description | Example |
| --- | --- | --- | --- |
| `VITE_API_BASE_URL` | yes | backend origin | `http://localhost:3000` |

**Anything in a client bundle is public.** Never put a secret here, and say so.

## Layout

```
src/
  components/   shared UI
  features/     feature folders
  lib/          the API client and shared utilities
  routes/       route definitions
  store/        state
```

## Conventions worth knowing on day one

- **All HTTP goes through the shared API client** at `src/lib/…` — never a bare `fetch`. It carries
  the base URL, the auth header, refresh-on-401 and the error mapping.
- Where auth tokens are stored, and the exact key name — a casing mismatch here is a classic
  half-day bug.
- The form and validation approach.
- The component library rules: what is generated, what must not be hand-edited.

## Gates

The commands CI runs. Where the build is the only gate, say so.

## Deployment

Where the build output goes, who serves it, and how the API origin differs per environment.
