# sdei-delivery

Ten skills that carry a project from pre-sales material to production, with a human gate at each
point where being wrong is expensive. Language and framework agnostic: every skill reads its stack,
its gate commands and its deploy mechanism out of the project profile rather than assuming Node.

Where [`sdei-review`](../sdei-review) judges whether code is *good*, this plugin establishes what
was *promised* — and then checks the two against each other.

## Install

```
/plugin marketplace add prity27/claude_plugins
/plugin install sdei-delivery@sdei-tools
/reload-plugins
```

`/reload-plugins` is **not optional.** Skills register on install, but the subagent registry is
snapshotted at session start — without a reload the three agents are not dispatchable and the
plugin looks broken.

Without installing:

```
claude --plugin-dir /path/to/claude-plugins/plugins/sdei-delivery
```

Nothing else is required — no API key, no configuration. Start with `/project-setup`.

## Updating

```
/plugin marketplace update sdei-tools       # refresh the marketplace from git
/plugin update sdei-delivery@sdei-tools     # move the installed copy to the new version
/reload-plugins                             # re-register the ten skills and three agents
```

`/plugin marketplace update` refreshes the clone; `/plugin update` moves your *installed* copy,
which is cached per version at `~/.claude/plugins/cache/sdei-tools/sdei-delivery/<version>/` — so
the marketplace can be current while your session still runs the old one. `claude plugin update`
says a **restart is required to apply** it.

**A brand-new skill may not appear in `/skills` until you restart**, even after a reload, because
the skill list and subagent registry are snapshotted at session start. Verify the file is really
there before concluding the update failed:

```bash
ls ~/.claude/plugins/marketplaces/sdei-tools/plugins/sdei-delivery/skills/
```

```
/plugin list                    # installed plugins and versions
/plugin details sdei-delivery   # component inventory and projected token cost
```

## The chain

Each skill writes an artefact the next one reads. Everything is committed to the project
repository, so a PR reviewer, a new joiner or a later Claude session picks it up cold.

```
/project-setup      →  docs/delivery/PROFILE.md
/ingest-knowledge   →  docs/knowledge/graph.json + sources/ + OPEN-QUESTIONS.md
/onboard-existing   →  docs/delivery/AS-BUILT.md + as-built stories   ← brownfield only, human gate
/write-stories      →  docs/delivery/stories/<epic>.md                ← human gate
/design-schema      →  docs/delivery/SCHEMA.md + mermaid ERD          ← human gate
/write-docs         →  README.md per repo, docs/API.md, docs/ARCHITECTURE.md, CLAUDE.md
/build-module       →  a plan-mode brief per epic, then the slice
/write-tests        →  tests named for the AC ids they prove
/validate-delivery  →  docs/delivery/VALIDATION.md                    ← human gate
/deploy             →  a CI pipeline, or a direct-to-host runbook
```

`PROFILE.md` is read first by every skill. Without it a skill stops and says so — it never guesses
the stack, the hosts or the compliance regime.

## Two entry points

**Greenfield**, in order:

```
/project-setup → /ingest-knowledge → /write-stories → /design-schema → /write-docs
   → then per epic:  /build-module → /write-tests → /validate-delivery → /deploy
```

**A project already under way:**

```
/project-setup → /onboard-existing → /write-docs → /validate-delivery
```

`/project-setup` records the stage honestly — `greenfield`, `scaffolded`, `partial` or `mature` —
and for the last two it produces three lists that are the point of a brownfield profile: what works
end to end, what was started and abandoned, and what contradicts the docs or the config.

Run `/ingest-knowledge` first if the original scope documents still exist; the reconciliation
`/onboard-existing` performs is only possible when there are two pictures to compare. From there
the normal chain resumes at `/write-stories` for the agreed-but-never-built backlog.

The governing rule of the brownfield path: **code is evidence of behaviour, never evidence of
intent.** Anything derived from code is marked `observed` and cannot become a requirement until a
human confirms it. That is what stops a half-abandoned feature from being read as a specification
and faithfully built out.

