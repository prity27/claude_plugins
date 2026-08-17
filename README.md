# sdei-tools

A Claude Code plugin marketplace. One plugin today: **`sdei-review`**.

## Install

```
/plugin marketplace add <owner>/claude-plugins
/plugin install sdei-review@sdei-tools
```

Replace `<owner>/claude-plugins` with wherever this repository ends up hosted. Private
repositories work — installs authenticate with your existing git credentials.

To try it without installing:

```
claude --plugin-dir /path/to/claude-plugins/plugins/sdei-review
```

## What `sdei-review` gives you

**`/deep-review [branch|path|commit]`** — resolves the diff once, runs the project's own
typecheck/lint/test gates once, fans out five specialists in parallel, deduplicates, then
challenges every finding with a skeptic before reporting. Findings that survive are grouped
Blocking / Should fix / Consider.

**Five specialists**, each usable on its own when you only care about one axis:

| Agent | Owns |
| --- | --- |
| `security-reviewer` | authorization, identity spoofing, IDOR, injection, mass assignment, secrets, crypto, uploads |
| `correctness-reviewer` | atomicity and partial writes, unchecked results, races, boundaries and timezones, silently dropped writes, error handling |
| `convention-reviewer` | layering, module structure, naming, result shapes, reuse of existing utilities |
| `performance-reviewer` | N+1 queries *and* requests, unbounded result sets, index coverage, over-fetching, algorithmic cost, fan-out |
| `contract-reviewer` | endpoint/schema/validator parity, response drift, breaking changes to published interfaces |

## Why these work on an unfamiliar codebase

Generic review advice is usually noise. These agents avoid that by **calibrating before
judging** — every one of them starts by reading the project's own rules (`CLAUDE.md`,
`CONTRIBUTING.md`, `docs/`), detecting the stack from its manifests, and locating the relevant
existing patterns. Then:

- **Counting before claiming.** A rule is only asserted as "the convention" after grepping for
  it. If the codebase does something 12 ways versus 10, the agents report that there *is* no
  convention rather than inventing one.
- **Sibling comparison over memorised rules.** The reference is the neighbouring file, not a
  style guide.
- **Documented project decisions win.** If the repo says a trade-off is deliberate, it is not
  reported as a defect.
- **Diff-scoped.** Pre-existing issues are out of scope unless the change depends on them.
- **Adversarial verification.** Every finding must survive an attempt to refute it.

Verified in practice: run against a React 19 + Vite frontend, the agents found the project's
axios wrapper convention, identified the existing API helpers the new code should have reused,
noticed a token key casing mismatch (`accessToken` vs the repo's `accesstoken`), and explicitly
scoped a pre-existing committed `.env` *out* as not belonging to the diff.

## Repository-specific reviewers

These are deliberately generic. A reviewer that carries a specific repo's file names, line
citations, and known exceptions will always beat a generic one on that repo — see
`HCTS_Backend/.claude/agents/` for that pattern.

Project agents in `.claude/agents/` override same-named plugin agents, so a repo can keep its
own specialists and still install this plugin for everything else. The names here are
axis-based (`security-reviewer`) rather than prefixed, so they don't collide with
repo-specific ones (`hcts-security-reviewer`).

## Adding a plugin to this marketplace

1. Create `plugins/<name>/.claude-plugin/plugin.json` (only `name` is required).
2. Put `agents/`, `skills/`, `hooks/` at the plugin root — **not** inside `.claude-plugin/`.
3. Add an entry to `.claude-plugin/marketplace.json` with `"source": "./plugins/<name>"`.
4. `claude plugin validate ./plugins/<name>` and `claude plugin validate .`

Note that agents do not hot-reload: after editing one, run `/reload-plugins` or restart the
session before it is dispatchable.
