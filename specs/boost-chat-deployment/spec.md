# Deployment, installation and documentation

This workstream owns packaging, installation, deployed verification and one consolidated guide. It consumes the integrated LCore/Cloud Agents, persistence, credentials, RBAC and optional sandbox infrastructure configuration owned by [boost-chat-rhdh-backend](../boost-chat-rhdh-backend/spec.md). Runtime execution and history implementation remain in [boost-chat-lcore-cloud-agents](../boost-chat-lcore-cloud-agents/spec.md).

## ADDED Requirements

### Requirement: RHDH installation bundle
The feature SHALL provide documented static backend registration and OCI dynamic installation of agentic-chat-backend, publish agentic-chat-common contracts and install the connection wiring needed by the native Boost frontend to use its execution/history routes. Installation SHALL consume the backend-owned configuration artifacts for runtime connections, service authentication, agents, credentials and RBAC rather than define those settings independently. Installation SHALL NOT expose model loops, privileged runtime endpoints or credentials to the browser.

#### Scenario: Static or dynamic installation

- **WHEN** an operator installs the supported package through either documented RHDH installation mode
- **THEN** the frontend can reach the authenticated agent-run, session-list and session-history routes using the same common contracts and backend-owned configuration

### Requirement: Install the configured LCore and Cloud Agents runtime
The installation SHALL deploy a compatible dedicated LCore runtime with the Cloud Agents execution dependency, required SDK/OGX packages and PostgreSQL conversation/workflow stores using the backend-owned configuration and tested version/image combination. It SHALL apply the supplied database/storage, secret-reference and RBAC artifacts and install gateway/worker infrastructure only where sandbox-enabled agents require it. A separate Cloud Agents API service SHALL NOT be assumed necessary without an integration finding that requires it. Configuration authoring and credential-delivery setup SHALL remain owned by the backend workstream.

#### Scenario: Deploy an unsandboxed-only profile

- **WHEN** every configured agent sets spawn "none" or "local"
- **THEN** installation applies the backend-owned configuration and controller execution and persistent history operate without a live sandbox gateway dependency

#### Scenario: Deploy a sandbox-enabled profile

- **WHEN** an agent sets spawn "ephemeral" or omits spawn while the LCore default is "ephemeral"
- **THEN** installation applies the supplied gateway/worker and secret references and the actual worker can reach required services with the configured static permissions

### Requirement: Verify persistent rollout and operational behavior
Installation SHALL follow and verify the backend-owned database migration/startup order, readiness settings and compatible rollback/disable/re-enable configuration, preserving existing history. Deployed acceptance SHALL verify authenticated execution, session list/retrieval/deletion, continuation after restart, tool filters, group/owner isolation, token redaction, rotation, cancellation, truthful completion and static/dynamic installation from the actual deployed components. Missing model, skill or tool bindings alone SHALL NOT reject an agent. Sandbox-enabled provisioning failure SHALL stop that run without fallback, unsandboxed agents SHALL remain independent of gateway availability and cleanup failures SHALL remain observable for bounded reconciliation.

#### Scenario: Restart or disable and reenable

- **WHEN** compatible services restart or the plugin is disabled and reenabled after migrations
- **THEN** owners can reopen saved conversations and continue using freshly supplied static configuration

#### Scenario: Verify supplied access and credential configuration

- **WHEN** the deployed stack is tested with authorized and unauthorized groups and valid, missing or rotated credentials
- **THEN** the backend-owned configuration enforces the expected access and authentication outcomes without exposing tokens or allowing cross-owner history access

#### Scenario: Sandbox dependency failure

- **WHEN** the sandbox gateway is unavailable
- **THEN** unsandboxed agents remain independent of it and sandbox-enabled runs fail safely without changing execution mode

#### Scenario: End-to-end verification

- **WHEN** the packaged deployment is evaluated for delivery
- **THEN** the documented execution/history, permissions, secrets, restart and applicable sandbox checks pass from the issuing backend/worker

### Requirement: One consolidated installation and configuration guide
Delivery SHALL include one RHDH-oriented installation and configuration guide covering package installation, tested LCore/Cloud Agents versions and images, database/migrations, service topology, frontend/backend wiring, agent examples, spawn modes and the omitted-spawn default, runtime connections, secret creation/injection/forwarding/rotation, group RBAC, sandbox setup, startup/readiness, verification, rollback and troubleshooting. The guide SHALL document the configuration delivered by the backend workstream; configuration implementation SHALL NOT become a separate documentation or installation responsibility. This guide SHALL be maintained as one documentation task rather than split across separate backend, LCore and sandbox documentation tasks. It SHALL distinguish production requirements from development examples and identify the two stretch goals without making them installation prerequisites.

#### Scenario: Operator follows the guide

- **WHEN** an operator follows the consolidated guide with deployment-specific secret values and group references
- **THEN** the operator can install, configure and verify the supported agentic-chat stack using the backend-owned configuration artifacts without discovering undocumented setup in another workstream

## Investigation inputs

- Use the [Intelligent Assistant configuration repository](https://github.com/redhat-developer/rhdh-intelligent-assistant-configs) as a reference for packaging and installation conventions. Adapt the backend-owned configuration artifacts rather than creating a second production configuration in this workstream.
- Confirm package distribution, supported installation charts/manifests, artifact application order and end-to-end acceptance prerequisites with both application workstreams. Configuration choices for persistence, secure credential delivery, RBAC and optional sandbox infrastructure are investigated and implemented in the RHDH backend workstream.
