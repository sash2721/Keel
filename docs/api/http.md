# HTTP API

Status: Proposed  
Date: 2026-10-20

The browser and any external client talk to the Go API. Python is not on this surface. All paths are under `/v1`.

## Conventions

- Authentication is `Authorization: Bearer <access token>`. The token resolves to one user and one organization.
- The organization is taken from the session. A body or header organization id that disagrees with the session is `403`.
- `X-Correlation-Id` is accepted when it is a UUID and generated otherwise. Every response echoes it.
- Timestamps are RFC 3339 in UTC.
- Errors use one shape:

```json
{
  "error": {
    "code": "approval_required",
    "message": "This action is waiting for approval.",
    "correlation_id": "7f3c..."
  }
}
```

| HTTP | When |
| --- | --- |
| 400 | Malformed JSON or failed schema validation. |
| 401 | Missing or invalid token. |
| 403 | Authenticated, not allowed. Includes cross-organization access. |
| 404 | Missing in this organization. Do not reveal that the id exists elsewhere. |
| 409 | Idempotency conflict or an illegal action transition. |
| 429 | Caller rate limit. |
| 202 | Accepted for work that finishes asynchronously. |

## Endpoints

| Method | Path | Purpose | Permission |
| --- | --- | --- | --- |
| POST | `/v1/query` | Grounded question. SSE stream, then a final answer. | `query.run` |
| POST | `/v1/workflows` | Start an action-capable workflow. | `action.propose` |
| GET | `/v1/workflows/{id}` | Workflow status and step summary. | The user who started it, or `action.approve` |
| GET | `/v1/actions/{id}` | Proposed, waiting, or finished action. | The requesting user, or `action.approve` |
| POST | `/v1/actions/{id}/approve` | Approve a waiting action. | `action.approve` |
| POST | `/v1/actions/{id}/reject` | Reject a waiting action. | `action.approve` |
| GET | `/v1/search` | Keyword and filter search without synthesis. | `query.run` |
| POST | `/v1/connectors` | Start a connector OAuth configuration. | `connector.manage` |
| POST | `/v1/connectors/{id}/sync` | Enqueue a backfill or incremental sync. | `connector.manage` |
| GET | `/v1/audit` | Query audit events for the organization. | `audit.read` |

Query streaming uses server-sent events. Event names: `stage` (`retrieving`, `ranking`, `synthesizing`) and `result`. The `result` payload is the answer object below. Clients that cannot read SSE may send `Accept: application/json` and receive only the final object.

Workflows return `202` with a workflow id. Clients poll `GET /v1/workflows/{id}`. The request does not stay open while a person approves.

## Query result

```json
{
  "answer": "The migration was driven by three factors...",
  "citations": [
    {
      "source": "github",
      "object_type": "pull_request",
      "object_id": "8c1a...",
      "title": "Move payments off Redis",
      "url": "https://github.com/example/payments/pull/482",
      "author": "Alice",
      "source_updated_at": "2026-09-14T00:00:00Z",
      "relevance": 0.93
    }
  ],
  "related_entities": [
    { "type": "repository", "id": "...", "name": "payments-service" },
    { "type": "issue", "id": "...", "name": "PAY-391" }
  ],
  "correlation_id": "7f3c..."
}
```

`relevance` is the fusion score, not a calibrated probability. The UI can sort on it and should not present it as a percentage of truth.

## Approve and reject

```json
{ "comment": "Looks right." }
```

The comment is optional. The response is the action object: id, status, tool name, risk, and, once executed, `result` or `error`.

## Search

`GET /v1/search?q=PAY-391&provider=jira&limit=20`

Returns matching source objects with title, url, and type. It does not call the model. It uses the same permission helper as grounded retrieval.

## Not in v1

No public registration flow beyond what Phase 1 needs for a local organization. No webhook receiver path documented as a stable public API, because each provider defines its own URL and signature scheme. Those routes live under `/v1/webhooks/{provider}` and are specified next to the connector that implements them.
