## ADDED Requirements

### Requirement: Explicit static tool allowlist
Actual tool dispatch SHALL enforce the intersection of the predefined agent allowlist, deployment policy and current user permission. Server selection, descriptions or prompt instructions alone SHALL NOT grant a tool. Global defaults and skill helpers SHALL NOT enlarge the effective list.

#### Scenario: Allowed tool

- **WHEN** an agent requests a named tool allowed by all three policies
- **THEN** the tool can execute

#### Scenario: Extra tool

- **WHEN** a server exposes another tool outside the configured list
- **THEN** that tool is not executable

#### Scenario: Revoked permission

- **WHEN** user permission is revoked before dispatch
- **THEN** the call is denied despite a valid supplied execution configuration

### Requirement: No interactive approval flow
The MVP SHALL execute permitted calls without human-in-the-loop pauses and SHALL reject capabilities requiring interactive approval or credential collection. Approval-like payloads SHALL NOT change tool policy.

#### Scenario: Approval dependent tool

- **WHEN** a required tool needs an interactive grant unavailable in the predefined binding
- **THEN** the run reports unsupported configuration instead of waiting for a person

#### Scenario: Policy bypass

- **WHEN** a caller marks an unlisted call as approved
- **THEN** the tool remains unavailable

### Requirement: Runtime reachable authenticated tools
The component issuing model/MCP/artifact calls SHALL use approved transport, network/TLS and backend-held credentials. Required binding or connectivity failure SHALL be surfaced without exposing secrets.

#### Scenario: Sandbox MCP access

- **WHEN** MCP calls originate in the configured worker
- **THEN** connectivity and scoped authentication are verified from that worker

#### Scenario: Skill capability request

- **WHEN** a skill requests a tool outside the allowlist
- **THEN** it does not execute
