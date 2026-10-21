<!-- After each completed task, commit the changes. -->

RHDHPLAN-1644 scope revised 2026-10-20 for the RHDH 2.2 static single-agent MVP. Every item is a slice of at most two hours and must be split further if needed. All tasks remain unchecked; this is a design revision. No application code is changed. See [design.md](./design.md) and [report](./design-report-1644.md).

## 1. Contract and runtime proof

- [ ] 1.1 Pin the LCore, OGX, OpenAI Agents SDK and Cloud Agents versions; record the one-step executor extension interface.
- [ ] 1.2 Define the transient snapshot contract, model/tool/skill/sandbox binding schema and bounded agent-run request/event fixtures.
- [ ] 1.3 Prove one Cloud Agents agent step using the LCore-owned SDK adapter and OGX without nested loops.
- [ ] 1.4 Select snapshot-per-execution or startup publication for the MVP; specify the handshake and continuation behavior.
- [ ] 1.5 Define trusted user identity, correlated events and current authorization checks without interactive approval.

## 2. RHDH native packages and adapter

- [ ] 2.1 Create backend/common packages with native services, browser-safe contracts and schema declarations.
- [ ] 2.2 Resolve one static authorized root and explicit limits; reject outgoing handoffs and unsupported profile capabilities.
- [ ] 2.3 Resolve model/default precedence and approved inference/MCP/tool/knowledge bindings.
- [ ] 2.4 Resolve immutable skill artifacts and static sandbox policy; define backend credential binding translation.
- [ ] 2.5 Build the per-run snapshot and test static binding validation, no secrets and no snapshot persistence in conversation history.
- [ ] 2.6 Implement frontend-facing session/turn/history proxy routes and stream/error normalization.

## 3. LCore single agent execution

- [ ] 3.1 Add POST /v1/agents/run with dedicated request/response/event models; test static snapshot/revision validation, payload bounds and rejected definition overrides.
- [ ] 3.2 Implement single-agent controller validation and reject any second agent or handoff request.
- [ ] 3.3 Translate the profile into one schema-validated workflow step and a separate invocation envelope.
- [ ] 3.4 Connect SDK tool dispatch to exact allowlist intersection; test permitted, extra global and revoked tools.
- [ ] 3.5 Materialize pinned skills and retrieval bindings in the actual execution filesystem.
- [ ] 3.6 Wire configured sandbox creation, worker execution, cancellation and teardown; test fixed-image policy and no fallback.
- [ ] 3.7 Record cleanup failure and provide bounded retry/reconciliation evidence.
- [ ] 3.8 Stream correlated events and finalize persistent history before success.

## 4. History and concurrency

- [ ] 4.1 Design additive conversation/turn attribution migrations without snapshot storage or references; preserve legacy rows.
- [ ] 4.2 Implement a valid active-turn representation and request-id uniqueness/idempotency.
- [ ] 4.3 Implement one authoritative context adapter and prove two-turn recall with RHDH-supplied configuration and no transcript duplication.
- [ ] 4.4 Link run, sandbox, tools and cleanup evidence without copying configuration; define separate workflow-store retention and test failed/cancelled attribution.
- [ ] 4.5 Wire owner-scoped list/reopen/delete and retention links; test two-user isolation and dependency failures.
- [ ] 4.6 Implement one active turn per conversation and owner cancellation without late output.

## 5. Installation deployment and verification

- [ ] 5.1 Package static and OCI dynamic backend installation and publish common contracts.
- [ ] 5.2 Document PostgreSQL, dedicated LCore and pinned runtime dependencies plus migration/rollback order.
- [ ] 5.3 Deploy and validate the static OpenShell gateway/worker profile where sandboxing is configured.
- [ ] 5.4 Test model/MCP/artifact connectivity, TLS and credential handling from the actual issuing worker.
- [ ] 5.5 Test restart continuation with freshly supplied configuration, allowed/denied tools, one-agent enforcement, sandbox cleanup and request retry.
- [ ] 5.6 Run affected application checks and regenerate OpenAPI where wire schemas change.
- [ ] 5.7 Add the target ADR and reconcile Boost CURRENT/frontend architecture only after implemented behavior is verified.
- [ ] 5.8 Validate/promote the six capability specs on archive, preserving the append-only journal.
