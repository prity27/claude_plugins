# `CLAUDE.md` — template

> Written by `/write-docs`. Generated from `docs/delivery/PROFILE.md` plus verified code, or
> proposed as a diff against an existing hand-written file — **never overwritten.**
>
> Roughly one page. It is read on every agent session, so length is a running cost.
> Delete any section that does not apply rather than filling it with something plausible.

---

# <project> — working rules

<One or two sentences: what this system does, in domain terms. Then, if the project is a reference
or has an unusual purpose, say so.>

## Layout

```
<dir>/    <what lives here, one line>          (<package or module name>)
<dir>/    <what lives here, one line>
docs/     delivery artefacts — profile, knowledge graph, stories, schema, validation
```

<Any repository-level rule: workspace root vs package, where to run installs, monorepo layout.>

## Commands

| Command | Does | Run from |
| --- | --- | --- |
| `<dev command>` | | |
| `<gate command>` | the full gate: <what it actually runs> | |
| `<test command>` | | |

Copy these from the profile's **gate commands** row — the ones that were **run**, not the ones the
manifest claims. Where a command fails by design, say so here and say why, or every reader treats it
as broken.

<State plainly whether there is a build step. "There is no build step" prevents an invented one.>

## Conventions

Group by area. Each rule states the rule; where a count matters, give it — "22 of 24 modules follow
this" is a convention, "we generally do this" is not.

**<Area, e.g. Server>**

- <Rule.> <One line of why, when the choice is not obvious.>
- Errors: <how they are raised and where they are translated>.
- <The response or result shape, exactly.>
- <Where configuration and environment values may be read, and that anywhere else is a defect.>

**<Area, e.g. Client>**

- <Rule.>

**Both**

- <Cross-cutting rules: secrets, logging, naming.>

## Delivery process

<Only where the project uses the delivery chain. Otherwise delete this section.>

Scope is not decided in a chat. It lives in `docs/`, and each artefact is produced by a skill that
cites its source:

| Artefact | Written by | Is |
| --- | --- | --- |
| `docs/delivery/PROFILE.md` | `/project-setup` | the stack, gates, hosts and security posture |
| `docs/knowledge/graph.json` | `/ingest-knowledge` | sourced entities, actors, processes, constraints |
| `docs/delivery/stories/*.md` | `/write-stories` | epics with Given/When/Then criteria |
| `docs/delivery/SCHEMA.md` | `/design-schema` | the data model, with an ERD |
| `docs/delivery/VALIDATION.md` | `/validate-delivery` | every criterion mapped to evidence |

Rules that follow:

- **Do not build an epic whose story file is not `status: approved`.**
- **Do not invent a requirement.** A gap becomes an entry in `docs/knowledge/OPEN-QUESTIONS.md`,
  not a reasonable default.
- Every claim in the delivery docs cites where it came from — a document and a quote, or a
  `file:line`.

## Known state, deliberately

> **The highest-value section in this file, and the easiest to omit.** Copy the profile's
> **known exceptions** row verbatim. Anything listed here is a recorded decision, and a reviewer —
> human or agent — that reports it as a defect is wasting everyone's time. Date it, because a
> deliberate gap and a forgotten one look identical after six months.

Do not report these as defects. Current as of <YYYY-MM-DD>:

- <Deliberate deviation, and what closes it.> <e.g. "No test runner. `npm test` exits 1 by design;
  `/write-tests` stands one up when the first epic needs it.">
- <Deliberate deviation, with the condition that makes it *stop* being acceptable.> <e.g. "No
  authentication yet — no write routes exist either. A write route merged without an auth check is
  a blocking defect, not a follow-up.">
- <A structural constraint and its consequence.> <e.g. "No migrations. The ORM models are the only
  schema, so a field rename is a hand-written data script — say so when proposing one.">
- <A technology choice someone will otherwise try to 'fix'.>

The second bullet's shape is worth copying: a deliberate gap plus **the condition under which it
becomes a defect**. An exception with no expiry becomes permanent by accident.
