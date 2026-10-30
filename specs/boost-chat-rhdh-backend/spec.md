# RHDH backend

This workstream owns the native backend/common packages, AI Catalog adaptation, frontend-facing execution and history routes, RHDH authorization, and the integrated agentic-chat configuration for LCore and Cloud Agents, including persistence, secure credential delivery, RBAC and optional sandbox infrastructure. LCore execution and history implementation are specified in [boost-chat-lcore-cloud-agents](../boost-chat-lcore-cloud-agents/spec.md); packaging, installation and consolidated documentation consume this configuration as specified in [boost-chat-deployment](../boost-chat-deployment/spec.md).

## ADDED Requirements

### Requirement: Native agentic-chat packages
RHDH SHALL provide plugins/agentic-chat-backend and browser-safe plugins/agentic-chat-common. The backend SHALL use native Backstage authentication, permissions, Catalog, config, HTTP routing and lifecycle services. Common SHALL publish request, event, history and shared validation contracts. RHDH SHALL adapt the native Boost frontend and proxy execution to LCore without running an agent loop or storing chat content in its plugin database.

#### Scenario: Native backend registration

- **WHEN** agentic-chat-backend is registered in RHDH
- **THEN** the frontend can use its typed execution and history routes without loading an execution SDK in the browser

### Requirement: Optional per-agent sandbox flag
The AI Catalog agent entity TypeScript contract and JSON schema SHALL add optional spec.useSandbox of boolean type while retaining the current OpenAI Agents SDK integration fields. The authoring/provider path SHALL preserve the field through Catalog ingestion. Omission SHALL mean false. True SHALL require execution using an administrator-predefined sandbox profile; false SHALL request execution without sandbox provisioning. The flag SHALL NOT permit callers to select sandbox images or permissions.

#### Scenario: Existing agent without sandbox flag

- **WHEN** an existing agent entity omits useSandbox
- **THEN** it remains valid and its execution snapshot resolves useSandbox to false

#### Scenario: Explicit sandbox choice

- **WHEN** an administrator configures useSandbox as true or false for an agent
- **THEN** Catalog ingestion and backend resolution preserve that boolean in the execution snapshot

#### Scenario: Invalid flag type

- **WHEN** useSandbox is a string or another non-boolean value
- **THEN** entity validation rejects that value instead of coercing it

### Requirement: Authorized static entity resolution
The run router SHALL read one predefined agent from the RHDH Catalog using authorized backend access. It SHALL extract the selected identity, instructions, available model/model settings, configured tools/MCP bindings, skills, knowledge, limits, output mapping, optional allowedTools and useSandbox needed by LCore and Cloud Agents. It SHALL preserve existing agent schema semantics and resolve administrator defaults and runtime-reachable references where available. It SHALL reject handoffs, discovery and caller-authored capability overrides. Missing model, skill or tool bindings alone SHALL NOT cause configuration rejection or creation of replacement bindings.

#### Scenario: Agent selected by entity reference

- **WHEN** an authorized user submits a query for a predefined agent reference
- **THEN** the backend reads that entity and constructs execution configuration from trusted Catalog and administrator configuration rather than browser-supplied definitions

#### Scenario: Missing optional capability bindings

- **WHEN** the entity has no model, skill or tool binding
- **THEN** the backend accepts the configuration without inventing capabilities and forwards the available configuration to LCore

#### Scenario: Unauthorized or multi-agent selection

- **WHEN** the user cannot access the selected entity or the selected agent requires a cross-agent handoff
- **THEN** the backend rejects the request without dispatching an execution

### Requirement: Agent execution router
The backend SHALL expose POST /v1/agents/run relative to its configured plugin base path and forward authorized turns to LCore POST /v1/agents/run. Each turn SHALL carry the query, selected agent snapshot, request identity, optional conversation identity and streaming preference. Authenticated owner identity SHALL come from trusted Backstage credentials. Static configuration SHALL be supplied on initial, continued and restarted sessions independently of saved chat history. Runtime identifiers and queries SHALL NOT alter administrator-predefined executable definitions or capabilities. The contract SHALL define payload bounds and safe errors without truncating configuration or including artifact bytes and secrets in the snapshot.

