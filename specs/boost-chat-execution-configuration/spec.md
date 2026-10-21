## ADDED Requirements

### Requirement: Static selected profile
RHDH SHALL resolve one predefined authorized agent and its explicit model, allowed tools/MCP, pinned skills, knowledge, limits and sandbox settings. It SHALL preserve the selected root and SHALL NOT discover agents or silently prune required capabilities.

#### Scenario: Explicit root

- **WHEN** an MVP profile selects the HR agent with no outgoing handoffs
- **THEN** that root is used without traversing other catalog agents

#### Scenario: Unsupported profile

- **WHEN** the selected profile requires handoffs or an unavailable binding
- **THEN** execution is rejected with a structured reason before inference

### Requirement: Resolved model and artifacts
RHDH SHALL resolve model environment expressions and explicit defaults. The executor SHALL materialize pinned skill resources and use runtime-reachable approved model/MCP/retrieval bindings. RHDH-local paths and boolean knowledge flags alone SHALL NOT constitute executable bindings.

#### Scenario: Empty model

- **WHEN** the configured agent model is empty and no explicit usable default exists
- **THEN** the profile is rejected

#### Scenario: Worker artifact access

- **WHEN** a selected skill has immutable resources
- **THEN** the resources are available at the execution filesystem before running

#### Scenario: Missing retrieval binding

- **WHEN** a profile requires knowledge access with no concrete approved source
- **THEN** execution fails compatibility validation

### Requirement: Versioned configuration transfer
The selected deployment mode SHALL use either snapshot-per-execution or startup publication with the same versioned configuration contract. LCore SHALL validate configuration for each run independently of saved conversation context and SHALL NOT save snapshots or snapshot references in conversation history. RHDH SHALL supply configuration on every execution or identify a published runtime profile. RHDH SHALL own static configuration compatibility. LCore SHALL keep secrets outside snapshots. Startup mode SHALL fail readiness or execution for unknown/unpublished revisions.

#### Scenario: Continued turn

- **WHEN** a continued turn supplies a valid snapshot from RHDH
- **THEN** LCore uses that configuration for the run and loads prior messages independently without retrieving a snapshot from history

#### Scenario: Changed configuration

- **WHEN** RHDH supplies a changed predefined configuration that satisfies its compatibility policy and current authorization
- **THEN** LCore validates the supplied configuration without requiring a new conversation solely because its content differs

#### Scenario: Startup mode reference

- **WHEN** a configured startup mode receives an unknown revision
- **THEN** the request fails without substituting another profile

### Requirement: One step translation and execution controller endpoint
LCore SHALL expose a new POST /v1/agents/run execution controller endpoint with dedicated typed request, response and stream contracts. It SHALL accept user input, request identity, optional conversation identity, the static snapshot or published revision, and streaming preference. The adapter SHALL build one schema-validated Cloud Agents agent step with static configuration and a distinct invocation envelope. The controller SHALL consume execution configuration before inference serialization and SHALL integrate context, correlated events and persistent history.

#### Scenario: Workflow translation

- **WHEN** a valid static profile is submitted
- **THEN** one step preserves its root, capabilities, bounds and spawn profile while runtime input and IDs remain separate

#### Scenario: Oversized snapshot

- **WHEN** the proposed execution envelope exceeds its negotiated limit
- **THEN** the service rejects it without truncation

#### Scenario: Controller request

- **WHEN** an authenticated RHDH client submits a valid single-agent turn to POST /v1/agents/run
- **THEN** the controller validates the static configuration, invokes the embedded workflow runner and returns correlated progress and a terminal outcome linked to persistent history
