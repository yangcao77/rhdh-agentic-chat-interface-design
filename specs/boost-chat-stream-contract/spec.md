## ADDED Requirements

### Requirement: Correlated single agent events
The integration SHALL stream agent start, available tool activity, text and safe outcomes with conversation, logical turn/request, agent and workflow-run correlation. It SHALL NOT advertise executed cross-agent transitions in 2.2.

#### Scenario: Single agent stream

- **WHEN** a run uses an allowed MCP tool and answers
- **THEN** events correlate to one agent and one logical turn

### Requirement: Truthful persistent completion
Terminal success SHALL follow authoritative history finalization. Storage failure SHALL NOT be represented as a saved successful answer. The new execution endpoint SHALL expose its own versioned agent-run event contract.

#### Scenario: History write failure

- **WHEN** inference finishes but final history storage fails
- **THEN** the terminal outcome reports the persistence failure rather than saved success

### Requirement: Cancellation and idempotency
Owner-scoped cancellation SHALL stop the active runner and configured sandbox lifecycle and suppress late output. Repeated request identities SHALL NOT create duplicate executions; active and terminal records SHALL be distinguishable.

#### Scenario: Cancel execution

- **WHEN** the owner cancels an active turn
- **THEN** the runtime stops applicable work, cleans up and emits no later answer deltas

#### Scenario: Retry after disconnect

- **WHEN** the same request ID is retried after a disconnect
- **THEN** the existing execution/outcome is returned or reported without a duplicate run
