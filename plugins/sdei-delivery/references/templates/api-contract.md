# API contract — <project>

Base URLs: local `http://localhost:3000` · staging `<url>` · production `<url>`
Version: `<v1>` · Last verified against the code: `<YYYY-MM-DD>`

Every endpoint below was verified to exist in the routers. If you cannot grep the route, it does not
belong in this document.

## Authentication

How a caller authenticates, in one paragraph: the header, the token format, the lifetime, how it is
refreshed, and what a rejected token returns.

```
Authorization: Bearer <access token>
```

| Failure | Status | Body |
| --- | --- | --- |
| No token | 401 | standard error envelope, `message: "Authentication required"` |
| Expired token | 401 | `message: "Token expired"` — the client should refresh and retry once |
| Valid token, insufficient permission | 403 | `message: "Forbidden"` — never leaks whether the resource exists |

## Response envelope

Every endpoint returns the same shape. Document it once here and never repeat it per endpoint.

```json
{
  "success": true,
  "message": "Campaign closed",
  "responseObject": { },
  "statusCode": 200
}
```

Errors use the same envelope with `success: false` and `responseObject: null`.

| Code | Meaning | When |
| --- | --- | --- |
| 400 | validation failed | a request body or query field violated a validator rule |
| 401 | not authenticated | missing, malformed or expired token |
| 403 | not permitted | authenticated, but not for this action or this record |
| 404 | not found | the resource does not exist, or the caller may not know that it does |
| 409 | conflict | the action is invalid in the resource's current state |
| 422 | unprocessable | syntactically valid, semantically impossible |
| 429 | rate limited | includes `Retry-After` |
| 500 | server error | generic message plus a correlation id — never a stack trace |

## Pagination

The contract every list endpoint follows: parameter names, defaults, **the enforced maximum**, the
sort contract and its deterministic tie-break, and the shape of the returned metadata.

```
GET /api/campaigns?page=1&limit=20&sortBy=startDate&sortOrder=desc
limit default 20, maximum 100 — a larger value is clamped, not honoured
```

## Rate limits

Which endpoints, which environments, the window and the ceiling. If limits apply in production
only, say so — that is a fact a client integrator needs.

---

## Endpoints

### `POST /api/campaigns` — create a campaign

**Story:** BE-03-01 · **Auth:** required · **Permission:** `campaign:create`
**Object-level check:** the caller's grower must match `growerId`

Request body

| Field | Type | Required | Rules |
| --- | --- | --- | --- |
| `campaignName` | string | yes | 1–120 chars, unique per grower |
| `growerId` | ObjectId | yes | must exist; must be the caller's grower |
| `startDate` | ISO date | yes | not before today in the grower's timezone |

```json
{ "campaignName": "North block spring", "growerId": "665f…", "startDate": "2026-04-01" }
```

Success — `201`

```json
{
  "success": true,
  "message": "Campaign created",
  "responseObject": { "_id": "6690…", "campaignName": "North block spring", "status": "draft" },
  "statusCode": 201
}
```

Errors

| Status | When |
| --- | --- |
| 400 | any validator rule above fails |
| 403 | `growerId` is not the caller's grower |
| 409 | a campaign with that name already exists for the grower |

---

Repeat the block per endpoint. Group by resource, and keep the order stable so diffs stay readable.

## Drift audit

Where the project generates documentation from annotations, this section replaces duplication:

| Finding | Route | Detail |
| --- | --- | --- |
| Undocumented | `PATCH /api/campaigns/:id/close` | exists in the router, absent from the annotations |
| Documented but absent | `DELETE /api/campaigns/:id` | annotated, no such route |
| Shape drift | `GET /api/campaigns` | annotation says an array, the service returns `{ items, total }` |
