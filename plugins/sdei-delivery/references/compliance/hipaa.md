# HIPAA — engineering obligations

Switched on per project in `PROFILE.md`. This is what HIPAA means for the build, not legal advice —
a compliance programme also needs policies, training, agreements and risk analysis that no skill
produces.

**Scope test first.** HIPAA applies when the system creates, receives, maintains or transmits
**PHI** — health information tied to an identifiable individual — as a covered entity or as a
business associate. Get this confirmed by a human before designing around it. Building HIPAA
controls into a system with no PHI wastes money; the reverse is a breach.

## What counts as PHI

Health information plus any of the 18 identifiers — name, address narrower than a state, all dates
tied to an individual (birth, admission, discharge, death), phone, email, SSN, medical record and
account numbers, device identifiers, IP address, biometrics, photographs, and any other unique
identifier.

**Consequences for `/design-schema`:** every field carrying one of these is classed `PHI` in the
entity table. A "date of birth" column and an "IP address" audit column are both PHI when
associated with a patient. Classify before designing, because the classification decides the
encryption, the logging, the retention and the export path.

## Technical safeguards, as acceptance criteria

**Access control (§164.312(a))** — unique identity per user, never a shared account. Role- or
attribute-based authorization enforced server-side. **Minimum necessary**: a user sees only the PHI
their role requires, which means field-level projections, not just route-level roles. Automatic
logoff after inactivity. An emergency access procedure that is itself logged.

**Audit controls (§164.312(b))** — record every access to PHI, not merely every modification.
Read access is auditable under HIPAA, which most systems get wrong. Each entry: who, what record,
what action, when, from where. Audit records are append-only and are themselves protected.

**Integrity (§164.312(c))** — PHI cannot be altered or destroyed without detection. Soft delete plus
an audit trail, not a hard delete.

**Authentication (§164.312(d))** — verify identity before access; MFA for administrative and remote
access.

**Transmission security (§164.312(e))** — TLS 1.2+ everywhere, including internal service hops and
the database connection. **A silent fallback to plain HTTP when a certificate is missing is a
violation** — startup must fail instead.

**Encryption at rest** — addressable rather than strictly required, which in practice means
implement it or document a defensible reason. Encrypt the database volume and any backup, and
consider field-level encryption for the most sensitive attributes.

## What this changes in the build

| Phase | Obligation |
| --- | --- |
| `/ingest-knowledge` | flag every PHI element the sources describe; redact real PHI out of digests — they are committed |
| `/design-schema` | classify every field; state retention per entity; soft delete; **no PHI in a natural key or a URL path segment that gets logged** |
| `/write-stories` | minimum-necessary as an AC on every read story; audit-on-read as an AC; MFA on admin stories |
| `/build-module` | no PHI in logs, error messages, analytics, or third-party telemetry — **including stack traces and query logs** |
| `/write-tests` | never real PHI in fixtures; assert that PHI does not appear in log output |
| `/deploy` | no PHI in CI logs; production data never copied to staging without de-identification |

## Common engineering failures

1. **PHI in logs.** A logged request body, a query with parameters, an error containing a record —
   all disclosures. This is the most frequent real failure.
2. **Read access not audited.** Modifications are logged, views are not, and the "who looked at this
   chart" question cannot be answered.
3. **Production data in staging.** Copied for a debug session and never removed.
4. **Third parties with no BAA.** Any vendor processing PHI — hosting, email, SMS, error tracking,
   analytics, an LLM API — needs a business associate agreement. **Error trackers and analytics are
   the ones teams forget**, and they receive PHI by default.
5. **Shared accounts** for support or operations, which destroys attribution.
6. **Backups unencrypted** or retained past the stated period.

## Breach notification

The system must make it possible to answer *which individuals' PHI was affected*. That is a design
requirement on the audit trail, and it cannot be retrofitted after an incident.
