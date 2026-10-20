# Keel

Keel is an operating layer for organizational knowledge, context, and action. It ingests GitHub, Slack, and Jira, answers questions with citations, and runs write actions only after a deterministic policy check.

This repository is in **Phase 0**. The documents below are proposed drafts. Implementation has not started. Change anything that should be different, then treat the accepted docs as the contract for Phase 1.

The engineering blueprint that these drafts interpret is `AI_Organizational_Intelligence_Action_Platform_Blueprint.pdf` in this folder. `.gitignore` excludes `*.pdf`, so that file stays local and will not be committed.

## Read in this order

1. [Product requirements](docs/prd.md) — who it is for, what the first version must do, and the finish line.
2. [Domain model](docs/architecture/domain-model.md) — tenants, source objects, relationships, chunks, actions, audit.
3. [ADRs 001–005](docs/adr/README.md) — the architecture choices to accept before code.
4. [HTTP API](docs/api/http.md) — the external contract.
5. [Connector and tool contracts](docs/architecture/connector-and-tool-contracts.md) — how GitHub, Slack, and Jira plug in.
6. [Action state machine](docs/architecture/action-state-machine.md) — propose, approve, execute, audit.
7. [Events and idempotency](docs/architecture/events-and-idempotency.md) — ingestion and safe retries.

## Confirm or edit these before implementation

These are decisions made in the drafts, not leftovers. If one is wrong, change the doc and the ADR that owns it.

| # | Decision | Where |
| --- | --- | --- |
| 1 | Two deployable processes: a Go API and a Python orchestrator. Other blueprint boxes are packages inside those processes until a boundary needs its own deploy. | ADR-001 |
| 2 | Go and Python talk over gRPC. The browser talks HTTP to Go only. | ADR-001, HTTP API |
| 3 | Ingestion is asynchronous. The first queue is a Postgres outbox. NATS or Kafka waits until workers need to scale independently. | ADR-002 |
| 4 | PostgreSQL holds tenants, source data, relationships, and vectors (`pgvector`). No dedicated graph database in the first version. | ADR-003 |
| 5 | Retrieval is hybrid: semantic, keyword, metadata filters, and relationship walk, then a fusion step. A learned reranker is added when evaluation shows fusion is not enough. | ADR-004 |
| 6 | Organization and permission filters run in the query, before any text is placed in a prompt. | ADR-005 |
| 7 | A user belongs to one organization in v1. Membership is still its own table. | Domain model |
| 8 | Roles are `owner`, `admin`, `member`, and `viewer`. | Domain model |
| 9 | Isolation in v1 is the organization boundary. Team and channel restrictions are columns from the start and are enforced as each connector learns real ACLs. | Domain model, ADR-005 |
| 10 | Low-risk reads auto-execute. Medium writes are configurable. High-risk writes always require approval. | Action state machine |
| 11 | The embedding provider sits behind an interface. No vendor is chosen yet. | ADR-003 |
| 12 | Phase 7 (approvals and audit, three connectors, grounded answers) is the finish line. Multi-step workflows and production hardening are follow-on. | PRD |

## Intended layout

Create this when implementation starts. It is not scaffolded yet.

```text
keel/
├── services/api/          Go: auth, query, actions, audit, connectors, ingestion
├── ai/                    Python: LangGraph orchestration, retrieval client, tools
├── proto/                 gRPC contracts between Go and Python
├── web/                   React + TypeScript
├── migrations/            PostgreSQL
├── docs/
│   ├── adr/
│   ├── api/
│   └── architecture/
└── README.md
```

## Stack

Go and Chi, Python and FastAPI, gRPC, PostgreSQL with pgvector, Redis when caching and rate limits show up, React and TypeScript, OpenTelemetry once there is a request path worth tracing.
