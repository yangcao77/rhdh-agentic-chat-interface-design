## Canonical Touchpoints

No canonical documents are updated during design. Implementation will add target `workspaces/boost/specifications/adr/0001-chat-cloud-agents-lcore.md` and reconcile target Boost `specifications/CURRENT.md` and frontend architecture. No existing ADR was found at that target location. The six capability deltas in [proposal.md](./proposal.md) are archive candidates under `openspec/specs/`. No PRD update is proposed.

## Context

RHDHPLAN-1644 is scoped on 2026-10-20 to the RHDH 2.2 static single-agent MVP. The [design report](./design-report-1644.md) provides the complete configuration, execution, history and deployment design. [Research](./explore.md) separates inspected foundations from proposed integration work.

## Goals / Non-Goals

**Goals:** native Boost backend/common packages; one predefined agent; explicit tool allowlist; static model, skills, MCP and sandbox policy; LCore controller and OGX/OpenAI Agents SDK integration; Cloud Agents workflow execution; persistent LCore history; correlated streaming; owner isolation; installation and deployment.

**Non-Goals:** agent discovery; human-in-the-loop; multi-agent execution/handoffs in 2.2; generated definitions; user/model changes to tools or sandbox settings; additional execution backends; live configuration administration; advanced history/export; infrastructure operator implementation.

## Decisions

### D1 Native Boost backend and common packages

Use new `boost-chat-backend` and browser-safe `boost-chat-common` packages, working plugin ID `boost-chat` and `/api/boost-chat` routes. RHDH authenticates, authorizes, resolves static configuration and proxies execution/history. Use native Backstage services and schema-driven Node contracts. It does not run the SDK loop.

### D2 LCore controller and Cloud Agents execution

LCore hosts the execution/handoff controller. The 2.2 mode dispatches exactly one agent and rejects cross-agent transfers before another invocation starts. Embed Cloud Agents as an execution dependency. The only backend is OGX with an OpenAI Agents SDK adapter owned by the LCore integration. Implement a Cloud Agents step executor that uses that adapter, sharing context, allowlists, events and cancellation; do not nest independent agent loops. Pin and prove this extension interface rather than assuming endpoint availability proves compatibility.

### D3 Static AI Catalog configuration and snapshots

RHDH selects one predefined `AiResource` with no handoffs, not a discovered or dynamically assembled graph. Resolve its prompt/model and explicit allowed tools, MCP, skill, knowledge, limits and static sandbox profile. Preserve root identity; reject required unsupported configuration. The reference configuration contains multi-agent examples; those routers are not MVP profiles. Provider-level root/limit fields are absent from current agent entities and need explicit static profile bindings.

Recommend a resolved versioned snapshot on every execution. LCore validates and uses it for that run without saving the snapshot or a snapshot reference in conversation history. RHDH supplies configuration again for continuation and restart; LCore loads prior messages independently. Do not pin conversations to revisions or require a new conversation solely because configuration changes. RHDH owns static configuration compatibility. Startup publication is an alternative independent runtime configuration mode using the same schema; it needs a proposed publication API and readiness handshake. Implement one selected mode for MVP. No agent discovery or live authoring registry is introduced. Credential bindings resolve server-side.

### D4 One-step workflow contract

Translate the static profile into one supported Cloud Agents agent step, carrying model/SDK adapter, instructions, exact tools/MCP bindings, skill artifacts, limits, input/history mapping, output mapping and static spawn settings. Keep per-turn owner/input/request/conversation IDs in the invocation envelope. The selected step name `agent` and output `result` follow the inspected request documentation and must be verified against the pinned package schema. The report provides the required field checklist.

Add a new LCore `POST /v1/agents/run` endpoint for the execution controller. Its typed request carries user input, request identity, the static execution snapshot, and streaming preference. Field names remain a versioned contract to finalize. The controller validates identity/configuration, loads conversation context, dispatches one Cloud Agents step, streams correlated events, and finalizes persistent history before returning the terminal result. Use the embedded runner directly; an internal workflow is the execution representation. Define dedicated request/response/event models and a bounded payload limit. Keep artifact bytes outside the request and consume configuration before constructing OGX inference calls. This change adds a controller API and requires no change to the existing inference endpoint contract.

