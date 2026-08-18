---
name: write-docs
description: Write the project's documentation from what the code actually does — a README per repository scoped to real scripts and real env vars, docs/API.md as the endpoint contract, and docs/ARCHITECTURE.md for the layering and data flow. Use when onboarding a repo, after an epic ships, or when the README no longer matches reality.
argument-hint: "[readme | api | architecture | all] [backend | frontend] — default: all, current repo"
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(ls:*), Bash(cat:*), Bash(grep:*), Bash(find:*), Bash(git log:*), Bash(git ls-files:*), Bash(head:*), Bash(test:*), Bash(npm run:*)
---

# Write docs

Three documents, each written from the repository rather than from intent:

| Document | Answers |
| --- | --- |
| `README.md` (per repo) | how do I run this, and what do I need first |
| `docs/API.md` | what is the contract, exactly |
| `docs/ARCHITECTURE.md` | where does code go, and why |

Read `docs/delivery/PROFILE.md` first — missing means `/project-setup`. Read the approved stories
and `SCHEMA.md` where they exist; they explain the *why* the code cannot.

## The rule that governs all three

**Document what the repository does, not what it should do.** Verify every claim against a file
before writing it, and run every command you document.

This exists because the failure is universal and expensive: a boilerplate README describing a
package manager the project does not use, a Docker setup that was never adopted, a test suite that
does not exist. A new engineer follows it, nothing works, and from then on nobody trusts any
documentation in the repository — including the accurate parts.

So: if `npm test` exits 1, the README says there are no tests. If there is no build step, it does
not describe one. If the docs and the code disagree, the code wins and the disagreement gets
reported.

## 1. README, per repository

Templates: `${CLAUDE_PLUGIN_ROOT}/references/templates/readme-backend.md` and `readme-frontend.md`.

Derive, never assume:

- **scripts** — read `package.json` scripts and say what each really does. A script that prints an
  error and exits is documented as not implemented;
- **prerequisites** — runtime version from `engines` or the toolchain config; the database and its
  version; anything that must exist before the first run;
- **environment** — every variable the code actually reads. Grep for `process.env` (or the stack's
  equivalent) and list what you find with a description and an example value. **Never a real
  secret.** If a variable is required and its absence breaks at boot, say so;
- **first run** — the exact sequence from clone to a working local instance. Run it if you can, and
  say which steps you verified;
- **gates** — the commands CI runs, and what a clean result looks like;
- **layout** — the directory map with one line each, generated from the tree.

Keep it short. A README nobody finishes is a README nobody follows.

## 2. `docs/API.md` — the contract

Template: `${CLAUDE_PLUGIN_ROOT}/references/templates/api-contract.md`. Build it from the routers,
validators and services — the code, not the annotations.

Every endpoint gets: method and path, purpose, **authentication and authorization requirement**,
path and query parameters with types and limits, request body with validation rules copied from the
validator, success response with the real shape and status, error responses with codes and when
each occurs, and the pagination contract for list endpoints.

Then the document-level sections: the base URL per environment, how authentication is presented,
the **response envelope** every endpoint shares, the error envelope and its codes, rate limits and
where they apply, and versioning.

**Where generated API documentation already exists** (OpenAPI, swagger annotations), do not
duplicate it — audit it. Compare the annotations to the routers and report: routes with no
documentation, documented routes that do not exist, and response shapes that disagree with what the
service returns. Drift, listed explicitly, is worth more than a second copy that will also drift.

Every endpoint listed must exist. Grep the router for it before writing the row.

## 3. `docs/ARCHITECTURE.md`

For an existing codebase this is descriptive, and its value is in the counting. Do not write "the
service layer handles business logic" — write how many modules follow the pattern, how many do not,
and name the exceptions. A convention claimed without a count is the kind of statement that makes
architecture documents useless.

Cover: the request lifecycle from entry point to response, with the middleware in order; the layers
and what each may import (state the rule *and* its violations); the module shape, with the file
count that follows it; cross-cutting concerns — auth, validation, error handling, logging, audit,
events, uploads; external dependencies and what happens when each is unavailable; and the
**deliberate deviations**, which belong in the document rather than being rediscovered by every new
reviewer.

Where the project already documents its conventions, link rather than restate. Two copies of a
convention diverge.

## 4. Report

What you wrote, what you verified by running, every place the code and the existing documentation
disagreed, and every claim you could not verify — listed, not quietly omitted.

## Standing rules

- **Never document a command you did not run**, or an env var you did not find in the code.
- Never leave upstream boilerplate in place. Replace it or delete it.
- Never put a real secret, token or connection string in a document. Placeholders only.
- Where you find documentation that is actively wrong, say so prominently. It is doing damage.
