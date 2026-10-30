# LCore and Cloud Agents

This workstream owns the execution controller, workflow dispatch, Cloud Agents integration and authoritative chat history, including history storage implementation and migrations. Its RHDH caller and integrated runtime configuration for persistence, credentials, RBAC and optional sandbox infrastructure are specified in [boost-chat-rhdh-backend](../boost-chat-rhdh-backend/spec.md); runtime packaging, installation and consolidated documentation are specified in [boost-chat-deployment](../boost-chat-deployment/spec.md).

## ADDED Requirements

### Requirement: Authenticated single-agent controller
LCore SHALL expose POST /v1/agents/run with dedicated versioned request, response and stream models. It SHALL validate the trusted RHDH caller and authenticated owner, request identity, optional conversation identity, supplied static agent snapshot and streaming preference. It SHALL enforce a bounded request size, reject unsupported handoffs and caller overrides, prepare context, enforce limits and return correlated normalized outcomes. Missing model, skill or tool bindings alone SHALL NOT reject configuration. Missing usable capabilities needed during execution SHALL produce safe runtime failures without discovery or invented bindings.

#### Scenario: Authorized controller request

- **WHEN** a trusted RHDH backend submits a valid single-agent turn
- **THEN** LCore accepts the static configuration, prepares the owner's context and controls execution through completion

#### Scenario: Missing bindings

- **WHEN** the snapshot omits model, skill or tool bindings
- **THEN** configuration validation accepts the snapshot and any unavailable capability needed during execution produces a safe runtime failure

#### Scenario: Untrusted or oversized request

- **WHEN** caller provenance or owner identity is invalid or the request exceeds its bound
- **THEN** LCore rejects it before workflow creation without persisting protected input or truncating the snapshot

### Requirement: Workflow creation through the workflow API
The controller SHALL translate the agent snapshot into exactly one schema-validated Cloud Agents agent step and dispatch workflow creation through LCore POST /v1/workflows/run. This is the concrete harness route for the requested /v1/workflow integration. A shared authenticated workflow submission service behind that route SHALL preserve the same validation, authorization and credential rules; bypassing them with a separate direct runner path SHALL NOT be allowed. Workflow run identity and query/context input SHALL remain distinct from the static executable definition. The controller SHALL retain run linkage and bridge workflow status, events and cancellation into the chat contract.

#### Scenario: Controller starts a workflow run

- **WHEN** a valid single-agent turn reaches the controller
- **THEN** one workflow is submitted through the workflow creation contract and its run ID is linked to the logical chat turn

#### Scenario: Workflow submission fails

- **WHEN** the workflow API rejects the translated definition or cannot create the run
- **THEN** the controller reports a correlated failure without starting a second execution path or reporting a completed answer

#### Scenario: Query changes between turns

- **WHEN** two turns use the same administrator configuration with different queries and run IDs
- **THEN** their executable definition, agent capabilities and sandbox policy remain unchanged

### Requirement: Proven LCore-owned SDK executor bridge
Cloud Agents SHALL provide or adapt an executor interface that invokes the LCore-owned OpenAI Agents SDK integration through the configured OGX route. The interface SHALL carry context, tool policy, model settings, skill resources, streaming, cancellation and terminal output across both sandboxed and unsandboxed execution. Exactly one agent/model-tool loop SHALL run per turn. Compatible LCore, Cloud Agents, SDK and OGX versions SHALL be pinned, and one real execution SHALL be demonstrated before support is claimed. Workflow endpoint availability or an existing alternate executor SHALL NOT establish SDK compatibility.

#### Scenario: Real SDK integration proof

- **WHEN** the pinned integration executes a predefined agent with an explicit allowed tools list
- **THEN** evidence shows one Cloud Agents step using the LCore-owned SDK adapter and configured inference route with correlated output and no nested agent loops

#### Scenario: Agent transfer requested

- **WHEN** a definition or running agent requests a cross-agent transfer
- **THEN** the request fails as unsupported and no second agent is invoked

