---
name: onboard-existing
description: Bring an already-built or half-built codebase under the delivery gates — inventory what exists with file:line evidence, derive a knowledge graph from the code marked as observed rather than agreed, back-fill stories for what was already delivered, and reconcile the code against whatever scope documents survive. Language and framework agnostic. Use on a partial or mature repository before /write-stories, /design-schema or /validate-delivery can mean anything.
argument-hint: "[backend | frontend | all] [module or path] — default: all, the repos in the profile"
allowed-tools: Read, Write, Edit, Glob, Grep, AskUserQuestion, Agent, Bash(ls:*), Bash(cat:*), Bash(grep:*), Bash(find:*), Bash(head:*), Bash(wc:*), Bash(test:*), Bash(git log:*), Bash(git ls-files:*), Bash(git blame:*)
---

# Onboard an existing codebase

The rest of this plugin assumes scope came first and code came second. Most real projects arrive the
other way round: there is a repository, some of it works, the person who wrote it has moved on, and
the scope document — if it exists — stopped matching reality months ago.

This skill produces the missing first half **without pretending the code is a requirement**. It
answers "what is here" precisely, and refuses to answer "what was wanted" on the code's behalf.

Read `docs/delivery/PROFILE.md` first. Missing → `/project-setup`. If its **stage** row says
`greenfield` or `scaffolded`, stop and say so: there is nothing to onboard, and `/ingest-knowledge`
or `/write-stories` is the right command.

## The rule that governs everything here

**Code is evidence of behaviour, never evidence of intent.**

A field exists. A route returns 403. A job runs nightly. All of that is fact, and you cite it with
`file:line`. Whether any of it is *correct*, *wanted*, or *still wanted* is not in the repository —
it is in someone's head, and the only honest move is to ask.

So every claim this skill derives from code is written with `confidence: "observed"`, and every
`observed` claim that a story or a schema would depend on becomes an open question. The failure this
prevents is specific and expensive: a half-finished feature gets read as a specification, gets
written into stories, gets built out faithfully, and nobody notices for two months that the original
developer abandoned it because the client changed their mind.

## 1. Inventory — what is actually here

Work outside-in, because the edges of a system are enumerable and its middle is not. For each repo
in the profile, find the surfaces first, then follow each one inward.

```bash
git ls-files | sed 's|/[^/]*$||' | sort | uniq -c | sort -rn | head -30   # where the code lives
```

Then locate, in whatever form the stack expresses them:

| Surface | Look for | Record |
| --- | --- | --- |
| HTTP endpoints | route registrations, controller annotations, framework decorators | method, path, auth applied, handler `file:line` |
| Persisted entities | ORM models, schema files, migrations, table DDL | name, fields, `file:line` |
| Background work | schedulers, queue consumers, cron definitions | trigger, what it mutates |
| External integrations | SDK clients, outbound HTTP, webhook receivers | which system, in which direction |
| Screens / views | router config, page components, templates | route, which endpoints it calls |
| Auth and roles | middleware, guards, policy checks, role enums | the actual enforcement points |

For each surface, follow one path all the way inward — route to handler to data access to model —
before recording it. A route listed from a routing table alone is a claim; a route you traced to the
write it performs is a fact.

**Three states, and the distinction is the whole point:**

- **Working** — traced end to end, and nothing on the path is a stub.
- **Partial** — reachable, but something on the path is unimplemented, hardcoded, or always returns
  the same value.
- **Dead** — defined and unreachable: no route registers it, no caller imports it, the feature flag
  is off with no path to on.

```bash
grep -rn "TODO\|FIXME\|XXX\|HACK\|not implemented\|NotImplementedError\|panic(\"unimplemented" \
  --exclude-dir=node_modules --exclude-dir=.venv --exclude-dir=target --exclude-dir=dist . | head -50
```

Do not fix, tidy, or delete anything. This skill observes. Remediation becomes work in
`/write-stories`, where a human decides whether it is worth doing.

Write the result to `docs/delivery/AS-BUILT.md` using
`${CLAUDE_PLUGIN_ROOT}/references/templates/as-built.md`.

