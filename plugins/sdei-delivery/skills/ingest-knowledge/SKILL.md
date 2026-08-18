---
name: ingest-knowledge
description: Turn pre-sales material — discovery-call transcripts, chat threads, proposal documents, SOWs, emails, wireframe notes — into a committed knowledge graph of entities, relations, decisions, constraints and open questions, each traceable to the source that said it. Then close the gaps by asking. Use before writing user stories or designing a schema.
argument-hint: "[path to the folder of source material] — default: docs/knowledge/inbox/"
allowed-tools: Read, Write, Edit, Glob, Grep, AskUserQuestion, Bash(ls:*), Bash(find:*), Bash(file:*), Bash(wc:*), Bash(head:*), Bash(cat:*), Bash(test:*)
---

# Ingest knowledge

Pre-sales material is where the real requirements live, and it is the first thing lost. This skill
converts it into `docs/knowledge/graph.json` — a committed, diffable, **sourced** model of what the
client actually asked for, plus an explicit list of what nobody has answered yet.

The value is entirely in the provenance. A graph that says "campaigns have crews" is worth little.
One that says "campaigns have crews — CTO, discovery call, 4 March, line 140" survives the argument
in month three.

Read `${CLAUDE_PLUGIN_ROOT}/references/templates/graph-schema.md` before writing anything.

## 0. Preconditions

Read `docs/delivery/PROFILE.md`. If it is missing, stop: **run `/project-setup` first.** The
profile's domain-expertise field decides how hard you interrogate domain terms, and its compliance
field decides which data-handling questions you must ask.

**Claude Code cannot read audio or video.** An `.mp3`, `.mp4`, `.m4a` or `.webm` recording must be
transcribed before it reaches you. Say this immediately and list what you can take:

| Accepted | Notes |
| --- | --- |
| `.txt`, `.md` | anything, including exported chat threads |
| `.vtt`, `.srt` | meeting transcripts with timestamps — keep the timestamps, they make good `loc` pointers |
| `.pdf` | read directly, page by page |
| `.docx`, `.xlsx`, `.pptx` | convert to text first; say so rather than guessing at contents |
| screenshots, wireframes | read as images; describe what they specify, mark `implied` |

If a recording exists with no transcript, list it as a **missing source** in the report and carry
on. Never infer the contents of a file you could not read.

## 1. Inventory before reading

List every file, its type, its size and its date. Show the inventory and the reading order —
highest authority first, so later low-authority sources are read as *deltas* against what is
already settled rather than as fresh claims.

Rank authority: signed proposal or SOW > written decision or requirements doc > call transcript >
chat thread > email aside. Ranking orders the reading; it never resolves a contradiction.

## 2. Digest each source

For every source, write `docs/knowledge/sources/<YYYY-MM-DD>-<slug>.md`:

- a header with kind, date, participants, authority;
- a **factual digest** — what this source establishes, in its own terms, with line or timestamp
  pointers;
- **verbatim quotes** for anything that will become a requirement, a constraint or a number;
- a "said but vague" list — the things that sound like requirements and are not yet specific enough
  to build against.

Do not editorialise, do not merge sources, and do not resolve anything at this stage. One file per
source, faithful to that source.

## 3. Extract into the graph

Now build `docs/knowledge/graph.json` per the schema reference. Work through each digest and emit:

- **entities** — the nouns that get persisted, with attributes and types where stated;
- **actors** — every role that acts on the system; these become the "As a …" clause of every story;
- **systems** — external integrations, with what crosses the boundary in each direction;
- **processes** — workflows spanning entities. Capture the **transactional** detail explicitly:
  what must succeed or fail together, what the volumes are, what the frequency is, what happens on
  retry. `/design-schema` and the atomicity questions depend entirely on this;
- **terms** — domain vocabulary. If the profile says domain expertise is `none`, every term needs a
  human-confirmed definition before a story is built on it;
- **constraints** — legal, regulatory, volumetric, temporal, budgetary;
- **decisions** — anything already settled. These are never re-litigated later, only cited;
- **non-goals** — explicitly out of scope. These are the cheapest artefact here and the most useful
  in month four.

Mark every item `stated`, `implied` or `assumed`, and give every one a source with a quote.
**Every `assumed` item must also become an open question** — that invariant is what stops the graph
from quietly becoming fiction.

## 4. File the conflicts — do not resolve them

Where two sources disagree, write an entry in `openQuestions` carrying **both positions with their
citations**, and what the answer `blocks`. Never pick the more recent one, the more senior speaker,
or the more convenient reading. You are not authorised to decide scope.

Also raise a question for: a missing cardinality, an entity with no owner or lifecycle, a process
with no failure path, a number nobody gave (retention, volume, concurrency), any PII or PHI whose
handling is unstated, and any acceptance expectation stated as a feeling ("it should be fast").

## 5. Ask — in priority order

Render `docs/knowledge/OPEN-QUESTIONS.md`, ordered by how much each answer unblocks, then run the
clarification loop with `AskUserQuestion`, four at a time, highest-cost first.

Make the cost visible in the question. *"Can a crew work two campaigns at once? This decides 1:N
versus M:N, and changes the schema plus four stories"* gets a real answer. *"Tell me about crews"*
does not. Where you can, offer the candidate answers you already extracted, with their sources, as
the options.

Fold each answer back in: update the entity or relation, flip `confidence` to `stated`, set the
question's `status`, `answer`, `answeredBy` and `answeredOn`. Keep answered questions in the file —
the record of *why* something is the way it is is the point.

Stop asking when the remaining questions no longer block schema or stories, and say which ones you
are deliberately leaving open.

## 6. Verify the invariants, then report

Check all six invariants in the schema reference — dangling source ids, dangling relation ends,
unsourced claims, `assumed` items with no question, and entities dropped since the last run. Fix
what you can, report what you cannot.

Report: sources ingested and skipped (with the reason), counts by entity type, the conflict list
and how each was resolved, questions still open and what they block, and the next command —
`/write-stories`, or `/design-schema` if the data model is the immediate need.

## Standing rules

- **No claim without a source.** If you cannot cite it, it does not go in the graph — it goes in
  `openQuestions`.
- **Never resolve a contradiction yourself**, however obvious the answer looks.
- Re-ingesting merges by `id` and reports what changed. Never silently drop an entity a human has
  already reviewed.
- Source material often contains credentials, client PII and commercial terms. Digests are
  committed to the repo — **redact secrets and personal data as you digest**, and say what you
  redacted.