### Requirement: Enforced static tool and capability policy
The executor SHALL enforce allowedTools at both tool exposure and dispatch. Omission SHALL allow available tools from the agent's configured tool/MCP bindings subject to deployment policy and current user permissions; an empty list SHALL allow none; a populated list SHALL restrict tools to those names. Missing tool bindings SHALL NOT create capabilities. Skills SHALL NOT bypass the filter. Configured immutable skill resources SHALL be materialized in the actual execution filesystem, and knowledge access SHALL use concrete approved retrieval bindings rather than RHDH-local paths or boolean flags alone. Interactive authorization and approval pauses SHALL remain unsupported.

#### Scenario: Omitted filter

- **WHEN** allowedTools is omitted and an available configured tool is authorized
- **THEN** the tool can execute without an additional agent-level filter

#### Scenario: Empty or restrictive filter

- **WHEN** allowedTools is empty or excludes a requested tool
- **THEN** the excluded call cannot execute, including when requested by a skill or marked approved by a caller

#### Scenario: Authorization revoked

- **WHEN** a tool is listed but its permission has been revoked before dispatch
- **THEN** the call is denied

#### Scenario: Runtime capability access

- **WHEN** the worker uses configured skills, MCP or retrieval services
- **THEN** resource materialization, connectivity, TLS and scoped authentication are verified from that worker, and missing resources or execution failures are reported safely

### Requirement: Explicit useSandbox translation and lifecycle
LCore SHALL consume the resolved per-agent useSandbox boolean and default omission to false. False SHALL explicitly select unsandboxed workflow execution without depending on a sandbox image, spawner or gateway. True SHALL select the administrator-predefined sandbox profile; Cloud Agents SHALL provision it before the agent executes and attempt cleanup after success, failure or cancellation, including partial provisioning. User/model input SHALL NOT change the image, resources, mounts, network or permissions. Provisioning failure SHALL stop the run without unsandboxed fallback. Cleanup failure SHALL record environment identity and a bounded retry or reconciliation obligation.

#### Scenario: Default false despite executor defaults

- **WHEN** useSandbox is false or omitted while the Cloud Agents package defaults an omitted spawn mode to ephemeral
- **THEN** translation explicitly selects the supported unsandboxed mode and no sandbox is provisioned

#### Scenario: Sandbox required

- **WHEN** useSandbox is true
- **THEN** the approved sandbox is provisioned before execution and cleanup is attempted on every terminal outcome

#### Scenario: Provisioning or cleanup failure

- **WHEN** sandbox provisioning fails or cleanup leaves resources behind
- **THEN** the run never switches execution mode and any remaining resources and cleanup obligation are recorded

### Requirement: Authoritative conversation context and history
LCore SHALL own persistent messages, context and agent/run attribution and expose owner-scoped session list and per-session history APIs compatible with the RHDH proxy. Each successful turn SHALL store one logical user message and final assistant answer. Context SHALL be assembled once from authoritative history. Cloud Agents transcripts SHALL remain linked execution evidence outside main chat history by default and SHALL NOT be replayed as duplicate context. Snapshots, serialized definitions, snapshot references and credentials SHALL NOT be saved in conversation/turn records or history evidence. Separate workflow definition/transcript retention SHALL remain independent of chat history.

#### Scenario: Two-turn recall after restart

- **WHEN** the owner reopens a completed session after service restart and RHDH supplies the static configuration again
- **THEN** the next run receives prior messages once without reconstructing configuration from history

#### Scenario: Transcript and final answer

- **WHEN** Cloud Agents produces internal transcripts and a final answer
- **THEN** main history stores the logical chat exchange and links execution evidence without copying internal transcripts by default

#### Scenario: List or retrieve history

- **WHEN** RHDH requests the authenticated owner's sessions or one session's history
- **THEN** LCore returns only owner-authorized persisted data and available agent/run attribution

### Requirement: Additive conversation and turn migrations
Conversation metadata SHALL support nullable agent identity/name, execution backend and contract version while preserving legacy rows. Turn metadata SHALL correlate request, agent, workflow run, actual provider/model when available, terminal status and safe error/finish reason, plus sandbox/cleanup evidence where applicable. Pending turns SHALL have a valid representation before completion. Migrations SHALL account for current non-null completion/model/provider fields without inventing values for unavailable bindings or historical attribution.

#### Scenario: Legacy history migration

- **WHEN** migrations apply to conversations without agent/run attribution
- **THEN** existing history remains readable without fabricated attribution

#### Scenario: Pending turn or missing model