#### Scenario: Start or continue a chat

- **WHEN** the frontend sends an authorized query with a new request ID and an optional existing conversation ID
- **THEN** the router forwards the resolved snapshot and trusted turn identity to LCore and returns its correlated response

#### Scenario: Browser attempts to override execution

- **WHEN** the request includes another owner, tool policy, model binding or sandbox image in place of trusted configuration
- **THEN** those fields cannot establish authority or widen the dispatched capabilities

#### Scenario: Oversized execution input

- **WHEN** the snapshot exceeds the negotiated request bound
- **THEN** the backend returns a structured size error without truncation or starting a run

### Requirement: Session list and session history endpoints
The backend SHALL expose GET /v2/conversations to list the authenticated user's chat sessions and GET /v2/conversations/:conversationId to retrieve one session's ordered history, relative to its plugin base path. These routes SHALL proxy LCore's authoritative conversation APIs using trusted identity, preserve supported pagination and identifiers, and return common-package history contracts including available agent/run attribution. They SHALL NOT build an independent chat store. Existing owner-scoped deletion SHALL be proxied consistently with the selected LCore contract.

#### Scenario: List own sessions

- **WHEN** an authorized user requests the session list
- **THEN** the backend returns only that user's LCore conversations, including an empty result when none exist

#### Scenario: Retrieve a session

- **WHEN** the owner requests a particular conversation ID
- **THEN** the backend returns its ordered messages and available attribution without copying Cloud Agents transcripts into main chat history by default

#### Scenario: Another user's session

- **WHEN** a user requests another owner's conversation or supplies a forged user identifier
- **THEN** the request does not disclose that conversation's content or execution evidence

#### Scenario: History dependency failure

- **WHEN** LCore cannot list, retrieve or delete history
- **THEN** the backend returns a safe upstream failure rather than fabricated history or successful deletion

### Requirement: Group-based chat and history authorization
The backend SHALL register agentic-chat permissions with the native Backstage permission framework and enforce authorization on execution and history routes. The permission contract SHALL distinguish chat use and history access and define coverage for cancellation and deletion. Administrators SHALL be able to grant these permissions to Catalog groups through RHDH RBAC roles. Group membership SHALL come from trusted identity; a group grant SHALL NOT bypass conversation ownership, Catalog authorization or tool permissions. Frontend visibility alone SHALL NOT enforce access.

#### Scenario: Granted group member

- **WHEN** an authenticated member of an authorized group invokes a route covered by its RBAC grant
- **THEN** the backend permits the operation subject to agent access and conversation ownership

#### Scenario: Missing permission

- **WHEN** an authenticated user lacks the route's required chat or history permission
- **THEN** the backend denies the direct HTTP request before forwarding it to LCore

#### Scenario: History grant without ownership

- **WHEN** a group member has history permission but requests another user's session
- **THEN** the history remains inaccessible

### Requirement: Correlated streaming and cancellation proxy
The backend SHALL forward correlated conversation, request/turn, agent and workflow-run identifiers with agent start, tool activity, text and terminal outcomes. It SHALL preserve LCore persistence failures and cancellation semantics rather than manufacturing success. Owner-scoped cancellation SHALL propagate to LCore and suppress late answer output. Request retries SHALL preserve the original request ID for LCore idempotency. Cross-agent transition events and interactive approval flows SHALL NOT be exposed as supported MVP operations.

#### Scenario: Streaming response

- **WHEN** LCore emits single-agent progress and a terminal outcome
- **THEN** the frontend receives the corresponding ordered events and safe outcome with the same correlation identifiers

#### Scenario: Cancel or retry a turn

- **WHEN** the owner cancels an active turn or retries its request ID after disconnect
- **THEN** the backend propagates that operation without creating a replacement request identity or emitting a late answer

