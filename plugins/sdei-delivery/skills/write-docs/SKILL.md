---
name: write-docs
description: Write the project's documentation from what the code actually does — a README per repository scoped to real scripts and real env vars, docs/API.md as the endpoint contract, docs/ARCHITECTURE.md for the layering and data flow, and CLAUDE.md as the agent-readable rulebook every later Claude session and code reviewer calibrates against. Use when onboarding a repo, after an epic ships, or when the README no longer matches reality.
argument-hint: "[readme | api | architecture | claude | all] [backend | frontend] — default: all, current repo"
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(ls:*), Bash(cat:*), Bash(grep:*), Bash(find:*), Bash(git log:*), Bash(git ls-files:*), Bash(head:*), Bash(test:*), Bash(wc:*), Bash(npm run:*), Bash(pnpm run:*), Bash(yarn run:*), Bash(make -n:*), Bash(python -m:*), Bash(python3 -m:*), Bash(pytest:*), Bash(go test:*), Bash(go vet:*), Bash(dotnet test:*), Bash(cargo check:*)
---

# Write docs

Four documents, each written from the repository rather than from intent:

| Document | Answers | Audience |
| --- | --- | --- |
| `README.md` (per repo) | how do I run this, and what do I need first | a human, on day one |
| `docs/API.md` | what is the contract, exactly | a client developer |
| `docs/ARCHITECTURE.md` | where does code go, and why | a human, in month three |
| `CLAUDE.md` | what are the rules a change must obey | **an agent, every session** |

`CLAUDE.md` is the one with a second consumer. Every later Claude Code session reads it, and so does
every code reviewer that calibrates against a project's own rules — including `sdei-review`'s five
specialists, which treat a documented project decision as outranking anything they would otherwise
flag. A project set up entirely by this plugin has a rich `docs/delivery/PROFILE.md` and, without
this skill, **no `CLAUDE.md` at all** — so the reviewer arrives with no rulebook and rediscovers,
or wrongly reports, decisions that were already made and written down.

Read `docs/delivery/PROFILE.md` first — missing means `/project-setup`. Read the approved stories
and `SCHEMA.md` where they exist; they explain the *why* the code cannot.

## The rule that governs all four

**Document what the repository does, not what it should do.** Verify every claim against a file
before writing it, and run every command you document.

This exists because the failure is universal and expensive: a boilerplate README describing a
package manager the project does not use, a Docker setup that was never adopted, a test suite that
does not exist. A new engineer follows it, nothing works, and from then on nobody trusts any
documentation in the repository — including the accurate parts.

So: if the project's test command exits non-zero or matches no files, the README says there are no
tests. If there is no build step, it does not describe one. If the docs and the code disagree, the
code wins and the disagreement gets reported.

## 1. README, per repository

Templates: `${CLAUDE_PLUGIN_ROOT}/references/templates/readme-backend.md` and `readme-frontend.md`.

Derive, never assume:

- **commands** — read the task definitions the stack uses (`package.json` scripts, `Makefile`
  targets, `pyproject.toml` tool entries, `Rakefile` tasks, gradle tasks) and say what each really
  does. A task that prints an error and exits is documented as not implemented;
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

## 4. `CLAUDE.md` — the rulebook an agent reads

Template: `${CLAUDE_PLUGIN_ROOT}/references/templates/claude-md.md`.

This is not a summary of the other three documents. It is the **short, operational** set of rules a
change must obey, written so that an agent reading only this file makes the same choices a
long-standing team member would.

Build it from `docs/delivery/PROFILE.md` plus what you verified in the code:

| Section | Source | Why it earns its place |
| --- | --- | --- |
| Layout | the directory tree | so a new file goes in the right place |
| Commands | the profile's **gate commands** row, run | so nobody invents a script that does not exist |
| Conventions | the patterns you counted, with their exception counts | the reference is the neighbouring file |
| Delivery process | the artefact table and the gate statuses | so an agent does not build an unapproved epic |
| **Known state, deliberately** | the profile's **known exceptions** row | **the do-not-flag list** |

The last row is the highest-value line in the file and the easiest to leave out. A reviewer that
knows "the test script exits 1 on purpose, there is no runner yet" reports nothing; one that does
not opens with it as a defect, every single run. Copy the profile's known-exceptions row verbatim
and say plainly that these must not be reported as defects.

Keep it to roughly a page. `CLAUDE.md` is read on every session, so length is a running cost, and a
rule buried on page four is a rule nobody follows.

### Never clobber a hand-written `CLAUDE.md`

This file is usually human-authored and often carries hard-won decisions that exist nowhere else.

- **Absent** → generate it, and say what you generated it from.
- **Present** → do **not** overwrite. Read it, then propose a diff: what the code contradicts, what
  the profile records that the file omits, and what the file claims that you could not verify. The
  human applies it.
- **Present and wrong** → say so prominently. A `CLAUDE.md` stating a convention the code abandoned
  is worse than silence, because both agents and humans trust it. That is a finding, not a
  formatting problem.

Never write a delivery decision into `CLAUDE.md` that is not already recorded in `PROFILE.md`, a
story, or `SCHEMA.md`. This file *reflects* decisions; it is not where they are taken.

## 5. Report

What you wrote, what you verified by running, every place the code and the existing documentation
disagreed, and every claim you could not verify — listed, not quietly omitted.

For `CLAUDE.md` specifically, say which it was: generated fresh, or a diff proposed against an
existing file. If you proposed a diff, the human has to apply it — do not report the rule as
documented until they have.

## Standing rules

- **Never document a command you did not run**, or an env var you did not find in the code.
- Never leave upstream boilerplate in place. Replace it or delete it.
- Never put a real secret, token or connection string in a document. Placeholders only.
- Where you find documentation that is actively wrong, say so prominently. It is doing damage.
- **Never overwrite an existing `CLAUDE.md`.** Propose the diff and let a human apply it.
