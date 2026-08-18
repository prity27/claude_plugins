# PM2 over SSH — manual runbook

For a Node service on a VM under PM2. Fill the placeholders from the profile's deployment section;
never guess a host, a path or a process name.

## Before you connect

```bash
git status --short              # must be empty
git log --oneline -1            # the ref you are about to ship
npm run typecheck && npm test --if-present
```

Record the **current** deployed commit, so rollback is one command rather than an investigation:

```bash
ssh <user>@<host> "cd <path> && git rev-parse --short HEAD"
```

## Deploy

```bash
ssh <user>@<host>
cd <path>

git rev-parse --short HEAD          # note this — it is your rollback target
git fetch --all
git checkout <ref>

npm ci                              # lockfile only, never `npm install`
# npm run build                     # only if the project actually has a build step

pm2 reload <process> --update-env
pm2 status
pm2 logs <process> --lines 50 --nostream
```

- **`reload`, not `restart`** — reload is zero-downtime for clustered apps; restart drops requests.
- **`--update-env`** whenever environment values changed. PM2 caches the environment from when the
  process was first started, and without this flag your new variable is silently absent — a
  genuinely baffling class of bug.
- **`npm ci` deletes and rebuilds `node_modules`** from the lockfile. That is the point: it makes
  the host match what was tested.

## Verify

```bash
curl -fsS <health-url>                       # non-zero exit → roll back now
pm2 describe <process> | grep -E 'status|uptime|restarts'
pm2 logs <process> --lines 100 --nostream | grep -iE 'error|unhandled'
```

A climbing restart count means the process is crash-looping and PM2 is hiding it behind
`autorestart`. Roll back; do not wait for it to settle.

Then issue one real request through something the release actually changed. A health check proves
the process is up, not that the change works.

## Roll back

```bash
ssh <user>@<host>
cd <path>
git checkout <previous ref>
npm ci
pm2 reload <process> --update-env
curl -fsS <health-url>
```

If the release included a schema change, rollback includes reversing it. **Where the change has no
reversal, say so before deploying** — that makes the release one-way, and the user should know that
before it happens, not after.

## Things that bite on this setup

| Trap | Why it hurts |
| --- | --- |
| `ecosystem.config.json` naming a different entry point than the project actually runs | works while someone starts it by hand, fails the moment PM2 restarts it alone — often at reboot, weeks later |
| A project with no build step deployed as if it had one | the build silently produces nothing and the old code keeps serving |
| `pm2 save` never run | after a host reboot nothing comes back; pair it with `pm2 startup` |
| TLS certificates at a hardcoded path | if the file is missing the app may fall back to plain HTTP **silently** — verify the scheme after every deploy |
| `.env` edited on the host only | the host and the repository diverge, and the next clean deploy loses the change |
| Log files unrotated | the disk fills, and the failure looks like anything but a disk |
| Runtime version drift between host and CI | it built on Node 24 and runs on Node 18 |

## Multi-service hosts

Where the backend and a static frontend share the VM, deploy them separately, verify separately, and
note which nginx site serves which. Reloading the wrong process is easy and looks like nothing
happened.

## First-time setup

`pm2 start ecosystem.config.json`, then `pm2 save`, then `pm2 startup` and run the command it
prints. Without the last two the service does not survive a reboot — the single most common cause of
"it was working on Friday".
