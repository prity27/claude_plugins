---
epic: BE-90
title: Campaign and reception hardening (real-code audit fixture)
unit: backend
status: approved
approved_by: fixture
approved_on: 2026-08-18
graph_entities: [ent-campaign, ent-reception, proc-reception-record]
---

Target codebase: `<TARGET_REPO>` — substitute the repository you are auditing. The verdicts depend on that repo, so build the answer key from it first (see ../README.md).

### BE-90-01 — Campaign access is controlled and attributable

**Source:** `ent-campaign`, `constraint-audit`

**Acceptance criteria**

- **AC-1** Given a request carrying no authentication token
  When it calls any `/api/campaign` route
  Then the response is 401 and no campaign data is returned

- **AC-2** Given an authenticated administrator
  When they create a campaign
  Then an audit entry is written recording the actor, the entity and the event

- **AC-3** Given an authenticated user who is not a system administrator
  When they call the role-creation endpoint under `/api/roles`
  Then the response is 403 and no role is created

- **AC-4** Given any endpoint in the service
  When it returns either success or failure
  Then the body is the standard envelope `{ success, message, responseObject, statusCode }`

### BE-90-02 — Reception recording is safe under load

**Source:** `proc-reception-record` — peak ~2000/day, several devices at one bridge

**Acceptance criteria**

- **AC-1** Given a reception is recorded against a campaign
  When the reception insert succeeds but the campaign running-total update fails
  Then neither write is persisted and the caller receives an error

- **AC-2** Given a single client recording receptions in a tight loop
  When it exceeds a documented request ceiling
  Then further requests receive 429 with `Retry-After`

- **AC-3** Given the authorization rules on reception routes
  When the test suite runs
  Then an automated test asserts that a caller without crew-management rights receives 403
