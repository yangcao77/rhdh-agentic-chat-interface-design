## ADDED Requirements

### Requirement: LCore authoritative history
LCore SHALL own messages, context and execution attribution. RHDH SHALL persist only necessary content-free mappings. Run transcripts SHALL remain linked evidence and SHALL NOT be independently replayed as duplicate chat context.

#### Scenario: Restart continuation

- **WHEN** production services restart after a completed turn
- **THEN** the owner reopens prior context and RHDH supplies the execution configuration again independently of history

#### Scenario: One logical answer

- **WHEN** a workflow produces internal transcripts and a final answer
- **THEN** user-visible history contains one logical user turn and final assistant answer

### Requirement: Configuration outside conversation history
LCore SHALL NOT save execution snapshots, serialized workflow definitions or snapshot references in conversation records, turn records or history evidence. Conversation continuation SHALL NOT depend on a history-pinned configuration revision. Startup-published runtime configuration and the workflow runner's separate definition store SHALL have a storage and retention boundary independent of conversation history.

#### Scenario: Snapshot is execution input

- **WHEN** a turn completes using a supplied snapshot
- **THEN** saved conversation history contains messages and agent/run attribution without the snapshot or a snapshot reference

#### Scenario: Missing execution configuration

- **WHEN** a new run lacks the configuration required by the selected transfer mode
- **THEN** execution fails rather than reconstructing configuration from conversation history

### Requirement: Agent related database migration
Conversation metadata SHALL include nullable agent identity/name, execution backend and contract version. Turn metadata SHALL correlate request, agent, workflow run and terminal state. Pending turns SHALL have a valid representation despite the current non-null completion timestamp.

#### Scenario: Legacy conversation

- **WHEN** migrations apply to rows without agent attribution
- **THEN** existing history remains readable without invented attribution

#### Scenario: Running turn

- **WHEN** a new execution starts before completion
- **THEN** a pending/active record can be persisted and later finalized

#### Scenario: Sandbox evidence

- **WHEN** a turn uses a configured sandbox
- **THEN** its execution record links sandbox identity and cleanup outcome to that turn

### Requirement: Owner scoped history lifecycle
List/reopen/continue/delete and linked execution evidence SHALL enforce ownership. Deletion SHALL remove conversation-owned content and links according to retention policy, preserve shared artifacts and report dependency failures. Failed/cancelled turns SHALL preserve earlier completed history.

#### Scenario: Cross user access

- **WHEN** another user requests a conversation or its run evidence
- **THEN** access is denied

#### Scenario: Delete conversation independently of runtime configuration

- **WHEN** an owner deletes a conversation
- **THEN** its content and links are removed without deleting the independently managed runtime configuration used by other runs
