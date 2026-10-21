# Execution lifecycle

```mermaid
stateDiagram-v2
  [*] --> Validating
  Validating --> Failed: Unsupported configuration
  Validating --> Provisioning: Static profile accepted
  Provisioning --> Running: Configured environment ready
  Provisioning --> Cleaning: Provisioning failure with resources
  Running --> Running: Allowed tool iteration within bounds
  Running --> Cleaning: Answer failure limit or cancellation
  Cleaning --> Finalizing: Record cleanup outcome and any retry obligation
  Finalizing --> Completed: Answer and history saved
  Finalizing --> Failed: Execution or persistence failure
  Finalizing --> Cancelled: Cancellation recorded
  Failed --> [*]
  Completed --> [*]
  Cancelled --> [*]
```

Cleanup failure is linked to the run and reconciled with bounded policy; it is never silently treated as destroyed. A completed answer and outstanding cleanup obligation are distinct stored states.
