# Connector and tool contracts

Status: Proposed  
Date: 2026-10-20

Connectors move data in. Tools let an agent read or propose a write. Both are typed. Neither is allowed to skip policy.

Go owns connector implementations for GitHub, Slack, and Jira. Python receives a tool catalog for the current organization and returns a proposed call. Go validates and authorizes that call, then invokes `execute`.

## Connector interface

Every provider implements the same surface. Provider-specific behavior stays inside the implementation. Callers see canonical objects and tool results.

| Method | Responsibility |
| --- | --- |
| `authenticate` | Exchange and refresh OAuth credentials via `credentials_ref`. |
| `capabilities` | Return the tool catalog this connection currently grants. |
| `sync` | Backfill or incremental pull from `sync_cursor`. Used by `POST /v1/connectors/{id}/sync`. Emits the same internal events a webhook would. |
| `search` | Provider-side search when an index miss must be checked at the source. Returns references, not prompts. |
| `get` | Fetch one object by external id and map it to the canonical table. |
| `execute` | Run one tool with validated arguments. Called only after policy allows it. |

`search` and `get` are reads used by ingestion and by the retriever when the local index is behind. They are not a side door around ADR-005. Results are still written or filtered under `organization_id`.

Mapping into canonical tables happens in a normalizer per provider. The rest of Keel does not branch on raw GitHub payloads.

## Tool definition

A tool is a versioned schema, not a free function call.

| Field | Meaning |
| --- | --- |
| name | Stable id, for example `jira.create_issue`. |
| version | Integer. Old versions stay executable until actions in flight finish. |
| provider | `github`, `slack`, `jira`, or `keel` for internal reads. |
| side_effect | `read`, `write`, `destructive`. |
| risk | Default `low`, `medium`, or `high`. Policy may raise risk, never lower it. |
| input_schema | JSON Schema. Unknown fields are rejected. |
| output_schema | JSON Schema for the result stored on the action. |
| idempotency | Which input fields define “the same operation”. |

Arguments are validated in Go before policy and again inside `execute`. Python’s proposal is untrusted. Extra fields, wrong types, and missing required fields fail the action as `rejected` with reason `invalid_arguments`.

## Risk defaults

| Risk | Examples | Default |
| --- | --- | --- |
| Low | Search, get, summarize. | Execute. |
| Medium | Create an issue, post a message, comment. | Follow organization policy. The default organization policy requires approval. |
| High | Merge a pull request, change permissions, delete a branch, close a project. | Approval is mandatory. Policy cannot set these to auto-execute in v1. |

The organization policy object is small:

| Field | Meaning |
| --- | --- |
| medium_risk | `require_approval` or `auto_execute`. Default `require_approval`. |
| high_risk | Always `require_approval` in v1. |
| disabled_tools | Tool names the agent may not propose. |

## Catalog for the first three providers

Ship read tools with the connector. Ship write tools with the action lifecycle (blueprint Phase 6), behind the state machine.

### GitHub

| Tool | Side effect | Risk |
| --- | --- | --- |
| `github.get_repository` | read | low |
| `github.search_code` | read | low |
| `github.get_pull_request` | read | low |
| `github.list_commits` | read | low |
| `github.get_issue` | read | low |
| `github.create_issue` | write | medium |
| `github.add_comment` | write | medium |
| `github.create_pull_request` | write | medium |
| `github.merge_pull_request` | destructive | high |
| `github.delete_branch` | destructive | high |

`github.create_branch` and permission changes stay out of the first catalog. Branch creation is easy to add once idempotency for refs is tested. Permission changes stay high-risk and unlisted until the approval inbox has been used for real.

### Slack

| Tool | Side effect | Risk |
| --- | --- | --- |
| `slack.list_channels` | read | low |
| `slack.get_thread` | read | low |
| `slack.search_messages` | read | low |
| `slack.post_message` | write | medium |

Posting is limited to channels the connector is a member of, and to `acl_team_ids` the caller may access.

### Jira

| Tool | Side effect | Risk |
| --- | --- | --- |
| `jira.search_issues` | read | low |
| `jira.get_issue` | read | low |
| `jira.get_project` | read | low |
| `jira.create_issue` | write | medium |
| `jira.add_comment` | write | medium |
| `jira.transition_issue` | write | medium |

`jira.create_issue` is the first write the demo must perform. Its input schema includes `project_key`, `issue_type`, `summary`, `description`, and optional `assignee_account_id` and `labels`. The output schema includes `key`, `id`, and `url`. Those three are what the audit row and the user see.

## How a call moves

1. Python selects a tool from the catalog Go supplied for this request. The catalog is already filtered by connector status and `disabled_tools`.
2. Python returns `tool_name`, `tool_version`, and `arguments`, plus a short reason and the citation ids it used.
3. Go checks the name against the catalog, validates arguments, computes risk, and runs policy.
4. On allow, Go creates the action row and, if required, the approval row, before any provider call.
5. On execute, the connector receives the action id and the idempotency key. It must persist the provider’s external id on the action as soon as the provider returns one, even if a later step of the same tool fails.

## Leaky abstraction rule

If a provider needs a field the canonical model does not have, add it to that tool’s input schema or to the object’s `metadata`. Do not add provider branches in the policy engine or the retriever. If three providers need the same field, promote it to a real column in the domain model.