---

## Skill reference

### `/project-setup [backend path] [frontend path]`

**Reads** the repositories. **Writes** `docs/delivery/PROFILE.md`.

Detects before it asks: identifies the stack family from whichever manifest is actually present
(`package.json`, `pyproject.toml`, `go.mod`, `pom.xml`, `*.csproj`, `Gemfile`, `composer.json`,
`Cargo.toml`), then reads that manifest for gates, build step, database and entry point.

**It runs the gate commands it records.** A gate copied from a manifest is a claim; a gate that was
run is a fact, and the profile says which. It also records contradictions rather than resolving
them silently — a process-manager config pointing at the wrong entry point is exactly what breaks a
deploy at 2am.

Re-run when the stack, hosts, team or deployment change. It asks before overwriting, because a
profile carries human answers that are expensive to recollect.

### `/ingest-knowledge [path to source material | interview]`

**Reads** `docs/knowledge/inbox/` by default. **Writes** `docs/knowledge/graph.json`,
`sources/`, `OPEN-QUESTIONS.md`.

Turns transcripts, chat threads, proposals, SOWs and wireframe notes into a sourced graph of
entities, actors, processes, constraints, decisions and non-goals. Accepts `.txt`, `.md`, `.vtt`,
`.srt`, `.pdf`, images; **cannot read audio or video** — it says so and lists the recording as a
missing source rather than guessing at its contents.

**With no material at all it runs interview mode**, writes your answers to
`docs/knowledge/sources/<date>-interview.md` as a citable digest at medium authority, and builds
the graph from that. The citation trail stays real even when there was never a document.

Confidence is tracked per claim: `stated` (someone said it, quoted), `implied` (follows from what
was said), `observed` (read out of code by `/onboard-existing`), `assumed` (a gap you filled — and
**every assumed item must also appear as an open question**).

### `/onboard-existing [backend | frontend | all] [path]`

**Reads** the code, plus any existing graph. **Writes** `docs/delivery/AS-BUILT.md`, extends
`graph.json`, back-fills `as-built` stories. **Human gate.**

Works outside-in: enumerates the surfaces (endpoints, persisted entities, background work,
integrations, screens, auth enforcement points), then follows each one inward. A route listed from a
routing table is a claim; a route traced to the write it performs is a fact.

Classifies every surface **working** (traced end to end, no stub on the path), **partial**
(reachable, but something is unimplemented or hardcoded) or **dead** (defined and unreachable).

Where scope documents exist, it reconciles into three lists: **built and agreed**, **built but
never agreed** (someone decided this — a human must say whether it was wanted), and **agreed but
never built** (usually the most valuable output of the whole skill).

It does not fix, tidy or delete anything. Not a typo, not a lint error. It records.

### `/write-stories [epic name | baseline | all]`

**Reads** `PROFILE.md`, `graph.json`, `OPEN-QUESTIONS.md`, `AS-BUILT.md`. **Writes**
`docs/delivery/stories/<NN>-<slug>.md`. **Human gate, per epic.**

Shows you the proposed epic list **before** drafting stories — restructuring after forty stories
exist is expensive, arguing about the list is cheap. Epics are cut along user-visible capability,
never along technical layer, then split into `backend` / `frontend` / `infra` / `shared` units.

Generates `00-foundation` first, from the baseline template: infrastructure, migrations, auth
scaffolding, backups, logging, pagination. These are the stories teams skip because they feel like
setup, and then retrofit at roughly ten times the cost. A story is dropped only where the profile
proves it already exists, and the skill says which and why.

Every story carries an actor from a graph `actor`, a `Source:` citation, and acceptance criteria
covering **success, invalid input, unauthorised, and already-in-that-state**. Authorization is
folded into the criteria, never split into a separate security story — the moment security is its
own backlog item it gets deprioritised.

Then `acceptance-validator` audits the set adversarially for coverage gaps, invented scope,
untestable criteria, missing unhappy paths and contradictions, before you ever see it.

