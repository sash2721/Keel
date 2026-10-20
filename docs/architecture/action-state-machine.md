# Action state machine

Status: Proposed  
Date: 2026-10-20

An action is a durable row, not a function call in flight. The process may restart at any state. Approval is a row, not a modal. Audit is an insert on each transition, not a final state.

Policy code in Go is the only thing that moves an action toward execution. The model proposes. It does not approve.

## States

| State | Meaning | Terminal |
| --- | --- | --- |
| `proposed` | Schema-valid tool call is stored. Policy has not been recorded. | |
| `awaiting_approval` | Policy requires a person. An `approvals` row exists. | |
| `rejected` | Policy denied it, or a person rejected it, or arguments were invalid. | Yes |
| `approved` | A person approved. Execution has not started. | |
| `executing` | A worker owns the provider call. | |
| `succeeded` | The provider result is stored. | Yes |
| `failed` | The provider call failed and will not be retried automatically. | Yes |

`cancelled` is reserved on the workflow, not the action. Cancelling a workflow rejects its still-open actions.

## Transitions

```text
proposed
  ├─ policy allow, risk may auto-execute ──► executing
  ├─ policy allow, approval required ──────► awaiting_approval
  └─ policy deny or invalid arguments ─────► rejected

awaiting_approval
  ├─ approve ──► approved ──► executing
  └─ reject  ──► rejected

executing
  ├─ provider success ──► succeeded
  ├─ retryable failure, attempts left ──► executing
  └─ terminal failure or attempt budget exhausted ──► failed
```

`approved → executing` happens in the same worker turn as the approval in the common case. The `approved` state exists so a crash after the decision and before the provider call can resume without asking the person again.

Illegal transitions are rejected by a conditional update:

```text
UPDATE actions
SET status = $next, updated_at = now()
WHERE id = $id
  AND organization_id = $org
  AND status = $expected
```

Zero rows means another worker took the transition. The loser stops.

## What an approval stores

The reviewer sees a snapshot, not a live re-plan. Re-planning after approval would change the thing they approved.

- Who requested it, and the correlation id.
- Tool name, version, and the validated arguments.
- Why the agent proposed it, as text.
- Citation ids and titles used as evidence.
- Risk and the policy version that required approval.
- Who approved or rejected, when, and an optional comment.
- After execution, the provider result or error on the action row.

The inbox lists `awaiting_approval` for callers with `action.approve`. Approving calls `POST /v1/actions/{id}/approve`. That handler checks role, checks organization, writes the approval decision, moves the action to `approved`, and enqueues execution.

A person cannot approve their own action in v1 when another admin exists. If they are the only admin, the audit event records `self_approved = true` and the transition is allowed. This keeps a single-operator demo possible and visible.

## Execution rules

- The action row and its idempotency key exist before `execute`.
- `attempt` increments when entering `executing` from `approved` or when retrying.
- Retryable errors: timeouts, 429, 5xx. Backoff is exponential, capped, with jitter. Budget is 5 attempts.
- Terminal errors: 4xx other than 429, schema mismatches, missing credentials. These go to `failed` immediately.
- The provider result, including the external id, is saved on the same update that sets `succeeded`.
- If the provider succeeded and the database update failed, the retry loads by idempotency key and reconciles. It does not create a second issue. See [Events and idempotency](events-and-idempotency.md).

## Workflow pause

A LangGraph run that needs approval persists its checkpoint, sets the workflow to `awaiting_approval`, and returns. The approve endpoint resumes the graph with the action result. The graph does not poll a person. The API process does not hold the HTTP request open across the wait.

Query-only workflows never create an action. They end in `succeeded` or `failed` with citations on the response and a `query.completed` audit event.

## Audit on each transition

| Transition | event_type |
| --- | --- |
| Create proposed | `action.proposed` |
| Policy denies | `action.refused` |
| Needs a person | `approval.requested` |
| Person approves | `approval.granted` |
| Person rejects | `approval.rejected` |
| Enter executing | `action.executing` |
| Succeeded | `action.succeeded` |
| Failed | `action.failed` |

Payloads include tool name, risk, and correlation id. They omit tokens and omit full source document bodies. Citation ids are enough to join back to chunks.
