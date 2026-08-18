# `docs/knowledge/graph.json` — shape and rules

One file, four arrays, every claim traceable to a source. Written by `/ingest-knowledge`, read by
`/write-stories`, `/design-schema`, `/write-docs` and `/validate-delivery`.

```json
{
  "version": 1,
  "project": "Harvest Ops",
  "generated": "2026-03-11",
  "sources": [
    {
      "id": "src-discovery-1",
      "path": "docs/knowledge/sources/2026-03-04-discovery-call.md",
      "kind": "call-transcript",
      "date": "2026-03-04",
      "participants": ["client CTO", "delivery lead"],
      "authority": "high"
    }
  ],
  "entities": [
    {
      "id": "ent-campaign",
      "type": "entity",
      "name": "Campaign",
      "description": "A harvest run scoped to one grower, one crop and a date window.",
      "attributes": [
        { "name": "campaignName", "type": "string", "required": true, "confidence": "stated" },
        { "name": "startDate", "type": "date", "required": true, "confidence": "stated" },
        { "name": "status", "type": "enum", "values": ["draft", "active", "closed"], "confidence": "implied" }
      ],
      "confidence": "stated",
      "sources": [
        { "id": "src-discovery-1", "loc": "L118-L131", "quote": "every campaign is one grower, one crop, and it opens and closes on dates" }
      ]
    }
  ],
  "relations": [
    {
      "id": "rel-campaign-crew",
      "from": "ent-campaign",
      "to": "ent-crew",
      "type": "has-many",
      "cardinality": "1:N",
      "confidence": "stated",
      "sources": [{ "id": "src-discovery-1", "loc": "L140", "quote": "a campaign runs several crews" }]
    }
  ],
  "openQuestions": [
    {
      "id": "q-1",
      "question": "Can a crew work two campaigns in the same week?",
      "why": "Decides whether crew↔campaign is 1:N or M:N, which changes the schema and four stories.",
      "blocks": ["rel-campaign-crew", "ent-crew"],
      "conflict": [
        { "source": "src-discovery-1", "says": "a crew belongs to one campaign" },
        { "source": "src-proposal-v2", "says": "crews are shared across concurrent campaigns" }
      ],
      "status": "open",
      "answer": null,
      "answeredBy": null,
      "answeredOn": null
    }
  ]
}
```

## Field rules

**`type`** on an entity is one of:

| Value | Means | Consumed by |
| --- | --- | --- |
| `entity` | a thing that gets persisted | `/design-schema` |
| `actor` | a human or role that acts on the system | `/write-stories` (the "As a …" clause) |
| `system` | an external system or integration | stories + architecture doc |
| `process` | a workflow or transaction spanning entities | stories + the transactional analysis |
| `term` | domain vocabulary needing a definition | the glossary; gated when domain expertise is `none` |
| `constraint` | a rule the build must honour (legal, volumetric, temporal) | stories + acceptance criteria |
| `decision` | something the client already settled | never re-litigate; cite it |
| `non-goal` | explicitly out of scope | scope defence — cite when refusing work |

**`confidence`** is one of:

- `stated` — someone said it, in as many words. Quote it.
- `implied` — it follows from what was said, but nobody said it. Must be confirmed before a schema
  or a story depends on it.
- `assumed` — you filled a gap. **Every `assumed` item must also appear in `openQuestions`.**

**`authority`** on a source ranks it when sources disagree: a signed proposal or a written decision
outranks a passing remark in a chat thread. Ranking does not resolve a conflict — it only orders
the question. Conflicts still go to a human.

**`sources`** is mandatory and non-empty on every entity, attribute, relation and open question. An
unsourced claim does not go in the graph. `loc` is a line or section pointer into the digest file;
`quote` is the shortest span that carries the claim.

## Invariants

Anything violating these is a bug in the ingest, not an acceptable state:

1. Every `sources[].id` resolves to an entry in `sources`.
2. Every `relations[].from` / `.to` resolves to an entity `id`.
3. Every `confidence: "assumed"` item is referenced by an open question.
4. Every open question names what it `blocks`, so its cost is visible.
5. `answer` is non-null if and only if `status` is `answered`, and carries `answeredBy` and
   `answeredOn`.
6. No entity is silently dropped between runs — re-ingesting merges by `id` and records what
   changed.