### `/design-schema [entity name]`

**Reads** `graph.json` and the approved stories. **Writes** `docs/delivery/SCHEMA.md` with a mermaid
ERD. **Human gate.** Emits model code only after sign-off.

Schema mistakes are the most expensive class of mistake in a project — a wrong service is one file,
a wrong cardinality is a migration, a backfill and a week of reconciliation. That asymmetry is why
the gate is in the middle rather than at the end.

Type precision is enforced: money as a minor-unit integer or decimal and never a float, timestamps
as instants with a stated timezone policy, enums enumerated, free text with a maximum, identifier
formats stated. Indexes are justified by the query that needs them, so an index nothing queries is
removed and a filter with no index is a defect found here rather than in production.

The **transactional analysis** section names, per process, which writes must succeed or fail
together — and a row without a decision is a blocking question for the gate, not a note for later.

The gate is a checklist walked item by item, not "does this look right": completeness, cardinality
read aloud in both directions, required-versus-optional, type precision, uniqueness, delete
behaviour, indexes, classification and retention, atomicity. Every decision is recorded in the
document with its rationale, which is what stops the same argument recurring in month five.

**On an existing database it designs a delta**, never a greenfield model — what changes, what
breaks, what backfills.

### `/write-docs [readme | api | architecture | claude | all] [backend | frontend]`

**Reads** the repository and `PROFILE.md`. **Writes** `README.md` per repo, `docs/API.md`,
`docs/ARCHITECTURE.md`, and `CLAUDE.md`.

Documents what the repository **does**, not what it should do, and verifies every claim against a
file before writing it. If the test command exits non-zero, the README says there are no tests. If
there is no build step, it does not describe one. Where docs and code disagree, the code wins and
the disagreement is reported.

This exists because the failure is universal: a boilerplate README describing tooling the project
never adopted, a new engineer follows it, nothing works, and from then on nobody trusts any
documentation in the repository — including the accurate parts.

**`CLAUDE.md` is the one with a non-human audience.** Every later Claude session reads it, and so
does any reviewer that calibrates against a project's own rules — `sdei-review`'s five specialists
included. It is built from `PROFILE.md` plus verified code, and its highest-value section is
**Known state, deliberately**: the profile's known-exceptions row, copied verbatim, so a reviewer
does not open every run by reporting a deliberate decision as a defect.

Without this, a project set up entirely by this plugin has a rich `PROFILE.md` and **no `CLAUDE.md`
at all**, and the reviewer arrives with no rulebook. So: **run `/write-docs` before `/deep-review`.**

An existing `CLAUDE.md` is **never overwritten** — it usually carries hard-won decisions recorded
nowhere else. Absent, it is generated; present, a diff is proposed for a human to apply; present and
contradicted by the code, that is reported prominently as a finding.

### `/build-module <epic id | story id>`

**Reads** the approved epic, `SCHEMA.md`, the project's conventions, and two or three sibling
modules. **Writes** a plan-mode brief, then the slice.

**Refuses to start** unless `PROFILE.md` exists, the epic is `status: approved`, `SCHEMA.md` is
`validated` where the epic persists data, and no open question blocks an entity the epic touches.
It states which precondition failed and what unblocks it. `as-built` is refused too — that code
already exists.

The brief goes to you before any file is written, and it is specific enough that someone else could
build from it: the file list, the function contracts, the acceptance criteria this slice must
satisfy, the OWASP requirements, the atomicity decisions, the gate commands, and what is explicitly
out of scope.

Then it builds bottom-up, matching the sibling module's file set, export style and error handling
exactly — consistency beats the model's preference every time. It **refuses work smaller than a
slice**: asked for one function, it says so and proposes the slice that function belongs to.

Finally it runs the gates, reports the real output, and maps each acceptance criterion to what
implements it — **naming the ones it did not implement and why.**

### `/write-tests [epic id | path] [backend | frontend]`

**Reads** the approved epic and its criteria. **Writes** tests named for the AC ids they prove.

