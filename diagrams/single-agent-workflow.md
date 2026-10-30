# Single agent workflow

```mermaid
flowchart TD
  Q[User query and conversation ID] --> R[RHDH authenticates and resolves static agent profile]
  R --> C[LCore POST /v1/agents/run validates configuration and current access]
  C --> P[Create logical turn and load authoritative history]
  P --> B{Agent spawn mode}
  B -->|none or local| A[Cloud Agents workflow runs]
  B -->|ephemeral or omitted - LCore default| S[Provision predefined sandbox]
  S -->|Provisioned| A
  S -->|Provisioning failed| E[Record correlated failure]
  A --> D{Model result}
  D -->|Tool request| T[Apply optional tool filter and existing authorization]
  T -->|Permitted| A
  T -->|Denied| E
  D -->|Cross-agent transfer| E
  D -->|Final answer| F[Record result]
  E --> X[Cleanup provisioned sandbox if any and save run evidence]
  F --> X
  X --> H[Finalize LCore conversation history]
  H --> U[Return final outcome through RHDH]
```

Configuration validation accepts missing model, skill or tool bindings without inventing capabilities. An unavailable capability needed during execution produces a safe runtime failure. A sandbox (spawn `ephemeral`, set or defaulted) that cannot be provisioned stops the run without an unsandboxed fallback.
