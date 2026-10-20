# Domain model

Status: Proposed  
Date: 2026-10-20

PostgreSQL is the system of record (ADR-003). Every tenant-owned row carries `organization_id`. Application queries include it. Identifiers are UUIDs unless the row is an external object, which also stores the provider’s id.

A user belongs to one organization in v1. The membership row still exists so a later multi-org user does not require a new table, only a relaxed uniqueness rule.

## Identity and access

### organizations

| Column | Notes |
| --- | --- |
| id | Primary key. |
| name | Display name. |
| slug | Unique, used in URLs. |
| plan | `free` until billing exists. Stored so the column is not a migration later. |
| created_at | |

### users

| Column | Notes |
| --- | --- |
| id | Primary key. |
| organization_id | Required in v1. |
| email | Unique inside the organization. |
| display_name | |
| identity_subject | OIDC subject, or a local-dev subject with the same shape. |
| status | `active`, `disabled`. |

### memberships

| Column | Notes |
| --- | --- |
| organization_id | |
| user_id | Unique together with organization_id in v1. |
| role | `owner`, `admin`, `member`, `viewer`. |

### teams and team_members

A team has `id`, `organization_id`, and `name`. `team_members` links `team_id` and `user_id`.

### Roles

| Role | Can |
| --- | --- |
| viewer | Run queries and open search. |
| member | Viewer, plus propose actions and read workflows they started. |
| admin | Member, plus manage connectors, approve actions, read the organization audit log, edit action policy, and read any workflow in the organization. |
| owner | Admin, plus manage memberships. |

Permission names used by the policy engine: `query.run`, `action.propose`, `action.approve`, `connector.manage`, `audit.read`, `policy.manage`, `membership.manage`. Roles map to these in code. A custom role table waits until a real organization needs one.

## Connectors

### connectors

| Column | Notes |
| --- | --- |
| id | |
| organization_id | |
| provider | `github`, `slack`, `jira`. |
| status | `pending`, `connected`, `error`, `disabled`. |
| external_account_id | Provider workspace, org, or site id. |
| credentials_ref | Pointer into the secret store. Never the token. |
| scopes | Granted scopes. |
| sync_cursor | Provider cursor for backfill and incremental sync. |
| last_synced_at | |
| last_error | Short operator-facing message. |

Unique `(organization_id, provider, external_account_id)`.

Secrets live outside the row. Local development may use an encrypted column or a file-backed secret store behind the same `credentials_ref` interface. Production uses a KMS-backed secret manager. The interface is `Put`, `Get`, `Delete` by reference. The database never stores a plaintext OAuth token.

## Source objects

Each source type is its own table. They share a column convention so ingestion and retrieval code can treat them uniformly.

Shared columns:

| Column | Notes |
| --- | --- |
| id | Internal UUID. This is what relationships and chunks point at. |
| organization_id | |
| connector_id | |
| external_id | Provider id. Unique with `(organization_id, connector_id)`. |
| title | |
| body | Text we are willing to chunk. Empty when the object is only a pointer. |
| url | Deep link. |
| author_external_id | Provider user id. |
| source_created_at, source_updated_at | From the provider. |
| metadata | JSON for provider fields we do not promote to columns yet. |
| acl_team_ids | Null means visible to the whole organization. A non-null list restricts to those teams. |
| deleted_at | Soft delete when the provider removes the object. Chunks of a deleted object leave the index. |

### repositories

Shared columns, plus `name`, `full_name`, `default_branch`, `owner_team_id` nullable.

### documents

Shared columns, plus `doc_type` (`readme`, `adr`, `page`, `other`). The first documents are repository files we choose to index, starting with README and ADR-style markdown. Arbitrary file indexing is a later switch.

### messages

Shared columns, plus `channel_external_id`, `thread_external_id`, `channel_name`. Body is the message text. The thread id groups a conversation for retrieval.

### issues

Shared columns, plus `key` (for example `PAY-391`), `project_id` nullable, `status`, `assignee_external_id`, `labels` as text array. Jira issues use this table. GitHub issues use it too, with the repo id in metadata and `key` set to the number string. `provider` on the connector distinguishes them.

### pull_requests

Shared columns, plus `repository_id`, `number`, `status` (`open`, `merged`, `closed`), `author_external_id`. The body stored for retrieval is the title, description, and a short review summary. Raw diffs are not embedded by default. The URL still opens the diff at the provider.

### projects

Shared columns, plus `key`, `status`, `owner_team_id` nullable. Jira projects are the first rows.

## Relationships

### entity_relationships

| Column | Notes |
| --- | --- |
| id | |
| organization_id | |
| subject_type, subject_id | Type is the table name. |
| predicate | Closed set below. |
| object_type, object_id | |
| provenance | JSON: connector id, event id, extractor (`rule` or `model`), confidence. |
| created_at | |

