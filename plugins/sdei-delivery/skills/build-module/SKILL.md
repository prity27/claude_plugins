---
name: build-module
description: Turn one approved epic into a plan-mode brief and then build it as a complete vertical slice — model through endpoint or component through screen — rather than as isolated functions. Carries the acceptance criteria, the security requirements and the gate commands into the build. Use to implement an approved epic or story.
argument-hint: "<epic id or story id> — e.g. BE-03 or BE-03-02"
allowed-tools: Read, Write, Edit, Glob, Grep, AskUserQuestion, Bash
---

# Build module

Build one approved epic as a working slice. The unit of work is a **module** — everything needed
for a user to exercise the behaviour end to end — not a function, not a layer.

Building by layer produces four half-finished layers and nothing demonstrable. Building by slice
produces something a human can accept against its acceptance criteria on the day it is written,
which is the only way `/validate-delivery` can mean anything.

## 0. Preconditions — refuse to start without them

1. `docs/delivery/PROFILE.md` exists. Missing → `/project-setup`.
2. The epic file exists and its header says `status: approved`. **Anything else is a stop.**
   `as-built` in particular is a stop, not a shortcut: the code is already there, and the criteria
   were derived from it rather than agreed. Promoting it to `approved` is a human decision made in
   `/onboard-existing`'s gate, not here.
   `draft` or `in-review` means the scope is not settled, and building it wastes the build.
3. Where the epic creates or changes persisted data, `SCHEMA.md` says `status: validated`.
4. No open question in `OPEN-QUESTIONS.md` `blocks` an entity this epic touches.

State which precondition failed and what unblocks it. Do not proceed on a partial pass.

## 1. Read the ground before proposing anything

- The epic's stories and every acceptance criterion — these are the specification, and nothing else
  is;
- `SCHEMA.md` for the entities involved;
- the project's own conventions — `CLAUDE.md`, `CONTRIBUTING.md`, `docs/` — which **outrank every
  generic pattern you know**;
- **two or three sibling modules that already do this**. The reference for how to build here is the
  neighbouring file, not a framework tutorial. Note the file set, the export style, the error
  handling, the response shape, the import conventions, the test placement;
- the OWASP rows that touch this slice, plus any compliance pack the profile names.

## 2. Write the plan-mode brief

The brief is the deliverable of this phase. It is specific enough that someone else could build the
slice from it, and it goes to the user for approval before any file is written.

```
Epic BE-03 — Campaign lifecycle · backend
Stories: BE-03-01 (create), BE-03-02 (close), BE-03-03 (list)

Files
  new    src/api/campaign/campaign.types.ts        request and response shapes
  new    src/api/campaign/campaign.service.ts      lifecycle rules, returns ServiceResponse
  new    src/api/campaign/campaign.controller.ts   destructure, one service call, hand off
  new    src/api/campaign/campaign.validator.ts    body and query rules per AC
  new    src/api/campaign/campaign.router.ts       auth → authz → audit → validate → handler
  edit   src/models/campaign.model.ts              add closedAt, closedBy  (SCHEMA.md §Campaign)

Contracts
  service.close(campaignId: string, actorId: string): Promise<ServiceResponse<Campaign>>
    → 409 when an open reception exists (AC-2); idempotent on an already-closed campaign (AC-4)

Acceptance criteria this slice must satisfy
  BE-03-02 AC-1..AC-4, BE-03-01 AC-1..AC-3

Security requirements for this slice
  A01 object-level check: the actor's grower must own the campaign  (BE-03-02 AC-3)
  A01 route declares its permission explicitly
  A03 no raw request object reaches a query filter
  A09 the close writes an audit entry with actor, target, action

Atomicity
  SCHEMA.md flags proc-campaign-close as a multi-document write with no transaction decision.
  → this is a question for the user, not a default. Ask before building.

Gates
  <the profile's gate commands, verbatim — e.g. npm run typecheck>

Out of scope
  reopening a closed campaign — non-goal-reopen
```

Every section is load-bearing. A brief without the AC list produces code nobody can validate; one
without the security section produces the authorization gap found later by a reviewer.

**Where the brief hits a documented project inconsistency** — two competing patterns, an unresolved
convention — do not pick a side silently. Name it, state which way the file you are editing goes,
and ask if it is a real fork.

## 3. Refuse work that is smaller than a slice

If what you were asked for is one function, say so and propose the slice it belongs to. A slice
means: the data layer, the business logic, the entry point, the validation, the authorization, the
documentation and the test — as much of that as the project's own architecture calls for.

The one exception is a genuine one-line fix to existing behaviour. Say that is what it is.

## 4. Build, bottom-up

Work in dependency order so each layer compiles against a real signature: types → data → business
logic → entry point → validation → wiring → documentation → test.

While building:

- **Match the sibling module exactly.** Same file set, same export style, same error handling, same
  naming, same import conventions. Consistency beats your preference every time.
- Implement the acceptance criteria, including the unhappy ones. The 403 case and the
  already-in-that-state case are criteria, not polish.
- Reuse what exists — the project's own helpers, wrappers and middleware. Grep before you write a
  utility; a second implementation of something already present is a defect.
- Do not fix unrelated problems you pass. Note them and move on. A slice that also refactors three
  neighbours cannot be reviewed.

## 5. Gate before you claim anything

Run the profile's gate commands. Report the real output. A slice with a failing typecheck is not
built, however complete it looks.

Then map each acceptance criterion to what implements it, and name the ones you did **not**
implement and why. Never imply completeness you cannot demonstrate.

## 6. Hand off

Report: files created and changed, the AC map, gate results verbatim, the questions you had to ask
and the answers you built on, anything left undone, and the next commands —
`/write-tests <epic>`, then `/validate-delivery <epic>`, and `/deep-review` from `sdei-review` for
code quality.

## Standing rules

- **Never build an unapproved epic.** The approval gate is the only thing making acceptance real.
- Never invent a requirement the stories do not state. Ask.
- Never claim done without the gate output.
- Never migrate a module's style as a side effect of adding to it.
