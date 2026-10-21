# RHDH LCore and Cloud Agents architecture

Proposed RHDH 2.2 static single-agent execution. The controller is hosted in LCore; cross-agent handoffs are disabled.

```mermaid
flowchart TD
  UI[Boost frontend] --> R[RHDH native Boost backend and common]
  C[Static AI Catalog and execution profiles] --> R
  R -->|Resolved profile on every run| L[LCore POST /v1/agents/run controller]
  L --> H[(Persistent conversations and execution metadata)]
  L --> W[Embedded Cloud Agents one-agent workflow]
  W --> S[SDK execution worker with optional sandbox]
  S --> O[OGX inference]
  S --> M[Configured tools and resources with optional filter]
  A[Approved immutable skill artifacts] --> S
  W -->|Events result and cleanup evidence| L
  L -->|Correlated stream and saved final answer| R
  R --> UI
```
