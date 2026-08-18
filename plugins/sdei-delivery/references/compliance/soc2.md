# SOC 2 — engineering obligations

Switched on per project in `PROFILE.md`. SOC 2 is an audit of controls against the Trust Services
Criteria, and most of it is organisational. What follows is the engineering half — and the framing
that matters is that **an auditor asks for evidence, not for assurances**. A control you perform but
cannot demonstrate does not exist.

Security is always in scope. Availability, confidentiality, processing integrity and privacy are
included only if the engagement says so — confirm with a human.

## Security (always in scope)

**Access control.** Unique accounts, no sharing. Least privilege by role. Joiner/mover/leaver
handled — access removed on departure, and demonstrably. Access reviewed periodically, with the
review recorded. Production access restricted, and privileged actions logged.

**Change management.** Every production change traceable to a reviewed, approved change. In
practice: no direct commits to the deploy branch, pull requests reviewed by someone other than the
author, CI gates that block merge, and a deployment record showing what shipped, when, and by whom.
**Branch protection is the control**; a convention that people follow is not, because it produces no
evidence.

**Vulnerability management.** Dependency scanning in CI with a documented remediation timeline;
patching cadence for hosts; findings tracked to closure.

**Monitoring and incident response.** Security-relevant events logged and retained; alerting on
authentication failures and privilege changes; an incident procedure that has been exercised.

**Encryption.** TLS in transit, encryption at rest, documented key management and rotation.

## Availability (if in scope)

Backups on a schedule, **restore tested and the test recorded**, uptime monitored against a stated
objective, capacity considered, and a recovery plan with RTO and RPO that someone has rehearsed.

## Confidentiality and privacy (if in scope)

Data classified, retention defined and **enforced by something that runs**, secure disposal, and
customer data segregated between tenants and between environments.

## Processing integrity (if in scope)

Inputs validated, processing complete and accurate, errors detected and corrected, and reconciliation
where financial or quantitative data is involved.

## What this changes in the build

| Phase | Obligation |
| --- | --- |
| `/write-stories` | the baseline epic's audit, access-review and backup stories are mandatory, not optional |
| `/design-schema` | classification and retention per entity; tenant isolation stated explicitly |
| `/build-module` | every privileged action writes an audit entry with actor, target, action, timestamp |
| `/write-tests` | authorization tests are evidence; keep them and keep their results |
| `/deploy` | branch protection, required review, CI gates blocking merge, a deployment record, environment protection on production |

## Evidence, which is the actual deliverable

For each control, know what an auditor would be shown:

| Control | Evidence |
| --- | --- |
| Change management | PR history with approvals; branch protection settings; CI run records |
| Access control | user list with roles; the access-review record; offboarding tickets |
| Vulnerability management | scan output over time; remediation tickets with dates |
| Backup and restore | schedule config; **the restore test record** |
| Monitoring | log retention configuration; alert definitions; incident records |
| Encryption | TLS configuration; at-rest settings; key rotation record |

If the evidence is a screenshot someone took once, the control will not survive the audit period.
Prefer controls the system produces continuously — branch protection, CI gates, automated scans —
over controls that depend on someone remembering.
