# RHDHPLAN 1644 design report

RHDH 2.2 single agent MVP | 20 October 2026

## Purpose and scope

RHDHPLAN-1644 delivers the RHDH 2.2 chat execution boundary. RHDH owns static AI Catalog configuration and the native Boost frontend adapter. LCore owns the execution controller, the OGX/OpenAI Agents SDK integration and persistent conversation history. Cloud Agents executes a predefined one-agent workflow under LCore control.

The MVP executes one agent with an explicit allowed tools list. Model bindings, prompts, skills, MCP connections, execution limits and sandbox spawn settings are predefined by administrators. Runtime identifiers and the user query vary per turn; executable definitions and capabilities do not.

**Out of scope.** Agent discovery, human-in-the-loop interaction, cross-agent handoffs, multiple execution backends, generated workflow definitions and user/model changes to sandbox settings. A sandbox provisioned from a static profile is in scope; dynamically choosing its image or permissions is not.

[Figure: architecture](./diagrams/architecture.md)

**Controller placement.** LCore hosts the handoff controller as the architectural boundary. In 2.2 it validates and dispatches one agent only. A transfer request fails as unsupported; it never invokes a second agent.

## Architecture and design decisions

**D1 Native Boost packages.** Create plugins/boost-chat-backend and plugins/boost-chat-common. Use native Backstage authentication, permissions, Catalog, config, HTTP routing and lifecycle services. Common contains browser-safe request, event and history types plus shared validation contracts.

**D2 LCore execution ownership.** LCore receives authenticated turn requests at POST /v1/agents/run, validates the supplied run configuration, prepares context, dispatches Cloud Agents, enforces limits, normalizes events and returns the final response. RHDH forwards requests and streams; it does not run an agent loop.

**D3 One backend for 2.2.** OGX with the OpenAI Agents SDK integration on the LCore side is the only execution backend. Cloud Agents provides workflow execution and configured sandbox lifecycle. There must be one SDK model/tool loop for the one agent step, not an SDK loop wrapped around another independent executor loop.

**Integration gate.** Define the Cloud Agents executor interface that invokes the LCore-owned SDK adapter and carries streaming, cancellation, tool policy and context. The current workflow endpoint does not prove this bridge exists. Pin compatible package versions and prove one real execution before treating this as supported.

**D4 LCore history.** LCore stores messages and agent/run attribution. Execution snapshots are not saved in conversation history or pinned to conversations. Cloud Agents transcripts are linked execution evidence, not a second independently replayed chat history. RHDH stores only necessary content-free identifiers and mappings.

**D5 Static configuration transfer.** RHDH remains the authoring authority. The recommended MVP sends a resolved snapshot on each execution. Startup synchronization is an alternative deployment mode using the same immutable snapshot contract; implementing both is not an MVP requirement.

**Failure boundary.** Reject incompatible configuration before running the model. Missing required tools, skills, model bindings or a configured sandbox are errors. No silent capability removal, extra backend or unsandboxed fallback is selected.

## AI Catalog configuration and synchronization

The Boost configuration example defines agents under ai-catalog.entityProviders.ogx. Agents contain instructions, model, tools, knowledge flags and optionally handoffs. The OGX entity provider maps them to AiResource entities. AI Catalog is the source of configuration, not the execution service.

