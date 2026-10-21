## ADDED Requirements

### Requirement: Optional static tool filter
The agent configuration MAY supply allowedTools. If omitted, the runtime SHALL allow all tools available through the agent's configured tool/MCP bindings without an additional agent-level filter. A supplied list SHALL restrict actual tool exposure and dispatch to listed names; an explicit empty list SHALL allow no tools. Existing deployment policy and user permissions SHALL still apply. Missing allowedTools SHALL NOT cause a configuration error. Adapters and workers SHALL preserve omission versus an empty list.

#### Scenario: Default tools

- **WHEN** a valid agent omits allowedTools and requests an available configured tool authorized by existing policy
- **THEN** the tool can execute without requiring an explicit list

#### Scenario: Allowed tool

- **WHEN** a supplied allowedTools list includes an available authorized tool
- **THEN** the tool can execute

#### Scenario: Extra tool with explicit filter

- **WHEN** a supplied allowedTools list excludes a tool exposed by the configured server
- **THEN** that tool is not executable

#### Scenario: Empty list

- **WHEN** allowedTools is explicitly an empty list
- **THEN** the run can proceed without tools and no tool call executes

#### Scenario: Revoked permission

- **WHEN** user permission is revoked before dispatch
- **THEN** the call is denied even when allowedTools is omitted or includes the tool

### Requirement: No interactive approval flow
The MVP SHALL execute permitted calls without human-in-the-loop pauses and SHALL reject capabilities requiring interactive approval or credential collection. Approval-like payloads SHALL NOT change tool policy.

#### Scenario: Approval dependent tool

- **WHEN** a required tool needs an interactive grant unavailable in the predefined binding
- **THEN** the run reports unsupported configuration instead of waiting for a person

#### Scenario: Policy bypass

- **WHEN** a caller marks a call excluded by an explicit filter as approved
- **THEN** the tool remains unavailable

### Requirement: Runtime reachable authenticated tools
The component issuing model/MCP/artifact calls SHALL use approved transport, network/TLS and backend-held credentials. Required binding or connectivity failure SHALL be surfaced without exposing secrets.

#### Scenario: Configured execution access

- **WHEN** MCP calls originate in an execution worker with sandboxing enabled or disabled
- **THEN** connectivity and scoped authentication are verified from the issuing component

#### Scenario: Skill capability request

- **WHEN** a skill requests a tool excluded by an explicit allowedTools list
- **THEN** it does not execute
