## ADDED Requirements

### Requirement: Static selected profile
RHDH SHALL resolve one predefined authorized agent and its model, configured tools/MCP bindings, pinned skills, knowledge, limits, optional allowedTools and optional sandbox settings. It SHALL preserve the selected root, default omitted allowedTools to all available configured tools and default sandboxing to disabled. Neither omission SHALL invalidate the profile. It SHALL NOT discover agents or silently prune required capabilities.

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
The MVP SHALL transfer the resolved static agent configuration from RHDH with every run, including continuation and restart. LCore SHALL validate it independently of saved context and SHALL NOT save snapshots or snapshot references in conversation history. Startup synchronization using the same contract SHALL remain a stretch goal. Dynamic reload, change detection and configuration pickup SHALL be out of scope. Secrets SHALL remain outside snapshots.

#### Scenario: Continued turn

- **WHEN** a continued turn supplies its predefined configuration from RHDH
- **THEN** LCore loads prior messages independently without retrieving configuration from history

#### Scenario: Optional settings omitted

- **WHEN** the resolved profile omits allowedTools and sandbox settings
- **THEN** LCore accepts it, permits all available configured tools and executes without a sandbox


### Requirement: One step translation and execution controller endpoint
LCore SHALL expose a new POST /v1/agents/run execution controller endpoint with dedicated typed request, response and stream contracts. It SHALL accept user input, request identity, optional conversation identity, the resolved static agent configuration, and streaming preference. LCore SHALL translate the resolved RHDH profile into one schema-validated Cloud Agents agent step with static configuration and a distinct invocation envelope. The controller SHALL consume execution configuration before inference serialization and SHALL integrate context, correlated events and persistent history.

#### Scenario: Workflow translation

- **WHEN** a valid static profile is submitted
- **THEN** one step preserves its root, capabilities, bounds, optional tool filter and optional sandbox profile while runtime input and IDs remain separate

#### Scenario: Oversized snapshot

- **WHEN** the proposed execution envelope exceeds its negotiated limit
- **THEN** the service rejects it without truncation

#### Scenario: Controller request

- **WHEN** an authenticated RHDH client submits a valid single-agent turn to POST /v1/agents/run
- **THEN** the controller validates the static configuration, invokes the embedded workflow runner and returns correlated progress and a terminal outcome linked to persistent history
