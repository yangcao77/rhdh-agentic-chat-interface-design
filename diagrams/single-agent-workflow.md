# Single agent workflow

```mermaid
flowchart TD
  Q[User query and conversation ID] --> R[RHDH authenticates and resolves static profile]
  R --> C[POST /v1/agents/run validates configuration and current access]
  C --> P[Create logical turn and load authoritative history]
  P --> S[Cloud Agents provisions predefined sandbox]
  S --> A[Run one SDK agent through OGX]
  A --> D{Model result}
  D -->|Allowed tool request| T[Validate allowlist and execute named tool]
  T --> A
  D -->|Unlisted tool or cross-agent transfer| E[Fail with correlated outcome]
  D -->|Final answer| F[Record result]
  E --> X[Cleanup and save terminal execution evidence]
  F --> X
  X --> H[Finalize LCore conversation history]
  H --> U[Return final outcome through RHDH]
```

Text/tool progress streams while execution runs. Success follows history finalization. Tool iterations do not create a second agent. Static bounds terminate loops; cancellation triggers cleanup and suppresses late output.
