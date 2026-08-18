# <project> — backend

One paragraph: what this service does, for whom, and what it talks to. Not a description of the
framework.

> Every command below has been run. Every environment variable below is read somewhere in `src/`.
> If something here is wrong, it is a bug — fix the README in the same commit as the code.

## Stack

Language and runtime with the **enforced** version · framework · database and driver · auth
mechanism · anything unusual a newcomer would trip over (no build step, ESM-only, explicit import
extensions, a required local service).

## Prerequisites

- Runtime `>= x.y` — enforced by `engines`, and the reason if it is unusual
- Database `x.y`, reachable locally
- Anything else that must exist before the first run

## First run

```bash
git clone <url> && cd <repo>
npm ci
cp .env.example .env      # then fill the values described below
npm run seed              # optional: local data, roles and permissions
npm run dev
```

Then: the health-check URL that proves it worked, and what a successful start prints.

## Scripts

| Command | What it does |
| --- | --- |
| `npm start` | production start — the real command, not the conventional one |
| `npm run dev` | watch mode |
| `npm run typecheck` | the gate |
| `npm test` | **state the truth** — including "not implemented; exits 1" if that is the case |

Do not list a script that does not exist. Do not omit one that does.

## Environment

Every variable the code reads. Required means the process should fail at boot without it.

| Variable | Required | Description | Example |
| --- | --- | --- | --- |
| `PORT` | no | HTTP port | `3000` |
| `MONGODB_URL` | yes | connection string | `mongodb://localhost:27017/app` |
| `JWT_SECRET` | yes | signs access tokens — **rotate on any exposure** | `<generate one>` |

Never a real value. If `.env` is tracked in git, say so here as a known issue with the remediation,
rather than leaving the next person to discover it.

## Layout

```
src/
  index.ts        entry point
  app.ts          middleware chain and route mounting
  api/<module>/   feature modules
  models/         data models
  ...
```

One line each, generated from the tree. Link `docs/ARCHITECTURE.md` for the reasoning.

## Gates

What CI runs, and what clean looks like. If there is only one gate, say that plainly — a reader
should not assume tests run when they do not.

## Deployment

One paragraph and a link to the runbook. Never credentials, never host secrets.

## Known issues

The things that will surprise someone in week one: a config that disagrees with reality, a silent
fallback, a limitation with a ticket. Honesty here saves days.
