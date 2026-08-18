---
name: write-stories
description: Turn the knowledge graph into epics and user stories with Given/When/Then acceptance criteria, split into backend, frontend and infrastructure units, including the baseline epic for infra, migrations, auth scaffolding and CRUD. Ends with a human sign-off gate on coverage and correctness. Use after /ingest-knowledge, or whenever scope needs to become buildable work.
argument-hint: "[epic name or 'baseline' or 'all'] — default: all epics not yet drafted"
allowed-tools: Read, Write, Edit, Glob, Grep, AskUserQuestion, Agent, Bash(ls:*), Bash(cat:*), Bash(test:*), Bash(grep:*)
---

# Write stories

Convert `docs/knowledge/graph.json` into epics and stories at `docs/delivery/stories/`, each
acceptance criterion mechanically checkable and each story traceable to the source that asked for
it. Nothing is built from an epic until a human marks it `approved`.

Formats and rules: `${CLAUDE_PLUGIN_ROOT}/references/story-template.md`. Baseline content:
`${CLAUDE_PLUGIN_ROOT}/references/templates/baseline-epic.md`. Security rows:
`${CLAUDE_PLUGIN_ROOT}/references/owasp-checklist.md`.

## 0. Preconditions

Read `docs/delivery/PROFILE.md` — missing means **run `/project-setup` first**. Read
`docs/knowledge/graph.json` — missing means **run `/ingest-knowledge` first**; do not write stories
from a conversation, because a story with no source cannot be defended later.

Check `OPEN-QUESTIONS.md`. A question whose `blocks` list names an entity you are about to write
stories for is a stop: **ask it now**, or write the story and mark it `blocked` with the question
id. Never quietly pick an answer.

## 1. Cut the epics

Group graph entities and processes into epics along **user-visible capability**, never along
technical layer. "Campaign lifecycle" is an epic; "controllers" is not.

Then split each epic into units — `backend`, `frontend`, `infra`, `shared` — because these are
scheduled and validated separately. Same capability, same epic number, different unit.

Order by dependency: `00-foundation` always first, then epics whose entities other epics reference.
Record `depends_on` in the header so the build order is derivable rather than remembered.

Show the proposed epic list and get agreement **before** drafting stories. Restructuring epics after
forty stories exist is expensive; the list is cheap to argue about.

## 2. Generate the baseline epic first

`00-foundation` comes from the baseline template, adapted to the profile's actual stack. Every story
in it is generated. Drop one only where the profile proves it already exists, and say which you
dropped and why.

This is not filler. Auth scaffolding, migrations, backups, logging and pagination are exactly what
gets skipped and then retrofitted under pressure.

## 3. Draft the feature epics

Dispatch the `story-writer` agent — one per epic, in parallel, single message. Give each: the
profile, the graph slice its epic covers (entities, actors, processes, constraints, decisions,
non-goals), the story template, and the OWASP rows that touch its capability.

Each story must carry:

- the actor from a graph `actor`, never invented;
- a `Source:` citation with the graph id and the quote;
- acceptance criteria covering **success, invalid input, unauthorised, and already-in-that-state**;
- authorization folded in as an AC — never a separate security story;
- explicit out-of-scope lines where a graph `non-goal` is adjacent.

Where the graph is silent on something a story needs, the agent does not invent it — it emits an
open question, which you add to `OPEN-QUESTIONS.md` and put to the user.

## 4. Audit before showing a human

Dispatch `acceptance-validator` against the full story set. Its job is to find what is missing, and
it reports:

- **coverage** — every graph entity, actor, process and constraint mapped to at least one story;
  anything unmapped listed by id;
- **invention** — every story with no source, or whose scope exceeds what its source says;
- **untestable criteria** — any AC you could not write a test for;
- **missing unhappy paths** — stories with only a success case;
- **cross-unit gaps** — a backend story with no frontend counterpart where the graph implies a UI,
  and the reverse;
- **contradictions** — two stories requiring incompatible behaviour.

Fix what is mechanical. Everything else goes to the human gate as a question, not a decision.

## 5. The human gate — the point of this skill

Present, compactly:

1. the **coverage matrix** — graph id → story ids, with unmapped rows called out first;
2. the **gap list** — what is deliberately not covered, and why;
3. the **question list** — every place the agents wanted to invent scope;
4. per epic: story count, size mix, and dependencies.

Then ask for sign-off **per epic**, not for the set — an approver who must accept forty stories at
once accepts none of them. Use `AskUserQuestion` and offer *approve* / *approve with the noted
changes* / *needs rework, here is what is wrong*.

On approval, set `status: approved`, `approved_by` and `approved_on` in that epic's header. That
field is the interlock: `/build-module` and `/write-tests` refuse to act on anything else.

Say plainly, in the profile's verbosity register, what approval means: *this is the scope, these are
the tests that will be written, and this is what "done" will be measured against.*

## 6. Report

Epics written and their status, total stories by unit, coverage percentage against the graph, open
questions still blocking, and the next command — `/design-schema` if the data model is not settled,
otherwise `/build-module <epic>`.

## Standing rules

- **A story with no graph source is invented scope.** Flag it; never ship it silently.
- Never mark an epic `approved` yourself. That field means a human read it.
- Do not renumber stories. Ids are permanent handles used by tests, commits and the validation
  matrix. Superseded stories get `status: superseded` and stay.
- Re-running merges: existing approved epics are left alone unless the user asks, and changes to an
  approved epic reset it to `in-review` with the diff shown.
