---
name: design-schema
description: Derive the database model from the knowledge graph's entities and transactional detail — fields with types, required-ness, defaults, indexes, PII/PHI classification and retention, plus relationships with cardinality and a mermaid ERD. Ends with a human validation gate on completeness and correctness, and only then emits model code in the project's stack.
argument-hint: "[entity name] — default: the whole model"
allowed-tools: Read, Write, Edit, Glob, Grep, AskUserQuestion, Agent, Bash(ls:*), Bash(cat:*), Bash(grep:*), Bash(test:*), Bash(find:*)
---

# Design schema

Produce `docs/delivery/SCHEMA.md` from `docs/knowledge/graph.json` — the entity model, its
relationships, its indexes, and its data classification — and get it validated by a human **before
a single model file is written**.

Schema mistakes are the most expensive class of mistake in the project. A wrong service is a
rewrite of one file; a wrong cardinality or a missing constraint is a data migration, a backfill
and a week of reconciliation. That asymmetry is why this skill has a hard gate in the middle.

Template: `${CLAUDE_PLUGIN_ROOT}/references/templates/schema.md`.

## 0. Preconditions

Read `docs/delivery/PROFILE.md` (missing → `/project-setup`) and `docs/knowledge/graph.json`
(missing → `/ingest-knowledge`). Read the approved stories if any exist — the queries they imply
determine the indexes, and a schema designed without knowing the access patterns is a guess.

**On an existing codebase, read the current models first** and design as a *delta*. Say what
already exists, what changes, and what a change would break. Never present a greenfield model for a
database that has rows in it.

## 1. Derive the entities

Dispatch `schema-designer` with the graph, the profile's database and ORM, and the approved
stories. For every graph entity it proposes:

| Column | Rule |
| --- | --- |
| Field | the name in the project's own casing convention |
| Type | precise, not approximate — see below |
| Required | required, optional, or **unknown** — never guessed |
| Default | stated, or none |
| Unique | including composite uniqueness |
| Indexed | and *why* — which query or sort needs it |
| Class | public / internal / PII / PHI / secret |
| Retention | how long it is kept, and what happens then |
| Source | graph id and quote |

**Type precision is where models rot.** Money is a minor-unit integer or a decimal, never a float.
Timestamps are instants with a stated timezone policy; a "date" that is really a business day in a
local timezone must say so. Enumerations are enumerated, with the full value list. Free text has a
maximum length. An identifier's type and format are stated.

**Never invent a field.** A field the graph does not mention is an open question, not a design
decision. Say which fields you are proposing and which the client actually asked for.

## 2. Derive the relationships

For each relation: both endpoints, cardinality, which side owns the reference, whether it is
required, and what happens on delete — cascade, restrict, orphan, or soft-delete. Every one carries
its graph source.

Then look for what the graph *implies* and never said: a many-to-many hiding behind a repeated
one-to-many, a join entity with its own attributes (assignments, memberships and line items nearly
always have their own fields), a hierarchy needing a parent reference, and any entity with no
lifecycle owner.

Each of those becomes an open question, not an assumption.

## 3. Analyse the transactional detail

Read the graph's `process` entities and answer, per process, in the schema doc:

- **Which writes must succeed or fail together?** Name them. This is the atomicity requirement, and
  it decides whether the project needs transactions — a decision that must be made deliberately,
  with a human, and not discovered later during a partial-write incident.
- **What are the volumes and the growth rate?** A table that grows per-scan behaves differently
  from one that grows per-customer.
- **Which queries run in a hot path?** Those are the indexes that matter.
- **What is concurrent?** Two users acting on one record — is that last-write-wins, rejected, or
  merged? Silence here becomes a race in production.
- **What must never be lost?** That decides soft versus hard delete and the audit requirement.

## 4. Write `SCHEMA.md` and the ERD

Fill the template: entity tables, relationship list, index list with justification, the
transactional analysis, the data-classification summary, and a **mermaid `erDiagram`** showing
every entity and relationship with cardinality. The diagram is for the human gate — a wrong
cardinality is visible in a diagram and invisible in a table.

Where the profile names a compliance regime, apply its pack
(`${CLAUDE_PLUGIN_ROOT}/references/compliance/`) to the classification columns now, not later.

## 5. The human validation gate

Do not ask "does this look right". Walk the checklist, item by item, and record each answer in the
doc:

1. **Completeness** — every graph entity present; every entity a story needs present; nothing here
   that no source asked for.
2. **Cardinality** — read each relationship aloud in both directions. *"A campaign has many crews; a
   crew belongs to exactly one campaign."* Wrong cardinality survives every review except this one.
3. **Required versus optional** — for each required field, is it truly available at creation time?
   This is the single most common cause of a blocked insert in week three.
4. **Type precision** — money, decimals, dates versus timestamps, timezone policy, enum values,
   string limits, identifier formats.
5. **Uniqueness** — what makes a row unique to the business, not just to the database. Composite
   keys included.
6. **Deletes** — per entity: hard, soft, or forbidden; and what happens to its children.
7. **Indexes** — every story filter and sort covered; no index nothing queries.
8. **Classification and retention** — every PII/PHI field labelled, retention stated, deletion and
   export paths defined where the regime requires them.
9. **Atomicity** — the multi-write processes named in step 3, and the decision recorded.

Use `AskUserQuestion` for the genuine forks, offering the candidate answers with their consequences
— *"1:N means a crew cannot be shared; M:N adds a join entity and changes three stories."*

Record the outcome in the doc header: `validated_by`, `validated_on`, and every decision made at the
gate with its rationale. That record is what stops the same argument recurring in month five.

## 6. Emit model code — only after sign-off

With `validated: true`, generate models in the profile's stack: Mongoose schemas, Prisma models, or
DDL plus a migration. Match the existing codebase's conventions exactly — read a sibling model
first and copy its shape, including its guards, timestamps and interface style.

Where the profile says there are no migrations, say explicitly what deploying this change requires
and what happens to existing rows. A schema change with no migration path is not finished.

## Standing rules

- **No model code before the gate.** Code makes a proposal feel settled, and the gate stops working.
- Never invent a field, an entity or a cardinality. Unknowns are questions.
- Never present a greenfield design for a live database. Delta, impact, migration.
- Do not restructure an existing model as a side effect of adding to it. Propose it separately.
