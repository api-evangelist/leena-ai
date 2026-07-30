---
name: leena-ai-run-agent-procedure
description: >-
  Trigger an Agent Operating Procedure on a Leena AI AI Colleague from an external system and
  poll it to completion, handling the paused (awaiting human approval) state correctly.
api: leena-ai:external-aop-api
spec: openapi/leena-ai-aop-openapi.yml
operations:
  - executeAop
  - getAopStatus
generated: '2026-07-19'
method: generated
source: https://docs.leena.ai/docs/external-aop-api-authentication-usage-guide
---

# Run a Leena AI Agent Operating Procedure

Starts an AOP on an AI Colleague and follows it to a terminal state. This is Leena AI's
asynchronous execute-then-poll model — the execute call returns immediately with `accepted`
and does not carry the result.

## Before you start

- Credentials (`client_id`, `client_secret`, `username`, `password`) are issued by a Leena AI
  representative. There is no self-service signup.
- Know your tenant's **region code**. Hosts are region-prefixed
  (`https://<region-code>-aic.leena.ai`); the default region (ap-south-1) is served
  unprefixed. Published regions: `us-east-1`, `eu-west-1`, `eu-central-1`, `canadacentral`,
  `ap-southeast-1`, `ap-south-1`, `qatarcentral`, `me-central2`.
- Know the `aop_id` you intend to run.

## Step 1 — Get an access token

`POST https://<region-code>-acl.leena.ai/api/v1.0/oauth/token`

- `Authorization: Basic <base64(client_id:client_secret)>`
- `Content-Type: application/json`
- Body: `{"username": "...", "password": "...", "grant_type": "password"}`

The response carries `access_token`, `token_type: Bearer`, `refresh_token` and
`expires_in: 3600`. Refresh proactively — an expired token returns 401 mid-poll.

## Step 2 — Execute the AOP (`executeAop`)

`POST /api/v1/external/aop/execute` on the `aic` host, with
`Authorization: Bearer <access_token>`.

```json
{
  "aop_id": "<aop identifier or ObjectId>",
  "message_to_start": "<optional natural-language instruction>",
  "context": { "<key>": "<value>" }
}
```

Returns `aop_item_id`, `request_id`, `run_id` and `status: "accepted"`. Persist
`aop_item_id` — it is the only handle for this run, and there is no list operation to
recover it.

> **No idempotency.** Leena AI publishes no idempotency key for this operation. A retried
> execute starts a **second agent run** that may act on real enterprise systems. Deduplicate
> on your side before retrying, and if you are unsure whether the first call landed, poll
> `getAopStatus` with the `aop_item_id` you already hold rather than re-executing.

## Step 3 — Poll for completion (`getAopStatus`)

`GET /api/v1/external/aop/items/{aop_item_id}/status`

Returns `aop_item_id`, `reference_id`, `status`, `initiated_at` and `completed_at`.

- **Terminal:** `completed`, `failed`, `aborted` — stop polling.
- **Non-terminal:** `in_progress`, `paused` — keep polling.

`paused` is **not** a failure. An AOP pauses when it hits a human approval step and will
resume on its own. Do not treat it as terminal and do not re-execute.

Poll with exponential backoff. No completion webhook or callback is published, so polling is
the only completion signal.

## Error handling

| Status | Meaning | Do |
|---|---|---|
| 400 | Invalid request | Check `aop_id` is present and well-formed |
| 401 | Auth failure or expired token | Refresh the token and retry once |
| 403 | Insufficient scope | Request a credential provisioned for AOP execution |
| 404 | AOP or item not found | Verify the id belongs to the authenticated tenant |

Errors are a plain JSON object with a `message` string — there is no RFC 9457
`problem+json` and no machine-readable error code, so branch on HTTP status.

Note the platform-wide inconsistency: insufficient scope is **403 here** but **401** on the
Audit Logs API. Handle both.

## Related

- `conventions/leena-ai-conventions.yml` — regions, async model, pagination
- `errors/leena-ai-problem-types.yml` — full error catalogue
- `authentication/leena-ai-authentication.yml` — token lifecycle
- `mcp/leena-ai-mcp.yml` — the MCP alternative to this REST flow
