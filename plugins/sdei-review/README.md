# sdei-review

Five code-review specialists that run in parallel, each owning one axis, with every finding
challenged by a skeptic before it reaches you.

Language and framework agnostic: the agents carry no hardcoded rules. They calibrate against the
repository they are reviewing — its own `CLAUDE.md`, its manifests, its neighbouring files — and
derive the convention rather than importing one.

Where [`sdei-delivery`](../sdei-delivery) establishes what was *promised*, this judges whether the
code is any *good*. Both matter, and neither substitutes for the other.

## Install

```
/plugin marketplace add prity27/claude_plugins
/plugin install sdei-review@sdei-tools
/reload-plugins
```

`/reload-plugins` is **not optional.** Skills register on install, but the subagent registry is
snapshotted at session start — without a reload the five agents are not dispatchable, the fan-out
fails, and the plugin looks broken.

Without installing:

```
claude --plugin-dir /path/to/claude-plugins/plugins/sdei-review
```

## Updating

```
/plugin marketplace update sdei-tools     # refresh the marketplace from git
/plugin update sdei-review@sdei-tools     # move the installed copy to the new version
/reload-plugins                           # re-register the skill and the five agents
```

`/plugin marketplace update` refreshes the clone; `/plugin update` moves your *installed* copy,
which is cached per version — so the marketplace can be current while your session still runs the
old one. `claude plugin update` says a **restart is required to apply** it, and for agents that is
literal: they never hot-reload.

If a change to an agent seems to have done nothing, you are almost certainly still running the old
prompt. Restart.

```
/plugin list                  # installed plugins and versions
/plugin details sdei-review   # component inventory and projected token cost
```

## Usage

### `/deep-review [branch | path | commit]`

| Invocation | Scope |
| --- | --- |
| `/deep-review` | uncommitted changes, plus this branch versus the default branch |
| `/deep-review feature/billing` | that branch against its merge base |
| `/deep-review src/payments` | that path |
| `/deep-review HEAD~3` | that commit range |

The default branch is **derived**, not assumed — `origin/HEAD` first, falling back to
`main`/`master`. On a clean tree it stops and says so rather than reviewing nothing.

### What a run actually does

1. **Resolves scope once**, centrally, so no specialist re-derives it. Produces a changed-file list
   and a short change summary.
2. **Calibrates once.** Reads the project's own rules, detects the stack from its manifests, and —
   most importantly — collects the project's documented trade-offs into a **do-not-flag list** that
   every specialist receives. This single step removes most false positives.
3. **Runs the project's own gates once** — typecheck, lint, fast tests — so five agents do not each
   run them, and so nobody reports what a compiler already catches. Slow, destructive or
   network-dependent commands are skipped, and it says so.
4. **Fans out all five specialists concurrently.** A specialist whose axis the diff cannot touch is
   skipped and named — no routes or schemas changed means `contract-reviewer` has nothing to do.
   Security and correctness are never skipped on executable code.
5. **Deduplicates** on `file:line` plus the claim. Overlap is expected: a missing guard draws both
   security and conventions.
6. **Verifies adversarially.** Each surviving finding goes to a skeptic told to *refute* it, with
   instructions to default to REFUTED when it cannot demonstrate the failure. A finding survives
   only if refutation fails.
7. **Reports** grouped Blocking / Should fix / Consider, each with `file:line`, the concrete input
   or interleaving that makes it fail, and the fix — preferring to point at an existing pattern in
   the codebase over writing a snippet.

It closes with one line per axis that came back clean. **Five agents finding nothing is a valid
result**, and it says so plainly rather than manufacturing something to justify the run.

### Using one specialist alone

Sometimes you only care about one axis, or you want the unfiltered list without the skeptic pass:

```
Use the security-reviewer agent on src/api/auth
Use the performance-reviewer agent on the current diff
Use the contract-reviewer agent on src/routes and docs/API.md
```

Run directly, an agent reports everything it finds. That is noisier than `/deep-review` by design —
the adversarial verification is what makes the orchestrated output short.

## The five specialists

