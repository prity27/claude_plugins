# User story format

One file per epic at `docs/delivery/stories/<NN>-<epic-slug>.md`. Stories are numbered inside the
epic and never renumbered — an id is a permanent handle that tests, commits and the validation
matrix all point at.

## Epic file header

```markdown
---
epic: BE-03
title: Campaign lifecycle
unit: backend            # backend | frontend | infra | shared
status: draft            # draft | in-review | approved | built | validated
approved_by:             # a human name, set only at the /write-stories gate
approved_on:
graph_entities: [ent-campaign, ent-crew, proc-campaign-close]
depends_on: [BE-01]
---
```

`status` is the human gate made mechanical. Nothing downstream — `/build-module`, `/write-tests` —
acts on an epic that is not `approved`.

## Story format

```markdown
### BE-03-02 — Close a campaign

**As a** grower administrator
**I want** to close a campaign that has finished harvesting
**So that** no further crew assignments or receptions can be recorded against it

**Source:** `ent-campaign` · discovery call 2026-03-04 L204 — "once it's closed nothing else
lands on it, that's the whole point"

**Acceptance criteria**

- **AC-1** Given an active campaign with no open receptions
  When the administrator closes it
  Then its status becomes `closed`
  And the closed-at timestamp and closing actor are recorded

- **AC-2** Given a campaign with an open reception
  When the administrator closes it
  Then the request fails with 409 and the standard error envelope
  And the campaign remains `active`

- **AC-3** Given a user without the campaign-management permission
  When they attempt to close any campaign
  Then the response is 403
  And the attempt is written to the audit log

- **AC-4** Given a campaign that is already `closed`
  When it is closed again
  Then the response is 200 and the state is unchanged  *(idempotent)*

**Out of scope:** reopening a closed campaign — see `non-goal-reopen`.

**Estimate:** M
**Notes:** the close is a multi-document write; see `/design-schema` atomicity note for `proc-campaign-close`.
```

## Rules that make these worth writing

**Every AC is Given / When / Then, and every AC is mechanically checkable.** If you cannot name the
test or the request that proves it, it is not an acceptance criterion — it is a hope. "Is
performant" fails; "returns within 500ms for a 10,000-row campaign" passes.

**Every story cites the graph.** `Source:` names the entity or process id and the quote it came
from. A story with no source is scope someone invented — flag it rather than shipping it.

**Cover the unhappy paths.** A story with only a success AC is half written. Every story needs, at
minimum: the success case, the invalid-input case, the unauthorised case, and the
already-in-that-state case where it applies.

**Authorization is an AC, never a separate story.** The moment security is its own backlog item it
gets deprioritised. Fold the relevant `references/owasp-checklist.md` rows into the AC list of the
story they belong to.

**Vertical slices, not layers.** "Create the campaign model" is not a story — nobody can accept it.
"Create a campaign" spanning model through endpoint is. Frontend and backend may be separate units
of work, but each is still a slice a human can accept.

**Size to a slice.** If a story cannot be built and validated in one sitting, split it on the
behaviour, never on the layer.

## Sizing

`S` — one endpoint or one screen, no new entity. `M` — a slice touching a new entity or a new flow.
`L` — split it. `L` is not an estimate, it is a signal that the story is really two.

## The baseline epic

Every project gets `00-foundation` before feature epics, covering infrastructure, migrations, auth
scaffolding and basic CRUD. `/write-stories` generates it from
`references/templates/baseline-epic.md` — it exists because these are the stories teams skip and
then retrofit at ten times the cost.
