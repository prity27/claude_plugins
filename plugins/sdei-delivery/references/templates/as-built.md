# As-built inventory — <project name>

> Written by `/onboard-existing`. Describes what the code **does today**, with `file:line` evidence.
> It is not a specification: nothing here is agreed scope until a human says so.
> Regenerate rather than hand-edit — a stale inventory is worse than none.

Generated: <YYYY-MM-DD> · Repos covered: <paths> · Commit: <sha>

## Headline

| | Count |
| --- | --- |
| Endpoints | working <n> · partial <n> · dead <n> |
| Persisted entities | <n> |
| Background jobs | <n> |
| External integrations | <n> |
| Screens | <n> |

Estimated completeness against intended scope: <n>% — **estimated by reading code, not measured.**

## Endpoints

| Method + path | Auth enforced | State | Handler | Notes |
| --- | --- | --- | --- | --- |
| `POST /api/jobs` | authenticated + role check | working | `src/routes/jobs.ts:41` | writes Job + Assignment, not transactional |
| `PATCH /api/jobs/:id/close` | authenticated only | partial | `src/routes/jobs.ts:88` | returns 200 without persisting; `TODO` at :94 |
| `DELETE /api/jobs/:id` | none | dead | `src/routes/jobs.ts:120` | not registered on the router |

`State` is `working` (traced end to end, no stub on the path), `partial` (reachable, something on
the path is unimplemented or hardcoded), or `dead` (defined and unreachable).

**Auth enforced** records what the code actually applies at that route, not what the route ought to
have. A blank here is a finding.

## Persisted entities

| Entity | Defined at | Fields | Referenced by |
| --- | --- | --- | --- |
| `Job` | `src/models/Job.ts:12` | 11 | 4 routes, 1 job |

For each, note fields the code declares but nothing reads or writes — they are usually either a
half-built feature or a leftover, and the difference matters.

## Background work and integrations

| What | Trigger | Mutates | Defined at |
| --- | --- | --- | --- |

## Screens

| Route | Calls | State | Defined at |
| --- | --- | --- | --- |

## Reconciliation against scope documents

Fill only when `docs/knowledge/sources/` has material. Each row cites both sides.

### Built and agreed

| What | Code | Document |
| --- | --- | --- |

### Built but never agreed

Each row is a decision for a human: essential-and-undocumented, or scope nobody bought.

| What | Code | Decision | Decided by / on |
| --- | --- | --- | --- |

### Agreed but never built

This is the backlog.

| What | Document | Blocking? |
| --- | --- | --- |

## Findings

Recorded, not fixed. `/write-stories` decides which become work.

| Finding | Evidence | Why it matters |
| --- | --- | --- |
