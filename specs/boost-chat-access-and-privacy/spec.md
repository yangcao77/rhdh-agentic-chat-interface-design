## ADDED Requirements

### Requirement: Native authentication and trusted identity
RHDH SHALL use native Backstage authentication, permissions and authorized Catalog access. LCore SHALL validate trusted integration provenance and authenticated owner identity; browser-authored owner/configuration fields SHALL NOT establish authority.

#### Scenario: Forged identity

- **WHEN** a caller supplies another owner or a capability-widening snapshot
- **THEN** the request cannot expand access or execution rights

### Requirement: Approved connections and secret handling
Inference/MCP/artifact/spawner destinations and credentials SHALL be deployment-approved and enforced at the issuing component. Snapshots, history, routine logs and safe errors SHALL omit secrets. Static policy SHALL NOT be replaceable by user/model input.

#### Scenario: Redirect outside policy

- **WHEN** an outbound request redirects to an unapproved endpoint
- **THEN** the issuing component stops before transmitting credentials or content

#### Scenario: Safe diagnostics

- **WHEN** a backend connection fails
- **THEN** returned errors omit credentials and protected payloads

### Requirement: Dedicated supported installation
The feature SHALL provide documented static/dynamic RHDH installation and a dedicated LCore/Cloud Agents deployment with PostgreSQL and required static dependencies. Readiness SHALL reflect required profile bindings. Disable/re-enable SHALL preserve saved history.

#### Scenario: No sandbox dependency by default

- **WHEN** a valid profile omits or disables sandboxing and no spawner is deployed
- **THEN** absence of a spawner does not prevent readiness or execution

#### Scenario: Missing required spawner

- **WHEN** the static profile requires sandbox execution but its spawner is unavailable
- **THEN** the affected profile does not report execution ready

#### Scenario: Reenable feature

- **WHEN** the plugin is disabled then reenabled with compatible versions
- **THEN** completed persistent history remains accessible to its owner
