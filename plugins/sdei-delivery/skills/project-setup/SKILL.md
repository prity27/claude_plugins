---
name: project-setup
description: Capture the project profile that every other delivery skill reads — team experience level, domain, backend and frontend stack and database, working directories, git remotes, deployment pipeline, security practices, OWASP baseline and compliance regimes. Works on any language or framework, greenfield or half-built. Use at the start of a project, when onboarding onto an existing one, or when the stack, hosts or team have changed.
argument-hint: "[backend path] [frontend path] — default: the current repo, plus sibling repos it detects"
allowed-tools: Read, Write, Glob, Grep, AskUserQuestion, Bash(git remote:*), Bash(git branch:*), Bash(git rev-parse:*), Bash(git log:*), Bash(git status:*), Bash(git config:*), Bash(git ls-files:*), Bash(npm run:*), Bash(npm test:*), Bash(pnpm run:*), Bash(yarn run:*), Bash(python -m:*), Bash(python3 -m:*), Bash(pytest:*), Bash(ruff:*), Bash(mypy:*), Bash(go build:*), Bash(go test:*), Bash(go vet:*), Bash(mvn -q:*), Bash(gradle -q:*), Bash(./gradlew -q:*), Bash(dotnet build:*), Bash(dotnet test:*), Bash(cargo check:*), Bash(cargo test:*), Bash(bundle exec:*), Bash(composer run:*), Bash(make -n:*), Bash(ls:*), Bash(cat:*), Bash(find:*), Bash(head:*), Bash(test:*), Bash(wc:*)
---

# Project setup

Write `docs/delivery/PROFILE.md` — the file every other `sdei-delivery` skill reads before it does
anything. Get it wrong and nine skills inherit the error, so **detect before you ask, and ask
before you assume**.

The whole discipline of this skill is in that order. A profile full of plausible guesses is worse
than no profile, because the later skills trust it.

## 1. Detect everything the repositories can tell you

Never ask a human for a fact sitting in a file. Run these first, in both the backend and frontend
working directories.

**1a. Universal probe — runs regardless of stack.**

```bash
git remote -v ; git rev-parse --abbrev-ref origin/HEAD ; git log --oneline -5
ls -A                                    # what manifest is actually here
ls .github/workflows/ .gitlab-ci.yml Jenkinsfile 2>/dev/null
ls Dockerfile docker-compose.yml Procfile Makefile 2>/dev/null
ls ecosystem.config.* vercel.json netlify.toml fly.toml render.yaml 2>/dev/null
ls .env.example .env.sample 2>/dev/null   # never read a real .env into context
```

**1b. Identify the stack family from the manifest, then read that manifest.** Do not assume Node
because most projects are — check which of these exists and branch:

| Manifest present | Stack family | Read it for |
| --- | --- | --- |
| `package.json` | Node / JS / TS | `scripts`, deps, `engines`, `"type"`; plus `tsconfig.json` |
| `pyproject.toml`, `requirements.txt`, `Pipfile` | Python | deps, `[tool.*]` gate config, Python version |
| `go.mod` | Go | module path, Go version, `Makefile` targets |
| `pom.xml`, `build.gradle(.kts)` | Java / Kotlin | JDK version, plugins, the run and test tasks |
| `*.csproj`, `*.sln` | .NET | target framework, the run and test commands |
| `Gemfile` | Ruby | Rails or not, Ruby version, `Rakefile` tasks |
| `composer.json` | PHP | Laravel/Symfony or not, `scripts` |
| `Cargo.toml` | Rust | edition, workspace members |
| several of the above | polyglot / monorepo | profile each service separately, and say so |

**1c. Read the signals, don't just record them.** The general form of every row below is *a claim in
config that may not match reality* — the value is in noticing the mismatch.

Universal:

| Signal | What it tells you |
| --- | --- |
| no lockfile beside the manifest | dependency versions are not reproducible; a deploy can drift |
| a CI config that only runs on PRs | nothing gates `main`; merges are unverified |
| `.env.example` listing more vars than the code reads (or fewer) | the environment contract is stale |
| a `Dockerfile` no pipeline references | someone tried containers and stopped; do not treat it as the deploy path |
| a test target that exits non-zero or matches no files | there are no tests, whatever the deps claim |
| a test framework in deps with no config file | a runner was intended and never stood up |

Node specifically (the most common case, kept because the traps are specific):

| Signal | What it tells you |
| --- | --- |
| `"type": "module"` + `allowImportingTsExtensions` | ESM with no build step — imports carry `.ts` |
| `noEmit: true` and no `build` script | **there is no build step**; the deploy skill must not invent one |
| `mongoose` / `prisma` / `pg` / `typeorm` in deps | the database and its access layer |
| no `prisma/` and no `migrations/` | **no migrations** — the ORM models are the only schema |
| `ecosystem.config.json` | PM2 — note the `script:` field and whether it matches the real entry point |
| a `build` script in a `noEmit` project | it is a typecheck under another name and emits nothing |

For any other stack, derive the equivalents rather than skipping the section: what is the build
step, where does the schema live, is there a migration tool, what is the entry point, and which
command does CI actually run.

**Run the gates you are about to record.** A gate command copied out of `package.json` is a
claim; a gate you ran is a fact. Run each one, record the real result and the date, and where you
could not run it say so in the profile — never inherit "the typecheck is clean" from a document.

