# sdei-tools

A [Claude Code](https://code.claude.com) plugin marketplace with two plugins:

| Plugin | What it does |
| --- | --- |
| **`sdei-review`** | Five code-review specialists run in parallel, then a skeptic tries to refute every finding before you see it. |
| **`sdei-delivery`** | Ten skills carry a project from pre-sales material to production, with a human gate at each point where being wrong is expensive. |

They pair. `sdei-delivery` establishes **what was promised** and checks the code against it;
`sdei-review` judges whether **that code is any good**. Well-built code implementing the wrong
scope still fails acceptance, and correct scope built badly still fails review.

> ### Status: early — v0.2.0
>
> Working and used on real projects, but young. Concretely:
>
> - **Artefact formats may change.** `PROFILE.md`, `graph.json`, `SCHEMA.md` and the story
>   front-matter are stable enough to build on and not frozen. Upgrades may need a re-run of the
>   skill that wrote them.
> - **`sdei-review` is the more proven half.** Its five agents have a regression suite with a
>   pre-registered answer key (baseline 30/30) and have been run against real diffs.
> - **`sdei-delivery` is stack-agnostic by design and exercised mainly on Node.** The detection
>   matrix covers Python, Go, Java/Kotlin, .NET, Ruby, PHP and Rust; those paths are written but
>   lightly travelled. Expect to correct it, and please say where it was wrong.
> - **`/onboard-existing` is the newest skill** and has not yet been run against a large legacy
>   codebase.
>
> Read [Pitfalls](#pitfalls) before your first real run. Most early frustration with these
> plugins is in that list.

---

## Install

From any project, inside Claude Code:

```
/plugin marketplace add prity27/claude_plugins
/plugin install sdei-review@sdei-tools
/plugin install sdei-delivery@sdei-tools
/reload-plugins
```

**`/reload-plugins` is not optional.** Skills register the moment they install, but the subagent
registry is snapshotted at session start. Without a reload — or a restart — the eight agents are
not dispatchable, `/deep-review` fails to fan out, and the plugins look broken.

### Verify it worked

```
/skills
```

You should see `sdei-review:deep-review` plus ten `sdei-delivery:*` skills. If `onboard-existing`
is missing you are on an older copy — see [Updating](#updating).

### Updating

Two different things can be stale, and they need different commands.

```
/plugin marketplace update sdei-tools     # 1. refresh the marketplace from git
/plugin update sdei-review@sdei-tools     # 2. move the installed plugin to the new version
/plugin update sdei-delivery@sdei-tools
/reload-plugins                           # 3. re-register skills and agents in this session
```

- **Step 1** pulls this repository's latest commit into the marketplace clone at
  `~/.claude/plugins/marketplaces/sdei-tools/`. Omit the name to update every marketplace you have.
- **Step 2** moves your *installed* copy to the new version. Installed plugins are cached per
  version at `~/.claude/plugins/cache/sdei-tools/<plugin>/<version>/`, so after a version bump the
  marketplace can be current while your session still runs the old cached copy. `claude plugin
  update` states plainly that a **restart is required to apply** it.
- **Step 3** re-registers skills and agents in the running session.

**If a new skill still does not appear in `/skills`, restart the session.** The subagent registry
and the skill list are snapshotted at session start; a reload does not always pick up a skill that
did not exist when the session began, even though the file is on disk. Check the marketplace clone
directly before concluding the update failed:

```bash
ls ~/.claude/plugins/marketplaces/sdei-tools/plugins/sdei-delivery/skills/
```

The update pulls from the **remote**. If you changed the plugins locally, push first — otherwise the
update fetches nothing and you will wonder why your change had no effect.

### Checking what you have

```
/plugin list                    # installed plugins and their versions
/plugin marketplace list        # configured marketplaces
/plugin details sdei-delivery   # component inventory and projected token cost
```

### Trying it without installing

```
claude --plugin-dir /path/to/claude-plugins/plugins/sdei-review
```

### Removing

```
/plugin uninstall sdei-review@sdei-tools
/plugin marketplace remove sdei-tools
```

Nothing is left behind in your project except the artefacts `sdei-delivery` wrote into `docs/`,
which are plain markdown and JSON and are yours to keep or delete.

## Requirements

Claude Code, and read access to this repository. Nothing else — no API key, no secret, no admin
rights, no configuration. Installs authenticate with your existing git credentials, so private
forks work too.

`sdei-delivery`'s `/deploy` benefits from the `gh` CLI but does not need it; without `gh` it tells
you which steps you must do in the browser instead of pretending it did them.

---

## Quickstart

### Reviewing a change

```
/deep-review                    # the current uncommitted diff
/deep-review feature/billing    # a branch against its merge base
/deep-review src/payments       # a path
/deep-review HEAD~3             # a commit range
```

One specialist at a time, when you only care about one axis:

```
Use the security-reviewer agent on src/api/auth
```

### Starting a project

```
/project-setup                  # always first — writes docs/delivery/PROFILE.md
/ingest-knowledge               # discovery docs → a sourced knowledge graph (or an interview)
/write-stories                  # epics + acceptance criteria       ← you sign off per epic
/design-schema                  # the data model + ERD              ← you sign off
/write-docs                     # README, API, architecture, and CLAUDE.md
/build-module BE-01             # one approved epic as a vertical slice
/write-tests BE-01              # tests named for the criteria they prove
/validate-delivery BE-01        # criteria → evidence               ← you sign off
/deep-review                    # and now: is the code any good
```

`/write-docs` is easy to skip and shouldn't be. It writes the `CLAUDE.md` that `/deep-review`
calibrates against — without it the reviewer has no rulebook and will report your deliberate
decisions as defects.

### Picking up a project already under way

```
/project-setup                  # records stage: partial or mature, honestly
/onboard-existing               # inventory the code, derive a graph, back-fill as-built stories
/write-docs                     # a README, API contract and architecture doc from what is real
/validate-delivery              # find out what the existing code actually proves
```

`/project-setup` and `/write-docs` are worth running alone on any unfamiliar repo. They produce an
honest baseline in about an hour, including the discrepancies nobody had written down.

---

## What `sdei-review` gives you

`/deep-review` resolves the diff once, runs the project's own typecheck/lint/test gates once, fans
out five specialists in parallel, deduplicates, then makes a skeptic try to refute each finding.
Survivors are grouped **Blocking / Should fix / Consider**.

| Agent | Owns |
| --- | --- |
| `security-reviewer` | authorization, identity spoofing, IDOR, injection, mass assignment, secrets, crypto, uploads |
| `correctness-reviewer` | atomicity and partial writes, unchecked results, races, boundaries and timezones, silently dropped writes, error handling |
| `convention-reviewer` | layering, module structure, naming, result shapes, reuse of existing utilities |
| `performance-reviewer` | N+1 queries *and* requests, unbounded result sets, index coverage, over-fetching, algorithmic cost, fan-out |
| `contract-reviewer` | endpoint/schema/validator parity, response drift, breaking changes to published interfaces |

Full usage: [`plugins/sdei-review/README.md`](plugins/sdei-review/README.md).

### Why they work on a codebase they have never seen

Generic review advice is noise. These agents avoid it by **calibrating before judging** — each one
starts by reading the project's own rules (`CLAUDE.md`, `CONTRIBUTING.md`, `docs/`), detecting the
stack from its manifests, and locating the existing patterns. Then:

- **Counting before claiming.** A rule is asserted as "the convention" only after grepping for it.
  If the codebase does something 12 ways versus 10, they report that there *is* no convention
  rather than inventing one.
- **Sibling comparison over memorised rules.** The reference is the neighbouring file, not a style
  guide.
- **Documented decisions win.** If the repo says a trade-off is deliberate, it is not a defect.
- **Diff-scoped.** Pre-existing issues are out of scope unless the change depends on them.
- **Adversarial verification.** Every finding must survive an attempt to refute it.

Observed in practice on a React 19 + Vite frontend: the agents found the project's axios wrapper
convention, identified the existing API helpers the new code should have reused, noticed a token
key casing mismatch (`accessToken` versus the repo's `accesstoken`), and explicitly scoped a
pre-existing committed `.env` *out* as not belonging to the diff.

---

## What `sdei-delivery` gives you

A chain of skills, each writing an artefact the next one reads. Everything lands in the project
repository, so scope, schema and acceptance evidence are diffable, reviewable in a PR, and survive
the session that produced them.

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

Three agents support it: `story-writer` drafts one epic per instance in parallel,
`schema-designer` proposes a data model without deciding it, and `acceptance-validator` audits
adversarially — first for scope the stories missed, later for acceptance criteria the code never
actually delivered.

Three principles run through all of it:

- **Nothing is unsourced.** Every entity, story and schema field cites the document and quote it
  came from. When scope is argued about in month three, the citation settles it.
- **Gaps become questions, never defaults.** An invented requirement is worse than a missing one,
  because the missing one gets noticed.
- **The gates are interlocks, not ceremony.** `/build-module` refuses an unapproved epic.
  `/validate-delivery` refuses to mark a criterion passed without a `file:line` citation it read.

Full usage, per skill: [`plugins/sdei-delivery/README.md`](plugins/sdei-delivery/README.md).

---

## Pitfalls

The things that will cost you an hour if nobody tells you first.

### Installation and reloading

- **A newly added skill may not appear until you restart the session.** `/reload-plugins` can
  report `0 skills` and `/skills` can still show the old list even though the file is on disk in
  `~/.claude/plugins/marketplaces/<name>/`. Check the marketplace clone directly before concluding
  the install failed.
- **Agents never hot-reload.** After editing an agent prompt, `/reload-plugins` or restart, or the
  old version keeps running and your change appears to have done nothing.
- **`/plugin marketplace update` pulls from the remote.** Local commits you have not pushed are not
  in the update.

### Using `sdei-delivery`

- **`/project-setup` is not optional and not a formality.** Every other skill reads
  `PROFILE.md` first and stops if it is missing. Skipping it does not save time; it just moves the
  stop.
- **Rows left `unknown` in `PROFILE.md` are load-bearing.** A later skill will stop and ask rather
  than guess. That is deliberate, and it means an unfilled profile makes the chain feel obstructive.
  Fill it, or accept the interruptions.
- **The chain is long and each gate is a round trip.** Budget for it. On a real domain,
  `/write-stories` can produce 60+ stories and several hundred acceptance criteria; that is the
  honest scope, not a bug, but you approve it epic by epic and that takes real attention.
- **Human gates batch at most four questions at a time.** With more than four epics, they get
  grouped — and it is easy for one epic to fall out of the batch and stay `draft` while you believe
  you approved everything. Check the status line of every epic file after a sign-off round.
- **Changing an approved epic is a scope change.** Adding a story to an epic that is already
  `approved` should send it back to `in-review` with the diff shown. If a skill adds one during a
  build, it must be flagged for sign-off rather than absorbed.
- **A validated `SCHEMA.md` can still drift.** If a build needs an entity the schema does not have,
  record it in `SCHEMA.md` as a post-validation addition. A schema document that silently diverges
  from the models is worse than none.
- **`/build-module` refuses anything that is not `status: approved`** — including `as-built`
  stories, which describe code that already exists and have nothing to build.
- **`/deploy` will not invent a host, a path or a credential.** Where the profile says `unknown` it
  asks. If `gh` is missing, secrets and branch protection are yours to set in the browser.
- **Compliance packs are opt-in per project**, switched on in `PROFILE.md`. OWASP is always on.

### Using `sdei-review`

- **It is diff-scoped on purpose.** Pre-existing problems are not reported unless your change
  depends on them. If you want a whole-codebase audit, point it at paths deliberately.
- **A clean review is not a clean bill of scope.** `/deep-review` never asks whether the code does
  what was promised. That is `/validate-delivery`.
- **Findings are challenged before you see them**, so the list is shorter than a naive review's.
  Shorter is the feature; if you want the unfiltered version, run a single agent directly.

### Both

- **These plugins read your project's own rules and defer to them.** A `CLAUDE.md` that is wrong,
  stale, or aspirational will actively mislead them. Fixing your own docs makes both plugins
  sharply better, and both will tell you where those docs disagree with the code.
- **`CLAUDE.md` is the bridge between the two plugins.** `sdei-delivery` writes it (via
  `/write-docs`) from the project profile; `sdei-review` calibrates against it. If you set a project
  up with the delivery chain and then review it, **run `/write-docs` before `/deep-review`** — the
  reviewer's most valuable input is the profile's known-exceptions list, and it reaches the reviewer
  through `CLAUDE.md`. An existing hand-written `CLAUDE.md` is never overwritten; `/write-docs`
  proposes a diff instead.

---

## Pairing with repository-specific reviewers

These agents are deliberately generic, and generic has a ceiling. A reviewer carrying one repo's
actual file names, line citations, call-site counts and documented exceptions beats a generic one
on that repo every time, because it can assert "this is the convention, in 22 of 24 modules"
instead of deriving it.

So for a codebase you work in daily, use **both**:

- Repo-specific specialists committed to that project's `.claude/agents/`, carrying its real
  evidence and its known-exception list.
- This plugin for everything else — other services, unfamiliar repos, one-off reviews.

They coexist cleanly. Project agents in `.claude/agents/` take precedence over same-named plugin
agents, and the names here are axis-based (`security-reviewer`) rather than prefixed, so they do
not collide with a repo-specific set (`myproject-security-reviewer`).

The fastest way to build the repo-specific version is to copy an agent from `agents/` and replace
its Phase 0 calibration section with the answers — the concrete file paths, counts, and "do not
flag these, they are deliberate" list for that codebase.

## Roadmap

- **Harness engineering.** Feed real project outcomes back into the agent prompts, so the
  instructions that keep failing get replaced rather than re-explained. This is the main planned
  direction.
- **A regression suite for the delivery skills**, matching the one the review agents already have.
- **More deploy packs.** GitHub Actions and PM2-over-SSH are worked examples; container, systemd,
  PaaS and static targets are currently translation guidance rather than templates.
- **Exercising the non-Node detection paths** against real Python, Go and .NET projects.

## Contributing

### Adding a plugin to this marketplace

1. Create `plugins/<name>/.claude-plugin/plugin.json` (only `name` is required).
2. Put `agents/`, `skills/`, `hooks/` at the plugin root — **not** inside `.claude-plugin/`.
3. Add an entry to `.claude-plugin/marketplace.json` with `"source": "./plugins/<name>"`.
4. Validate: `claude plugin validate ./plugins/<name>` and `claude plugin validate .`

### Cutting a release

```bash
# bump "version" in plugins/<name>/.claude-plugin/plugin.json AND the matching
# marketplace.json entry — claude plugin tag fails if the two disagree
claude plugin validate .
claude plugin tag ./plugins/sdei-delivery     # creates sdei-delivery--v<version>
claude plugin tag ./plugins/sdei-review
git push --follow-tags
```

Then add a `CHANGELOG.md` entry saying which artefact formats changed and what to re-run. Consumers
pick the release up with the [Updating](#updating) commands.

### Changing an agent

Run the regression suite in `plugins/sdei-delivery/tests/` afterwards. It is a synthetic project
with defects planted on purpose and a pre-registered answer key, so a regression shows up as a
specific numbered row and you can tell which instruction stopped working. Baseline is 30/30.

Then `/reload-plugins` before testing, because agents do not hot-reload.

### Reporting something wrong

The most useful report names the stack, the skill, and what it assumed that was not true. The
non-Node paths and `/onboard-existing` are where that is most valuable right now.
