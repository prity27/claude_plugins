# Changelog

Versions are per plugin but released together, since both share this marketplace.

This project is early. Artefact formats are stable enough to build on and not frozen — where a
release changes one, the entry says what to re-run.

## 0.2.0 — 2026-08-19

First release documented for other people to install and use.

### Added

- **`/onboard-existing`** — the brownfield entry point that was missing. Inventories what the code
  actually does with `file:line` evidence, classifies every surface `working` / `partial` / `dead`,
  derives graph entries marked `observed`, back-fills `as-built` stories, and reconciles code
  against surviving scope documents into built-and-agreed / built-but-never-agreed /
  agreed-but-never-built.

  Its governing rule: **code is evidence of behaviour, never evidence of intent.** Anything derived
  from code cannot become a requirement until a human confirms it, which is what stops a
  half-abandoned feature from being read as a specification.

- **Interview mode in `/ingest-knowledge`.** With no source material, it runs a structured interview
  and writes the answers to `docs/knowledge/sources/` as a citable digest at medium authority, so
  the graph's citation trail stays real even when there was never a document. Previously this path
  effectively required inventing a graph or skipping to `/write-stories` uncited.

- **`docs/delivery/AS-BUILT.md` template**, and a `PROFILE.md` **project stage** section
  (`greenfield` / `scaffolded` / `partial` / `mature`) with the three brownfield lists: what works
  end to end, what was started and abandoned, what contradicts the docs or config.

- **`CLAUDE.md` is now a fourth output of `/write-docs`**, generated from `PROFILE.md` plus verified
  code, with a template at `references/templates/claude-md.md`.

  This closes a real gap between the two plugins. `sdei-delivery` *read* `CLAUDE.md` and never wrote
  it; `sdei-review` reads it and has no fallback to `PROFILE.md`. So a project set up entirely by
  the delivery chain had a rich profile and **no `CLAUDE.md` at all**, and the reviewer arrived with
  no rulebook — rediscovering, or wrongly reporting, decisions already made and written down.

  Its highest-value section is **Known state, deliberately**: the profile's known-exceptions row,
  copied verbatim, which is precisely the do-not-flag list `/deep-review`'s calibration step asks
  for. **An existing `CLAUDE.md` is never overwritten** — absent, it is generated; present, a diff
  is proposed for a human to apply; present and contradicted by the code, that is reported as a
  finding.

  Recommended ordering is now explicit in both READMEs: **`/write-docs` before `/deep-review`.**

- **`sdei-review/README.md`** — the plugin had none.

- Usage documentation: per-skill reference, both entry-point paths, an artefact/status table, and a
  pitfalls section in every README. Install, **update**, inspect (`/plugin list`, `/plugin details`)
  and uninstall commands are now spelled out, including the distinction between refreshing the
  marketplace clone and moving the installed copy to a new version — and a release procedure using
  `claude plugin tag`.

### Changed — multi-stack support

`sdei-review` was already stack-agnostic. `sdei-delivery` was not, despite the claim: its detection
was npm-only, its tool whitelist permitted only `npm run`, and its deploy paths knew only GitHub
Actions and PM2. A Python, Go, Java or .NET project produced a half-empty profile.

- **`/project-setup`** now branches on a manifest-detection matrix — `package.json`,
  `pyproject.toml` / `requirements.txt` / `Pipfile`, `go.mod`, `pom.xml` / `build.gradle`,
  `*.csproj`, `Gemfile`, `composer.json`, `Cargo.toml`, and polyglot repos — with a universal
  signals table (missing lockfile, CI that only runs on PRs, a stale `.env.example`, an unreferenced
  Dockerfile) plus the Node-specific traps retained as a labelled example. Tool whitelist widened to
  the equivalent gate commands per stack.

- **`/deploy`** splits into a CI path and a direct-to-host path. The runbook is now five beats —
  fetch the ref, install deterministically from a lockfile, build if there is a build, replace the
  running process, prove it is alive — with translations for PM2, systemd, containers, PaaS and
  static hosts. Commands come from the profile instead of being hardcoded. The GitHub Actions and
  PM2 reference packs are now framed as worked examples to translate, not as the only options.

- **`/write-tests`, `/write-docs`, `/build-module`** read the test, script and gate commands from
  `PROFILE.md` rather than from `package.json`.

- **`PROFILE.md` template** gained `Stack family`, `Package manager`, `Install command on target`
  and `Restart / release command` rows; the deployment section now separates **CI** from **delivery
  mechanism**, and the frontend section accepts `not applicable` for API-only services.

### Changed — the graph and the interlocks

- **New confidence level `observed`** in `graph.json`, for claims read out of existing code. Treated
  exactly like `implied` at every gate: it may describe the system, but it may not become a
  requirement until a human confirms it. An `observed` item contradicting a `stated` one is a
  conflict for a human — the code does not win merely by being more recent.

- **New source kind `source-code`**, with high authority on behaviour and **none on intent**.

- **New story status `as-built`.** `/build-module` refuses it exactly as it refuses a draft — the
  code already exists, there is nothing to build. `/validate-delivery` accepts it, with the caveat
  stated in its report that criteria derived from the code they check prove internal consistency,
  not that the behaviour was wanted.

- **`/write-stories` reads `AS-BUILT.md` first** where it exists, so the agreed-but-never-built list
  becomes the backlog and already-delivered capability does not get a duplicate epic.

- New graph invariant: every `observed` item cites a real `source-code` path and line range and
  appears in `AS-BUILT.md`.

### Migration from 0.1.0

Nothing breaks. Existing artefacts stay valid.

- To pick up the stage section and the new stack rows in an existing `PROFILE.md`, re-run
  `/project-setup`. It asks before overwriting and shows what changed.
- Existing graphs need no change; `observed` and `source-code` are additive.
- Existing stories need no change; `as-built` is only produced by `/onboard-existing`.
- To get a `CLAUDE.md` on a project the delivery chain already set up, run `/write-docs`. If you
  hand-wrote one, it is not touched — you get a proposed diff instead.

## 0.1.0

Initial internal version.

- `sdei-review`: `/deep-review` plus five specialists (security, correctness, conventions,
  performance, contract) with adversarial verification of every finding.
- `sdei-delivery`: nine skills from project profile through deployment, three supporting agents,
  and a regression suite for the agent prompts with a pre-registered answer key.
