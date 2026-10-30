<!-- After each completed task, commit the changes. -->

Implementation is organized into the same three workstreams as the recreated specs. All tasks are unchecked; this revision changes planning artifacts only. Investigation tasks must record the selected contracts and evidence before dependent implementation. Keep implementation slices within two hours where practical and split them further when needed; the consolidated documentation deliverable remains one task as requested.

Configuration authoring for the integrated runtime, persistence, secure credential delivery, RBAC and optional sandbox infrastructure belongs to workstream 1. LCore history implementation and migrations remain in workstream 2. Workstream 3 packages and installs those deliverables, verifies the deployed stack and documents it in one consolidated task.

## 1. RHDH backend and integrated configuration

Spec: [boost-chat-rhdh-backend](./specs/boost-chat-rhdh-backend/spec.md).

- [ ] 1.1 Investigate Intelligent Assistant's router, identity middleware, history proxy, common permissions and RBAC policy; record the reusable patterns and agentic-chat permission/route contracts with request/response fixtures.
- [ ] 1.2 Trace AI Catalog agent schema/type validation and authoring/provider mappings; identify every path that must preserve spec.useSandbox and the existing SDK configuration fields.
- [ ] 1.3 Investigate the Intelligent Assistant configuration repository and harness secret examples; select the backend-owned integrated configuration location, persistence settings, service-authentication/owner-forwarding contract, credential delivery/rotation, RBAC examples and optional sandbox infrastructure settings, distinguishing production setup from library-mode/noop/SQLite development examples.
- [ ] 1.4 Scaffold plugins/agentic-chat-backend and plugins/agentic-chat-common with native Backstage services, backend config declarations and registration; verify common has no browser-unsafe runtime imports.
- [ ] 1.5 Add optional useSandbox?: boolean to the AI Catalog agent TypeScript and JSON schema contracts; verify omitted/false/true values and reject non-booleans while retaining existing entity compatibility.
- [ ] 1.6 Update the identified authoring/provider mappings to preserve useSandbox through Catalog ingestion; verify true survives ingestion and omission resolves to false at execution.
- [ ] 1.7 Implement authorized entity resolution and static snapshot mapping for instructions, available model/settings, tools/MCP, skills, knowledge, limits, output mapping and useSandbox; test missing bindings, no invented capabilities and rejection of handoffs/overrides.
- [ ] 1.8 Implement POST /v1/agents/run under the configured plugin base path, forwarding the query, snapshot, trusted owner, request ID, optional conversation ID and stream preference to LCore; test payload bounds and continuation without local chat storage.
- [ ] 1.9 Implement GET /v2/conversations and GET /v2/conversations/:conversationId against the agreed LCore history contract; test empty results, pagination, ordered messages, missing sessions, upstream failure and cross-user denial.
- [ ] 1.10 Register chat-use/history permissions and enforce their agreed coverage on run/list/history/cancel/delete routes; verify Catalog group-to-role grants, missing grants and ownership isolation independently of frontend visibility.
- [ ] 1.11 Implement backend-only LCore service authentication and trusted owner forwarding using the contract agreed in 2.1 and 1.3; verify no browser-supplied identity authority, no token leakage and no credential forwarding to unapproved destinations.
- [ ] 1.12 Wire the native Boost frontend adapter to the new routes, correlated stream, safe failures and owner cancellation; preserve request IDs on retry and verify no late answer output or duplicate local history.
- [ ] 1.13 Complete the owner-scoped delete/cancel proxy contract and run focused backend/common checks, including rejected direct unauthorized requests and dependency failures.
- [ ] 1.14 Assemble the backend-owned RHDH/LCore/Cloud Agents connection and capability configuration using the tested runtime version/image bindings from 2.2; verify controller/workflow endpoints, approved inference/MCP/artifact references and configuration acceptance with missing optional bindings.
- [ ] 1.15 Supply PostgreSQL conversation/workflow connection and storage configuration, database secret references, readiness settings and migration/startup/rollback/disable-re-enable settings; verify compatibility with LCore's migrations without moving history storage into RHDH.
- [ ] 1.16 Add placeholder-only Kubernetes/OpenShift Secret or external-secret configuration for service/model/MCP/artifact/database/sandbox tokens and credentials; verify backend-only config visibility, TLS, safe RHDH-to-LCore authentication, secret-reference injection, any transient forwarding and credential rotation without persistence or log leakage.
- [ ] 1.17 Supply static agent examples with omitted/false/true useSandbox and RHDH RBAC group-to-role configuration for chat/history permissions; verify group grants, denied nonmembers and cross-owner denial.
- [ ] 1.18 Supply the approved sandbox gateway/worker image, service-account and resource/network/mount/cleanup configuration where needed; verify actual worker connectivity, no gateway dependency for unsandboxed agents and no fallback on enabled provisioning failure, coordinating lifecycle implementation with 2.8.

## 2. LCore and Cloud Agents

Spec: [boost-chat-lcore-cloud-agents](./specs/boost-chat-lcore-cloud-agents/spec.md).

