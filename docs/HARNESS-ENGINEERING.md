# Harness engineering programme

> Status: **draft, not yet accepted.** Written 2026-08-21 against `sdei-review` 0.2.0 and
> `sdei-delivery` 0.2.0. Every factual claim below was verified during planning; the ones that
> were not are marked **UNVERIFIED** and say what would settle them.

## What this is

A systematic way to make these plugins measurably better from the failures they actually hit,
by changing the harness — tool grants, subagent topology, gates, scope resolution, calibration
inputs, CI wiring, provenance — instead of rewriting prompts. It is the roadmap item at
`README.md:335-337` made concrete:

> "Feed real project outcomes back into the agent prompts, so the instructions that keep
> failing get replaced rather than re-explained."

The programme's entire claim is the routing table in §2: a symptom maps to a **layer**, and in
six of eight classes that layer is not a prompt. If a failure report ends in "add a sentence to
the agent," the programme has failed.

---

## 1. Why now: eight verified defects

Not speculative. Each was confirmed by reading code, logs or the GitHub API.

1. **The pilot reviewer burns money and produces no reviews, reporting success.**
   `sdeitech/HCTS_Backend` PR #116 attempt 1: SDK result `subtype: success`,
   `is_error: false`, `permission_denials_count: 0`, `total_cost_usd: 0.9436776`. It gathered
   context, found the skill, ran the gate — then ended its turn at
   `- [ ] Fan out specialists`. PR #112 did the same at 2m 42s. Nothing was denied. It treated
   the tag-mode progress checklist as its deliverable and stopped one step before the review.
2. **The action silently skips itself.** PR #116 attempt 2: `Skipping action due to workflow
   validation: the workflow file must … have identical content to the version on the
   repository's default branch. Error is not retryable, giving up immediately.` The step still
   reported `success`. **Any PR that edits the reviewer disables the reviewer for that PR.**
3. **No `skeptic` agent exists.** `plugins/sdei-review/agents/` holds exactly five files.
   `skills/deep-review/SKILL.md:91` dispatches "a **skeptic**", `README.md:4` advertises one,
   and `SKILL.md:13` says *"The verification step is the point."* The most important subagent
   in the plugin has no file, no model pin, no version and no test — it is retyped as prose on
   every run. This is the programme's thesis in miniature.
4. **A skill that cannot execute its own step.** `SKILL.md:56` says run the project's
   typecheck/lint/test gates; `SKILL.md:5` grants only `Bash(git …:*)`.
5. **The grant topology is inverted.** All five specialists carry `tools: Read, Grep, Glob,
   Bash` — unrestricted (`agents/*.md:4`) — while the orchestrator instructed to run gates
   centrally is git-only. The five told to "skip anything a compiler would catch" are the only
   ones that *can* run the compiler.
6. **Scope resolution assumes an interactive checkout.** `hcts-review/SKILL.md:22` uses
   `git diff --stat main...HEAD`; `actions/checkout` leaves HEAD detached on the PR merge ref
   with no local branch. `deep-review/SKILL.md:22` derives `origin/HEAD`, which is better and
   still assumes a resolvable local ref. Both CI prompts spend ~10 lines undoing this.
7. **A live privilege-escalation path.** `claude-code-action` restores `.claude/`, `CLAUDE.md`
   and `.mcp.json` from the base branch but **not `package.json`**. Any granted
   `Bash(npm run <script>)` therefore executes a PR-branch script body. HCTS grants
   `Bash(npm run typecheck)` (`.github/workflows/claude-code-review.yml:135`), and
   `prity27/claude_field_ops` ships a root `"gate": "npm run gate --workspace=server && npm run
   gate --workspace=client"` — so the tension has a file and a line, not just a theory. Threat
   model is a **collaborator**, not an outsider (fork PRs receive no secrets), but a
   collaborator gains arbitrary execution in a job holding `id-token: write` plus the auth
   token, without touching the one file that *is* validated. One-line fix: grant the
   toolchain binary (`Bash(npx tsc:*)`), never `npm run`.
8. **Documented false assurance.** `README.md:21-22` claims the five review agents "have a
   regression suite with a pre-registered answer key (baseline 30/30)". They have none — that
   baseline belongs to the three *delivery* agents in `plugins/sdei-delivery/tests/`.
   `plugins/sdei-review/README.md:172-174` repeats the error. **The false claim is what stops
   anyone looking.**

### Drift is observed, not predicted

`docs/ci-review-template.yml` is 256 lines meant to be copied per repo. There are now three
copies and no two agree: the template at 256, `HCTS_Backend` at 136, and
`claude_field_ops` at 87. They diverge in the prompt, the setup block and `--allowedTools`.

---

## 2. The failure taxonomy

A report resolves to **exactly one** class, and the class **names the file that must change**.
The rightmost column is the part that matters: every class has a tempting wrong layer, and in
every verified case so far the tempting wrong layer is prose.

