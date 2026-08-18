---
name: deploy
description: Deploy the project the way the profile says it deploys — a GitHub Actions pipeline, or a PM2-over-SSH runbook, or both — with pre-flight gates that refuse a bad deploy, a secrets checklist, a post-deploy health check and a tested rollback. Use to set up a pipeline, or to run or walk through a release.
argument-hint: "[setup | release] [staging | production] [backend | frontend] — default: setup for whatever the profile lacks"
allowed-tools: Read, Write, Edit, Glob, Grep, AskUserQuestion, Bash
---

# Deploy

Two modes, driven by `docs/delivery/PROFILE.md`:

- **setup** — create the pipeline: the workflow file, the secrets list, the runbook.
- **release** — take a specific change to a specific environment, pre-flight through health check.

Missing profile → `/project-setup`. The profile's deployment section is the source of truth for
hosts, environments, secrets and the rollback command; where it says `unknown`, **ask — never
infer a host or a credential.**

References: `${CLAUDE_PLUGIN_ROOT}/references/deploy/github-actions.md` and
`${CLAUDE_PLUGIN_ROOT}/references/deploy/pm2-ssh.md`.

## 1. Verify what actually runs — before anything else

Configuration lies. Check reality:

```bash
cat package.json                     # the real start command, and whether a build exists
cat ecosystem.config.json 2>/dev/null # PM2: does `script:` match the real entry point?
ls .github/workflows/                 # which of these actually deploy?
git status --short                    # a dirty tree is not deployable
```

The classic failure: a PM2 config pointing at `src/index.js` while the project runs
`tsx src/index.ts` and has no build step. It works because someone starts it by hand, and it fails
the first time PM2 restarts it on its own. **Report every such discrepancy before deploying, and do
not paper over it** — a deploy skill that quietly works around a broken config guarantees the
outage happens later, without you.

Confirm too: does a build step exist, what is the artefact, and does the host have the runtime
version the project requires?

## 2. Pre-flight — refuse, do not warn

A deploy proceeds only when **all** of these hold. State each result explicitly:

1. The working tree is clean and the branch is the intended one.
2. Every gate in the profile passes — typecheck, lint, tests, build. Verbatim output.
3. The epic being released is `validated` in its story file, or the user explicitly overrides.
4. Every environment variable the target needs exists on the target, and no new one is missing.
   Compare the code's `process.env` reads against the environment's configured set.
5. Schema changes have a migration path, and it has been rehearsed against a copy.
6. A rollback target exists — the previous release is identified by commit or tag.
7. For production: a backup is current, and someone is watching.

A failed check stops the deploy. **Do not deploy with a warning.** The entire value of this list is
that it is a gate rather than advice.

## 3. Secrets

Every secret the target needs, where it lives, and who can rotate it. Verify presence, never print a
value. If a secret is missing, the deploy stops here.

If you find a credential in a tracked file, say so prominently: deleting the file is not
remediation, because git history keeps it. **The credential must be rotated.**

## 4. GitHub Actions path

Per the reference pack: gates run first and a failure blocks the deploy job; environment protection
on production; `npm ci` from the committed lockfile; secrets from repository or environment
secrets, never from the repo; concurrency configured so two merges cannot deploy over each other;
the deploy step ending in a health check that **fails the job** when it fails.

Before writing a workflow, list what a human must do that you cannot: create the secrets, add the
deploy key, configure the environment and its reviewers. Say it plainly, because until those exist
the workflow fails on the first run and looks broken.

## 5. PM2 over SSH path

Per the reference pack — the runbook, with the real commands from the profile:

```bash
ssh <host>
cd <path> && git fetch --all && git checkout <ref>
npm ci                      # from the lockfile, never `npm install`
<build, only if the project has one>
pm2 reload <process> --update-env
pm2 status && pm2 logs <process> --lines 50
curl -fsS <health-url>      # non-zero means roll back now
```

Note where the process name, path and host came from. `pm2 reload` over `restart` for zero-downtime;
`--update-env` when environment values changed, because PM2 otherwise keeps the old ones and the
symptom is baffling.

**Do not run a production deploy autonomously.** Show the exact commands, confirm with the user, and
have them run it or explicitly authorise you to. A deploy is outward-facing and hard to reverse.

## 6. Verify, and be ready to roll back

After deploy: the health check, the log tail for errors, one real request through a changed
endpoint, and the version actually running.

Have the rollback command ready **before** you deploy, not after something breaks:

```bash
git checkout <previous ref> && npm ci && pm2 reload <process>
```

If the release included a schema change, the rollback includes its reversal — and if there is no
reversal, say so before deploying, because that makes the release one-way.

## 7. Report

Environment and ref deployed, pre-flight results, what ran, health-check output, what to watch, the
rollback command, and anything a human must still do.

## Standing rules

- **Never deploy on a failing gate or a dirty tree.**
- Never deploy to production without explicit confirmation in that message.
- Never print, log or commit a secret value.
- Never invent a host, a path, a process name or a credential. Unknown means ask.
- Never silently work around a broken configuration — report it and fix it properly.