Each test is named for its criterion — `BE-04-06 AC-2 — cancelling without a reason returns 400` —
which is what makes `/validate-delivery` a mechanical check rather than an opinion.

Establishes whether a runner actually exists first, in whatever form the stack expresses it. Where
there is none, standing one up is a real decision (which runner, how the database is handled,
whether CI runs it) and it **asks before writing config**. It will not scatter tests using a
framework the project has not adopted — an orphan test file nobody can run is worse than none.

### `/validate-delivery [epic id]`

**Reads** the epic, its criteria, and the code. **Writes** `docs/delivery/VALIDATION.md`.
**Human gate.**

Answers one question with evidence: was what was promised actually delivered? Runs the project's
gates and reports verbatim output, then maps **every** acceptance criterion to a verdict — PASS,
FAIL or NOT COVERED — via `acceptance-validator` in adversarial mode.

It **refuses to mark a criterion PASS without a `file:line` citation it actually read.** It notes
explicitly when there are no tests, because "the suite passes" and "there is no suite" are opposite
facts and the difference must never be blurred.

It also validates `as-built` stories, with the caveat stated plainly in the report: criteria derived
from the code they are being checked against prove internal consistency, not that the behaviour was
ever wanted.

### `/deploy [setup | release] [staging | production] [backend | frontend]`

**Reads** `PROFILE.md`'s deployment section. **Writes** a pipeline, or walks a release.

Verifies what actually runs before anything else, because configuration lies. The classic failure
has the same shape in every stack — **the declared start command is not the one a human uses** —
and it reports the discrepancy rather than papering over it.

Pre-flight is a gate that **refuses, not warns**: clean tree, gates passing verbatim, the epic
validated, every environment variable present on the target, a rehearsed migration path, an
identified rollback target, and for production a current backup and someone watching.

Supports a CI path and a direct-to-host path. The runbook has the same five beats whatever the
mechanism — fetch the ref, install deterministically from a lockfile, build if there is a build,
replace the running process, prove it is alive — with translations for PM2, systemd, containers,
PaaS and static hosts.

**It will not run a production deploy autonomously**, and it prepares the rollback before deploying
rather than after something breaks.

---

## The agents

| Agent | Role |
| --- | --- |
| `story-writer` | Drafts one epic from its graph slice. Run one per epic, in parallel. May not invent scope — gaps become questions. |
| `schema-designer` | Proposes entities, types, indexes and cardinalities. **Proposes; never decides.** |
| `acceptance-validator` | Adversarial auditor. Mode A finds gaps in stories against the graph; Mode B finds acceptance criteria that were never actually delivered. |

You rarely call these directly — the skills dispatch them. They are separate agents so that
drafting runs in parallel per epic, and so the auditor's context is clean of the reasoning that
produced the thing it audits.

## Artefact reference

| File | Written by | Status values |
| --- | --- | --- |
| `docs/delivery/PROFILE.md` | `/project-setup` | rows marked *detected* / *asked* / `unknown` |
| `docs/knowledge/graph.json` | `/ingest-knowledge`, `/onboard-existing` | confidence: `stated` / `implied` / `observed` / `assumed` |
| `docs/knowledge/OPEN-QUESTIONS.md` | both of the above | `open` / `answered` |
| `docs/delivery/AS-BUILT.md` | `/onboard-existing` | surfaces: `working` / `partial` / `dead` |
| `docs/delivery/stories/*.md` | `/write-stories`, `/onboard-existing` | `draft` / `in-review` / `approved` / `built` / `validated` / `as-built` |
| `docs/delivery/SCHEMA.md` | `/design-schema` | `proposed` / `validated` |
| `docs/delivery/VALIDATION.md` | `/validate-delivery` | per criterion: PASS / FAIL / NOT COVERED |
| `CLAUDE.md` | `/write-docs` | not status-tracked — generated when absent, proposed as a diff when present |

The `status` fields are the interlocks. `/build-module` reads the story status and `SCHEMA.md`'s
status and refuses on anything but `approved` and `validated`. That refusal is the product.