| # | Class | Verified example | Detection signal | Fix belongs in | Anti-fix |
| --- | --- | --- | --- | --- | --- |
| **A** | Capability–instruction mismatch — told to do what its grant cannot express | defects 4, 5 | Static lint: skill verbs vs `allowed-tools`. Runtime: `tool_used` grader; `friction.denied` | **Tool grant**, generated from stack profiles; **subagent topology** | "If you cannot run the gate, say so" — converts a fixable grant bug into a permanent capability regression the model honours forever |
| **B** | Environment assumption — true interactively, false in CI or a detached checkout | defect 6 | **Any override paragraph in a CI prompt is a bug report against a skill.** Both our CI prompts contain one. Lint: grep CI prompts for `git diff` | **Skill step** — an environment-derived fallback chain | Keeping the override in the CI prompt. It works, which is why it survived twice — and guarantees every new consumer re-derives it |
| **C** | Unmeasured surface + false assurance | defect 8 | Coverage inventory: `claude plugin details` components joined against case dirs. Free | **Eval programme** *and* the doc correction, since the false claim suppresses the signal | Writing more agent prose. You cannot improve what you cannot score |
| **D** | Stack blind spot — a path that exists in prose and has never executed | only fixture is MERN; Angular appears solely in commented-out YAML (`ci-review-template.yml:82-91`) | Profile × fixture matrix lint; run records grouped by detected stack | **Stack profile data** + one fixture per profile | A bigger detection matrix in the prompt. Detection is not the failure; acting correctly after it is |
| **E** | Resolution & provenance — you cannot tell which copy ran | project `.claude/agents/` shadows same-named plugin agents (`HCTS_Backend/plugins/sync-from-project.sh:10-14`); zero tags means no version to attribute to | `aggregate-result.json` → `suite.plugins` lists the plugin with no blocking `problem` | **Eval invocation discipline + release tags** | "Run it from a scratch directory" — that is `tests/README.md:30` today, and it is advice, not a mechanism |
| **F** | Unobservable run | neither review skill emits machine-readable output; the whole delivery-suite baseline is one sentence at `tests/README.md:77` | Absence is the signal | **Instrumentation** (§3) and the consumers built on it | Asking developers to paste reports somewhere. Every manual capture step is where data stops |
| **G** | Calibration starvation — judged against generic rules because the repo's rulebook is missing or wrong | `claude_field_ops/docs/delivery/PROFILE.md:87` says `CI \| **none** — no .github/workflows/` while that workflow exists. `SKILL.md:44-52` calls the do-not-flag list the thing that "removes most false positives" but can require no artifact | Precision decoy cases; reports naming a documented decision | **Skill step** (make the do-not-flag list a required input) + profile `false_positive_traps` + `/write-docs` | More hedging in the prompt. Hedging trades recall for precision globally; a do-not-flag list trades neither |
| **H** | Topology & cost — the subagent graph and model routing are prose with no artifact behind them | defect 3; `SKILL.md:98` says route models per axis but never names the mechanism; `ci-review-template.yml:41-44` concedes monthly cost is unknown | `tool_used` on `Agent` with `min`/`max`; `tool_order`; `costUsd` | **Subagent topology** (`agents/skeptic.md`) + explicit `model:` at the dispatch site | "Be efficient." Cost is a topology property; prose cannot bound how many agents spawn on how much context |

If triage cannot name a file, the report is `needs-repro` — or the taxonomy needs a ninth
class, added deliberately with a fix location, **never** dumped into G.

---

## 3. Phase 0 — Truth, tags, and a free gate

Days, near-zero API spend. The smallest slice that produces real signal, and it is what makes
every later phase measurable.

1. **Cut the tags first.** `claude plugin tag ./plugins/sdei-review --push`, same for
   `sdei-delivery`. Verified: a dry run creates `sdei-review--v0.2.0` and cross-checks
   `plugin.json` against the `marketplace.json` entry. Pinning, rings, "fixed in v0.3.1" and
   report attribution all depend on a version existing. **`git tag -l` is empty today.**
2. **Fix the false-assurance lines** (`README.md:21-22`,
   `plugins/sdei-review/README.md:172-174`). Class C stays invisible until this is true.
3. **Fix scope resolution at the source** (class B), retiring both CI override paragraphs:
   ```bash
   base="${GITHUB_BASE_REF:+origin/$GITHUB_BASE_REF}"
   base="${base:-$(git rev-parse --abbrev-ref --symbolic-full-name origin/HEAD 2>/dev/null)}"
   base="${base:-$(git rev-parse --verify -q origin/main >/dev/null && echo origin/main)}"
   base="${base:-$(git rev-parse --verify -q origin/master >/dev/null && echo origin/master)}"
   git diff --stat "$(git merge-base "$base" HEAD)"..HEAD
   ```
   Works detached, with no remote, and with a non-`main` default. Needs only
   `Bash(git merge-base:*)`, already granted at `SKILL.md:5`.
4. **Fix the grant defects** (class A + defect 7). Add `Bash(npx tsc:*)`, `Bash(npx ng lint)`,
   `Bash(dotnet build:*)`, `Bash(dotnet format:*)` to `deep-review/SKILL.md:5` — deliberately
   **not** `Bash(npm run:*)`. Replace `Bash(npm run typecheck)` with `Bash(npx tsc:*)` in the
   consumer workflows for the same reason.