[Boost AI Catalog configuration example lines 80 to 250](https://github.com/redhat-developer/rhdh-plugins/blob/970c37037eea21f19b4871d4035da7e3ddded907/workspaces/boost/app-config.yaml#L80-L250)

**MVP root.** Select one predefined agent with no outgoing handoffs, for example the HR specialist in the example. The sample router with specialist dependencies is not an MVP execution profile. Reject a selected agent requiring handoffs; do not silently drop its targets or traverse the graph.

**Configuration adapter.** Resolve the selected agent through authorized Catalog access. Preserve its explicit root identity and execution limits. Resolve its prompt, concrete model, exact allowed tool names, MCP bindings, pinned skill artifacts and required knowledge bindings. No agent search or runtime enrollment is required.

**Configuration gaps.** Current agent entities do not carry provider-level defaultAgent and maxAgentTurns. Supply an explicit static execution profile for the selected root, limits and sandbox settings. Skills, credentials and sandbox policy also need deployment-owned bindings rather than assumed agent entity fields. New configuration keys need declared schemas and documentation.

**Model resolution.** Resolve environment expressions in RHDH. Use the agent model when set, otherwise an explicit approved default. Reject an unresolved model. Send the concrete model and approved OGX connection binding, not environment syntax or browser-provided credentials.

**Snapshot per execution.** Send input, conversation identity, request identity and the small configuration snapshot with each run. LCore validates and uses it for that execution without saving the snapshot or a snapshot reference in conversation history. RHDH supplies configuration again on continuation or after restart; LCore loads prior messages separately. No conversation-pinned revision or automatic new-conversation rule is required. Current permission checks still apply.

**Startup synchronization alternative.** RHDH validates and publishes predefined profiles to a proposed LCore configuration API before reporting ready. Execution references profile ID and revision in runtime configuration, independently of conversation history. Unknown or unavailable revisions fail. RHDH owns configuration compatibility; revisions are not retained because a conversation pins them. This publication API is additional work and is not available today.

## How the executor accesses capabilities

**Models.** The snapshot identifies the approved OGX connection and concrete model. LCore resolves credentials from deployment-owned secrets and provides the execution worker with the approved inference route. The selected SDK adapter must use this route; no second model registry is authored in LCore.

**MCP.** For each predefined server, resolve a stable binding ID, runtime-reachable URL, supported transport, exact allowed tool names and authentication binding. LCore validates these against deployment policy and exposes only those tools to the SDK. Validate tool names and schemas against the configured contract without automatically adding other tools.

**Network and credentials.** The process issuing the MCP request must have network, DNS, TLS and authentication access. If calls originate inside a sandbox worker, test connectivity there; RHDH connectivity is insufficient. Resolve service credentials at the issuing backend or use a scoped credential broker. Do not put secret values in snapshots, history or frontend payloads. Interactive credential collection is out of scope.

**Tool enforcement.** Enforce the intersection of the static agent allowlist, administrator policy and current user permissions at actual tool dispatch. Prompt instructions and UI filtering are insufficient. Skills, globally installed tools and backend defaults cannot add privileges. Allowed calls execute without an approval pause; disallowed or approval-dependent calls fail.

**Skills and knowledge.** Represent selected skills by immutable artifact digest/revision and required resource paths. LCore or its worker retrieves them from an approved Git/OCI/artifact service and stages SKILL.md and resources into the execution filesystem. An RHDH-local path or usage command is insufficient. Resolve knowledge flags into a concrete approved retrieval binding; unavailable required material fails before execution.

**Static sandbox profile.** Administrators fix spawn mode, image digest, spawner binding, timeouts, resource limits, workspace/mount policy, network destinations and cleanup policy. LCore checks the supplied profile against deployed policy. Cloud Agents provisions the configured ephemeral environment for the agent step and cleans it up on success, failure or cancellation. Cleanup failures are recorded for bounded retry or operator reconciliation.

**SDK and sandbox placement.** The SDK adapter belongs to the LCore integration. For sandboxed execution, the workflow worker must contain the compatible adapter or receive an approved runner binding. Prove that this path still uses exactly one agent loop and the same tool policy. Resource/network constraints may be enforced by the gateway deployment rather than existing workflow fields.

## One agent workflow and execution flow

LCore translates the selected profile into a deterministic one-step Cloud Agents workflow. The user message is the step input. The history adapter supplies prior context. A workflow session grouping ID alone does not supply conversation memory.

[Figure: workflow](./diagrams/single-agent-workflow.md)

**Streaming.** Correlate conversation ID, logical turn/request ID, agent reference and workflow run ID. Emit agent start, tool activity, text deltas and safe error or terminal events. Stream progress during execution and emit success only after history finalization. Do not advertise cross-agent transition events in the MVP.

**Concurrency and cancellation.** Allow one active turn per conversation. Cancellation stops the SDK run, remote tool work where supported and the configured sandbox; suppress late output. Record failed/cancelled outcomes and preserve earlier completed turns.

## Information needed to create a Cloud Agents workflow

The following is the adapter input checklist, not a claim that a complete executable workflow schema already exists. Finalize field names against the pinned Cloud Agents package before publishing an example as deployable configuration.

**Definition identity.** Supported apiVersion and kind, deterministic workflow/profile ID, definition revision and exactly one agent step. Preserve the selected root agent reference. Use the supported one-step naming convention agent with output key result after schema verification.

**Agent and inference.** Instructions, selected model, OGX connection/SDK runner binding, supported model settings, input/context mapping and expected output shape. Provider credentials are selected by LCore, not by caller-supplied secret selectors.

**Capabilities.** Exact tool allowlist, static tool schemas or validated schema revisions, MCP endpoints/transports and auth bindings, skill artifact digests/resource paths, and any approved retrieval configuration.

**Bounds and spawn.** Agent invocations per turn fixed to one; explicit limits for SDK iterations, tool calls, tokens and wall-clock time. Static spawn mode, sandbox image digest and spawner profile, plus enforced resource/network/mount/cleanup policy. A sandbox image alone does not define isolation.

**Invocation envelope.** User input, authenticated owner, conversation ID, request/turn ID, saved context and trace correlation. These are runtime values and must not alter the static executable definition. Distinguish the workflow run ID from the chat conversation ID.

**Result contract.** Final answer mapping, normalized event stream, tool summaries, optional citations/artifact references, terminal outcome, cancellation signal and linkage to persistent execution evidence.

**Agent execution controller API.** Add a new POST /v1/agents/run endpoint with dedicated request, response and event models. Accept user input, request identity, optional conversation identity, the static snapshot or published revision, and streaming preference. The controller validates the request, loads saved context, invokes the embedded Cloud Agents runner and finalizes history before the terminal result. Define its own payload bound and consume configuration before OGX inference serialization. Keep skill bytes outside the request.

**Required implementation proof.** A real one-step run must show the selected SDK adapter calling OGX, an allowed MCP tool succeeding, a non-allowed tool failing, the configured sandbox lifecycle, two-turn context reuse and a history record that links the same run. Confirm the executor extension point before finalizing workflow JSON.

## Conversation history data model

The current LCore metadata schema has user_conversation and user_turn tables. Conversation rows contain ownership, last provider/model, timestamps, response identity and summary. Turn rows contain ordering, timestamps, provider/model and response ID. They do not yet contain the required agent or workflow attribution.

**Conversation metadata migration.** Add nullable agent_ref, agent_name_at_creation, execution_backend fixed to ogx for this feature and execution_contract_version. Keep current ownership and model/provider fields. Nullable defaults preserve existing conversations; do not invent historical agent attribution.

**Configuration stays outside history.** Do not add an execution_snapshot table, snapshot column or snapshot foreign key to conversation history. Keep prompts, model/tool bindings, skill configuration and sandbox settings outside conversation and turn records. RHDH supplies configuration per run, or LCore resolves it from the independent startup configuration mode.

**Turn metadata migration.** Add a unique request/turn ID, agent_ref, workflow_run_id, terminal status, safe error code, finish reason and optional sandbox_execution_id. Reuse current actual model/provider fields. Support pending/running turns by making completed_at nullable until terminal completion or by introducing a separate active-run record; finalize the migration choice before implementation.

**Execution evidence.** Retain ordered tool summaries, citations/artifact references and sandbox cleanup outcome in a linked execution record. Use a bounded metadata payload for summaries; keep detailed Cloud Agents transcripts under retention policy without copying the execution snapshot into history evidence. The current runner separately persists workflow definitions in its run store; define that store's retention boundary independently from conversation history. A chat turn, workflow run and individual model response are distinct identifiers.

**Consistency.** Create one logical user turn and one final assistant answer in the authoritative LCore conversation. Feed prior context once. Persist the run linkage and terminal outcome before reporting success. A disconnect after model completion must not create a duplicate run on a repeated request; enforce request-id idempotency and a conversation-level active-turn guard.

**Failure and retention.** Save explicit failed/cancelled outcomes without replacing prior successful turns. List, reopen, continue and delete remain owner-scoped. Delete conversation-owned content and links under retention policy; preserve shared artifacts needed elsewhere. A storage failure must not be reported as a successfully saved answer.

## Installation deployment and delivery plan

**RHDH installation.** Provide static backend registration and an OCI dynamic plugin package for boost-chat-backend. Publish boost-chat-common types. Configure LCore service URL, service authentication, static profile selection and configuration transfer mode. Register native permissions and wire the frontend chat/history client to the new routes.

**LCore deployment.** Deploy a dedicated Boost LCore service with pinned OGX/OpenAI Agents SDK and Cloud Agents integration versions. Configure PostgreSQL, workflow state/transcript stores, trusted RHDH identity, approved model/MCP/artifact bindings, and the static spawner profile. A separate Cloud Agents API service is not required by this design.

**Sandbox deployment.** The inspected harness supports the OpenShell spawner adapter. Deploy its gateway and approved worker image when ephemeral execution is configured. Supply gateway identity/TLS, fixed workspace and capacity, image pull access, network policy and cleanup monitoring. Do not expose sandbox/image overrides to chat users.

**Startup order and health.** Apply database migrations; start storage, OGX, MCP/artifact dependencies and the sandbox gateway; start LCore; validate or publish the static profile from RHDH; enable chat only when required bindings are usable. Runtime-specific dependencies must be tested from their issuing process. Missing mandatory dependencies fail readiness for the affected profile.

**RHDH work.** Build the native backend/common packages, frontend-facing adapter, static configuration resolver and model/MCP/skill/sandbox translation. Forward snapshot and turn requests, normalize streams, expose owner-scoped history and propagate cancellation.

**LCore work.** Implement POST /v1/agents/run and its typed execution contracts, one-agent controller, SDK/Cloud Agents bridge, dispatch allowlist, context adapter, correlated events, request idempotency and history migrations. Keep cross-agent transfers rejected in 2.2.

**Cloud Agents work.** Verify the one-step schema and executor extension; carry capability/context bindings into the selected worker, apply static spawn policy, expose progress/cancellation and return answer plus run/cleanup evidence.

**Acceptance gate.** Prove one configured agent, allowed and denied tools, no second agent, fixed sandbox settings, startup/snapshot compatibility for the selected mode, two-user isolation, restart continuation with RHDH-supplied configuration, no snapshot retention in history, cancellation and truthful history completion. Prove clean static and dynamic plugin installation.

**Remaining decisions.** Pin versions and SDK executor interface; confirm exact snapshot/events schema and payload limit; confirm credential delivery and sandbox network/resource enforcement; finalize the history migration. These are implementation gates, not added discovery or approval features.
