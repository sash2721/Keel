# ADR-004: Hybrid retrieval

Status: Proposed  
Date: 2026-10-20

## Context

Questions Keel must answer are not one shape. “Why did we move payments from Redis to Kafka?” is semantic. “PAY-391” and “PR #482” are exact. “What changed in authentication in the last six months?” is a metadata constraint. “Show the Slack thread behind this decision” is a relationship the vector index does not store.

A vector-only retrieval stack answers the first shape and misses the others, then the model fills the gap. That is the failure mode this product exists to avoid.

## Decision

Every question-answering retrieval runs four candidate generators, then a fusion step, then a context builder. All of them receive the permission scope from ADR-005 and apply it inside the query.

| Generator | Retrieves |
| --- | --- |
| Semantic | Nearest chunks by embedding, filtered by organization and ACL. |
| Keyword | Full text and exact identifiers (issue keys, PR numbers, repo names). |
| Metadata | Equality and range filters: provider, repository, project, author, team, time window. The planner may emit these from the question. |
| Graph | From the top entities, follow `entity_relationships` up to 2 hops and pull the chunks of the neighbors. |

Fusion in the first implementation is reciprocal rank fusion over the candidate lists, with a stable tie-break toward newer source timestamps and exact identifier matches. A cross-encoder or LLM reranker is a later stage in the same pipeline. Add it when the evaluation set shows fusion ranking the right evidence outside the context window. Do not block the first cited answers on a reranker.

The context builder packs citations until a token budget. Each packed chunk carries source, object id, title, URL, author, timestamp, and the retrieval reason (which generators proposed it). The synthesizer may only cite ids that were packed. The response returns those citations and the related entity ids.

Entity resolution (“payments service” → repository) is its own step before graph walk. The first version resolves by alias table and exact or fuzzy name match against repositories, projects, and teams. Unresolved mentions stay in the keyword query. They do not get invented ids.

## Consequences

- Retrieval quality is debuggable. A bad answer can be traced to a generator, a fusion rank, or a missing relationship.
- Indexing must maintain three representations together: the canonical row, the `search_vector`, and the embedding. The ingestion stage machine in ADR-002 is what keeps them aligned.
- The evaluation set needs cases that fail if any one generator is removed. Otherwise hybrid retrieval will rot into “whatever the vector search returned.”

## Alternatives

| Alternative | Why it was not chosen |
| --- | --- |
| Vector search only | Fails exact keys, time bounds, and cross-system links. |
| Keyword search only | Fails paraphrased questions, which are the common ask-the-organization path. |
| LLM chooses chunks with no ranker | Hides the failure. We cannot tell retrieval failure from synthesis failure. |
| Learned reranker as a launch dependency | Useful, and slower and costlier. Fusion is good enough to measure against. |