5. **Narrow the five specialists** from unrestricted `Bash` to `Read, Grep, Glob`. Fixes the
   inverted topology and removes five duplicate-gate cost paths.
6. **Add `agents/skeptic.md`** (plus `skeptic-deep.md`, opus, for security and correctness
   claims), lifting the refutation prompt inline at `SKILL.md:94-97` into a real file with a
   pinned model — and name `model:` explicitly at the dispatch site, since `SKILL.md:98`
   describes model routing without naming any mechanism.
7. **Make the reviewer actually review.** In each consumer workflow: `timeout-minutes: 25` (the
   job currently inherits the 360-minute default and the SDK is documented to hang on
   `pull_request` runs); a fork guard; a `PROCESS` rewrite making the fan-out the deliverable
   and stating that a progress checklist is not a review; and the workflow-validation gate
   documented in the header so the first PR of every future workflow change is expected to be
   skipped.
8. **`.github/workflows/harness-lint.yml` — no model, no API key, seconds, every PR.** The
   highest value-per-hour artifact in the programme. It checks: `claude plugin validate` on the
   marketplace and each plugin; `claude plugin tag --dry-run` (catches a version disagreement
   at PR time, not release time); generated grants match frontmatter; every profile has ≥1
   tagged case; every gate has a parseable grant; no commented-out `uses:`/`run:` in workflows;
   no `.claude/agents|skills` under eval fixtures; no `git diff <literal-branch>...HEAD` in any
   skill (class B regression guard); `CHANGELOG.md` touched when a version changes; and
   case-before-cure (§7).
9. **Seed `docs/harness/PRESSURE.md`** by hand from the eight defects above. Five minutes, and
   it turns the next report from a Slack message into a row.
10. **Get `.claude_secret` out of the working tree** of a public-by-intent repo. Gitignored is
    not absent, and one `git add -f` ends it.

---

## 4. Phase 1 — Instrumentation: make every run legible

Nothing downstream is measurable until a run leaves a record.

### 4.1 Emit via `--json-schema`, not a comment footer

Add `--json-schema '{…}'` to `claude_args`; read `steps.review.outputs.structured_output`.
Prose reaches humans through the comment tool, JSON reaches machines through the SDK's final
result. Separate rails, so a malformed record cannot damage the review.

Two findings decide this:

- **An HTML-comment footer is impossible.** The comment tool sanitises every body it writes and
  `stripHtmlComments` deletes `<!-- … -->` outright. A JSON footer vanishes **silently** — the
  worst available failure mode. Ruled out.
- **`--json-schema` requires `continue-on-error: true` on the step.** With no schema-valid
  output the action calls `setFailed` and throws. Without that line a bad record fails the job
  and red-flags the PR even though the prose review already posted.

Encoding rules forced by the `claude_args` parser: **one physical line, single-quoted, no
apostrophes anywhere** (including in `description` text), no `#`. Command substitution is
escaped rather than executed, so `--json-schema "$(cat schema.json)"` cannot work. Every
`description` is billed as input tokens on every run.

Documented fallback for consumers who cannot edit `claude_args`: a `<details>` + fenced-JSON
block, which survives sanitising. Rejected: a narrow `Write` grant (trades the read-only
posture for a capability we already have) and a second Claude step structuring the prose (a
full extra model call, guessing at facts the first agent already held).

### 4.2 Three sources, one record

Never ask the model for a fact CI holds for free — every field the model fills is a field it
can get wrong.

| Section | Filled by | Carries |
| --- | --- | --- |
| `run` | the workflow, from the `github` context | repo, PR, base/head sha, run id and attempt, run URL, comment URL, **`plugin@version`** |
| `review` | the model, via `--json-schema` | `status`, `review_path`, scope + diff command, detected stack/profile, gates attempted with outcome (`pass`/`fail`/`unavailable`/`denied`/`skipped`), per-axis outcome with a **reason** for `skipped`, findings (severity, axis, file, line, category, claim, `confirmed`/`unverified`), drop counts, **`friction`** |
| `harness` | the workflow, from `execution_file` | turns, duration, cost, and the **observed** `permission_denials` |

`verdict` (`blocking`/`findings`/`clean`/`incomplete`) and `record_status`
(`ok`/`absent`/`malformed`) are **derived by CI, never by the model** — the routing key must not
be model-controlled, or a confused reviewer can silence its own alarm.

`friction` is `required` in the schema, so the SDK rejects a record without it. It carries tools
refused, gates unavailable, scope trouble, context searched for and not found, and places the
skill was silent or wrong for this repo. **Its fields are deliberately identical to the
failure-report issue form (§7.3)**, so triage needs one parser — and `friction` is a
machine-authored answer to "what did it assume that was not true."

The quiet win: print **self-reported `friction.denied` beside observed
`harness.permission_denials`** in the job summary, every run. `0 self / 4 observed` means the
self-report is unreliable, which is itself a finding about the harness.

### 4.3 Where the text goes — forced by the base-branch restore

