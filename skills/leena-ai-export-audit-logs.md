---
name: leena-ai-export-audit-logs
description: >-
  Incrementally export Leena AI audit log records into a SIEM or data warehouse using cursor
  pagination, staying inside the 60 requests-per-minute rate limit and handling the PII the
  records carry.
api: leena-ai:audit-logs-external-api
spec: openapi/leena-ai-audit-logs-openapi.yml
operations:
  - listAuditLogs
generated: '2026-07-19'
method: generated
source: https://docs.leena.ai/docs/audit-logs-external-api-authentication-usage-guide-beta
---

# Export Leena AI audit logs

Pulls audit records for incremental sync. The endpoint is built for exactly this: a stable
ascending `(updatedAt, _id)` sort with an opaque cursor, so a resumable watermark loop is the
intended pattern.

## Before you start

- The access token **must** carry the `audit-logs:read` scope and a `botId` claim. A token
  missing either is rejected with 401.
- Hosts are region-prefixed: `https://<region-code>-auditlogs.leena.ai`, with the default
  region (ap-south-1) served unprefixed as `https://auditlogs.leena.ai`.

## Step 1 — Get a scoped access token

`POST https://<region-code>-acl.leena.ai/api/v1.0/oauth/token`

- `Authorization: Basic <base64(clientId:clientSecret)>`
- Body: `{"username": "...", "password": "...", "grant_type": "password"}`

Tokens last 3600 seconds. A long backfill will outlive one token — refresh mid-run.

## Step 2 — First page (`listAuditLogs`)

`GET /external/v1/audit-logs?updatedAt=<ISO-8601>&limit=1000`

with `Authorization: Bearer <access_token>`.

- `updatedAt` is a **strictly greater-than** lower bound and is required when no cursor is
  supplied.
- `limit` accepts 1–1000, defaulting to 100. Use 1000 to minimise call volume against the
  rate limit.

## Step 3 — Walk the cursor

The response is:

```json
{ "data": [ ... ], "nextCursor": "<base64 or null>", "hasMore": true }
```

Loop: while `hasMore` is true, call
`GET /external/v1/audit-logs?cursor=<nextCursor>&limit=1000`.

`cursor` overrides `updatedAt` — send one or the other, not both.

> **Treat the cursor as opaque.** It happens to decode to
> `base64({"updatedAt": "...", "_id": "..."})`, but Leena AI documents it as opaque. Pass
> `nextCursor` back verbatim; never construct or mutate one. A malformed cursor returns
> 400 `"Invalid cursor"`.

## Step 4 — Checkpoint for the next run

Persist **both** the highest `updatedAt` seen and the last `nextCursor`.

- Resume a **cleanly finished** run from the stored `updatedAt`.
- Resume an **interrupted** run from the stored cursor — records sharing a timestamp are
  disambiguated by the `_id` component, which the bare watermark loses.

## Rate limiting

60 requests per minute per OAuth client, keyed on the JWT `id` claim, over a fixed 60-second
window. Breaching it returns 429 with
`"Rate limit exceeded. Please retry after some time."`

There is **no `Retry-After` header and no quota headers**, so use exponential backoff. At
`limit=1000` you can move 60,000 records a minute without approaching the ceiling. Remember
that token refresh calls also consume request budget.

## Handling the records

Each `ExternalAuditLog` carries an `actor` and optional `targetUser`, both of which include
`email`, `phone` and `employeeId`, plus top-level `ip` and `useragent`.

**This is personal data.** Minimise retention, restrict access downstream, and confirm the
export is covered by your DPA before landing it in a warehouse. Leena AI holds ISO/IEC 27701
and lists GDPR, CCPA, VCDPA and LGPD on its trust center.

## Error handling

| Status | Message | Do |
|---|---|---|
| 400 | `Invalid cursor` | Restart from a stored `updatedAt` watermark |
| 400 | validation error | Check `updatedAt` is ISO-8601 and `limit` is 1–1000 |
| 401 | `Missing or invalid Authorization header` | Send `Authorization: Bearer <token>` |
| 401 | `Invalid token` / `Token expired` | Re-issue or refresh the token |
| 401 | `Insufficient OAuth scope` | Get a credential with `audit-logs:read` |
| 401 | `Token is missing botId` | Request a bot-bound token |
| 429 | `Rate limit exceeded...` | Exponential backoff |

Note that insufficient scope is **401 here** but **403** on the External AOP API.

## Related

- `scopes/leena-ai-scopes.yml` — the `audit-logs:read` scope
- `rate-limits/leena-ai-rate-limits.yml` — limits and the missing headers
- `conventions/leena-ai-conventions.yml` — pagination and region model
- `data-model/leena-ai-data-model.yml` — the AuditLog and Actor entities
