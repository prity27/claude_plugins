---
epic: BE-03
title: Campaign lifecycle
unit: backend
status: draft
approved_by:
approved_on:
graph_entities: [ent-campaign, proc-campaign-close]
depends_on: [BE-01]
---

### BE-03-01 — List campaigns

**As a** grower administrator
**I want** to see my campaigns
**So that** I can pick one to work on

**Source:** `ent-campaign` · discovery call 2026-03-04 L101 — "A campaign is one grower, one crop, one date window"

**Acceptance criteria**

- **AC-1** Given campaigns exist for my grower
  When I request the campaign list
  Then I receive them paginated 20 per page, newest first

- **AC-2** Given a campaign belonging to another grower
  When I request the campaign list
  Then that campaign does not appear

- **AC-3** Given no authentication
  When I request the campaign list
  Then the response is 401

**Estimate:** M

### BE-03-02 — Bulk export campaigns to Excel

**As a** grower administrator
**I want** to export all campaigns to an Excel workbook
**So that** I can send them to my accountant

**Acceptance criteria**

- **AC-1** Given campaigns exist
  When I request an export
  Then an .xlsx file downloads containing every campaign field

- **AC-2** Given no authentication
  When I request an export
  Then the response is 401

**Estimate:** M

### BE-03-03 — Campaign dashboard

**As a** grower administrator
**I want** a dashboard of my active campaigns
**So that** I can see progress

**Source:** `ent-campaign` · proposal v2 3.4 — "Receptions update the campaign running total so the dashboard is current"

**Acceptance criteria**

- **AC-1** Given active campaigns
  When I open the dashboard
  Then it loads quickly and the numbers look right

- **AC-2** Given no active campaigns
  When I open the dashboard
  Then I see an empty state

**Estimate:** M

### BE-03-04 — Close a campaign

**As a** grower administrator
**I want** to close a finished campaign
**So that** nothing further is recorded against it

**Source:** `proc-campaign-close` · discovery call 2026-03-04 L204 — "Once a campaign is closed nothing else lands on it"

**Acceptance criteria**

- **AC-1** Given an active campaign
  When the administrator closes it
  Then its status becomes `closed`

**Estimate:** S

### BE-03-05 — Reopen a closed campaign

**As a** grower administrator
**I want** to reopen a campaign I closed by mistake
**So that** I can carry on recording against it

**Source:** `proc-campaign-close` · discovery call 2026-03-04 L204

**Acceptance criteria**

- **AC-1** Given a closed campaign
  When the administrator reopens it
  Then its status returns to `active`

- **AC-2** Given a user without campaign-management permission
  When they attempt to reopen a campaign
  Then the response is 403

**Estimate:** S

### BE-03-06 — Assign a crew to a campaign

**As a** grower administrator
**I want** to assign a crew to a campaign
**So that** the crew knows where to work

**Source:** `ent-crew` · discovery call 2026-03-04 L140 — "A crew belongs to one campaign"

**Acceptance criteria**

- **AC-1** Given a crew with no current campaign
  When the administrator assigns it to a campaign
  Then the crew's campaign is set

- **AC-2** Given a crew already assigned to a campaign
  When the administrator assigns it to a second campaign
  Then the request fails with 409 because a crew may only work one campaign

**Estimate:** M