| Location | Takes effect | Carries |
| --- | --- | --- |
| the workflow `prompt:` | on the PR that adds it | the record **contract** — that a record is required and goes in the final message, never the comment |
| a consumer's `.claude/skills/**` | only **after merge to the default branch** (restored from base) | destination-agnostic **discipline** — keep a friction log from step 1, tally per axis |
| `plugins/sdei-review/skills/deep-review/SKILL.md` | next CI run in **every** consumer, no PR needed | the same, generically worded |

**Rule: transport in the workflow, discipline in the skills.** A skill that says "put JSON in
your final message" is wrong the moment someone runs it interactively — there is no consumer for
that message locally.

**Corollary: a `SKILL.md` change cannot be tested on the PR that makes it.** The only working
sequence is workflow first (self-testing), then merge the skill, then a throwaway PR.

Add one line to each specialist: end your report with `FRICTION:` naming any refused tool or
missing file, or `FRICTION: none`. Subagents run against the session allowlist, so a specialist
reaching for `rg` is denied and the orchestrator never hears about it.

### 4.4 Where records accumulate

A **ledger issue** in the consumer repo: one long-lived issue, one comment per run, a one-line
human summary plus the record in a fenced block. The only durable option with **zero owner
dependency** — Issues are enabled, the workflow already holds `issues: write`, and
`GITHUB_TOKEN` needs no secret. Two minutes to set up. Pair with `upload-artifact` for the
record plus the raw transcript, 90 days, for debugging.

The alternatives are blocked or unfit: a collector repo needs a cross-repo PAT (a secret);
Discussions are disabled on HCTS and enabling them is an admin setting; Slack only ever sees
`blocking`/`incomplete`, so it has no denominator.

**Warn in the template:** the raw `execution_file` is unsanitised and holds the full diff and
every tool result. Acceptable in a private repo; **must be dropped** if the template is copied
into a public one — which now includes `claude_field_ops`.

Limits, stated up front: no aggregation UI (every trend is a `jq` script); rotate the issue
yearly; comments are editable, so it is a log and not an audit trail; single-repo until a PAT
exists.

---

## 5. Phase 2 — Tell developers, proportionally

Severity-routed from `verdict`. The asymmetry is the design: `blocking` is loud at the
**author**, `incomplete` is loud at the **harness owner**, and `findings`/`clean` are quiet at
everyone.

| `verdict` | Tier 0 — no new secrets | Tier 1 — Slack |
| --- | --- | --- |
| `blocking` | `REQUEST_CHANGES` review listing the findings → emails the author, red banner that must be actively dismissed. Label `review:blocking` | `<!here>`, red header, one line per finding, deep-linked to the `#issuecomment-` anchor |
| `findings` | Nothing extra — the bot comment already notified the author. Label only | **Nothing.** Routing should-fix to Slack is how a channel gets muted in week two |
| `clean` | Silent. Label + ledger | Silent |
| `incomplete` | A distinct comment: *"the automatic review did not complete — treat this PR as unreviewed"*, with the run link. **Never silent** | Orange, no `<!here>` — a harness problem, not a developer emergency |

**Build `incomplete` first.** Given defects 1 and 2 it is currently ~100% of runs. It is not the
edge case, it is the status quo, and it is what will tell us why the reviewer stops.

Topology: everything needing `steps.review.outputs.*` runs **in** the review job under
`if: always()`. A second `notify-void` job (`needs: [review]`, `if: always() &&
needs.review.result != 'success' && != 'skipped'`) covers only what the first cannot —
cancellation and timeout, where no in-job step reached the end. Exactly one notification per run
in every state. Read the record from the step output, not by parsing comments (which
reintroduces the prose-parsing we are removing) and not from artifacts (a round trip for data
already in memory).

Two details worth fixing once: the comment author is **`claude[bot]`**, not
`github-actions[bot]`, so select on `user.type == "Bot"`; and each comment's first line embeds
its own run URL, an exact per-run selector when several bot comments exist.

**Tier 1 needs admin on the consumer repo** to add `SLACK_WEBHOOK_URL`. Ship the step guarded on
`env.SLACK_WEBHOOK_URL != ''` *before* the secret exists — correct and silent, then it lights up
with no further edit.

> **This is the strongest argument for canarying in `claude_field_ops`.** On
> `sdeitech/HCTS_Backend` we hold `push`/`triage` only, so tier 1 is blocked on someone else.
> On `prity27/claude_field_ops` the API reports `"admin": true` — so Slack *and* branch
> protection can both be proven end to end there before either is asked of a repo we do not own.

**Disputes reuse what exists.** Slack → comment anchor → the developer comments `@claude`
quoting the finding → the interactive workflow already fires on `issue_comment` containing
`@claude`. Say so in the message so nobody invents a second protocol.

---

## 6. Phase 3 — Stack profiles, then distribution

### 6.1 Profiles: three artifacts, one direction of flow

| Artifact | Path | Consumed by |
| --- | --- | --- |
| Stack defaults | `plugins/sdei-review/profiles/{mern,dotnet,angular}.yaml` | reusable workflow at expansion time; skill at runtime; eval fixtures |
| Repo override | `<consumer>/.claude/stack-profile.yaml` | same |
| Prose profile (exists) | `<consumer>/docs/delivery/PROFILE.md` | humans; `/project-setup` generates the YAML from it |

