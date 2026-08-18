---
name: validate-delivery
description: Validate delivered code against the approved user stories and their acceptance criteria — run the project's gates, map every criterion to concrete evidence, mark it PASS, FAIL or NOT COVERED, and take a human through sign-off. Scope conformance, not code quality. Use before calling an epic done, and before a demo or a release.
argument-hint: "<epic id> — default: every epic with status 'built'"
allowed-tools: Read, Write, Edit, Glob, Grep, AskUserQuestion, Agent, Bash
---

# Validate delivery

Answer one question, with evidence: **was what was promised actually delivered?**

This is deliberately not a code review. `sdei-review`'s `/deep-review` judges whether the code is
good; this judges whether it is what the approved stories said. Both matter, and neither substitutes
for the other — well-built code implementing the wrong scope still fails acceptance.

## 0. Preconditions

Read `docs/delivery/PROFILE.md`, the epic file, and every story's acceptance criteria. The epic must
be `approved` — an epic whose scope was never agreed cannot be validated against it.

Establish what you are validating: the epic's commits, the branch, or the working tree. Say which.

## 1. Run the gates first

Run the profile's gate commands — typecheck, lint, tests, build — and report the **verbatim**
output. A failing gate does not stop the validation, but it is reported first and it blocks
sign-off.

Note explicitly if there are no tests. "The suite passes" and "there is no suite" are opposite
facts, and the difference must never be blurred.

## 2. Build the evidence matrix

Dispatch `acceptance-validator` in Mode B against the epic. For **every** acceptance criterion in
every story, one row:

| Story | AC | Verdict | Evidence |
| --- | --- | --- | --- |
| BE-03-02 | AC-1 | PASS | `campaign.service.ts:88` sets status and stamps `closedBy`; `campaign.test.ts:41` asserts both |
| BE-03-02 | AC-2 | PASS | `campaign.service.ts:79` returns 409; `campaign.test.ts:58` asserts status unchanged |
| BE-03-02 | AC-3 | **NOT COVERED** | no permission check on the close route; grepped `campaign:manage` — no hits |
| BE-03-02 | AC-4 | FAIL | second close returns 409, AC requires an idempotent 200 |

Three verdicts, and no fourth:

- **PASS** — cited at `file:line`, and you read the code or test and confirmed it asserts the thing.
- **FAIL** — found, and it does not do what the AC says.
- **NOT COVERED** — no evidence found. **This is not a soft pass.** Say what you searched for so a
  naming mismatch can be distinguished from a genuine gap.

**A green suite is not evidence.** A test named for an AC that asserts something else is a FAIL, and
finding those is most of the value here.

For criteria that live in the code rather than in tests — authorization enforced on the route,
validation applied to the field, the audit entry written, the index created, the error envelope
matching — verify in the source directly.

## 3. Check what surrounds the criteria

Beyond the AC list, verify and report:

- **the OWASP rows** the epic's stories carry — each one is an AC and gets a row;
- **compliance obligations** from the profile's packs where the epic touches classified data;
- **the API contract** — every endpoint the epic added is in `docs/API.md`, with the real shapes;
- **the schema** — the delivered models match `SCHEMA.md`, including indexes; report any drift;
- **scope creep** — behaviour delivered that no approved story asked for. It is as much a finding
  as a gap, because it was neither specified nor agreed, and nobody will maintain it knowingly.

## 4. Demonstrate what a table cannot

Some criteria are only provable by exercising the thing. Where the profile makes it possible, run
the app and issue the real requests, or drive the real screen, and paste the actual
request/response. For UI criteria, a screenshot per state — loaded, empty, error — beats any
assertion about a component tree.

Where you cannot demonstrate a criterion, say so and name what a human must check by hand. Never
convert an inability to verify into a PASS.

## 5. The human sign-off gate

Write `docs/delivery/VALIDATION.md`: the headline count, the gate output, the full matrix, the
findings from step 3, and the manual checks outstanding.

Then take the human through it in the profile's verbosity register:

1. the headline — `31 PASS · 2 FAIL · 5 NOT COVERED`;
2. every FAIL and NOT COVERED, with what it would take to close;
3. the manual checks only they can do;
4. scope delivered that nobody asked for.

Ask for sign-off with `AskUserQuestion`, per epic: *accept* / *accept with these known gaps
recorded* / *reject, fix these first*. On acceptance, set `status: validated`, `validated_by` and
`validated_on` in the epic header, and record accepted gaps **in the epic file** — an accepted gap
that only lives in a chat message is a gap that gets rediscovered as a bug.

## 6. Report

The matrix summary, the sign-off outcome, accepted gaps, and the next command — `/deploy` when the
epic is validated, or `/deep-review` if code quality has not been assessed yet.

## Standing rules

- **Never mark PASS without citing evidence you actually read.** A plausible function name is not
  evidence.
- **Never sign off yourself.** The whole point of this gate is that a human accepts.
- Never fix what you find here — that reopens the build and invalidates the matrix. Report it, and
  let the user decide whether it returns to `/build-module`.
- Never soften a NOT COVERED because the rest of the epic looks good.