Unique `(organization_id, subject_type, subject_id, predicate, object_type, object_id)`.

Rules write relationships first. A model may add relationships later, and those rows must keep `extractor = model` and a confidence. A rule-derived row is not overwritten by a model row.

| Predicate | Example |
| --- | --- |
| belongs_to | User → team. Issue → project. |
| member_of | User → team. |
| owns | User or team → repository or service. |
| implemented_by | Service → repository. Service is a row in `repositories` or a later `services` table. Until then, “service” is a repository alias. |
| contains | Repository → pull request. |
| references | Pull request → issue. Message → issue. |
| describes | Document → repository. |
| discusses | Message → issue or pull request. |
| authored | User → pull request or message. |
| assigned_to | Issue → user. |

User and service nodes that are not source objects live in `directory_entities` (`id`, `organization_id`, `kind` = `person` or `service`, `name`, `external_id`). People discovered from commits and Slack are directory entities linked to `users` when the email or identity matches.

Aliases (`payments service`, `payments-service`) live in `entity_aliases` (`organization_id`, `entity_type`, `entity_id`, `alias`). Entity resolution reads this table before it guesses.

## Retrieval

### chunks

| Column | Notes |
| --- | --- |
| id | |
| organization_id | |
| source_type, source_id | Parent object. |
| ordinal | Order inside the parent. |
| text | |
| token_count | |
| embedding | `vector`. Nullable until the embed stage finishes. |
| embedding_model | Required once `embedding` is set. |
| search_vector | `tsvector` on `text`. |
| acl_team_ids | Copied from the parent at index time. |
| source_updated_at | Copied so fusion can prefer newer evidence. |
| metadata | JSON citation fields: provider, title, url, author, repository, project. |

Unique `(source_type, source_id, ordinal)`.

Chunking defaults, open to change once real text is ingested:

- Target 500–800 tokens, overlap about 100.
- Split on markdown headings, message thread boundaries, and issue description versus comments.
- Skip chunks whose parent is soft-deleted.

A chunk enters hybrid search only when both `embedding` and `search_vector` are set, and the parent is not deleted.

## Actions and audit

The action lifecycle is specified in [Action state machine](action-state-machine.md). The tables are here so the model stays in one place.

### workflows

| Column | Notes |
| --- | --- |
| id | |
| organization_id | |
| correlation_id | |
| user_id | Who started it. |
| kind | `query` or `action`. |
| status | `running`, `awaiting_approval`, `succeeded`, `failed`, `cancelled`. |
| checkpoint_ref | LangGraph checkpoint key. |
| created_at, updated_at | |

### actions

| Column | Notes |
| --- | --- |
| id | |
| organization_id | |
| workflow_id | |
| correlation_id | |
| actor_user_id | |
| connector_id | |
| tool_name, tool_version | |
| arguments | JSON, schema-valid. |
| arguments_hash | Canonical JSON hash, part of the idempotency key. |
| risk | `low`, `medium`, `high`. |
| policy_id | Policy version that decided. |
| status | See the state machine. |
| idempotency_key | Unique per organization. |
| attempt | Integer. |
| result | JSON provider result, including external id. |
| error | Last error, if failed. |
| created_at, updated_at | |

### approvals

| Column | Notes |
| --- | --- |
| id | |
| action_id | Unique. One open approval per action. |
| organization_id | |
| requested_by | |
| payload | The snapshot a reviewer must see: tool, arguments, risk, reason, evidence citations, policy name. |
| decided_by | Null until a decision. |
| decision | `approved` or `rejected`. |
| comment | |
| decided_at | |

### audit_events

Append-only. Application code inserts. It does not update or delete.

| Column | Notes |
| --- | --- |
| id | |
| organization_id | |
| correlation_id | |
| actor_type | `user`, `system`, `agent`. |
| actor_id | |
| event_type | For example `query.completed`, `action.proposed`, `action.refused`, `approval.granted`, `action.succeeded`. |
| target_type, target_id | |
| payload | JSON. Arguments may be stored. Secrets and raw tokens must not. |
| created_at | |

## Isolation rules

- Session organization is the only organization a request may name.
- Retrieval SQL is built by one helper that requires `organization_id` and applies `acl_team_ids`.
- `acl_team_ids` null means every member of the organization. That is the v1 default until a connector records real ACLs.
- A non-member, and any member of another organization, receives no rows and no “this exists but you cannot see it” detail.
- Actions, approvals, and audit reads use the same organization predicate.

## What is deliberately absent

No billing tables. No `services` table beyond repository aliases. No per-user grant table. No cross-organization sharing. Add them when a journey needs them, in a new ADR.