Profiles live **in the plugin**, because that is what makes one fix reach N repos with a single
`/plugin update`. Each declares: `detect` globs; `gates` with `command`, `grant`,
`budget_seconds`, **`kind`** and **`ci_safe`**; `never_run`; `conventions` globs;
`false_positive_traps` feeding the class-G do-not-flag list; `cost` hints; and
`verification.last_verified`, set **only** by a green eval run.

**`gates[].kind` is load-bearing, not tidiness.** `claude_field_ops` declares a gate that
includes 86 tests (`docs/delivery/PROFILE.md:53`), while `deep-review/SKILL.md:60-61` says *"Do
not run anything slow… If the gates take minutes, say you skipped them."* The repo's declared
gate and the skill's gate policy contradict each other. `kind` resolves it: the skill consumes
`typecheck` and `lint`, never `test`.

**`ci_safe` is the security boundary.** `claude_field_ops` root `package.json` has
`"gate": "npm run gate --workspace=server && …"`. The profile records that command with
`ci_safe: false` and records the direct equivalents (`npx eslint`, `npx tsc`) as `ci_safe: true`.
CI composes `--allowedTools` from `ci_safe: true` entries only, so `npm run <anything>` never
appears in a CI grant (defect 7).

**Why the override lives in `.claude/` and not `docs/`:** the action deletes and restores
`.claude/` from the base branch, so a profile there is structurally **not PR-author
controllable** — exactly the property you want for the file that decides which commands execute.
And CI needs the gate commands *before* the model starts, so parsing a Markdown table in bash is
a liability.

**Monorepo shape, corrected by evidence.** The live case is not Angular-beside-.NET; it is npm
workspaces — `claude_field_ops` declares `"workspaces": ["server", "client"]`. Two workspaces
sharing a toolchain but needing different gates is both more common and cheaper to get right:
one `setup-node`, two gate sets, and `gate_selection: touched-only` doing the real work — map
changed paths to workspaces, so a client-only PR never installs or gates the server. The
polyglot case keeps schema support but needs a **synthetic** fixture, since no such repo is
reachable here.

**The two-place-edit hazard dies structurally, not by comment.** `scripts/gen-grants.mjs` prints
the sorted union of `profiles/*.yaml` `gates[].grant` filtered by `ci_safe`, and `harness-lint`
fails if it differs from `deep-review/SKILL.md:5`. Add a gate → regenerate → the lint proves
they agree.

Profiles stay honest by lint: no commented-out anything (if a stack is unsupported its file does
not exist — this retires `ci-review-template.yml:82-127`); every profile has a fixture; every
gate has a parseable grant; `never_run` commands appear in no `gates[]`; and a
`false_positive_traps` entry with no decoy case proving it is needed gets pruned at 90 days, or
the list grows into the prose swamp we are leaving.

### 6.2 Distribution: a reusable workflow

Three drifted copies of one 256-line template is the argument. Recommend
`prity27/claude_plugins/.github/workflows/review.yml` with `on: workflow_call`: a `plan` job
that reads the profiles and emits **both** the setup plan **and** the `--allowedTools` string
from the same `gates[].grant` fields, and a `review` job with **one `if:`-guarded setup step per
supported stack**.

GitHub Actions cannot generate `uses:` steps at runtime, so the verbosity the template tried to
solve by commenting five blocks out is solved instead by those blocks existing exactly once, in
a repo we control, behind a computed condition. `continue-on-error: true` stays on every setup
step — a broken restore must degrade the review, never cancel it.

**Be honest: the consumer writes ~14 lines, not 6.** A called workflow's token permissions come
from the caller's job, so the `permissions:` block cannot be centralised.

What it cannot do: hand a consumer the auth secret (`secrets: inherit` passes the *caller's*
secrets, and adding one needs admin **there** — that, not YAML, is the real ceiling on adoption,
and it belongs in one sentence at the top of the adoption doc); review fork PRs; or accept an
expression in its own `uses:` ref, so the consumer's pin is a literal and ref *naming* must be
stable.

**UNVERIFIED — check before building:** whether `plugin_marketplaces` accepts a ref. If not,
`plugin_ref` needs a clone-at-ref fallback passing a local path. Design that in; do not discover
it late.

Rejected: a composite action (cannot declare job `permissions`, so the highest-risk lines stay
copy-paste in N repos — keep as a documented escape hatch); a generator as *distribution*
(reintroduces drift the moment it runs) — though a generator is exactly right for an
`/adopt-review` onboarding skill.

---

## 7. Phase 4 — Evals, split by what is blocked

The five review agents have zero coverage and the only fixture is MERN-shaped.
**The runner is early-access and OFF here** — verified: `claude plugin eval` in an empty
directory prints `` `plugin eval` is currently in early access ``. So this phase splits, and
nothing else queues behind it.

### 7.1 Authoring (unblocked)

