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

`/reload-plugins` is **not optional.** Skills register the moment they install, but the subagent
registry is snapshotted at session start — without a reload or a restart, the three agents are not
dispatchable and the plugin looks broken.

To try it without installing:

```
claude --plugin-dir /path/to/claude-plugins/plugins/sdei-delivery
```

Nothing else is required — no API key, no configuration. Start with `/project-setup`.

## The chain

Each skill writes an artefact the next one reads. Everything is committed to the project
repository, so a PR reviewer, a new joiner or a later Claude session picks it up cold.

```
/project-setup      →  docs/delivery/PROFILE.md
/ingest-knowledge   →  docs/knowledge/graph.json + sources/ + OPEN-QUESTIONS.md
/onboard-existing   →  docs/delivery/AS-BUILT.md + as-built stories   ← brownfield only, human gate
/write-stories      →  docs/delivery/stories/<epic>.md          ← human gate
/design-schema      →  docs/delivery/SCHEMA.md + mermaid ERD    ← human gate
/write-docs         →  README.md per repo, docs/API.md, docs/ARCHITECTURE.md
/build-module       →  a plan-mode brief per epic, then the slice
/write-tests        →  tests named for the AC ids they prove
/validate-delivery  →  docs/delivery/VALIDATION.md              ← human gate
/deploy             →  a GitHub Actions pipeline, or a PM2 runbook
```

`PROFILE.md` is read first by every skill. Without it a skill stops and says so — it never guesses
the stack, the hosts or the compliance regime.

## The skills

| Skill | Does |
| --- | --- |
| `project-setup` | Captures the profile: experience level, domain, both stacks and databases, working directories, git remotes, deployment, security posture, OWASP baseline, compliance regimes. Detects everything the repos can tell it before asking a human anything. |
| `ingest-knowledge` | Turns discovery calls, chat threads, proposals and SOWs into a sourced knowledge graph of entities, actors, processes, constraints, decisions and non-goals — then closes the gaps by asking. With no material, runs a structured interview and makes that the source. |
| `onboard-existing` | Brownfield entry point. Inventories what the code actually does with `file:line` evidence, derives a graph marked `observed` rather than agreed, back-fills `as-built` stories, and reconciles code against surviving scope documents into built-and-agreed / built-but-never-agreed / agreed-but-never-built. |
| `write-stories` | Epics and stories with Given/When/Then criteria, split backend/frontend/infra, including the baseline epic for infra, migrations, auth scaffolding and CRUD. Ends in per-epic sign-off. |
| `design-schema` | Fields with precise types, indexes justified by real queries, relationships with cardinality, data classification and retention, a transactional analysis, and an ERD. Model code only after sign-off. |
| `write-docs` | A README per repo written from real scripts and real env vars, an API contract, an architecture document. |
| `build-module` | An approved epic becomes a plan-mode brief and then a complete vertical slice — never a loose function. |
| `write-tests` | Backend and frontend tests, each named for the AC it proves, standing up the runner first when there isn't one. |
| `validate-delivery` | Every acceptance criterion mapped to evidence: PASS, FAIL or NOT COVERED. Then a human accepts. |
| `deploy` | GitHub Actions or PM2 over SSH, with pre-flight gates that refuse rather than warn, and a rollback ready before the deploy. |

## The agents

| Agent | Role |
| --- | --- |
| `story-writer` | Drafts one epic from its graph slice. Run one per epic, in parallel. May not invent scope — gaps become questions. |
| `schema-designer` | Proposes entities, types, indexes and cardinalities. Proposes; never decides. |
| `acceptance-validator` | Adversarial auditor. Mode A finds gaps in stories against the graph; Mode B finds acceptance criteria that were never actually delivered. |

## What makes it different from prompting your way through

**Nothing is unsourced.** Every entity in the graph, every story, every field in the schema cites
the document and the quote it came from. When scope is argued about in month three, the citation
settles it.

**Gaps become questions, never defaults.** The agents are explicitly forbidden from filling silence
with something reasonable. An invented requirement is worse than a missing one — the missing one
gets noticed.

**The human gates are interlocks, not ceremony.** `/build-module` refuses an epic that is not
`approved`. `/validate-delivery` refuses to mark a criterion PASS without a `file:line` citation it
actually read. The gates are the product.

**It documents what is, not what should be.** Every skill reports the discrepancies it finds —
a PM2 config pointing at the wrong entry point, a logger disabled in production, a README describing
tooling that was never adopted — rather than working around them silently.

## Using it

Greenfield, in order: `/project-setup` → `/ingest-knowledge` → `/write-stories` → `/design-schema`
→ `/write-docs` → then per epic `/build-module` → `/write-tests` → `/validate-delivery` →
`/deploy`.

On an existing codebase: `/project-setup` → `/onboard-existing` → `/write-docs`, then
`/validate-delivery` against the back-filled `as-built` stories to find out what the code really
proves. Add `/ingest-knowledge` first when the original scope documents still exist — the
reconciliation is only possible when there are two pictures to compare. From there the normal chain
resumes at `/write-stories` for the agreed-but-never-built backlog.

The governing rule of that path: **code is evidence of behaviour, never evidence of intent.**
Anything derived from code is marked `observed` and cannot become a requirement until a human
confirms it, which is what stops a half-abandoned feature from being read as a specification and
faithfully built out.

Pair `/validate-delivery` with `sdei-review`'s `/deep-review`. They answer different questions, and
well-built code implementing the wrong scope still fails acceptance.

## Configuring it for a project

Everything project-specific lives in `docs/delivery/PROFILE.md`, generated by `/project-setup`. The
compliance packs under `references/compliance/` are switched on there per project; OWASP always
applies.

## Testing changes to the agents

`tests/` holds a regression suite: a synthetic project with defects planted on purpose, and a
pre-registered answer key. Run it after editing any agent prompt — a regression shows up as a
specific numbered row, so you can tell which instruction stopped working. Baseline is 30/30.
