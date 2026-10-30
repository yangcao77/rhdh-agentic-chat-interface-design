## Canonical Touchpoints

No canonical documents are updated during design. Implementation will add target `workspaces/boost/specifications/adr/0001-chat-cloud-agents-lcore.md` and reconcile target Boost `specifications/CURRENT.md` and frontend architecture. No existing ADR was found at that target location. The three capability deltas in [proposal.md](./proposal.md) are archive candidates under `openspec/specs/`. No PRD update is proposed.

## Context

RHDHPLAN-1644 is scoped to the RHDH 2.2 static single-agent MVP, clarified on 2026-10-28. The [design report](./design-report-1644.md) provides the complete configuration, execution, history and deployment design. [Research](./explore.md) separates inspected foundations from proposed integration work.

## Goals / Non-Goals

**Goals:** native Boost backend/common packages; one predefined agent; optional tool filter defaulting to all available tools; optional per-agent spawn mode deferring to the LCore default when omitted; static model, skills, MCP and sandbox policy; LCore controller and OGX/OpenAI Agents SDK integration; Cloud Agents workflow execution; persistent LCore history; correlated streaming; owner isolation; installation and deployment.

**Non-Goals:** agent discovery; human-in-the-loop; multi-agent execution/handoffs in 2.2; generated definitions; user/model changes to tools or sandbox settings; additional execution backends; live configuration administration; advanced history/export; infrastructure operator implementation.

## Decisions

### D1 Native RHDH plugin packages

Create `plugins/agentic-chat-backend` and browser-safe `plugins/agentic-chat-common`, retaining working plugin ID `boost-chat` and `/api/boost-chat` routes. RHDH owns the native Boost frontend adapter, authenticates, authorizes, resolves static configuration and proxies execution/history. Use native Backstage authentication, permissions, Catalog, config, HTTP routing and lifecycle services. Common contains browser-safe request, event and history types plus shared validation contracts. RHDH does not run the SDK loop.

### D2 LCore controller and Cloud Agents execution

LCore hosts the execution/handoff controller. The 2.2 mode dispatches exactly one agent and rejects cross-agent transfers before another invocation starts. Embed Cloud Agents as an execution dependency. Implement a Cloud Agents step executor that uses that adapter, sharing context, allowlists, events and cancellation; do not nest independent agent loops. Pin and prove this extension interface rather than assuming endpoint availability proves compatibility.

### D3 Static AI Catalog configuration and snapshots

Retain the current AI Catalog agent configuration schema for the OpenAI Agents SDK integration in 2.2. RHDH selects one predefined `AiResource` with no handoffs, not a discovered or dynamically assembled graph. Resolve its configured prompt/model, tools/MCP bindings, skills, knowledge, limits, optional allowedTools and optional sandbox settings into the snapshot sent by agentic-chat-backend. LCore owns SDK integration and Cloud Agents workflow translation; do not require an AI Catalog workflow-schema migration. The intended MVP profile supplies an explicit allowed tools list, but omission remains valid and allows all available configured tools. A missing spawn mode is passed through so LCore applies its default. Missing model, skill or tool bindings do not reject configuration. Preserve root identity and reject unsupported handoffs or caller overrides. The reference configuration contains multi-agent examples; those routers are not MVP profiles. Provider-level root/limit fields are absent from current agent entities and need explicit static profile bindings.

Add optional `spec.spawn` to the existing agent schema, matching the LCore workflow step type `Optional[Literal["none", "local", "ephemeral"]] = None`, and preserve it through Catalog ingestion. `none` runs the agent in the LCore process, `local` runs it in a child process (neither is sandboxed), and `ephemeral` runs it in the administrator-predefined sandbox profile. Omission stays unset in the snapshot and LCore applies the Cloud Agents default, currently `ephemeral`, so an agent without `spawn` needs a configured spawner. LCore currently allows `none` and `local` only for admin principals, so the RHDH service identity must be authorized for any mode an agent selects.

Resolve a model from the agent or an explicit administrator default when available. Missing bindings alone are accepted by both RHDH and LCore validation, without inventing a model, skill, tool or connection. If execution needs an unavailable capability, report a safe runtime failure. Sandbox provisioning failure (spawn `ephemeral`, set or defaulted) stops that run without changing execution mode.

Send the resolved static agent configuration with every execution, including continuation and restart. LCore uses it for that run without saving the snapshot or a reference in conversation history, and loads prior messages independently. Startup configuration synchronization is a stretch goal using the same contract. Dynamic reload, change detection and configuration pickup are out of scope. Credential bindings resolve server-side.

### D4 One-step workflow contract

RHDH sends the resolved agent profile; LCore translates it into one supported Cloud Agents agent step with available model/SDK adapter, instructions, configured tools/MCP bindings, optional allowedTools, skills, limits, context/output mapping and optional sandbox settings. Preserve omitted allowedTools versus an explicit empty list and pass spawn through unchanged, including omission. Keep per-turn workflow run ID, owner/input/request/conversation IDs in the invocation envelope; executable definitions and capabilities remain administrator-predefined. The selected step name `agent` and output `result` follow the inspected request documentation and must be verified against the pinned package schema. The report provides the required field checklist.