- **WHEN** a turn is accepted before completion or fails before a model/provider can be resolved
- **THEN** its active and terminal states can be persisted consistently without fictitious completion times or model/provider values

### Requirement: Bounded idempotent execution and truthful completion
The controller SHALL enforce one active turn per conversation and separate agent invocation, model/tool iteration, token and elapsed-time bounds. Repeated request IDs SHALL NOT create duplicate workflow runs, including retries after disconnect. Owner cancellation SHALL stop applicable SDK/tool/workflow work, perform required sandbox cleanup and suppress late output. Terminal success SHALL follow authoritative history finalization. Failed/cancelled turns SHALL preserve earlier completed history and retain their terminal attribution.

#### Scenario: Retry or competing turn

- **WHEN** the same request ID is retried or another request arrives while a conversation has an active turn
- **THEN** LCore reports the existing run/outcome or rejects the competing turn without starting a duplicate execution

#### Scenario: Runtime limit or cancellation

- **WHEN** a limit is reached or the owner cancels the active run
- **THEN** the runtime stops applicable work, records the correlated outcome, cleans up provisioned resources and emits no later answer deltas

#### Scenario: History persistence fails

- **WHEN** inference completes but final history storage fails
- **THEN** the terminal result reports persistence failure rather than claiming a successfully saved answer

### Requirement: Owner and credential boundaries
LCore SHALL validate trusted service identity and forwarded user identity independently of browser fields. Ownership SHALL apply to continuation, cancellation, list/retrieve/delete and linked execution evidence. Deletion SHALL remove conversation-owned content and links according to retention policy, preserve shared runtime configuration/artifacts and surface dependency failures. Provider/MCP credentials SHALL be resolved from approved secret bindings at the issuing component or a verified transient channel and SHALL NOT enter persisted workflow definitions/providers, transcripts, history, routine logs or error payloads. Secret selectors and destinations SHALL NOT be caller-controlled.

#### Scenario: Cross-user access or forged credential selector

- **WHEN** a caller requests another owner's history/evidence or attempts to select an unapproved credential
- **THEN** the operation is denied without disclosing protected data

#### Scenario: Delete with shared resources

- **WHEN** the owner deletes a conversation
- **THEN** conversation-owned content and links are removed under retention policy while shared definitions/artifacts are preserved and dependency failures are reported

#### Scenario: Secret-bearing workflow input

- **WHEN** execution needs a provider or MCP token
- **THEN** the token reaches only its approved runtime consumer and is absent from the workflow input that is persisted and from user-visible outputs

## Investigation inputs

- [Harness workflow endpoint](https://github.com/jameswnl/lightspeed-stack/blob/f2d5911092d4591c2c6766b38c7c38a8aaa56915/src/app/endpoints/workflows.py) and [request model](https://github.com/jameswnl/lightspeed-stack/blob/f2d5911092d4591c2c6766b38c7c38a8aaa56915/src/models/api/requests/agents.py): the observed creation path is /v1/workflows/run. Confirm shared dispatch, workflow streaming/status/cancel contracts and authorization without assuming a singular /v1/workflow endpoint exists.
- [Cloud Agents executor dispatch](https://github.com/jameswnl/lightspeed-cloud-agents/blob/1788b17d84420ca07c8df423a460bcae5b369f66/src/cloud_agents/workflow/executor/step/dispatch.py) currently defaults spawn to ephemeral; spawn none selects the direct executor. [The direct executor](https://github.com/jameswnl/lightspeed-cloud-agents/blob/1788b17d84420ca07c8df423a460bcae5b369f66/src/cloud_agents/workflow/executor/step/direct.py) uses Pydantic AI. Prove the required SDK adapter and both sandbox paths rather than treating this as delivered support.
- [Conversation metadata](https://github.com/jameswnl/lightspeed-stack/blob/f2d5911092d4591c2c6766b38c7c38a8aaa56915/src/models/database/conversations.py): completed_at, provider and model are currently non-null on turns. Finalize additive migrations, active-turn representation, owner-scoped API mapping and one authoritative context source.
- Verify transient credential delivery against workflow persistence before enabling forwarding. Startup configuration synchronization and optional intermediate-output/thought-process history storage remain stretch goals outside MVP acceptance.