- [ ] 2.1 Investigate the harness /v1/workflows/run creation path, shared validation/authorization, trusted RHDH identity, secret selection, event/status/cancel APIs and owner-scoped conversation APIs; record the controller request/event/history contract and payload bounds with RHDH.
- [ ] 2.2 Investigate Cloud Agents executor interfaces and pin compatible LCore/Cloud Agents/SDK/OGX revisions; record gaps between the current Pydantic AI direct executor and the required LCore-owned OpenAI Agents SDK bridge, including both spawn modes.
- [ ] 2.3 Prove one real Cloud Agents agent step through the LCore-owned SDK adapter and configured OGX route with an explicit tool list; retain execution evidence and identify remaining integration work without claiming endpoint presence proves support.
- [ ] 2.4 Add POST /v1/agents/run and its typed request/response/event models; test trusted caller/owner validation, snapshot bounds, accepted missing model/skill/tool bindings and rejected handoffs/definition overrides.
- [ ] 2.5 Translate the snapshot into one static workflow step and a separate invocation/context envelope; dispatch through POST /v1/workflows/run or its shared authenticated submission service and verify inherited validation, credential rules and one run ID per turn.
- [ ] 2.6 Implement the required SDK executor bridge across the controller and Cloud Agents; carry model settings, authoritative context, streams, cancellation and final output without a second agent loop.
- [ ] 2.7 Explicitly map useSandbox false/omitted to the supported unsandboxed spawn mode and true to the fixed sandbox profile; prove the Cloud Agents ephemeral default cannot accidentally provision a sandbox.
- [ ] 2.8 Wire configured sandbox provisioning and terminal cleanup; verify provision failure stops the run without fallback, partial resources are cleaned up and cleanup failures retain bounded reconciliation evidence.
- [ ] 2.9 Enforce allowedTools in SDK/worker exposure and dispatch; verify omission, empty/populated lists, missing tool bindings, revoked permissions and skill/approval bypass attempts.
- [ ] 2.10 Materialize configured immutable skills and approved MCP/retrieval bindings in the issuing process; verify worker connectivity/TLS, accepted missing bindings and safe failures for capabilities needed but unavailable during execution.
- [ ] 2.11 Finalize and implement additive conversation/turn migrations for nullable attribution, request/run linkage and pending/failed states; account for current non-null completed_at/provider/model fields and verify legacy rows and pre-inference failures.
- [ ] 2.12 Implement one authoritative history/context adapter and owner-scoped list/retrieve integration; prove two-turn recall and restart continuation with resupplied configuration, no history snapshots and no transcript copying or duplicate replay by default.
- [ ] 2.13 Enforce request-id idempotency, one active turn per conversation and agent/model/tool/token/time limits; test repeated requests, competing turns and correlated limit outcomes without duplicate workflow creation.
- [ ] 2.14 Connect normalized execution events, owner cancellation and history finalization; verify no late output, failed/cancelled attribution and persistence failure reported instead of saved success.
- [ ] 2.15 Enforce ownership on deletion and linked run evidence, independent workflow-store retention and safe credential resolution; verify deletion failures/shared resources and absence of secret values in persisted workflow inputs, transcripts, history and diagnostics.
- [ ] 2.16 Run affected LCore/Cloud Agents tests and regenerate OpenAPI for the new wire contracts; retain real execution, history and cancellation evidence for deployment acceptance.

## 3. Deployment, installation and consolidated documentation

Spec: [boost-chat-deployment](./specs/boost-chat-deployment/spec.md).

- [ ] 3.1 Investigate supported RHDH package distribution and installation charts/manifests; define how installation applies the backend-owned runtime, storage, secret-reference, RBAC and optional sandbox artifacts without introducing another configuration contract.
- [ ] 3.2 Package static and OCI dynamic RHDH backend installation and frontend/common wiring; verify both modes expose the run, session-list and per-session history routes.
- [ ] 3.3 Install the pinned LCore/Cloud Agents runtime, PostgreSQL stores and required sandbox infrastructure using the backend-owned configuration artifacts; follow the supplied migration/startup order and verify readiness, saved history after restart and compatible rollback/disable-re-enable behavior.
- [ ] 3.4 Run deployed acceptance checks for execution, tool-filter defaults/restrictions, list/retrieve/delete history, two-turn/restart continuation, group/owner isolation, secret handling, cancellation, truthful persistence outcomes and applicable sandbox cleanup; capture results for the consolidated guide.
- [ ] 3.5 Produce one consolidated RHDH installation/configuration/operations guide covering installation and the backend-owned configuration for the entire stack: packages, version/image pins, topology, databases/migrations, frontend/backend routes, agent/useSandbox examples, connections, secret creation/injection/forwarding/rotation, group RBAC, sandbox setup, readiness, verification, rollback and troubleshooting. In this same documentation task, reconcile the implemented Boost CURRENT/frontend architecture and add target workspaces/boost/specifications/adr/0001-chat-cloud-agents-lcore.md; identify both stretch goals without making them prerequisites.
- [ ] 3.6 Validate the implemented change against these three specs and promote them to openspec/specs/ on archive, preserving the append-only journal.

## Stretch goals outside MVP acceptance

Startup configuration synchronization and optional storage of available intermediate execution output or thought-process traces in chat history remain stretch goals. The MVP sends configuration per run and keeps Cloud Agents transcripts as linked evidence outside main chat history by default.
