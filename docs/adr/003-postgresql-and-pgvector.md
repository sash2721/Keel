# ADR-003: PostgreSQL for records, relationships, and vectors

Status: Proposed  
Date: 2026-10-20

## Context

Keel needs a transactional store for tenants and actions, a place for embeddings, and a way to walk relationships such as “this pull request references this issue.” The blueprint allows a dedicated vector database and a dedicated graph database later. It recommends starting in PostgreSQL.

The first version is one organization, then a small number of tenants, on a single primary. The hard problems are correctness of tenancy, idempotent writes, and cited retrieval. They are not graph-query latency.

## Decision

PostgreSQL is the system of record for organizations, identity, connectors, source objects, relationships, chunks, actions, approvals, audit, and ingestion state.

Vectors live in `pgvector` on the chunk table. Approximate index type (HNSW) is chosen when the first real corpus is loaded and can be measured. Keyword search uses PostgreSQL full text on the same rows. A chunk is visible to both paths only after its embedding and its `search_vector` are written, so hybrid search does not return half-indexed text.

Relationships are rows in `entity_relationships` with a subject, a predicate, an object, and provenance. Traversal is SQL with a depth limit (start at 2 hops). Predicates are a closed set in code, not free-form strings.

A dedicated graph database is in bounds later if a real query needs variable-length paths, repeated multi-hop traversals, or analytical graph workloads that are slow or unreadable in SQL. “We wanted a graph picture in the architecture” is not that condition.

Embeddings are accessed through a small interface (`Embed(ctx, texts) ([][]float32, error)` or the Python equivalent). The vendor and model are configuration. Swapping them requires a re-embed of affected chunks, so store `embedding_model` on each chunk.

## Consequences

- One backup story covers records and vectors.
- Tenant isolation is a `organization_id` predicate on SQL, which is straightforward to test.
- Large attachment bodies and raw diffs do not belong in the hot row. Store text we intend to retrieve. Keep provider URLs for the rest.
- If vector recall becomes the bottleneck, a separate vector store can take the embedding column without moving the system of record.

## Alternatives

| Alternative | Why it was not chosen |
| --- | --- |
| A vector database from day one | Another store to keep consistent with the canonical rows, before we know corpus size. |
| Neo4j or similar from day one | The relationship set is small and explicit. SQL can traverse it. The blueprint calls this out directly. |
| One document table with JSON only | Typed tables for issues, pull requests, and messages keep constraints and queries honest. Shared columns are a convention, documented in the domain model. |
