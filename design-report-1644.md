# RHDHPLAN 1644 design report

RHDH 2.2 single agent MVP | Clarified 28 October 2026

## Purpose and scope

RHDHPLAN-1644 delivers the RHDH 2.2 chat execution boundary. RHDH owns static AI Catalog configuration and the native Boost frontend adapter. LCore owns the execution controller, the OGX/OpenAI Agents SDK integration and persistent conversation history. Cloud Agents executes a predefined one-agent workflow under LCore control.

The intended MVP profile executes one agent with an explicit allowed tools list. Omission of allowedTools remains valid and allows all tools available through the agent's configured tool/MCP bindings; an explicit list restricts them and an empty list allows none. The static agent configuration may set spawn to `none`, `local` or `ephemeral`; omission uses the LCore default, currently `ephemeral` (sandboxed). Model bindings, prompts, skills, MCP connections, limits and any sandbox settings are predefined by administrators. Runtime identifiers, including workflow run ID, and the user query vary per turn; executable definitions and capabilities do not.

**Out of scope.** Agent discovery, human-in-the-loop interaction, cross-agent handoffs, multiple execution backends, generated workflow definitions and user/model changes to sandbox settings. A sandbox provisioned from a static profile is in scope; dynamically choosing its image or permissions is not.

[Figure: architecture](./diagrams/architecture.md)

**Controller placement.** LCore hosts the handoff controller as the architectural boundary. In 2.2 it validates and dispatches one agent only. A transfer request fails as unsupported; it never invokes a second agent.

## Architecture and design decisions

**D1 Native RHDH plugin packages.** Create plugins/agentic-chat-backend and plugins/agentic-chat-common. Use native Backstage authentication, permissions, Catalog, config, HTTP routing and lifecycle services. Common contains browser-safe request, event and history types plus shared validation contracts.

**D2 LCore execution ownership.** LCore receives authenticated turn requests at POST /v1/agents/run, validates the supplied run configuration, prepares context, dispatches Cloud Agents, enforces limits, normalizes events and returns the final response. RHDH forwards requests and streams; it does not run an agent loop.

**D3 Retain the AI Catalog agent configuration schema for 2.2.** RHDH AI Catalog agent definitions retain the current OpenAI Agents SDK integration schema. agentic-chat-backend sends the resolved agent snapshot to the LCore controller; LCore owns the SDK integration and workflow translation. Cloud Agents provides workflow execution and configured sandbox lifecycle. OGX is the only inference backend. There must be one SDK model/tool loop for the one agent step, not an SDK loop wrapped around another independent executor loop. No AI Catalog workflow-schema migration is required.

**Integration gate.** Define the Cloud Agents executor interface that invokes the LCore-owned SDK adapter and carries streaming, cancellation, tool policy and context. The current workflow endpoint does not prove this bridge exists. Pin compatible package versions and prove one real execution before treating this as supported.

**D4 LCore chat history.** LCore stores messages and agent/run attribution. Execution snapshots are not saved in conversation history or pinned to conversations. Cloud Agents transcripts are linked execution evidence and are not stored as part of main chat history by default. RHDH stores only necessary content-free identifiers and mappings.

**History stretch goal.** An option can store available intermediate execution output and thought-process traces as part of chat history. This is outside MVP acceptance and must preserve one authoritative context without duplicate transcript replay.

**D5 Static configuration transfer.** RHDH owns static configuration and sends the resolved agent profile with every execution, including continued conversations. LCore constructs the Cloud Agents workflow. Startup synchronization using the same configuration contract is a stretch goal. Dynamic configuration changes, hot reload and change pickup are out of scope.

**Failure boundary.** An omitted spawn or allowedTools setting is valid. By default the agent runs in the LCore default spawn mode (currently `ephemeral`) and may use all available configured tools. Missing model, skill or tool bindings do not cause configuration rejection. If the resolved spawn mode is `ephemeral`, provisioning failure stops that run without silently changing its execution mode.

Accepting missing bindings does not create capabilities or guarantee a successful run. Use administrator-defined defaults where available; if execution needs an unavailable model, skill or tool, report a safe runtime failure without inventing or substituting a binding.

## AI Catalog configuration and synchronization