Fixtures are **synthetic, with planted defects and planted decoys**, for four reasons: the
answer key must precede the run (`tests/RUBRIC.md:3-4`, and `tests/README.md:52-73` records the
key being *wrong* and the agent right); cost scales with fixture size, so **cap at ≤25 files and
≤400 diff lines**; a fixture must not need live infrastructure (`claude_field_ops`
`PROFILE.md:51` requires Mongo as a replica set for transactions — a fixture needing that is a
fixture that gets skipped, so MERN gates are lint/typecheck only); and the existing philosophy
is already right — `tests/README.md:12`, *"The defects are not random. Each one targets a
**temptation** rather than a knowledge gap."*

Every case carries all three grader families:

- **Recall** (weight 3) — a planted defect needing two files read to see. Graded on finding it,
  at the right severity, **with a citation that says what it claims** — the failure mode named
  at `tests/README.md:48-50`.
- **Precision** (weight 2) — a decoy licensed by the fixture's own `CLAUDE.md`. Flagging it
  fails; *naming it as a documented decision* passes, because `SKILL.md:126-128` says that is
  the correct behaviour.
- **Harness** (deterministic, free) — `tool_used` on the gate command (**this grader would have
  caught defect 4 the day it landed**); `tool_used` on `Agent` with `min: 4, max: 9` (lower
  bound catches "the orchestrator reviewed it itself", upper catches a cost blowout).

**Report recall and precision as two headline numbers, never only an aggregate.** An aggregate
hides which direction a prompt edit traded, and trading recall for precision is exactly what a
nervous prompt edit does.

Mandatory floor cases: a docs-only **should-not-fire** case (guards the reviewer that always
finds something) and a **detached-HEAD** case — the fixture should *reproduce* the class-B
condition, not sanitise it.

**Δ is the headline, not pass rate.** Near-zero Δ on a recall grader means the base model would
have found it anyway; make the defect subtler or retire the case. **Δ on *precision* is the
plugin's real, never-measured product** — the bare model flags decoys; calibration and the
skeptic are supposed to stop it. Negative Δ anywhere blocks the merge.

Provenance (class E) is handled at four layers: fixtures contain no `.claude/agents|skills`
(lint) but **do** carry `CLAUDE.md`, which is the decoy licence; assert `suite.plugins` lists the
plugin with no blocking `problem` — **the machine-checkable answer to "which copy ran"**,
replacing `tests/README.md:30`'s advice; target by path from the marketplace root; and make
name-prefixing of repo-local reviewers a lint rather than a footnote. `claude_field_ops` has no
`.claude/agents|skills` today, so locking the prefix rule in **before** anyone adds them costs
nothing.

Because `claude_field_ops` is **public**, its real merged PRs are legitimate public fixture
material — which partly dissolves the private-tier question. Transcript-replay cases still carry
code and stay private-tier.

### 7.2 Running (blocked externally)

Pilot at `--runs 1` to confirm `suite.plugins` and read `costUsd`, then set `--max-cost-usd`
from measurement rather than a guess, and treat exit 2 (`cost_ceiling`) as **inconclusive, not
pass**. Partition thresholds: a small `must-find` set at `--threshold 1.0`, the broad set at
`0.75` and non-blocking. A 1.0 threshold across the whole suite with LLM graders will flake, and
a gate people learn to re-run until green is worse than no gate. `--runs 3` is the floor; cost
conversations cut **cases**, not runs.

`--scaffold` runs author-supplied bash as you, so `fixtures/*/scaffold.sh` is reviewed like
production code, and eval PRs from untrusted forks need human review of the scaffold.

**Contingency if enablement never lands:** a ~120-line runner shelling
`claude -p --output-format stream-json` per case, applying only the deterministic grader types.
Those already cover classes A, B, E and H — the ones Phases 0–3 fix. Case files are unchanged
when the real runner arrives.

**First milestone:** three cases, one stack, one number — the first Δ `sdei-review` has ever had,
plus a measured dollar cost. Do not attempt the full matrix first; grader calibration is hours
per case and cannot be skipped, because an uncalibrated grader produces confident nonsense in
both directions.

> **Open fork — judge and agent models.** The authoring rules require a sonnet-tier-or-larger
> judge that is **not** the agent model (self-preference). `agents/security-reviewer.md:5` pins
> `model: opus` in its own frontmatter, so `--model` may not control the expensive part of the
> fan-out. Recommend `--model sonnet --judge-model opus`, and verify override behaviour in the
> pilot — the documented haiku judge default is too weak for rubrics asserting citation accuracy.

---

## 8. Phase 5 — Governance and the channel

### 8.1 Case before cure

**Every accepted failure report becomes an eval case before the prompt changes.** Three
mechanisms, descending strength:

1. **A CI check** (`case-before-cure` in `harness-lint`): a PR touching
   `plugins/*/agents/**` or `plugins/*/skills/**` must add a directory under
   `plugins/*/evals/*/`, or carry a `harness:no-case-needed` label plus a stated reason. ~30
   lines of bash, no model, no judgement. The escape hatch is labelled and therefore queryable,
   so abuse is visible. The stricter "case commit must precede prompt commit" variant ships as a
   **warning** — it fights squash-merge, and a check people learn to defeat is worse than one
   that advises.
