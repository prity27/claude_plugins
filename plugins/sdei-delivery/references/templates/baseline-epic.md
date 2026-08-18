# Baseline epic `00-foundation`

Generated for every project before any feature epic. These are the stories teams skip because they
feel like setup rather than scope — and then retrofit at ten times the cost, usually during the
first security review or the first deploy that goes wrong.

Adapt each story to the profile's stack. Drop a story only when the profile shows it is genuinely
already done, and say which and why. **Never drop one because it looks boring.**

---

## Infrastructure

**INF-01 — Runnable from a clean clone.** A new engineer clones, follows the README, and has the
app running. AC: documented commands only; every required env var listed with a description and an
example; the app fails at startup with a clear message when one is missing, rather than at the
first request.

**INF-02 — Environment separation.** Local, staging and production configuration are separate and
none of them is a code change to switch. AC: no environment-specific values compiled into source;
production config is loaded and validated at boot.

**INF-03 — Secrets are not in the repository.** AC: `git ls-files` returns no `.env` or key
material; `.gitignore` covers them; every secret has a documented home; **any credential already
committed is rotated, not just deleted** — git history keeps it otherwise.

**INF-04 — Gates run in CI.** AC: typecheck, lint and tests run on every pull request; a failing
gate blocks merge; the commands are the same ones a developer runs locally.

**INF-05 — Health check.** AC: an unauthenticated endpoint reports liveness and dependency
readiness (database reachable), and the deploy pipeline calls it.

**INF-06 — Logging that works in production.** AC: a structured logger active in every environment,
not gated on development; request id correlation; no credentials or PII in output.

**INF-07 — Error handling and the response envelope.** AC: one error shape across every endpoint;
unhandled errors return a generic message plus a correlation id and never a stack trace; the shape
is documented in the API contract.

## Migrations and data

**MIG-01 — Schema changes are versioned and repeatable.** AC: a documented mechanism to move a
database from any prior version to current; every change is a reviewable file. *If the profile says
there are no migrations and the ORM models are the only schema, this story is where that decision
gets made explicitly — with a human — rather than by default.*

**MIG-02 — Rollback for the last change.** AC: the previous version can be restored, and it has
been tested at least once against a copy of real-shaped data.

**MIG-03 — Seed data.** AC: a repeatable seed producing a usable local dataset, including the roles
and permissions the auth stories need; safe to run twice; never runnable against production.

**MIG-04 — Backups and restore.** AC: backup schedule and retention documented; **a restore has
actually been performed once** — an untested backup is not a backup.

**MIG-05 — Indexes match the queries.** AC: every filter and sort field used by an approved story
is covered; verified against `/design-schema`'s index list.

## Authentication scaffolding

**AUTH-01 — Registration or provisioning.** AC: the real account-creation path for this product;
password stored with a modern KDF; duplicate identity rejected without leaking whether the account
exists.

**AUTH-02 — Login issues credentials.** AC: valid credentials return an access token and a refresh
token with stated lifetimes; invalid credentials return one generic failure regardless of cause;
attempts are rate-limited and logged.

**AUTH-03 — Refresh and logout.** AC: refresh issues a new access token; logout invalidates the
refresh token; a token invalidated by logout or password change cannot be reused.

**AUTH-04 — Password reset.** AC: single-use, time-limited token delivered out of band; the request
endpoint reveals nothing about whether the address exists; reset invalidates all existing sessions.

**AUTH-05 — Route protection by default.** AC: every route requires authentication unless it
explicitly opts out; the opt-out list is short, reviewed and documented.

**AUTH-06 — One authorization model.** AC: roles and permissions defined once, enforced by one
mechanism; a route's requirement is declared at the route; adding a route without a declaration
fails review. *Where a project already has several competing mechanisms, this story is the decision
to converge — put it to a human before writing it.*

**AUTH-07 — Object-level authorization.** AC: fetching, updating or deleting by id verifies the
caller may act on **that record**, not merely that their role may use the route. Tested with a
second user's id on every owned resource.

**AUTH-08 — Audit trail for sensitive actions.** AC: authentication events, authorization failures,
privilege changes and destructive operations recorded with actor, target, action and timestamp.

## Basic CRUD

One slice per core entity from the graph, each following the same shape:

**CRUD-\<entity\>-01 Create** — validated input, only whitelisted fields accepted (**no mass
assignment**), authorization enforced, returns the created record in the standard envelope.

**CRUD-\<entity\>-02 Read one** — 404 for missing, 403 for not-permitted, and the two are
distinguishable only to someone entitled to know.

**CRUD-\<entity\>-03 List** — paginated with an enforced maximum page size, filterable on the
fields stories actually need, sorted deterministically (ties broken by id, so pages don't shuffle),
scoped to what the caller may see.

**CRUD-\<entity\>-04 Update** — partial update, whitelisted fields, optimistic-concurrency or
last-write-wins stated explicitly rather than left to chance.

**CRUD-\<entity\>-05 Delete** — soft or hard delete decided per entity and recorded in the schema;
referential consequences defined; authorization and audit enforced.

## Frontend foundation

**FE-01 — App shell and routing**, with protected and public route groups.
**FE-02 — One API client**: base URL, auth header, token refresh on 401, and one error-to-message
mapping. Every feature uses it; nothing calls `fetch` directly.
**FE-03 — Auth flows**: login, logout, refresh, password reset, and what a user sees when their
session expires mid-action.
**FE-04 — Loading, empty and error states** as first-class states of every data view, not an
afterthought.
**FE-05 — Form validation** mirroring the server's rules, with server-side errors surfaced against
the right field.
**FE-06 — Accessibility baseline**: keyboard reachable, labelled controls, visible focus, contrast.
