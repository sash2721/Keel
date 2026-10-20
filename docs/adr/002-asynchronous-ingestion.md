# ADR-002: Asynchronous ingestion with a Postgres outbox

Status: Proposed  
Date: 2026-10-20

## Context

GitHub, Slack, and Jira notify Keel through webhooks. Handling a webhook means validating the signature, upserting a canonical object, chunking text, calling an embedding model, and updating relationships. Embedding calls are slow and fail independently of the webhook. Providers retry when we respond late, which creates duplicates. A handler that does all of this inline will time out, and a retry will race the first attempt.

The blueprint names Kafka or NATS as the event bus. That solves decoupling. It also adds a system to run before we have a single connector.

## Decision

Webhook handlers return quickly. They authenticate the event, insert a normalized event row, and insert an outbox row in the same database transaction. The HTTP response is 2xx after that commit. A worker in the Go process consumes the outbox and runs the indexing pipeline.

The pipeline is staged and restartable:

`received → validated → upserted → chunked → embedded → related → indexed`

Each stage records its status on the event. A retry resumes at the first incomplete stage. It does not repeat a completed upsert.

Duplicate delivery is absorbed by a unique key on `(organization_id, provider, external_event_id)`. A conflict is a successful no-op, and the handler still returns 2xx.

Introduce NATS or Kafka when one of these becomes true: webhook bursts need a buffer the database should not hold, ingestion workers need to scale separately from the API, or a stage (embedding) needs its own consumer group. Until then, `SELECT … FOR UPDATE SKIP LOCKED` on the outbox is the queue.

Provider calls from workers use per-provider rate limits and exponential backoff. A message that fails past the retry budget goes to a dead-letter table with the stage name and the last error. Dead-lettered events are replayable after a fix.

## Consequences

- Indexing lag is normal. Reads of the canonical row may succeed before search can see the new chunk. The sync status on the connector is how the UI explains that.
- The outbox and the canonical upsert for “event stored” commit together. The later stages commit on their own and must be idempotent.
- Operators inspect `ingestion_events` and `ingestion_dead_letters` before they need a broker dashboard.
- Moving to NATS or Kafka later means changing the delivery mechanism, not the normalized event shape or the stage machine.

## Alternatives

| Alternative | Why it was not chosen |
| --- | --- |
| Synchronous indexing in the webhook | Provider timeouts and duplicate deliveries become data races. |
| Kafka or NATS on day one | Correct at higher volume. Extra operational cost before the first connector works. |
| Cron full-sync only | Backfill still needs a sync job. It misses the “what just changed” path, and webhooks are how the demo stays current. Keep a manual sync endpoint for backfill. |
