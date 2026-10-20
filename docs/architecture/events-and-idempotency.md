# Events and idempotency

Status: Proposed  
Date: 2026-10-20

Two different duplicates have to be safe: a provider delivering the same webhook twice, and a worker retrying a write that already happened.

## Normalized event

Webhook handlers and the manual sync job both write this shape. Downstream stages do not see provider-specific envelopes.

| Field | Meaning |
| --- | --- |
| id | Internal UUID. |
| organization_id | From the connector the event authenticated to. Never from the payload body alone. |
| connector_id | |
| provider | `github`, `slack`, `jira`. |
| external_event_id | Provider delivery id, or a hash of `(external_object_id, source_updated_at, event_type)` when the provider has no delivery id. |
| event_type | `upsert` or `delete`. |
| object_type | Canonical table name. |
| external_object_id | |
| occurred_at | Provider time. |
| payload | Raw JSON needed to normalize. Dropped or reduced after the object is indexed, according to retention once retention exists. |
| stage | `received`, `validated`, `upserted`, `chunked`, `embedded`, `related`, `indexed`, `dead`. |
| attempt | |
| last_error | |

Unique `(organization_id, provider, external_event_id)`.

On conflict, the handler commits nothing new and returns 2xx. The provider’s retry is done.

## Webhook acceptance

1. Resolve the connector by the provider’s installation or workspace id. Unknown installations get 404 and no insert.
2. Verify the signature with the connector’s signing secret. Reject missing, invalid, and stale timestamps. The replay window is 5 minutes.
3. Insert the event and an outbox row in one transaction.
4. Return 2xx.

Slack’s URL verification challenge is answered before this path, with the challenge body, and it writes no event.

## Pipeline

A worker claims outbox rows with `FOR UPDATE SKIP LOCKED`, ordered by `occurred_at`, in batches. For each event it runs the next incomplete stage.

| Stage | Does | Idempotent because |
| --- | --- | --- |
| validated | Checks the payload has the fields the normalizer needs. | No write except stage. |
| upserted | Maps to the canonical table and upserts on `(organization_id, connector_id, external_id)`. | Upsert. |
| chunked | Rebuilds chunks for that object. | Upsert on `(source_type, source_id, ordinal)`. Unchanged text keeps its embedding. Changed text clears `embedding`. Ordinals that disappeared are deleted. |
| embedded | Fills embeddings for chunks where `embedding` is null. | Skips chunks that already have an embedding for this `embedding_model`. |
| related | Recomputes rule-based relationships for the object. | Unique relationship key. |
| indexed | Marks the event `indexed` and advances `connectors.last_synced_at` when this event is the newest. | Stage compare-and-set. |

A stage records success before the next one starts. A crash mid-embed leaves the event at `chunked` and the retry embeds the missing chunks only.

Deletes set `deleted_at` on the object, remove its chunks from search by deleting those chunk rows, and drop relationships whose subject or object is that id. The event then goes to `indexed`.

## Failures

| Problem | Behavior |
| --- | --- |
| Duplicate event | Unique key, ack, no second pipeline run. |
| Transient provider or model error | Exponential backoff, attempt increment, same event. |
| Poison payload | After 5 attempts, `stage = dead`, copy to `ingestion_dead_letters`, ack the outbox so the queue moves. |
| Partial index | Resume at the recorded stage. |
| Provider rate limit | Per-connector token bucket in the worker. 429 uses the provider’s retry-after when present. |
| Concurrent updates of one object | Upsert uses `source_updated_at`. An older event does not overwrite a newer row. |

Dead letters are replayed by resetting `stage` to the failed stage and inserting a new outbox row. Replay is an operator action, not automatic.

## Action idempotency

The key is computed in Go, not by the model:

```text
idempotency_key = sha256(
  organization_id + tool_name + tool_version + canonical_json(arguments) + workflow_step_id
)
```

`canonical_json` sorts object keys and drops insignificant whitespace. Unique `(organization_id, idempotency_key)`.

| Existing action | Retry behavior |
| --- | --- |
| `succeeded` | Return the stored result. Do not call the provider. |
| `executing` | Do not start a second call. The owner is the worker that won the status update. |
| `failed` with a stored external id | Reconcile with `get` on that id. If it exists, mark `succeeded`. |
| `failed` with no external id | Automatic retries stop at `failed`. A person or operator may replay, and that replay may call the provider. Jira and GitHub issue creation are not always idempotent on their side, so the worker records the returned id immediately, before any follow-up such as assignment. |
| `awaiting_approval` | Do not create a second action for the same key. |

`workflow_step_id` is required so two intentional “create issue” steps in one workflow do not collapse into one, and so a replay of the same step does.

Where the provider accepts an idempotency token, send the Keel action id. Where it does not, the store-the-external-id-immediately rule is the protection.

## Correlation

`correlation_id` is generated at the HTTP edge if the caller did not send one, and is echoed back. The same id is stored on the workflow, every action, every audit event, and passed through gRPC metadata into Python. Logs include it. A single request can be walked from query to retrieval to approval to provider call without a new tracing backend. OpenTelemetry spans use this id as an attribute once tracing exists.
