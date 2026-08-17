---
name: deep-review
description: Review the current changes with five specialists in parallel (security, correctness, conventions, performance, contract), challenge every finding adversarially, then report what survives. Works in any language or framework. Use for reviewing a diff, branch, PR, or path.
argument-hint: "[branch | path | commit] — default: uncommitted changes plus branch vs the default branch"
allowed-tools: Read, Grep, Glob, Bash(git status:*), Bash(git diff:*), Bash(git log:*), Bash(git show:*), Bash(git rev-parse:*), Bash(git merge-base:*), Bash(git ls-files:*), Agent
---

# Deep review

Five specialists review the same change in parallel, each owning one axis. You resolve the scope
once, calibrate once, fan them out, then **challenge every finding before reporting it**.

The verification step is the point. Five agents reporting unverified findings would bury the real
ones, and a review that cries wolf gets ignored — which is worse than no review.

## 1. Resolve the scope — once, centrally

Do this yourself so no specialist re-derives it.

```bash
git status --short
git rev-parse --abbrev-ref origin/HEAD    # the default branch; fall back to main/master
git diff --stat HEAD                      # uncommitted
git diff --stat <default>...HEAD          # branch vs default
```

- **No argument** → uncommitted changes plus this branch versus the default branch.
- **A branch, path, or commit** → scope to that.

Note the default branch may not be `main`. Derive it; don't assume.

Produce two artefacts:

1. The **changed-file list**, grouped by area.
2. A **3–6 sentence change summary**: what it does, which layers it touches, whether it adds
   entry points, schema changes, or dependencies.

**If nothing changed, stop and say so.** Never review a clean tree.

## 2. Calibrate — once, centrally

The specialists are stack-agnostic, so this step is what makes their findings concrete rather
than generic. Spend a few tool calls and pass the results to every agent.

- **Project rules**: does `CLAUDE.md`, `AGENTS.md`, `CONTRIBUTING.md`, or `docs/` exist? Note what
  they mandate. **Documented project decisions outrank every generic rule the specialists carry**,
  and must be passed along explicitly.
- **Stack**: language, framework, datastore, and test/lint/typecheck tooling, from the manifests.
- **Gates**: the project's own check commands (from the manifest scripts or CI config).
- **Known trade-offs**: if the project documents unresolved inconsistencies, deliberate
  exceptions, or "do not fix X as a side effect" rules, **collect them and pass them to every
  specialist as a do-not-flag list.** This single step removes most false positives.

## 3. Run the project's own gates — once

If you found a typecheck, lint, or test command, run the fast ones now and note real failures.
Running them centrally avoids five duplicate runs, and it means the specialists can skip anything
a compiler would catch.

Do not run anything slow, destructive, or network-dependent. If the gates take minutes, say you
skipped them and why.

## 4. Fan out — all five in a single message

Launch these concurrently (one message, five `Agent` calls):

| Subagent | Axis |
| --- | --- |
| `security-reviewer` | authorization, identity, IDOR, injection, mass assignment, secrets, crypto |
| `correctness-reviewer` | atomicity, unchecked results, races, boundaries, dropped writes, error handling |
| `convention-reviewer` | layering, module structure, naming, result shapes, reuse |
| `performance-reviewer` | N+1, unbounded reads, indexes, over-fetching, algorithmic cost |
| `contract-reviewer` | surface/request/response parity, breaking changes |

Give each the same context: the changed-file list, the change summary, the calibration findings
(especially the do-not-flag list), the gate results, and the instruction *review only these
changes; report only on your own axis*.

**Skip a specialist when the diff cannot touch its axis** — no routes, schemas, or shared types
changed means `contract-reviewer` has nothing to do; a docs-only change needs none of them. Say
which you skipped and why. Never skip security or correctness on a change to executable code.

## 5. Deduplicate

Merge findings keyed on `file:line` plus the claim. Overlap is expected: a missing guard draws
security and conventions; an unindexed filter added with a new endpoint draws performance and
contract. Keep the version from the agent that owns the axis and fold the other's detail in.

## 6. Verify adversarially

For each surviving finding, dispatch a **skeptic** instructed to *refute* it. Run these in
parallel.

- Prompt shape: *"Here is a claimed defect: `<finding>` at `<file:line>`. Read the actual code and
  try to disprove it. Consider whether it is pre-existing, unreachable, already handled
  elsewhere, or a deliberate project decision. Default to REFUTED if you cannot demonstrate the
  failure. Reply `CONFIRMED` with the concrete failing input, or `REFUTED` with the reason."*
- Use a cheap model for mechanical axes (conventions, contract) and a stronger one for security
  and correctness.
- A finding survives **only if refutation fails**.

Drop, without reporting: anything refuted; anything the gates in step 3 would catch; pre-existing
issues outside the diff; and anything on the do-not-flag list from step 2.

## 7. Report

Group by severity, most severe first:

- **Blocking** — data corruption, authorization bypass, secret leakage, a crash, unbounded
  user-driven growth, a broken published contract, or something that fails at runtime.
- **Should fix** — real bugs in edge cases, convention violations with sibling evidence, N+1
  queries, contract drift.
- **Consider** — naming, reuse, minor simplifications.

For each finding: **what** is wrong (one sentence, with `file:line`), **how it fails** in practice
(the concrete input, request, or interleaving), and **the fix**. Prefer pointing at an existing
pattern in the codebase over writing a snippet.

Close with one line per axis that came back clean, and note any specialist you skipped.

**If the change is good, say so plainly.** Five agents finding nothing is a valid, useful result.

## Standing rules

- **Do not modify files.** This skill reviews; the caller decides what to apply.
- Cite `file:line` for everything. An uncited finding does not get reported.
- Where a finding collides with a documented project decision, name the decision and say a human
  must choose — do not pick a side, and do not fix it as a side effect of the review.