2. **A skill that makes compliance the easy path** — `/harness-report` classifies a failure
   against the eight classes and **writes the case**, tagged `known-gap`. A rule enforced only
   by a gate gets resented; a rule whose compliant path is one command gets followed. Same
   insight as the do-not-flag list: remove friction rather than exhort.
3. A PR template, so the CI failure is not the first anyone hears of it. Documentation, not
   enforcement.

**`known-gap` is the part people forget.** A case filed for a live failure must land **red**, and
there is no xfail concept — so new cases are tagged `known-gap`, excluded from the blocking run,
and listed in `docs/harness/KNOWN-GAPS.md` with class, report ids, current Δ and assigned fix
location. **Moving the tag from `known-gap` to `must-find` is the definition of done, and it is a
diff.** Publish the count: a suite with zero known gaps is either finished or dishonest. That
replaces `tests/README.md:77`'s single prose baseline with a ledger that cannot silently stay
true.

### 8.2 Versioning and rings

Semver with the contract `README.md:17-20` already promises: MAJOR = artefact-format change
requiring a re-run; MINOR = anything that can alter a verdict (agent prompt, skill step, profile
gate, topology); PATCH = docs only. **A prompt edit is never a PATCH** — that distinction is what
rings are for.

Rings work without an org, via `extraKnownMarketplaces`, which supports `ref` ("Commit SHA,
branch, or tag. Leave empty to track the default branch") and `expectedName` ("Rejects the
marketplace if its manifest name differs" — cheap insurance against a silent substitution; set it
always):

| Ring | Who | `ref` | Blast radius of a bad prompt edit |
| --- | --- | --- | --- |
| 0 canary | this repo + `claude_field_ops` | unset (tracks `main`) | 2 repos, same day |
| 1 default | most repos | `stable`, fast-forwarded only after Ring 0 is green ≥3 days and ≥5 real reviews | none until `stable` moves |
| 2 pinned | anywhere a bad review is expensive | an immutable tag or a 40-char SHA | none until someone edits the file |

A repo's `.claude/settings.json` pin beats a developer's personal settings — right for a shared
reviewer; an individual can still opt out in `settings.local.json`. CI pins independently through
the workflow's `plugin_ref`, so a repo can be Ring 1 locally and Ring 2 in CI, the right default
where CI comments on PRs.

**State plainly: with no GitHub organisation there is no enforcement.** `strictKnownMarketplaces`
is managed-settings-only. Rings are committed settings files plus a slow-moving branch. What
makes them work is that `stable` moves in a reviewed PR carrying the eval Δ, and that
`CHANGELOG.md` says what to re-run.

### 8.3 The show-and-tell channel

**Store first, channel second. Canonical store: GitHub issues on this repo**, with a structured
issue form. Slack is intake, not memory — threads are unqueryable in six weeks, which is exactly
when you want to ask "how many times has this instruction failed?" Issues are free, queryable by
`gh`, labelable by the taxonomy, and they live where the fix lands.

Required fields, each mapping to a class or to attribution: `plugin@version` (without it you
cannot tell a live bug from one fixed two releases ago — another reason tags come first),
surface, component, stack, failure-class guess, run id — and the one that matters, **"what it
assumed that was not true."** That phrasing is already what `README.md:404` asks for and rarely
gets; making it a required form field is the difference between asking and getting.

Three intake paths, one store:

1. **`/harness-report`** reads the current session, pre-fills six of eight fields, asks for one
   sentence, and files it. Target: one command and one sentence.
2. **Automatic from CI** — a review that errored, timed out, hit the ceiling, or self-reported a
   gate unavailable opens a `source:auto` issue with the record attached. Classes A and B are
   *machine-detectable*; they should never wait for a human to notice. **This is the
   highest-yield half of the intake and the least discussed.**
3. **Slack `#claude-harness`** for narration. Claude in Slack could do in-channel triage, but it
   is enabled by an **org owner** and there is no org — verify before it becomes load-bearing;
   the fallback is barely worse.

**Triage: weekly, by a scheduled agent, with one hard boundary — the robot writes cases, humans
write fixes.** A scheduled agent with write access that edits prompts is how you ship a silent
regression to every Ring-1 repo on a Monday morning. Restricting it to cases is safe (a red case
breaks nothing), useful (it is the tedious part), and enforces case-before-cure structurally
rather than by policy.

**`docs/harness/PRESSURE.md`** is the direct implementation of `README.md:335-337` —
regenerated, never hand-edited, one row per instruction: location, reports since it last changed,
when it last changed, class, verdict. **Three reports against one instruction with no change
since means that instruction is scheduled for replacement, not re-explanation.** That is the
roadmap's sentence turned into a number a robot maintains. Seed it from the eight defects in §1.

