# Project profile — <project name>

> Written by `/project-setup`. Every `sdei-delivery` skill reads this file first.
> Detected values were read from the repository; asked values came from a human.
> When a detected value goes stale, re-run `/project-setup` rather than hand-editing.

Last updated: <YYYY-MM-DD> · Updated by: <name>

## Team

| Field | Value |
| --- | --- |
| Experience level | junior / mid / senior / mixed |
| Verbosity contract | see below |
| Domain expertise | none / familiar / expert — in <domain> |
| Domain | e.g. healthcare claims, agricultural logistics, fintech payments |

**Verbosity contract** — how every later skill talks to this team:

- `junior` — explain the reasoning before the instruction, name the concept, link the reference.
  Never hand over a command without saying what it does and what a bad outcome looks like.
- `mid` — state the instruction, add one line of why when the choice is not obvious.
- `senior` — instruction and `file:line` citation only. No tutorial, no restating the stack.
- `mixed` — write for `mid`, and put the `junior` explanation in a collapsed "Why" note.

**Domain expertise** governs how hard the skills push back. `none` means every domain term in the
knowledge graph gets defined and confirmed with a human before a story is written on top of it.
`expert` means domain terms pass through unchallenged and the questions stay technical.

## Backend

| Field | Value |
| --- | --- |
| Working directory | absolute path |
| Git remote | URL |
| Default branch | |
| Branch convention | e.g. `<name>-dev` merged to main via PR |
| Language / runtime | e.g. TypeScript ESM on Node >= 24.18 |
| Framework | e.g. Express 4 |
| Build step | e.g. none — run with tsx / tsc to dist / esbuild |
| Database | e.g. MongoDB 7 via Mongoose 9 |
| Migrations | tool, or **none — schema lives only in the ORM models** |
| Gate commands | e.g. `npm run typecheck` |
| Test runner | e.g. none configured / jest / vitest |
| Entry point | e.g. `src/index.ts` |

## Frontend

| Field | Value |
| --- | --- |
| Working directory | absolute path |
| Git remote | URL |
| Default branch | |
| Language / framework | e.g. TypeScript + React 19 |
| Build tool | e.g. Vite 7 |
| State management | e.g. Redux Toolkit |
| UI system | e.g. Radix primitives + Tailwind 4 |
| Data fetching | e.g. axios wrapper at `src/lib/api.ts` |
| Gate commands | e.g. `npm run lint`, `npm run build` |
| Test runner | e.g. none configured / vitest + testing-library |

## Deployment

| Field | Value |
| --- | --- |
| Pipeline | GitHub Actions / PM2 over SSH / both / manual |
| Environments | e.g. staging, production |
| Backend host | e.g. EC2 `ubuntu@1.2.3.4`, PM2 process `app` |
| Frontend host | e.g. same VM behind nginx / S3 + CloudFront / Vercel |
| Secrets live in | e.g. GitHub Actions secrets, `.env` on the host |
| Rollback | the actual command |
| Health check | the URL that proves the deploy worked |

## Security practices

| Field | Value |
| --- | --- |
| OWASP baseline | **always on** — `references/owasp-checklist.md` |
| Auth mechanism | e.g. JWT access + refresh, `authMiddleware` on every router |
| Authorization model | e.g. RBAC by role name / permission table / policy engine |
| Secret management | how secrets reach the app, and whether `.env` is git-ignored |
| Transport | TLS termination point, and what happens if the cert is missing |
| Audit logging | where the operational record actually lives |
| Known exceptions | deliberate deviations a review must not flag |

## Compliance

Applicable regimes (delete what does not apply):

- [ ] OWASP ASVS + Top 10 — always applies
- [ ] HIPAA / PHI — `references/compliance/hipaa.md`
- [ ] SOC 2 — `references/compliance/soc2.md`
- [ ] GDPR — `references/compliance/gdpr.md`

Notes: <data residency, retention periods, BAAs, anything a story must honour>

## Artefact locations

| Artefact | Path |
| --- | --- |
| This profile | `docs/delivery/PROFILE.md` |
| Knowledge graph | `docs/knowledge/graph.json` |
| Source digests | `docs/knowledge/sources/` |
| Open questions | `docs/knowledge/OPEN-QUESTIONS.md` |
| User stories | `docs/delivery/stories/` |
| Database model | `docs/delivery/SCHEMA.md` |
| API contract | `docs/API.md` |
| Architecture | `docs/ARCHITECTURE.md` |
| Validation matrix | `docs/delivery/VALIDATION.md` |
