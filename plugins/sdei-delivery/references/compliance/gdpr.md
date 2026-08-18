# GDPR — engineering obligations

Switched on per project in `PROFILE.md`. Applies to personal data of people in the EU/UK regardless
of where the company is. Confirm applicability and the controller/processor role with a human before
designing around it.

## The two principles that change the design

**Data minimisation.** Collect only what a stated purpose requires. In `/design-schema` this is a
real constraint: for every personal-data field, name the purpose it serves. **A field with no
purpose is removed, not stored "in case it is useful"** — that instinct is precisely what the
regulation prohibits.

**Purpose limitation and lawful basis.** Every processing activity needs a lawful basis — consent,
contract, legal obligation, vital interests, public task, or legitimate interests. Record which per
entity in `SCHEMA.md`. Where the basis is consent, the system must record what was consented to,
when, and how it can be withdrawn — and withdrawal must be as easy as giving it.

## Data subject rights, as build requirements

Each of these is a feature someone must implement. They are the most commonly retrofitted, at the
highest cost, so they belong in the baseline epic:

| Right | What must exist |
| --- | --- |
| Access | export everything held about one person, in a readable form, within one month |
| Rectification | correct inaccurate data, and propagate the correction downstream |
| **Erasure** | delete on request — including from backups, logs, caches, search indexes and analytics |
| Portability | export in a structured, machine-readable, commonly used format |
| Restriction | mark data as restricted from processing without deleting it |
| Objection | stop processing based on legitimate interests |
| Automated decisions | explain the logic; provide human review |

**Erasure is the one that breaks architectures.** Denormalised copies, event logs, audit trails,
analytics warehouses and backups all hold personal data, and "delete the row" reaches none of them.
Decide the approach *at schema design time* — crypto-shredding a per-subject key is usually more
practical than chasing copies — and record the decision.

Note the direct tension with an append-only audit trail. Resolve it explicitly with a human:
typically by keeping the audit event and pseudonymising its subject.

## Technical and organisational measures (Art. 32)

Pseudonymisation and encryption where appropriate; confidentiality, integrity, availability and
resilience; restoration after an incident; and **regular testing of the measures** — untested
controls do not satisfy the article.

## Retention

Every entity holding personal data carries a retention period and what happens at its end — deletion
or anonymisation. **Enforced by something that runs on a schedule**, not by a sentence in a policy.
`/design-schema` requires this per entity, and a project with no deletion job has no retention
policy regardless of what its documentation says.

## Cross-border transfer

Where personal data leaves the EU/UK — hosting region, backup region, a support team, a third-party
API, an LLM provider — a transfer mechanism is required. Record the data residency requirement in
the profile, because it constrains where you may deploy and which vendors you may call.

## Breach notification

72 hours to the supervisory authority. The system must support answering *whose data, which fields,
what period* — an audit-trail design requirement, and impossible to retrofit during an incident.

## What this changes in the build

| Phase | Obligation |
| --- | --- |
| `/ingest-knowledge` | identify personal-data elements; redact real personal data from committed digests |
| `/design-schema` | per field: purpose, lawful basis, retention; the erasure strategy decided up front |
| `/write-stories` | subject-rights stories in the baseline epic; consent capture and withdrawal where consent is the basis |
| `/build-module` | no personal data in logs or telemetry beyond what is necessary; deletion actually propagates |
| `/write-tests` | assert that erasure removes the data everywhere it claims to |
| `/deploy` | data residency constrains the region; a vendor processing personal data needs a data processing agreement |

## Common engineering failures

1. **Deletion that only deletes the primary row**, leaving copies in logs, caches, search indexes
   and analytics.
2. **Retention policies with no job** — data kept forever while a document says two years.
3. **Personal data in application logs**, indefinitely, with no retention or access control.
4. **Third parties receiving personal data by default** — error trackers capture request bodies and
   user context unless configured not to.
5. **Consent recorded as a boolean**, with no record of what version of what was consented to, when,
   or how it was withdrawn.