| Degradation | What prevents it |
| --- | --- |
| Venting channel nobody reads | Reports missing `plugin@version` or the assumption field are auto-closed with a template and the one-command alternative |
| Nobody believes it is read — the strongest predictor a channel dies | The bot *answers*: "fixed in v0.3.1, run `/plugin update`". The digest names outcomes per person. Proof of reading beats a promise to read |
| Reports become prompt patches | `case-before-cure` + the triage boundary |
| The suite becomes a vanity metric | Δ is the headline, not pass rate; near-zero-Δ cases retired; `KNOWN-GAPS.md` count published and non-zero |
| Triage debt | Publish the `state:accepted` unfixed count. A hard merge cap only after a month of real numbers — calibrated, not invented |
| "We discussed it in Slack" as evidence | A report is not a regression. Only a case is. Put it in the channel topic |
| Class G becomes the dumping ground | Triage must name a file it would change, or the report is `needs-repro` |

---

## 9. Phasing and honest cost

Four dependencies force the order:

1. **Attribution before measurement.** Zero tags means no version to pin, roll back to, or
   attribute a report to. Ten minutes, and everything later depends on it.
2. **Static checks before generated artifacts.** Profiles stay honest only because
   `harness-lint` enforces grants-match-frontmatter and profiles-have-fixtures. Building
   profiles first just creates a second thing to drift.
3. **Profiles before both the workflow and the evals.** The workflow's whole claim — one source
   for setup steps and grants — *is* the profile. A case cannot declare `allowed_tools` for a
   gate nobody wrote down.
4. **Intake before the second fixture.** Which stack gets covered second should be decided by
   which stack generates reports, not by guessing. Open intake in Phase 3, before triage
   automation — the only genuinely reorderable piece.

**Cheap (hours, no API spend):** tags; the doc truth-fix; scope resolution; the grant and
security fixes; narrowing the specialists; `agents/skeptic.md`; `harness-lint`; profile YAML;
the issue form; `PRESSURE.md`; `KNOWN-GAPS.md`.

**Moderate (days, no per-run cost):** the record schema and assembly; the notification tiers;
`plan-review.mjs` and the reusable workflow; `gen-grants.mjs`; `/harness-report`; the fallback
eval runner.

**Expensive (weeks and/or recurring tokens):** eval fixtures and grader calibration — the true
bottleneck; every full suite run; the weekly triage agent; keeping fixtures current.

**Blocked on someone else:** `claude plugin eval` enablement; a repo secret on any consumer
where we lack admin; Claude in Slack, which wants an org owner where there is no org.

---

## 10. Verification

- **Phase 0** — `harness-lint` red on a deliberately mismatched grant, green after
  regeneration. `git tag -l` non-empty. A PR touching one source file produces a comment with
  **every checklist box ticked** and real findings, against today's baseline of stopping at
  fan-out. Expect the *first* such PR to be skipped by the workflow-validation gate; re-test
  after merge.
- **Phase 1** — `gh run view <id> --log | grep "Set structured_output"`; the PR comment holds
  prose and **no JSON**. **The important negative test:** add a bogus `required` field so the
  schema cannot be satisfied, then confirm the step fails, the **job stays green**,
  `record_status=absent`, and the prose review still posted. Then compare `friction.denied`
  against `harness.permission_denials` — that comparison *is* the test of whether the discipline
  took.
- **Phase 2** — three PRs: one dropping an authorization guard (expect `REQUEST_CHANGES`, red
  banner, author email); one comment-only (expect `review:clean`, silence); one with
  `timeout-minutes: 1` (expect `notify-void` only, nothing else). Check the Slack payload in
  Block Kit Builder before spending a CI round trip.
- **Phase 3** — migrate one consumer, keeping its old workflow for one cycle to compare outputs
  side by side. Force the plugin path by removing the repo-local skill on a test branch off the
  default branch; confirm `review_path` names the plugin and the profile's gates were found.
- **Phase 4** — `claude plugin eval ./plugins/sdei-review --tag must-find --ablation
  with-without`, run from a scratch directory outside any repo carrying its own `.claude/`;
  assert `suite.plugins` loaded, Δ ≥ 0, precision not worse. Record the baseline before trusting
  any later delta.

---

## 11. Open questions for a human

- **Which repo is the Ring-0 canary?** Recommend **`prity27/claude_field_ops`**: we hold
  `"admin": true` there (verified), so the Slack tier and branch protection can be proven end to
  end, and it is public, so its real PRs are legal fixture material. `sdeitech/HCTS_Backend` is
  `push`/`triage` only and blocks on someone else for both.
- **Which uncovered stack second, .NET or Angular?** Recommend deciding from Phase-3 intake
  rather than now. If forced, **.NET** — the axes that differ most from the Node baseline live
  there, and Angular reuses most of the Node profile.
- **Transcript-replay eval cases: skip them, or a private companion suite?** Transcripts carry
  code, so they cannot live in this public repo. A private tier via `--eval-dir` costs a second
  CI setup.
- **Gate grants in skill frontmatter (keeps the ablation honest) or CI-computed (safe against
  PR-authored script bodies)?** Recommend both, differently: toolchain-direct commands only in
  CI, via `ci_safe`. A repo whose typecheck is a bespoke npm script loses its CI gate until it
  exposes a direct command — a conscious trade, and `claude_field_ops` is the first case.
- **Does the Slack workspace have Claude in Slack?** Decides whether triage is in-channel or a
  scheduled agent.