## 2. Derive the graph from the code

Extend `docs/knowledge/graph.json` per `${CLAUDE_PLUGIN_ROOT}/references/templates/graph-schema.md`
— extend, never overwrite. If a graph already exists from `/ingest-knowledge`, its `stated` entities
outrank anything you derive here.

Register each file you cite as a source with `"kind": "source-code"`, and cite it with a `loc` line
range like any other source. Then:

- **entities** from persisted models — fields with the types the code enforces, not the types you
  would choose;
- **actors** from the role enum and the authorization checks that reference it;
- **processes** from the multi-step writes: what a handler does in sequence, and — important —
  whether it is transactional today;
- **constraints** from validation that is actually enforced at runtime. A constraint in a comment is
  not enforced; say so;
- **relations** from foreign keys, references and joins, with the cardinality the code implies.

Everything above is `confidence: "observed"`. Nothing here is `stated` — no one stated it, you read
it. Where the code's shape is ambiguous (a nullable field that is never null in practice, an enum
value nothing sets), record the ambiguity as an open question rather than picking the tidy reading.

## 3. Reconcile against whatever scope documents exist

If `docs/knowledge/sources/` has anything in it, compare the two pictures and produce three lists —
this is where the real value of the skill lands:

1. **Built and agreed** — `observed` matches `stated`. Cite both. These are safe.
2. **Built but never agreed** — in the code, in no document. Someone decided this. It may be
   essential and undocumented, or it may be scope nobody bought. **A human must say which**, and
   until they do it does not become a story.
3. **Agreed but never built** — in a document, absent from the code. This is the backlog, and it is
   usually the most valuable output of the whole skill.

Where the code and a document disagree about the same thing, that is a conflict entry in
`openQuestions`, with both sides quoted. Do not resolve it by preferring the code.

## 4. Back-fill stories for what exists

Only for the **built and agreed** list, and for **built but never agreed** items a human has since
confirmed. Write them per `${CLAUDE_PLUGIN_ROOT}/references/story-template.md`, into
`docs/delivery/stories/`, with two differences from a normal story:

- `status: as-built` — the code exists; the criteria were reverse-engineered from it, not agreed
  before it. `/build-module` must refuse an `as-built` story exactly as it refuses an unapproved
  one: there is nothing to build.
- Each acceptance criterion carries the `file:line` it was derived from, and is marked
  **unverified** — it describes what the code appears to do, not something you proved. Proving it is
  `/validate-delivery`'s job, and running it against these stories is the natural next step.

Write Given/When/Then from the **observable behaviour**, not from the implementation. "Returns 403
when the requester does not own the record" is a criterion; "calls `assertOwner()`" is a restatement
of the code and proves nothing.

Where a story would need an answer you do not have, leave the criterion out and list it as a gap. A
back-filled story with an invented criterion is worse than a missing one, because `/validate-delivery`
will later mark it PASS against code that was never checked against anyone's intent.

## 5. Gate

Do not proceed without a human. Put these in front of them:

- the inventory headline: how many surfaces, and how many are working / partial / dead;
- **built but never agreed** — item by item, because each one is a decision;
- **agreed but never built** — the backlog this uncovered;
- every conflict between code and document;
- the open questions blocking a schema or a story, ordered by what they block.

Use `AskUserQuestion`, batched, and record each answer in the graph with `answeredBy` and
`answeredOn`. Then report the next command: `/design-schema` to formalise the model you derived,
`/write-stories` for the never-built backlog, or `/validate-delivery` to check the back-filled
criteria against the code that claims to satisfy them.

## Standing rules

- **Never write `stated` for something you read in code.** `observed` exists precisely so that a
  later reader can tell the difference between what was agreed and what was found.
- **Never change code here.** Not a typo, not a lint error, not an obvious bug. Record it.
- Never delete or rewrite an existing graph, story or profile — extend, and report what changed.
- Never infer intent from a commit message, a variable name, or a comment. They are hints worth
  quoting and are not evidence.
- If a live credential is in a tracked file, say so prominently: deleting the file is not
  remediation, because git history keeps it. **It must be rotated.**
