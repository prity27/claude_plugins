# GitHub Actions deployment

The shape a deploy workflow must have, and what a human must do before it can run.

## Non-negotiables

1. **Gates before deploy.** The deploy job `needs` the gate job. A failing typecheck, lint or test
   must make deployment impossible, not merely noisy.
2. **`npm ci`, never `npm install`.** `ci` installs exactly the committed lockfile. `install` can
   resolve a different tree than the one that was tested.
3. **Concurrency control.** Two merges landing together must not deploy over each other.
4. **Environment protection on production** — a required reviewer, so production is never one merge
   away from an accident.
5. **The health check fails the job.** A deploy that "succeeded" while the service is down is worse
   than a failure, because nobody looks.
6. **Least privilege.** Set `permissions:` explicitly; the default token grants more than a deploy
   needs.
7. **Pin actions** to a major version at minimum.

## Skeleton

```yaml
name: Deploy

on:
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      environment:
        type: choice
        options: [staging, production]

concurrency:
  group: deploy-${{ github.ref }}
  cancel-in-progress: false      # never cancel a deploy mid-flight

permissions:
  contents: read

jobs:
  gates:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version-file: package.json    # the version the project declares
          cache: npm
      - run: npm ci
      - run: npm run typecheck
      - run: npm run lint --if-present
      - run: npm test --if-present
      - run: npm run build --if-present

  deploy:
    needs: gates
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment || 'staging' }}   # protection rules apply here
    steps:
      - uses: actions/checkout@v4

      - name: Deploy over SSH
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.DEPLOY_HOST }}
          username: ${{ secrets.DEPLOY_USER }}
          key: ${{ secrets.DEPLOY_KEY }}
          script: |
            set -euo pipefail
            cd ${{ secrets.DEPLOY_PATH }}
            git fetch --all
            git checkout ${{ github.sha }}
            npm ci
            pm2 reload ${{ secrets.PM2_PROCESS }} --update-env

      - name: Health check
        run: |
          for i in $(seq 1 10); do
            curl -fsS "${{ secrets.HEALTH_URL }}" && exit 0
            sleep 5
          done
          echo "health check failed after 50s"; exit 1
```

`set -euo pipefail` in the remote script matters: without it a failed `npm ci` still reports success
and PM2 reloads the old code while the job goes green.

## What a human must do first

You cannot do any of these — list them explicitly, because until they exist the workflow fails on
its first run and looks broken:

| Item | Where |
| --- | --- |
| `DEPLOY_HOST`, `DEPLOY_USER`, `DEPLOY_PATH`, `PM2_PROCESS`, `HEALTH_URL` | Settings → Secrets and variables → Actions |
| `DEPLOY_KEY` | a **deploy-only** private key, its public half in the host's `authorized_keys` |
| The `production` environment and its required reviewers | Settings → Environments |
| Branch protection on the deploy branch | Settings → Branches |

Use **environment** secrets rather than repository secrets where staging and production differ —
that is what stops a staging change reaching production credentials.

## Frontend variants

A static frontend replaces the SSH step with a build and an upload — object storage plus a CDN
invalidation, or the host's own action. Two things to get right:

- **Build-time variables are baked into the bundle and are public.** Never a secret, and a per
  environment build, because the API origin differs.
- **Invalidate the CDN cache after upload.** Otherwise the deploy succeeds and users keep the old
  bundle, which is the most confusing possible outcome.

## Auditing an existing workflow

Report, do not rewrite: does it gate before deploying; does it use `npm ci`; is production
protected; does the health check fail the job; are secrets scoped to environments; can two runs
overlap; and does the remote script use `set -e`.