### Requirement: Trusted connection and credential forwarding
The backend SHALL connect only to the configured LCore destination using authenticated transport and validated TLS. Service tokens SHALL be read from backend-only secret configuration and sent only through the agreed authentication channel. Forwarded user identity SHALL be accepted by LCore only under a verified service trust contract. Model/MCP/artifact credentials SHALL remain outside Catalog entities, execution snapshots, browser responses, routine logs and chat history. Any required per-request credential forwarding SHALL use a dedicated transient backend channel whose persistence behavior is verified before use.

#### Scenario: Authenticated backend request

- **WHEN** RHDH forwards a turn or history request to LCore
- **THEN** it authenticates the service connection and conveys trusted user identity without exposing service or capability tokens to the browser

#### Scenario: Connection redirects or fails

- **WHEN** the configured upstream redirects to an unapproved destination or rejects authentication
- **THEN** the backend does not disclose credentials or protected payloads and returns a safe failure

### Requirement: Integrated LCore and Cloud Agents configuration
The RHDH backend workstream SHALL supply the integrated agentic-chat configuration for a compatible dedicated LCore runtime with the Cloud Agents execution dependency and required SDK/OGX packages. RHDH-facing configuration SHALL group the LCore connection, service authentication, trusted user identity, static agent/profile selection, limits, tool/MCP/skill/artifact references and RBAC policy. Application and worker image/package revisions SHALL match the tested runtime combination. Configuration SHALL enable the controller/workflow APIs and approved runtime connections without requiring a separate Cloud Agents API service unless an integration finding requires it. Packaging and installation SHALL consume these configuration artifacts rather than define a second configuration contract.

#### Scenario: Configure the integrated runtime

- **WHEN** an administrator supplies the supported agentic-chat configuration
- **THEN** RHDH, LCore and Cloud Agents use compatible endpoints, trusted identity, runtime dependencies and approved capability bindings

#### Scenario: Omitted capability bindings

- **WHEN** an agent has missing model, skill or tool bindings
- **THEN** configuration and startup checks do not reject the agent solely for those omissions

### Requirement: Persistence and runtime lifecycle configuration
The backend workstream SHALL supply PostgreSQL connection/storage configuration for LCore conversation and workflow stores, database secret references, readiness settings, migration/startup ordering and compatible rollback/disable/re-enable settings. LCore SHALL remain responsible for history storage implementation and schema migrations. Configuration SHALL preserve saved history through compatible service restarts and plugin disable/re-enable. An unsandboxed agent SHALL NOT depend on sandbox readiness.

#### Scenario: Configured persistent stores

- **WHEN** the supported runtime uses the backend-owned storage and lifecycle configuration
- **THEN** LCore stores conversation/workflow data in the configured PostgreSQL stores and owners can reopen saved conversations after compatible restarts or plugin reenablement

#### Scenario: Independent sandbox readiness

- **WHEN** no configured agent requires sandboxing
- **THEN** runtime readiness and execution do not require a live sandbox gateway or spawner

### Requirement: Secret-backed tokens and trusted transport configuration
The backend workstream SHALL supply Kubernetes/OpenShift Secret or supported external-secret configuration for production credentials, exposing values only to approved backend runtime consumers. RHDH config declarations SHALL mark token fields as secret/backend-only. The RHDH-to-LCore service token SHALL be injected into RHDH and sent over validated TLS through the agreed authentication channel, with trusted user identity conveyed separately. Provider, MCP, artifact, database and sandbox credentials SHALL be mounted/injected into approved LCore/worker consumers by secret reference. Any required token forwarding SHALL use a tested transient channel and SHALL NOT persist secret values in Catalog entities, snapshots, workflow definitions/provider inputs, transcripts, history or routine diagnostics. Examples SHALL contain placeholders only; credential rotation SHALL be supported with its procedure recorded in the consolidated guide.

#### Scenario: Service token delivery

- **WHEN** RHDH sends a turn or history request using the configured service credential
- **THEN** LCore verifies that credential and trusted owner identity without exposing the credential to the browser or stored execution configuration

#### Scenario: Provider or MCP authentication

- **WHEN** the issuing process calls an approved model or tool endpoint
- **THEN** it receives the scoped credential through the configured secret binding or verified transient channel without persisting the secret value as workflow input

