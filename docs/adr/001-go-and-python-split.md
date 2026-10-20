# ADR-001: Go for product APIs, Python for orchestration

Status: Proposed  
Date: 2026-10-20

## Context

Keel has two kinds of work. The product path is authentication, multi-tenant data access, connectors, approvals, and audit. The AI path is LangGraph state, retrieval planning, prompts, and model calls. Those ecosystems do not share a strong library story. Putting both in one language means either a weaker agent stack or a weaker service stack.

The blueprint also draws many services: gateway, query, action, auth, approval, audit, connector, and ingestion. Shipping each as its own deployable before there is traffic creates operational surface with nothing to operate.

## Decision

Run two processes.

**Go** owns everything the product guarantees: HTTP API, identity, organizations, RBAC, PostgreSQL access, connector OAuth, ingestion workers, the action state machine, policy checks, and audit writes. Use Chi for HTTP. Keep auth, query, action, audit, connectors, and ingestion as packages in one module. Split a package into its own process only when it has an independent scaling or failure reason.

**Python** owns LangGraph: intent routing, planning, retrieval calls, synthesis, and tool selection. Expose it with FastAPI for health and local debugging. The Go API is the only caller of the orchestration RPCs, over gRPC with protobuf.

The browser talks only to Go. Python never accepts end-user credentials and never decides that an action is allowed. Go passes the caller’s organization, user, and roles into the RPC. Python returns a proposed tool call or a grounded answer. Go validates the proposal and runs policy.

Shared identifiers (`organization_id`, `user_id`, `correlation_id`, `workflow_id`) cross the RPC boundary as strings in the protobuf contract. PostgreSQL remains the source of truth for actions, approvals, and audit. LangGraph checkpoints may live in Postgres as well, keyed by `workflow_id`, so a paused approval can resume after a restart.

## Consequences

- Inter-service contracts live in `proto/` and are reviewed like APIs.
- A query pays one network hop from Go to Python. That is acceptable. Hiding it by embedding a model call inside a random Go handler is not.
- Local development is two processes plus PostgreSQL. Docker Compose is the expected dev environment once implementation starts.
- Package boundaries inside Go have to stay real. A connector package that imports the HTTP router has erased the boundary this ADR is buying.

## Alternatives

| Alternative | Why it was not chosen |
| --- | --- |
| Go only | The LangGraph and evaluation tooling we intend to use is Python-first. |
| Python only | The API, worker, and concurrency path is the larger body of deterministic code. Go fits that half. |
| One process per blueprint box, from day one | The blueprint lists premature microservice explosion as something to postpone. The boxes remain packages. |
| JSON over HTTP between Go and Python | Fine for a spike. gRPC makes the tool-proposal and stream contracts explicit, which is the contract we will get wrong if it stays implicit. |
