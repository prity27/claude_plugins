# sdei-tools

A [Claude Code](https://code.claude.com) plugin marketplace. One plugin today:
**`sdei-review`** — five code-review specialists that run in parallel, each finding challenged
before it is reported.

## Install

In Claude Code, from any project:

```
/plugin marketplace add PLACEHOLDER_OWNER/claude-plugins
/plugin install sdei-review@sdei-tools
/reload-plugins
```

`/reload-plugins` is not optional. Skills register the moment they are installed, but the
subagent registry is snapshotted at session start — without a reload (or a restart) the five
agents are not dispatchable yet, and the plugin looks broken.

To try it without installing anything:

```
claude --plugin-dir /path/to/claude-plugins/plugins/sdei-review
```

## Requirements

Claude Code, and read access to this repository. Nothing else — no API key, no secret, no
admin rights, and no configuration. Installs authenticate with your existing git credentials,
so private forks work too.

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

## Pairing with repository-specific reviewers

These agents are deliberately generic, and generic has a ceiling. A reviewer that carries one
repo's actual file names, line citations, call-site counts, and documented exceptions will beat
a generic one on that repo every time — because it can assert "this is the convention, in 22 of
24 modules" instead of having to derive it.

So the recommended setup for a codebase you work in daily is **both**:

- Repo-specific specialists committed to that project's `.claude/agents/`, carrying its real
  evidence and its known-exception list.
- This plugin for everything else — other services, unfamiliar repos, and one-off reviews.

They coexist cleanly. Project agents in `.claude/agents/` take precedence over same-named
plugin agents, and the names here are axis-based (`security-reviewer`) rather than prefixed, so
they don't collide with a repo-specific set (`myproject-security-reviewer`).

The fastest way to build the repo-specific version is to copy an agent from `agents/` and
replace its Phase 0 calibration section with the answers — the concrete file paths, counts, and
"do not flag these, they're deliberate" list for that codebase.

## Adding a plugin to this marketplace

1. Create `plugins/<name>/.claude-plugin/plugin.json` (only `name` is required).
2. Put `agents/`, `skills/`, `hooks/` at the plugin root — **not** inside `.claude-plugin/`.
3. Add an entry to `.claude-plugin/marketplace.json` with `"source": "./plugins/<name>"`.
4. `claude plugin validate ./plugins/<name>` and `claude plugin validate .`

Note that agents do not hot-reload: after editing one, run `/reload-plugins` or restart the
session before it is dispatchable.