Add a new LCore `POST /v1/agents/run` endpoint for the execution controller. Its typed request carries user input, request identity, the static execution snapshot, and streaming preference. Field names remain a versioned contract to finalize. The controller validates identity/configuration, loads conversation context, dispatches one Cloud Agents step, streams correlated events, and finalizes persistent history before returning the terminal result. Dispatch workflow creation through `POST /v1/workflows/run` or its shared authenticated submission service, preserving workflow validation, authorization and credential rules; an internal workflow is the execution representation. Define dedicated request/response/event models and a bounded payload limit. Keep artifact bytes outside the request and consume configuration before constructing OGX inference calls. This change adds a controller API and requires no change to the existing inference endpoint contract.

### D5 Allowed tools and capability access

When allowedTools is omitted, all tools available through the agent's configured tool/MCP bindings are allowed without an extra agent-level filter. When supplied, enforce the list at actual SDK/tool dispatch; [] allows no tools. Existing deployment policy and user authorization still apply. Preserve omission versus empty-list semantics through the adapter and worker transport. Skill instructions cannot bypass a supplied filter. Allowed actions execute automatically; approval-dependent flows are unsupported.

LCore/worker connections require approved runtime-reachable endpoints, transport, TLS and backend authentication. Materialize configured immutable skill artifacts into the execution filesystem and map knowledge flags to real retrieval bindings. Missing model, skill or tool bindings alone do not reject configuration; unavailable capabilities cannot be invented or used. RHDH filesystem/network access does not imply worker access. No secrets in snapshots, routine traces or history.

### D6 Static sandbox spawning

The spawn mode is optional per static agent configuration. Spawn `none` or `local` runs the agent without a sandbox; no spawner or sandbox image is required. An omitted spawn mode is not a configuration error and resolves to the LCore default, currently `ephemeral`. For `ephemeral`, the administrator fixes image, spawner, limits and mount/network/cleanup policy. For sandboxed execution Cloud Agents creates the configured environment before the one agent step, uses the compatible LCore-owned SDK worker adapter, and tears it down after success/failure/cancel. No model/user image or spawn overrides. No silent local fallback if provisioning fails. A sandbox covers the configured execution worker; remote MCP servers retain their own execution boundary.

The inspected harness has an OpenShell spawner; gateway compute-driver selection is deployment-owned. Define enforceable policy fields and map unsupported resource/network fields to the gateway's static configuration rather than inventing existing workflow schema support. Cleanup failures need recorded bounded retry/reconciliation.

### D7 Persistent LCore history

Extend current `user_conversation` with nullable agent reference/name, execution backend and contract version. Do not add snapshot storage, snapshot columns or snapshot references to conversation history. Extend `user_turn` with request ID, agent/run attribution and terminal status/error metadata. Its current non-null completion timestamp requires migration or a separate active-run row for pending turns. Link sandbox/tool/citation/cleanup evidence without duplicating model transcripts into user history. Preserve legacy rows without invented attribution. Keep execution configuration out of history evidence. The current Cloud Agents runner separately persists workflow definitions in its run store; define retention there independently rather than treating it as conversation configuration storage.

Use one authoritative context assembly, one logical user/final assistant turn, request-id idempotency and one active turn per conversation. Finalize history before terminal success. Ownership applies to list/reopen/continue/cancel/delete and linked run evidence. Production uses PostgreSQL, including the current workflow stores; broader local storage modes are not MVP promises.

Cloud Agents transcripts remain linked execution evidence and are not stored as main chat history by default. An option to include available intermediate execution output or thought-process traces in chat history is a stretch goal, outside MVP acceptance. It must not duplicate context replay or store execution configuration in history.

### D8 Installation and deployment

Ship static and OCI dynamic agentic-chat-backend installation plus agentic-chat-common contracts. Deploy dedicated Boost LCore with pinned SDK/Cloud Agents/OGX dependencies, PostgreSQL and static inference/MCP/artifact/spawner configuration. Cloud Agents is embedded; sandbox workers and its gateway are deployed when the profile requires them. Readiness checks service dependencies and configured runtime connections without rejecting a profile solely for missing model, skill or tool bindings. Disablement preserves history; additive database migrations preserve legacy clients.

## Diagrams

- [Architecture](./diagrams/architecture.md)
- [Single agent workflow](./diagrams/single-agent-workflow.md)
- [Request sequence](./diagrams/chat-sequence.md)

## Risks / Trade-offs

- SDK/Cloud Agents executor mismatch: prove a real one-step invocation using OGX before dependent work.
- Static authoring with runtime credential/network differences: validate from the actual issuing worker.
- Snapshot resend overhead: bounded configuration envelopes and artifact references limit the payload; startup synchronization remains a stretch goal.
- Tool default mismatch: preserve omitted allowedTools as all available configured tools, [] as none, and populated lists as dispatch restrictions.
- History duplication or persistence failure: one context source, idempotent requests and truthful terminal state.
- Sandbox cleanup or policy gaps: fixed profiles, enforced gateway policy and observable reconciliation.

## Remaining implementation decisions

Version/interface, exact API, credential, sandbox policy and history migration choices remain implementation gates described in the [design report](./design-report-1644.md). Architecture and 2.2 scope are selected; startup synchronization and optional intermediate-output history storage remain stretch goals. No application code is implemented by this revision.
