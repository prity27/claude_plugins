---
name: deploy
description: Deploy the project the way the profile says it deploys — a CI pipeline, a direct-to-host runbook (PM2, systemd, container, PaaS or static upload), or both — with pre-flight gates that refuse a bad deploy, a secrets checklist, a post-deploy health check and a tested rollback. Language and platform agnostic. Use to set up a pipeline, or to run or walk through a release.
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
`${CLAUDE_PLUGIN_ROOT}/references/deploy/pm2-ssh.md`. Both are worked examples in one stack — take
their **structure** (gate before deploy, pinned install, health check that fails the job, rollback
prepared first) and translate the commands to whatever the profile records.

## 1. Verify what actually runs — before anything else

Configuration lies. Check reality — the manifest the profile's **stack family** row names, plus
whatever declares the release:

```bash
git status --short                    # a dirty tree is not deployable
ls .github/workflows/ .gitlab-ci.yml Jenkinsfile 2>/dev/null   # which of these actually deploy?
```

Then, for the delivery mechanism the profile records:

| Mechanism | Read | The question it answers |
| --- | --- | --- |
| process manager over SSH | `ecosystem.config.*`, the systemd unit, the supervisor conf | does the start command match the real entry point? |
| container | `Dockerfile`, compose file, the registry tag in CI | what is in the image, and is the tag pinned? |
| PaaS push | `vercel.json`, `fly.toml`, `render.yaml`, `Procfile` | which command does the platform run, and from which branch? |
| static upload | the build output directory and the bucket/CDN target | is the artefact the one CI produced? |

The classic failure has the same shape in every stack: **the declared start command is not the one
a human uses.** A PM2 config pointing at `src/index.js` while the project runs `tsx src/index.ts`
with no build step; a Dockerfile `CMD` running a module that was renamed; a Procfile referencing a
gunicorn app path that moved. It works because someone starts it by hand, and it fails the first
time the platform restarts it on its own. **Report every such discrepancy before deploying, and do
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
   Compare the environment reads in the code — `process.env`, `os.environ`, `os.Getenv`,
   `System.getenv`, `ENV[]`, or the stack's equivalent — against the environment's configured set.
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

## 4. CI pipeline path

The reference pack is written for GitHub Actions; the structure transfers to GitLab CI or Jenkins
unchanged, so translate rather than refuse if that is what the profile records.

Per the reference pack: gates run first and a failure blocks the deploy job; environment protection
on production; a **deterministic install from the committed lockfile** — `npm ci`, `pip install -r
requirements.txt` against a pinned file, `go mod download`, `mvn -B -o`, `bundle install
--deployment` — never a resolver that can pick a new version at deploy time; secrets from
repository or environment secrets, never from the repo; concurrency configured so two merges cannot
deploy over each other; the deploy step ending in a health check that **fails the job** when it
fails.

Take every command from the profile's **install command on target** and **gate commands** rows. If
a row is `unknown`, ask — a plausible-looking command in a pipeline is how a deploy breaks in a way
nobody can read.

Before writing a workflow, list what a human must do that you cannot: create the secrets, add the
deploy key, configure the environment and its reviewers. Say it plainly, because until those exist
the workflow fails on the first run and looks broken.

## 5. Direct-to-host path

The runbook has the same five beats whatever the mechanism — **fetch the ref, install
deterministically, build if there is a build, replace the running process, prove it is alive.**
Fill each beat from the profile; never from memory of another project.

The reference pack covers PM2 over SSH in full:

```bash
ssh <host>
cd <path> && git fetch --all && git checkout <ref>
npm ci                      # from the lockfile, never `npm install`
<build, only if the project has one>
pm2 reload <process> --update-env
pm2 status && pm2 logs <process> --lines 50
curl -fsS <health-url>      # non-zero means roll back now
```

The equivalents, for the same five beats:

| Mechanism | Install | Replace the process | Prove it |
| --- | --- | --- | --- |
| systemd | the stack's lockfile install | `sudo systemctl restart <unit>` | `systemctl is-active <unit>` + health URL |
| container | `docker pull <image>@<digest>` | `docker compose up -d` / `kubectl rollout restart deploy/<name>` | `kubectl rollout status` + health URL |
| PaaS | the platform builds it | `vercel deploy --prod` / `flyctl deploy` | the platform's status command + health URL |
| static | build locally or in CI | sync the artefact, then invalidate the CDN | fetch the deployed asset and check its hash |

Note where the process name, path and host came from. Prefer the zero-downtime verb the platform
offers — `pm2 reload` over `restart`, a rolling update over a recreate. `--update-env` when
environment values changed, because PM2 otherwise keeps the old ones and the symptom is baffling;
every platform has its own version of that trap, so check whether the new environment actually
reached the new process rather than assuming it did.

**Do not run a production deploy autonomously.** Show the exact commands, confirm with the user, and
have them run it or explicitly authorise you to. A deploy is outward-facing and hard to reverse.

## 6. Verify, and be ready to roll back

After deploy: the health check, the log tail for errors, one real request through a changed
endpoint, and the version actually running.

Have the rollback command ready **before** you deploy, not after something breaks:

```bash
git checkout <previous ref> && <install command> && <restart command>
```

Both commands come from the profile. For a container or PaaS deploy the rollback is usually
redeploying the previous **immutable** artefact — the prior image digest or release id — which is
safer than rebuilding from a previous ref, because a rebuild can pick up different dependencies.
Record which of the two you are relying on.

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
