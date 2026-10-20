# Architecture decisions

Status values: **Proposed** until you accept or rewrite them. Accepted decisions are the ones implementation should follow.

| ADR | Title | Status |
| --- | --- | --- |
| [001](001-go-and-python-split.md) | Go for product APIs, Python for orchestration | Proposed |
| [002](002-asynchronous-ingestion.md) | Asynchronous ingestion with a Postgres outbox | Proposed |
| [003](003-postgresql-and-pgvector.md) | PostgreSQL for records, relationships, and vectors | Proposed |
| [004](004-hybrid-retrieval.md) | Hybrid retrieval | Proposed |
| [005](005-permission-filtered-context.md) | Permission filters before model context | Proposed |

ADR-006 through ADR-010 in the blueprint (deterministic tool authorization, human approval, action idempotency, tenant isolation details, and when to add a graph database) are covered inside the contracts and these five ADRs. Write them as separate ADRs only if a later change needs its own record.