| Agent | Owns | Typical finding |
| --- | --- | --- |
| `security-reviewer` | authorization, identity spoofing, IDOR, injection, mass assignment, secret leakage, weak crypto, unsafe uploads | a route that checks the caller's *role* but not whether they own *this record* |
| `correctness-reviewer` | atomicity and partial writes, unchecked nullable results, races, boundaries and timezones, silently dropped writes, error handling | a two-step write with no transaction, where a failure between steps leaves an inconsistent state |
| `convention-reviewer` | layering, module boundaries, naming, error and response shapes, reuse of existing utilities | a new module that reimplements a helper the repo already has, in a different shape |
| `performance-reviewer` | N+1 queries *and* requests, unbounded result sets, index coverage, over-fetching, algorithmic cost, fan-out | a list endpoint with no maximum page size, or a loop issuing one query per item |
| `contract-reviewer` | endpoint/schema/validator parity, response drift, breaking changes to published interfaces | a handler returning a field the documented schema does not have, or a required field made optional |

## Why they work on a codebase they have never seen

Generic review advice is noise, and a review that cries wolf gets ignored — which is worse than no
review. These agents avoid that by **calibrating before judging**:

- **Counting before claiming.** A rule is asserted as "the convention" only after grepping for it.
  If the codebase does something 12 ways versus 10, they report that there *is* no convention
  rather than inventing one.
- **Sibling comparison over memorised rules.** The reference is the neighbouring file, not a style
  guide the agent brought with it.
- **Documented project decisions win.** If the repo says a trade-off is deliberate, it is not a
  defect. Where a finding collides with a documented decision, the agent names the decision and
  says a human must choose — it does not pick a side.
- **Diff-scoped.** Pre-existing issues are out of scope unless the change depends on them.
- **Adversarial verification.** Every finding must survive an attempt to refute it.
- **Never modifies files.** This skill reviews; you decide what to apply.

Observed on a React 19 + Vite frontend: the agents found the project's axios wrapper convention,
identified the existing API helpers the new code should have reused, noticed a token key casing
mismatch (`accessToken` versus the repo's `accesstoken`), and explicitly scoped a pre-existing
committed `.env` *out* as not belonging to the diff.

## Pitfalls

- **It is diff-scoped on purpose.** Pre-existing problems are not reported unless your change
  depends on them. For a whole-codebase audit, point it at paths deliberately.
- **A clean review is not a clean bill of scope.** `/deep-review` never asks whether the code does
  what was promised. That is `sdei-delivery`'s `/validate-delivery`, and it is a different question
  with a different answer.
- **The output is shorter than a naive review's.** That is the adversarial pass working. If you want
  everything an axis can see, run that agent directly.
- **A wrong or stale `CLAUDE.md` will actively mislead it.** These agents defer to your documented
  decisions, so aspirational documentation gets treated as fact. They will, however, tell you where
  your docs disagree with your code.
- **Agents do not hot-reload.** After editing an agent prompt, `/reload-plugins` or restart, or the
  old version keeps running.
- **Skipping is normal and is reported.** If a specialist was skipped, the report says which and
  why. Read that line — it tells you what was *not* examined.

## Building a repo-specific version

These agents are deliberately generic, and generic has a ceiling. A reviewer carrying one repo's
actual file names, call-site counts and documented exceptions beats a generic one on that repo,
because it can assert "this is the convention, in 22 of 24 modules" instead of deriving it.

The fastest way to build one: copy an agent from `agents/` into the project's `.claude/agents/` and
replace its Phase 0 calibration section with the answers — concrete file paths, counts, and the
"do not flag these, they are deliberate" list.

Project agents in `.claude/agents/` take precedence over same-named plugin agents, and the names
here are axis-based rather than prefixed, so a repo-specific set named
`myproject-security-reviewer` coexists cleanly with this one.

## Changing an agent

Agents live in `agents/` at the plugin root. After editing one, run `/reload-plugins` or restart
before testing — otherwise you are testing the old prompt.

`sdei-delivery/tests/` holds a regression suite: a synthetic project with defects planted on
purpose and a pre-registered answer key. Run it after any agent edit; a regression shows up as a
specific numbered row so you can tell which instruction stopped working. Baseline is 30/30.