#### Scenario: Missing or rotated credential

- **WHEN** a required credential is absent, invalid or rotated
- **THEN** runtime authentication fails safely or resumes through the configured rotation mechanism without credential disclosure or unauthenticated fallback

### Requirement: Group and agent configuration examples
The backend workstream SHALL supply RHDH RBAC examples mapping operator-selected Catalog groups to the registered chat-use and history permissions, including cancellation/deletion coverage. Group examples SHALL use placeholders rather than grant access to every authenticated user. Agent examples SHALL demonstrate omitted, false and true useSandbox values. Group grants SHALL preserve LCore service authorization, Catalog access and owner isolation.

#### Scenario: Group configuration applied

- **WHEN** an operator assigns the supplied role configuration to a chosen group
- **THEN** permitted members can use the intended routes while nonmembers and attempts to access another owner's session remain denied

#### Scenario: Agent sandbox configuration examples

- **WHEN** the supplied examples omit useSandbox or set it to false or true
- **THEN** omitted/false selects unsandboxed execution and true selects only the predefined sandbox profile

### Requirement: Optional sandbox infrastructure configuration
The backend workstream SHALL supply the approved gateway/worker image, service-account and resource/network/mount/cleanup configuration when an agent requires sandboxing. Settings SHALL remain administrator-defined and enforceable from the actual execution worker. The configuration SHALL support provisioning-failure handling without unsandboxed fallback and observable cleanup failure with bounded reconciliation. The LCore/Cloud Agents workstream SHALL implement the provisioning and cleanup lifecycle; the installation workstream SHALL apply the supplied infrastructure configuration.

#### Scenario: Sandbox-enabled agent configured

- **WHEN** an agent sets useSandbox true
- **THEN** the supplied profile identifies the approved infrastructure and credentials needed for worker connectivity and statically enforced permissions

#### Scenario: Sandbox gateway unavailable

- **WHEN** the configured gateway cannot provision a required sandbox
- **THEN** the run stops without changing execution mode, unsandboxed agents remain independent of that gateway, and any resources requiring reconciliation remain observable

## Investigation inputs

- [AI Catalog agent model](https://github.com/redhat-developer/rhdh-plugins/tree/main/workspaces/ai-integrations/plugins/catalog-model-ai-resource-agent): confirm the schema, TypeScript type, validator tests and every provider mapping that must preserve spec.useSandbox.
- [Intelligent Assistant router](https://github.com/redhat-developer/rhdh-plugins/blob/b1c2c39bbd7dac6dc4c1eb5f180bd7117acba696/workspaces/intelligent-assistant/plugins/intelligent-assistant-backend/src/service/router.ts), [permissions](https://github.com/redhat-developer/rhdh-plugins/blob/b1c2c39bbd7dac6dc4c1eb5f180bd7117acba696/workspaces/intelligent-assistant/plugins/intelligent-assistant-common/src/permissions.ts) and [RBAC policy](https://github.com/redhat-developer/rhdh-plugins/blob/b1c2c39bbd7dac6dc4c1eb5f180bd7117acba696/workspaces/intelligent-assistant/rbac-policy.csv): reuse the authenticated history/proxy and group-to-role patterns; finalize agentic-chat permission names, route schemas, cancellation/deletion mapping and pagination before implementation.
- [Intelligent Assistant configuration repository](https://github.com/redhat-developer/rhdh-intelligent-assistant-configs) and [harness secret examples](https://github.com/jameswnl/lightspeed-stack/blob/f2d5911092d4591c2c6766b38c7c38a8aaa56915/examples/workflows/k8s-secrets.yaml): select the backend-owned configuration location, runtime image/package bindings, PostgreSQL settings, service trust, secret provisioning/rotation, RBAC examples and optional sandbox infrastructure configuration. Development library-mode/noop/SQLite examples do not establish production authentication or persistence.
- Coordinate service authentication, forwarded identity and transient token handling with the LCore workstream; provide the resulting configuration artifacts to installation and consolidated documentation. Verify actual backend/worker access and persistence behavior before enabling credential forwarding.
