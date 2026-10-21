## ADDED Requirements

### Requirement: Native Boost packages
The system SHALL provide a native Boost backend and browser-safe common package supporting static registration and OCI dynamic installation. RHDH SHALL use native services and forward execution to LCore.

#### Scenario: Install plugin

- **WHEN** a supported backend package is installed and configured
- **THEN** frontend-facing chat/history contracts are available without a browser runtime SDK

### Requirement: One LCore controlled agent
LCore SHALL host the execution/handoff controller and Cloud Agents execution integration. The 2.2 runtime SHALL support exactly one agent per turn using OGX and a LCore-owned OpenAI Agents SDK adapter with one model/tool loop.

#### Scenario: Single agent proof

- **WHEN** a valid static profile executes a query
- **THEN** one agent step runs through Cloud Agents and the selected SDK/OGX adapter with no Node agent loop

#### Scenario: Cross agent request

- **WHEN** a definition or run requests another agent or handoff
- **THEN** execution rejects the unsupported capability and no second agent runs

### Requirement: Static sandbox lifecycle
The runtime SHALL apply administrator-predefined spawn/image/spawner/resource/network/mount/cleanup settings. User or model input SHALL NOT change them. Required provisioning failure SHALL stop execution without unsandboxed fallback. Cleanup SHALL be attempted on every terminal outcome and failed cleanup SHALL be recorded for bounded reconciliation.

#### Scenario: Spawn fixed environment

- **WHEN** the profile requires an ephemeral sandbox
- **THEN** the approved environment is created before the agent step and cleaned up after it

#### Scenario: Reject override

- **WHEN** a query or caller payload requests another image or spawn policy
- **THEN** the configured policy remains authoritative and the override is rejected

#### Scenario: Spawn failure

- **WHEN** the required sandbox cannot be provisioned
- **THEN** the agent is not run locally as a fallback

#### Scenario: Cleanup failure

- **WHEN** sandbox teardown fails after a run
- **THEN** the failure and environment identity are recorded for bounded retry or reconciliation

### Requirement: Bounded execution
The runtime SHALL enforce one agent invocation per turn, separate model/tool/token/time bounds and one active turn per conversation.

#### Scenario: Iteration limit

- **WHEN** a single agent repeatedly calls tools until its iteration budget is exhausted
- **THEN** the runtime stops with a correlated limit outcome

#### Scenario: Concurrent turn

- **WHEN** another turn is active on the conversation
- **THEN** the competing request cannot start a second concurrent turn