### D5 Allowed tools and capability access

Enforce static agent allowlist intersected with administrator policy and current user authorization at the actual SDK/tool dispatch boundary. MCP server selection alone is insufficient: include exact permitted tool names and schema expectations. Global defaults and skill instructions cannot enlarge the set. Allowed actions execute automatically; approval-dependent flows are unsupported.

LCore/worker connections require approved runtime-reachable endpoints, transport, TLS and backend authentication. Materialize immutable skill artifacts into the execution filesystem and map knowledge flags to real retrieval bindings. RHDH filesystem/network access does not imply worker access. No secrets in snapshots, routine traces or history.

### D6 Static sandbox spawning

The administrator fixes spawn mode, image digest, spawner binding, limits and mount/network/cleanup policy. For ephemeral execution Cloud Agents creates the configured environment before the one agent step, uses the compatible LCore-owned SDK worker adapter, and tears it down after success/failure/cancel. No model/user image or spawn overrides. No silent local fallback if provisioning fails. A sandbox covers the configured execution worker; remote MCP servers retain their own execution boundary.

The inspected harness has an OpenShell spawner; gateway compute-driver selection is deployment-owned. Define enforceable policy fields and map unsupported resource/network fields to the gateway's static configuration rather than inventing existing workflow schema support. Cleanup failures need recorded bounded retry/reconciliation.

### D7 Persistent LCore history

Extend current `user_conversation` with nullable agent reference/name, execution backend and contract version. Do not add snapshot storage, snapshot columns or snapshot references to conversation history. Extend `user_turn` with request ID, agent/run attribution and terminal status/error metadata. Its current non-null completion timestamp requires migration or a separate active-run row for pending turns. Link sandbox/tool/citation/cleanup evidence without duplicating model transcripts into user history. Preserve legacy rows without invented attribution. Keep execution configuration out of history evidence. The current Cloud Agents runner separately persists workflow definitions in its run store; define retention there independently rather than treating it as conversation configuration storage.

Use one authoritative context assembly, one logical user/final assistant turn, request-id idempotency and one active turn per conversation. Finalize history before terminal success. Ownership applies to list/reopen/continue/cancel/delete and linked run evidence. Production uses PostgreSQL, including the current workflow stores; broader local storage modes are not MVP promises.

### D8 Installation and deployment

Ship static and OCI dynamic Boost backend installation plus common contracts. Deploy dedicated Boost LCore with pinned SDK/Cloud Agents/OGX dependencies, PostgreSQL and static inference/MCP/artifact/spawner configuration. Cloud Agents is embedded; sandbox workers and its gateway are deployed when the profile requires them. Readiness validates the selected configuration mode and required runtime reachability. Disablement preserves history; additive database migrations preserve legacy clients.

## Diagrams

- [Architecture](./diagrams/architecture.md)
- [Single agent workflow](./diagrams/single-agent-workflow.md)
- [Request sequence](./diagrams/chat-sequence.md)
- [Execution lifecycle](./diagrams/execution-lifecycle.md)

## Risks / Trade-offs

- SDK/Cloud Agents executor mismatch: prove a real one-step invocation using OGX before dependent work.
- Static authoring with runtime credential/network differences: validate from the actual issuing worker.
- Snapshot resend overhead: bounded configuration envelopes and artifact references limit the payload; startup publication remains an alternative.
- Missing dispatch enforcement: deny unlisted tools even if a model, global default or skill asks for them.
- History duplication or persistence failure: one context source, idempotent requests and truthful terminal state.
- Sandbox cleanup or policy gaps: fixed profiles, enforced gateway policy and observable reconciliation.

## Remaining implementation decisions

[open-question.md](./open-question.md) records version/interface, exact API, credential, sandbox policy and history migration gates. Architecture and 2.2 scope are selected; no application code is implemented by this revision.