## What makes it different from prompting your way through

**Nothing is unsourced.** Every entity in the graph, every story, every schema field cites the
document and the quote it came from. When scope is argued about in month three, the citation settles
it.

**Gaps become questions, never defaults.** The agents are explicitly forbidden from filling silence
with something reasonable. An invented requirement is worse than a missing one — the missing one
gets noticed.

**The human gates are interlocks, not ceremony.** `/build-module` refuses an epic that is not
`approved`. `/validate-delivery` refuses to mark a criterion PASS without a citation it read.

**It documents what is, not what should be.** Every skill reports the discrepancies it finds — a
process-manager config pointing at the wrong entry point, a logger disabled in production, a README
describing tooling that was never adopted — rather than working around them silently.

## Pitfalls

- **`/project-setup` is not a formality.** Every other skill reads `PROFILE.md` first and stops if
  it is missing. Skipping it does not save time, it moves the stop.
- **`unknown` rows are load-bearing.** A later skill stops and asks rather than guessing. That is
  deliberate, and it means an unfilled profile makes the chain feel obstructive. Fill it or accept
  the interruptions.
- **The chain is long and every gate is a round trip.** On a real domain `/write-stories` can
  produce 60+ stories and several hundred criteria. That is the honest scope, not a bug — but you
  approve it epic by epic, and that takes genuine attention.
- **Gates batch at most four questions.** With more than four epics they get grouped, and an epic
  can fall out of the batch and stay `draft` while you believe you approved everything. **Check the
  status line of every epic file after a sign-off round.**
- **Changing an approved epic is a scope change.** Adding a story to an `approved` epic should send
  it back to `in-review` with the diff shown. If a build adds one, it must be flagged for sign-off
  rather than absorbed.
- **A validated `SCHEMA.md` can still drift.** If a build needs an entity the schema lacks, record
  it in `SCHEMA.md` as a post-validation addition. A schema document that silently diverges from the
  models is worse than none.
- **Blocked stories are meant to stay blocked.** A story marked blocked on an open question should
  not be built — the alternative is building it twice.
- **`/write-tests` will not adopt a runner for you.** If the project has none, it asks. Expect that
  conversation rather than a pile of test files.
- **`/deploy` invents nothing.** No host, no path, no process name, no credential. Where the profile
  says `unknown`, it asks. Without the `gh` CLI, secrets and branch protection are yours to set in
  the browser — it will tell you which.
- **Multi-document atomicity is a real infrastructure question, not a code style.** The schema step
  surfaces it; expect to make a decision there (for MongoDB specifically, transactions need a
  replica set — a standalone server cannot do them at all).
- **Compliance packs are opt-in per project**, switched on in `PROFILE.md`. OWASP is always on.
- **Non-Node stacks are written but lightly travelled.** The detection matrix covers Python, Go,
  Java/Kotlin, .NET, Ruby, PHP and Rust; expect to correct it, and please report where it was wrong.
- **`/onboard-existing` is the newest skill** and has not yet met a large legacy codebase. Its
  inventory step is the part most likely to need a second pass.

## Configuring it for a project

Everything project-specific lives in `docs/delivery/PROFILE.md`, generated by `/project-setup`. The
compliance packs under `references/compliance/` are switched on there per project; OWASP always
applies.

The profile's **verbosity contract** is not decoration — it is the instruction every other skill
follows when writing for your team. `junior` explains the reasoning before the instruction;
`senior` gives the instruction and a `file:line` and nothing else; `mixed` writes for `mid` and puts
the junior explanation in a collapsed note.

## Testing changes to the agents

`tests/` holds a regression suite: a synthetic project with defects planted on purpose, and a
pre-registered answer key. Run it after editing any agent prompt — a regression shows up as a
specific numbered row, so you can tell which instruction stopped working. Baseline is 30/30.

Agents do not hot-reload. `/reload-plugins` or restart before testing, or you are testing the old
prompt.