The Boost configuration example defines agents under ai-catalog.entityProviders.ogx. Agents contain instructions, model, tools, knowledge flags and optionally handoffs. The OGX entity provider maps them to AiResource entities. AI Catalog is the source of configuration, not the execution service.

[Boost AI Catalog configuration example lines 80 to 250](https://github.com/redhat-developer/rhdh-plugins/blob/970c37037eea21f19b4871d4035da7e3ddded907/workspaces/boost/app-config.yaml#L80-L250)

**MVP root.** Select one predefined agent with no outgoing handoffs, for example the HR specialist in the example. The sample router with specialist dependencies is not an MVP execution profile. Reject a selected agent requiring handoffs; do not silently drop its targets or traverse the graph.

**Configuration adapter.** Resolve the selected agent through authorized Catalog access, retaining the current OpenAI Agents SDK integration schema. Preserve root identity, limits, prompt and available model, tools/MCP bindings, skill artifacts and required knowledge bindings. Preserve the difference between an omitted allowedTools field, a populated list and an empty list. Resolve the optional spawn mode from static agent configuration and leave it unset when omitted. Send the resolved profile to LCore for workflow construction without rejecting missing model, skill or tool bindings.

**Configuration gaps.** Current agent entities do not carry provider-level defaultAgent and maxAgentTurns. Supply static profile bindings for root and limits, optional allowedTools, and optional spawn mode/sandbox settings. Skills, credentials and sandbox policy need declared schemas and executable bindings rather than assumed agent entity fields. Exact new field names remain a contract implementation gate.

**Model resolution.** Resolve environment expressions in RHDH. Use the agent model when set, otherwise an explicit approved default where available. Neither RHDH nor LCore configuration validation rejects the agent solely because a model binding is absent. Send concrete bindings when available, not environment syntax or browser-provided credentials. If no usable model is available when inference is attempted, report a safe runtime failure.

**Snapshot per execution.** Send input, conversation identity, request identity and the small resolved configuration snapshot with every run. LCore validates and uses it without saving it or a snapshot reference in conversation history. RHDH supplies the same static configuration on continuation or restart; LCore loads prior messages separately. Configuration refresh and change detection are out of scope. Current permission checks still apply.

**Startup synchronization stretch goal.** As a stretch goal, RHDH publishes predefined profiles to a proposed LCore configuration API at startup. Execution then references a runtime profile independently of conversation history. Unknown or unavailable references fail. The publication API and readiness handshake are additional work; they are not required for the per-run MVP. This does not add dynamic reload.

## How the executor accesses capabilities

**Models.** The snapshot identifies the approved OGX connection and concrete model. LCore resolves credentials from deployment-owned secrets and provides the execution worker with the approved inference route. The selected SDK adapter must use this route; no second model registry is authored in LCore.

**MCP.** For each configured server, resolve a stable binding ID, runtime-reachable URL, supported transport and authentication binding. With allowedTools omitted, expose all tools available from the agent's configured bindings subject to existing authorization. When a list is supplied, expose and dispatch only its permitted names; an empty list exposes none. Do not add unconfigured servers or tools.

**Network and credentials.** The process issuing the MCP request must have network, DNS, TLS and authentication access. If calls originate inside a sandbox worker, test connectivity there; RHDH connectivity is insufficient. Resolve service credentials at the issuing backend or use a scoped credential broker. Do not put secret values in snapshots, history or frontend payloads. Interactive credential collection is out of scope.

**Tool enforcement.** An omitted allowedTools field adds no agent-level filter. A supplied list restricts actual tool dispatch; [] means no tools. Apply existing deployment policy and user authorization in either case. Preserve this distinction through normalization and worker transport; do not turn omission into an empty list or drop an explicit empty list. Permitted calls execute without an approval pause.

**Skills and knowledge.** Represent selected skills by immutable artifact digest/revision and required resource paths. LCore or its worker retrieves them from an approved Git/OCI/artifact service and stages SKILL.md and resources into the execution filesystem. An RHDH-local path or usage command is insufficient. Resolve configured knowledge flags into an approved retrieval binding where available. Missing skill or tool bindings do not cause configuration rejection.

**Static sandbox profile.** Spawn `none` or `local` runs the agent without provisioning a sandbox and requires no spawner readiness. Spawn `ephemeral`, set or defaulted, uses administrator-defined image, spawner, timeouts, resources, mounts, network and cleanup policy. In that case Cloud Agents provisions the environment and cleans it up on success, failure or cancellation; provisioning and cleanup failures retain their defined outcomes.

**SDK and sandbox placement.** The LCore-owned SDK adapter supports execution without a sandbox and execution in the configured sandbox worker. Both paths use one agent loop and the same optional tool filter. Sandboxed workers need the compatible adapter or approved runner binding. Verify both executor paths against the pinned package; resource/network constraints may require gateway deployment settings.

## One agent workflow and execution flow

LCore translates the selected profile into a deterministic one-step Cloud Agents workflow. The user message is the step input. The history adapter supplies prior context. A workflow session grouping ID alone does not supply conversation memory.

[Figure: workflow](./diagrams/single-agent-workflow.md)

**Streaming.** Correlate conversation ID, logical turn/request ID, agent reference and workflow run ID. Emit agent start, tool activity, text deltas and safe error or terminal events. Stream progress during execution and emit success only after history finalization. Do not advertise cross-agent transition events in the MVP.

**Concurrency and cancellation.** Allow one active turn per conversation. Cancellation stops the SDK run, remote tool work where supported and the configured sandbox; suppress late output. Record failed/cancelled outcomes and preserve earlier completed turns.

## Information needed to create a Cloud Agents workflow

The following is the adapter input checklist, not a claim that a complete executable workflow schema already exists. Finalize field names against the pinned Cloud Agents package before publishing an example as deployable configuration.

**Definition identity.** Supported apiVersion and kind, static profile/definition identity, definition revision and exactly one agent step. Preserve the selected root agent reference. Keep per-turn workflow run identity separate from the static definition. Use the supported one-step naming convention agent with output key result after schema verification.

**Agent and inference.** Instructions, selected model, OGX connection/SDK runner binding, supported model settings, input/context mapping and expected output shape. Provider credentials are selected by LCore, not by caller-supplied secret selectors.

**Capabilities.** Configured tools/MCP bindings and schemas, optional allowedTools with omission versus empty-list semantics preserved, authentication bindings, skill artifact digests/resource paths and any approved retrieval configuration.

**Bounds and spawn.** One agent invocation per turn with model/tool/token/time limits. Spawn is optional; omission uses the LCore default, currently `ephemeral`. Image/spawner/resource/network/mount/cleanup settings apply only to `ephemeral`. Spawn `none` or `local` requires no provisioning.

**Invocation envelope.** User input, authenticated owner, conversation ID, request/turn ID, workflow run ID, saved context and trace correlation. These are runtime values and must not alter the static executable definition. Distinguish the workflow run ID from both static definition identity and chat conversation ID; exact wire field names remain an integration gate.

**Result contract.** Final answer mapping, normalized event stream, tool summaries, optional citations/artifact references, terminal outcome, cancellation signal and linkage to persistent execution evidence.

**Agent execution controller API.** Add a new POST /v1/agents/run endpoint with dedicated request, response and event models. Accept user input, request identity, optional conversation identity, the resolved static agent configuration, and streaming preference. The controller validates the request, loads saved context, invokes the embedded Cloud Agents runner and finalizes history before the terminal result. Define its own payload bound and consume configuration before OGX inference serialization. Keep skill bytes outside the request.

**Required implementation proof.** Prove a spawn `none` run without an allowedTools field succeeds through the SDK/OGX adapter. Also prove explicit tool filtering, [] denying all tools, optional sandbox provisioning/cleanup, two-turn context reuse and history linkage to the run. Confirm executor extension points before finalizing workflow JSON.

## Conversation history data model

The current LCore metadata schema has user_conversation and user_turn tables. Conversation rows contain ownership, last provider/model, timestamps, response identity and summary. Turn rows contain ordering, timestamps, provider/model and response ID. They do not yet contain the required agent or workflow attribution.

**Conversation metadata migration.** Add nullable agent_ref, agent_name_at_creation, execution_backend fixed to ogx for this feature and execution_contract_version. Keep current ownership and model/provider fields. Nullable defaults preserve existing conversations; do not invent historical agent attribution.

**Configuration stays outside history.** Do not add an execution_snapshot table, snapshot column or snapshot foreign key to conversation history. Keep prompts, model/tool bindings, skill configuration and sandbox settings outside conversation and turn records. RHDH supplies configuration per run, with startup publication retained as a stretch goal.

**Turn metadata migration.** Add a unique request/turn ID, agent_ref, workflow_run_id, terminal status, safe error code, finish reason and optional sandbox_execution_id. Reuse current actual model/provider fields. Support pending/running turns by making completed_at nullable until terminal completion or by introducing a separate active-run record; finalize the migration choice before implementation.

**Execution evidence.** Retain ordered tool summaries, citations/artifact references and sandbox cleanup outcome in a linked execution record. Use a bounded metadata payload for summaries; keep detailed Cloud Agents transcripts under retention policy without copying the execution snapshot into history evidence. The current runner separately persists workflow definitions in its run store; define that store's retention boundary independently from conversation history. A chat turn, workflow run and individual model response are distinct identifiers.

**Consistency.** Create one logical user turn and one final assistant answer in the authoritative LCore conversation. Feed prior context once. Persist the run linkage and terminal outcome before reporting success. A disconnect after model completion must not create a duplicate run on a repeated request; enforce request-id idempotency and a conversation-level active-turn guard.

**Failure and retention.** Save explicit failed/cancelled outcomes without replacing prior successful turns. List, reopen, continue and delete remain owner-scoped. Delete conversation-owned content and links under retention policy; preserve shared artifacts needed elsewhere. A storage failure must not be reported as a successfully saved answer.

## Installation deployment and delivery plan

**RHDH installation.** Provide static backend registration and an OCI dynamic plugin package for agentic-chat-backend. Publish agentic-chat-common types. Configure LCore service URL, service authentication, static profile selection and configuration transfer mode. Register native permissions and wire the frontend chat/history client to the new routes.

**LCore deployment.** Deploy a dedicated Boost LCore service with pinned OGX/OpenAI Agents SDK and Cloud Agents integration versions. Configure PostgreSQL, workflow state/transcript stores, trusted RHDH identity, approved model/MCP/artifact bindings, and the static spawner profile. A separate Cloud Agents API service is not required by this design.

**Sandbox deployment.** The inspected harness supports the OpenShell spawner adapter. Deploy its gateway and approved worker image when ephemeral execution is configured. Supply gateway identity/TLS, fixed workspace and capacity, image pull access, network policy and cleanup monitoring. Do not expose sandbox/image overrides to chat users.

**Startup order and health.** Apply migrations; start storage, OGX and required MCP/artifact services; start LCore; accept the static RHDH profile without rejecting missing model, skill or tool bindings. Start and check the sandbox gateway only for agents whose spawn mode is or defaults to `ephemeral`. An agent with spawn `none` or `local` does not depend on spawner availability. Startup configuration publication remains a stretch goal.

**RHDH work.** Build the native backend/common packages, frontend-facing adapter, static configuration resolver and model/MCP/skill/sandbox translation. Forward snapshot and turn requests, normalize streams, expose owner-scoped history and propagate cancellation.

**LCore work.** Implement POST /v1/agents/run and its typed execution contracts, one-agent controller, SDK/Cloud Agents bridge, dispatch allowlist, context adapter, correlated events, request idempotency and history migrations. Keep cross-agent transfers rejected in 2.2.

**Cloud Agents work.** Verify the one-step schema and executor extension; carry capability/context bindings into the selected worker, apply static spawn policy, expose progress/cancellation and return answer plus run/cleanup evidence.

**Acceptance gate.** Prove the intended single-agent profile with an explicit allowed tools list, spawn `none` execution without allowedTools, omitted spawn resolving to the LCore default, and [] denying all tools. Verify missing model, skill and tool bindings pass configuration validation and unavailable capabilities report safe runtime failures when needed. Also prove optional sandbox lifecycle with no fallback on provisioning failure, one-agent enforcement, two-user isolation, continuation/restart with per-run configuration, no snapshot retention or transcript copying in main history, cancellation, truthful completion, and static/dynamic plugin installation. Startup synchronization and intermediate-output history storage are stretch goals, not MVP acceptance requirements.

**Remaining decisions.** Pin versions and SDK executor interface; confirm exact snapshot/events schema and payload limit; confirm credential delivery and sandbox network/resource enforcement; finalize the history migration. These are implementation gates, not added discovery or approval features.
