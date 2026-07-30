---
name: leena-ai-sync-knowledge-base
description: >-
  Migrate or continuously sync knowledge base articles and their attachments from a source
  system into the Leena AI Knowledge Base, with permission scoping and a final indexing
  trigger.
api: leena-ai:knowledge-management-rest-connector
spec: openapi/leena-ai-knowledge-management-openapi.yml
operations:
  - createKnowledgeAccessToken
  - uploadKnowledgeAttachment
  - syncKnowledgeArticle
  - completeKnowledgeSync
generated: '2026-07-19'
method: generated
source: https://docs.leena.ai/docs/rest
---

# Sync a knowledge base into Leena AI

Pushes articles from your source system into the Leena Knowledge Base, which is the corpus
that grounds AI Colleague answers. Content quality here propagates directly into
employee-facing responses.

## Before you start

- Your Leena AI representative provides `clientId`, `clientSecret`, `username`, `password`
  **and the connector host** — Leena AI does not publish the article-endpoint host, so do not
  assume the regional pattern used by its other APIs.
- Decide the stable identifier from your source system that will become `reference_id`. This
  choice is load-bearing (see idempotency below).

## Step 1 — Get a token (`createKnowledgeAccessToken`)

`POST /api/v1.0/oauth/token`

Body: `{"clientId": "...", "clientSecret": "...", "username": "...", "password": "..."}`

> **Watch the TTL.** This connector's token is valid for **30 minutes** — half the 3600
> seconds the AOP and Audit Logs APIs issue. Any real bulk migration outlives a single
> token, so implement refresh in your middleware, as Leena AI explicitly recommends.

## Step 2 — Stage attachments (`uploadKnowledgeAttachment`)

`POST /api/integration/articles/upload/` as `multipart/form-data` with the file and its MIME
type, and `Authorization: Bearer <token>`.

Returns an `attachment_id`.

Supported types: PDF, DOC, DOCX, TXT, HTML, XLS, XLSX, PPT, PPTX, PNG, JPEG.

Attachments must be staged **before** the article that references them — they cannot be
attached retroactively in one step.

## Step 3 — Push the article (`syncKnowledgeArticle`)

`POST /api/integration/articles/sync/`

```json
{
  "reference_id": "<stable source-system id>",
  "html_content": "<valid HTML>",
  "attachments": ["<attachment_id>"],
  "permissions": {
    "department": ["..."],
    "location": ["..."],
    "user_groups": ["..."]
  }
}
```

> **Idempotency by key.** There is no `Idempotency-Key` header anywhere in Leena AI's API
> surface, but this operation **upserts on `reference_id`**. Re-syncing the same
> `reference_id` updates the existing article instead of duplicating it — which makes a
> stable source identifier the difference between a safe re-runnable sync and a duplicated
> corpus. Never generate a fresh id per run.

`html_content` must be **valid HTML**; malformed markup is the documented failure mode for
this call.

Set `permissions` deliberately. It is the only control over who the article is surfaced to —
omitting it does not fail the call, it just widens visibility.

## Step 4 — Trigger indexing (`completeKnowledgeSync`)

`POST /sync/complete/`

Call this **once, after every article in the migration has been synced**. It triggers
indexing; calling it mid-migration indexes a partial corpus.

For ongoing incremental syncs rather than a bulk load, call it at the end of each batch.

## Ordering summary

```
token → upload attachments → sync articles → complete
```

## Error handling

| Status | Meaning | Do |
|---|---|---|
| 400 | Invalid payload, commonly malformed `html_content` | Validate the HTML before sending |
| 401 | Missing or expired bearer token | Refresh — remember the 30-minute TTL |

Errors return a plain JSON `message` string; there is no RFC 9457 problem+json and no
machine-readable error code.

## Rate limiting

Leena AI publishes **no** rate limit for this connector, which matters for bulk migrations.
Throttle conservatively and back off on any 429 rather than assuming headroom — no
rate-limit or `Retry-After` headers are returned anywhere on the platform.

## Related

- `conventions/leena-ai-conventions.yml` — auth, idempotency, error envelope
- `data-model/leena-ai-data-model.yml` — KnowledgeArticle, Attachment, ArticlePermissions
- `lifecycle/leena-ai-lifecycle.yml` — staging vs production environments
- Article lifecycle docs: https://docs.leena.ai/docs/lifecycle-of-knowledge-articles-in-leena-km
