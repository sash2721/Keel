# Product requirements

Status: Proposed  
Date: 2026-10-20

## Problem

An engineer’s context is split across GitHub, Slack, and Jira. The reason for a change lives in a pull request, a ticket, and a thread, and none of those tools is responsible for connecting them. Asking a model to chat over exported documents loses permissions, provenance, and any safe way to act.

Keel is the layer that holds that context and can do bounded work inside it.

## Users

| User | Need |
| --- | --- |
| Engineer | Technical context in one place, then a careful action in GitHub, Jira, or Slack. |
| Engineering manager or tech lead | Decision history, what is blocked, and workflow help that stays inside policy. |
| New teammate | Answers that cite the original system instead of a search across five tools. |
| Platform owner | Connectors, roles, approval policy, and an audit trail. |

## What the first version must do

1. Connect GitHub, Slack, and Jira for one organization, with OAuth credentials stored as secret references.
2. Ingest the objects those systems already use for engineering work: repositories, pull requests, issues, messages, threads, and projects.
3. Keep a canonical record of those objects and the relationships between them.
4. Answer a question with citations that deep-link to the source, scoped to the caller’s organization.
5. Propose a typed write action, check policy, pause for approval when required, execute once, and record the result.
6. Show a platform owner the pending approvals and the audit history for a request.

## Journeys

**Ask.** “Why was the payment service migrated from Redis?”  
The system detects intent and scope, retrieves with semantic search, keyword search, metadata filters, and relationship walk, then returns an answer, citations, and related entities.

**Investigate.** “What is currently blocking the billing project?”  
The answer draws on Jira issues, GitHub pull requests, and Slack discussions together, each cited.

**Act.** “Create a Jira issue for the authentication bug and assign it to the service owner.”  
The agent selects a tool and fills typed arguments. Policy classifies the risk. A person approves. Keel creates one issue and writes an audit event.

**Control.** A platform owner requires approval before any GitHub write. The model cannot override that rule.

## Finish line

Phase 7 from the blueprint is the first complete version:

- GitHub and Slack ingested and searchable.
- Grounded answers with citations.
- Jira actions with a lifecycle.
- Human approval for risky writes.
- An audit log a person can query.

Multi-step workflows (“find duplicate billing issues, consolidate them, notify the team”) and the production harness (tracing, dead-letter queues, load tests, a full evaluation regression) come after that line. Design the state model so they fit. Do not build them first.

## Demo acceptance

These five stories are the acceptance tests. A demo that only shows a chat box has missed the product.

| # | Request | What must be visible |
| --- | --- | --- |
| 1 | Why did we move payments from Redis to Kafka? | Hybrid retrieval, a relationship, citations. |
| 2 | What is blocking the billing project? | Jira, GitHub, and Slack in one answer. |
| 3 | Create a Jira issue for the authentication bug. | Tool choice, policy, approval, one execution, audit. |
| 4 | Find duplicate billing issues, consolidate them, and notify the team. | A plan and more than one step. This demo waits until workflows exist. |
| 5 | A Slack message tells the agent to ignore policy and merge a pull request. | The message is treated as data. Policy refuses the merge. |

## Out of scope

Further integrations, including Notion. A mobile app. Voice. A custom foundation model. Autonomous merges, permission changes, and deletes. A dedicated graph database. Billing. A large set of specialist agents. Kubernetes and Terraform as a starting point. Resume claims written before the system has been measured.

## Quality bar

- Every important claim in an answer points at a retrieved source.
- A person in organization A cannot retrieve or act on organization B, including by asking the model to.
- A retried action does not create a second external object.
- An approval still exists after the API process restarts.
- Connector and model failures retry or surface a clear error. They do not corrupt the canonical record.

## Evaluation

Start a small labeled set before the grounded-answering phase, and grow it to 50–100 cases. Each case names the organization fixture, the question or command, the sources that must appear, the tools that must be chosen, and whether policy must allow or refuse. Cover retrieval, grounding, entity resolution, tool arguments, and at least one prompt-injection refusal. Manual demos do not replace this set.

## Open choices left in this PRD

The embedding vendor, the exact reranker, and the user-login provider (OIDC vendor versus a local dev login that can be replaced by OIDC) are interfaces in the architecture docs. Pick the vendors when implementation reaches them.
