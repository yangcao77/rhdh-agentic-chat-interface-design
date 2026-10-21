# RHDH LCore and Cloud Agents architecture

Proposed RHDH 2.2 static single-agent execution. The controller is hosted in LCore; cross-agent handoffs are disabled.

```mermaid
flowchart TD
  UI[Boost frontend] --> R[RHDH native Boost backend and common]
  C[Static AI Catalog and execution profiles] --> R
  R -->|Snapshot per execution or published revision| L[LCore POST /v1/agents/run controller]
  L --> H[(Persistent conversations and execution metadata)]
  L --> W[Embedded Cloud Agents one-agent workflow]
  W --> S[Predefined sandbox worker and SDK adapter]
  S --> O[OGX inference]
  S --> M[Allowed MCP tools and approved resources]
  A[Approved immutable skill artifacts] --> S
  W -->|Events result and cleanup evidence| L
  L -->|Correlated stream and saved final answer| R
  R --> UI
```

The SDK adapter belongs to the LCore integration. The compatible executor/worker interface is an implementation gate. Static spawn settings control provisioning; the query cannot alter them. Execution configuration is runtime input and is not saved or pinned in conversation history. History stores messages and agent/run attribution.