**Record contradictions, do not silently resolve them.** If `ecosystem.config.json` runs
`src/index.js` while the project starts with `tsx src/index.ts`, that goes in the profile as a
flagged discrepancy — it is exactly the kind of thing that breaks a deploy at 2am.

Find the sibling repo if you were not given it: the frontend is usually a sibling directory of the
backend. Check, then confirm with the user — never profile a repo you guessed at.

## 2. Read the project's own rules

If `CLAUDE.md`, `AGENTS.md`, `CONTRIBUTING.md` or `docs/` exist, read them. They frequently already
state the branch convention, the gate command, the security posture and the deliberate exceptions.
Anything documented there **outranks anything you infer**, and the "known exceptions" row of the
profile is usually copied straight out of them.

Also note plainly what the docs get wrong. An untouched boilerplate `README.md` describing tooling
the project does not use is a finding, not a source.

## 2b. If code already exists — say how far along it really is

Most projects that reach this skill are not empty. A half-built repo profiled as though it were
greenfield produces a profile that reads like a plan, and every later skill then trusts a plan.
Record the **stage**, with evidence:

| Stage | What you must see to claim it |
| --- | --- |
| `greenfield` | no source beyond scaffolding |
| `scaffolded` | frameworks wired, routing or entry point exists, no domain logic yet |
| `partial` | some features work end to end; others are stubs, TODOs or dead routes |
| `mature` | the domain is broadly implemented; work is now change rather than construction |

For `partial` and `mature`, walk the code enough to fill three lists — these are the rows that make
a brownfield profile worth having:

- **Works end to end.** A feature you traced from entry point to persistence. Cite `file:line`.
- **Started and abandoned.** Stub handlers, `TODO`, commented-out routes, a model with no
  reader, a directory nobody imports. Cite them; do not tidy them.
- **Contradicts the docs or the config.** The README's setup step that fails, the env var nothing
  reads, the route the API doc omits.

```bash
grep -rn "TODO\|FIXME\|not implemented\|NotImplemented" --exclude-dir=node_modules . | head -40
git log --oneline -20                     # is this active, or dormant since a burst?
git log --format='%an' | sort | uniq -c | sort -rn | head   # who actually knows this code
```

A repo whose last twelve commits are three months old and all from one author is a different
delivery risk than one with weekly commits from four, and the profile should say which it is.

Finish this section with the honest headline: **what fraction of the intended product exists**, and
say explicitly that it is an estimate from reading code rather than a measured figure.

## 3. Ask only what is genuinely unknowable

Use `AskUserQuestion`, batched, four at a time. These are the fields no file contains:

**Batch A — team.** Experience level (junior / mid / senior / mixed), domain expertise
(none / familiar / expert), the domain itself, and the branch convention if the git history is
ambiguous.

**Batch B — deployment.** Which pipeline is real (GitHub Actions / PM2 over SSH / both / manual),
the hosts and environments, where secrets live, and the rollback command.

**Batch C — security and compliance.** Which compliance regimes apply — OWASP is always on; HIPAA,
SOC 2 and GDPR are per project. Ask where secrets are supposed to live versus where they actually
are. Ask which known deviations are deliberate.

Offer detected values as the recommended option rather than asking open questions: *"Deploy looks
like PM2 on a VM (`ecosystem.config.json`) with no deploy workflow in `.github/`. Correct?"* is a
better question than *"how do you deploy?"*.

## 4. Write the profile

Copy `${CLAUDE_PLUGIN_ROOT}/references/templates/profile.md` to `docs/delivery/PROFILE.md` and fill
every row. Rules:

- Mark each value **detected** or **asked**. A later reader must be able to tell which facts came
  from the code and which from someone's memory.
- Leave a row as `unknown` rather than filling it with something reasonable. `unknown` makes the
  next skill stop and ask; a plausible guess makes it proceed and be wrong.
- Absolute paths for working directories. Later skills are run from either repo.
- The **verbosity contract** section is not decoration — it is the instruction every other skill
  follows when it writes for this team. Fill it from the experience level.
- Record the security posture **as it is**, not as it should be. A committed `.env`, a hardcoded
  certificate path, rate limiting that only applies in production — all of it goes in. The profile
  is the honest baseline the OWASP work is measured against; sanitising it defeats the purpose.

## 5. Report and point at the next step

Summarise in the profile's own verbosity register:

- what was detected, in one table;
- what a human supplied;
- every row left `unknown`, and who can answer it;
- every contradiction found (config versus reality, docs versus code);
- for `partial` or `mature`, the three lists from §2b and the honest completeness estimate;
- the next command, chosen by stage:

| Stage | Next |
| --- | --- |
| `greenfield` / `scaffolded`, pre-sales material exists | `/ingest-knowledge` |
| `greenfield` / `scaffolded`, no material | `/write-stories` — it will interview you |
| `partial` / `mature` | `/onboard-existing` — derives the graph and back-fills stories from the code that is already there |

## Standing rules

- **Ask before overwriting an existing `PROFILE.md`.** Show the diff of what changed; a profile
  carries human answers that are expensive to recollect.
- Never write secrets into the profile. Record *where* a secret lives, never its value — and if you
  find a live credential in a tracked file, say so prominently in the report.
- Do not fix anything you find here. This skill observes; `/write-stories` is where remediation
  becomes work.
