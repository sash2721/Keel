# ADR-005: Permission filters before model context

Status: Proposed  
Date: 2026-10-20

## Context

Retrieved text is both sensitive and untrusted. A chunk from another organization, a private Slack channel, or a repository the caller cannot see must never enter a prompt. A prompt that says “do not reveal unauthorized documents” is not an access-control mechanism. The model will see the text, and it may repeat it.

The same hole exists for tools. A model that has read a malicious issue can request a merge or a permission change. If the model’s own judgment is what allows the call, the issue is now an operator.

## Decision

**Tenant and ACL filters are SQL predicates on every retrieval query.** The organization id comes from the authenticated session inside Go. The client cannot supply a different organization. Python receives an already-scoped retrieval request and cannot widen it. There is no code path that loads chunks for a prompt and then asks the model to hide some of them.

**Retrieved text is untrusted data.** Prompts label source chunks as data, not as instructions. Tool selection uses the user request and structured retrieval results. Text inside a chunk cannot add a tool, change a risk level, or mark an action as approved. The policy engine in Go is deterministic: it sees the authenticated user, the tool name, the validated arguments, and the organization’s policy. It does not see a model-written “allowed: true”.

**Writes are proposed, then authorized.** Python returns a tool name and arguments. Go validates them against the tool schema, computes the risk, and applies policy. Low-risk reads may run. Medium-risk writes follow the organization’s setting. High-risk writes (merge, permission change, delete) always wait for a human approval record.

**Audit the decision, including refusals.** A refused retrieval is an audit event when it is a cross-tenant attempt or an explicit policy denial. Ordinary empty search results are not.

## Consequences

- Connectors must record enough ACL to enforce. Until a connector has real per-object ACLs, objects are visible to every member of the organization and to nobody outside it. That rule is documented in the domain model so we do not pretend private channels are filtered before the Slack connector learns membership.
- Tests for this ADR are security tests: a second organization, a user without `action.approve`, and a fixture document that instructs the agent to merge a pull request.
- Prompt templates and policy code are reviewed separately. Improving the prompt is not a substitute for a policy change.

## Alternatives

| Alternative | Why it was not chosen |
| --- | --- |
| Retrieve broadly, instruct the model to respect permissions | The model has already seen the secret. Instruction-following is not authorization. |
| Let the agent call tools and reject after the fact | Side effects will already have happened. Approval has to be a pause before execution. |
| Share one index across tenants with application-side filtering only | A missing `WHERE` becomes a data leak. The predicate belongs in the query helper that every retrieval path uses, with a test that a query without `organization_id` does not compile or does not run. |
