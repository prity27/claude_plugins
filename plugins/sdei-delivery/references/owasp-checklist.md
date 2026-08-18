# OWASP baseline — always on

Applies to every project regardless of compliance regime. Used three ways:

1. `/write-stories` turns the relevant rows into **acceptance criteria**, not a separate "security
   story" nobody prioritises.
2. `/build-module` carries the rows that touch its slice into the plan-mode brief.
3. `/validate-delivery` demands evidence per row before sign-off.

A row is only meaningful with a project answer. "Uses parameterised queries" is a claim; "no string
concatenation into a query — verified by grep across `src/`" is evidence.

## A01 Broken access control

- Every route states its authorization requirement explicitly. A route with no authorization is a
  deliberate, documented decision — never an omission.
- **Authorization is checked server-side on every request**, never inferred from the client, a
  hidden field, or a role name in a token the client can swap.
- Object-level checks: fetching by id proves the caller owns or may see *that* object. This is the
  single most common real defect — IDOR is not caught by a route-level role check.
- One authorization mechanism per codebase. Three parallel mechanisms means nobody knows which
  routes are protected.
- Deny by default. A new route is unreachable until it declares who may call it.

## A02 Cryptographic failures

- TLS everywhere, and **failure to load a certificate is a startup failure**, not a silent
  downgrade to HTTP.
- Secrets come from the environment or a secret manager. `.env` is git-ignored — verify with
  `git ls-files | grep -i env`, not by reading `.gitignore`.
- No secret is reused for a second purpose. One key derived from another means one leak is two.
- Passwords: a modern KDF (bcrypt/argon2/scrypt), never a raw hash. Tokens: signed, short-lived,
  with a refresh path and a revocation story.
- Sensitive data at rest is classified before it is encrypted — you cannot protect a field nobody
  labelled.

## A03 Injection

- Parameterised queries or an ORM's own builders. Never string concatenation into a query, a shell
  command, or a template.
- **NoSQL is not exempt.** A raw request object spread into a filter allows operator injection
  (`{"$ne": null}`) — cast and validate every value before it reaches a query.
- Validate at the boundary with a schema, and **use the validated output**, not the raw request.
- Output encoding on render; no `dangerouslySetInnerHTML` or equivalent on user-controlled data.

## A04 Insecure design

- Rate limits on authentication, password reset, token refresh, search and any expensive endpoint —
  **in every environment**, not just production.
- Business-logic limits: quantities, date ranges, page sizes, upload sizes, batch sizes. Every
  list endpoint is paginated with an enforced maximum.
- The failure path is designed, not discovered. Multi-step writes state what happens when step two
  fails.

## A05 Security misconfiguration

- Errors return a generic message and a correlation id; stack traces and driver errors never reach
  a client.
- Security headers set; CORS lists real origins, never `*` with credentials.
- Default credentials, sample routes, seeders and debug endpoints are absent from production.
- Dependencies pinned via a committed lockfile.

## A06 Vulnerable and outdated components

- `npm audit` (or the stack equivalent) runs in CI and its result is visible.
- Runtime versions are declared and enforced, not implied by whatever the VM happens to have.

## A07 Identification and authentication failures

- Session and token lifetimes are explicit; logout and password change invalidate existing tokens.
- Brute-force protection on login, lockout or backoff, and no user enumeration through differing
  error messages or response timing.
- MFA where the compliance regime or the data class requires it.

## A08 Software and data integrity failures

- CI installs from the lockfile (`npm ci`, not `npm install`).
- Uploads: validate content type by inspecting the file, not by trusting the extension or the
  client's header. Cap the size. Store outside the web root or in object storage. Never execute
  what was uploaded.
- Deserialisation of untrusted input is schema-validated.

## A09 Security logging and monitoring failures

- Authentication, authorization failures, privilege changes and destructive operations are logged
  with actor, target, action and timestamp.
- **Logging works in production.** A logger gated on `NODE_ENV === 'development'` means production
  has no logs — a common and load-bearing mistake.
- Logs never contain credentials, tokens, full PII or PHI.
- Someone can answer "who changed this record, and when" from the logs alone.

## A10 Server-side request forgery

- Any URL supplied by a user and fetched by the server is validated against an allowlist; internal
  ranges and metadata endpoints are blocked.

## Turning a row into an acceptance criterion

Bad: *"The API is secure."*

Good:

```
AC-4  Given a user authenticated as `worker`
      When they GET /api/campaigns/:id for a campaign belonging to another grower
      Then the response is 403 with the standard error envelope
      And no campaign field appears in the body
      And the attempt is written to the audit log with actor, target and action
```

That is testable, reviewable, and it fails loudly when someone removes the check.
