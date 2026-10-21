# Evidence and integration gaps for 1644

Revised 2026-10-20. [Design](./design.md) and [report](./design-report-1644.md) define the selected 2.2 scope. This file distinguishes inspected foundations from planned integration.

- RHDH source `workspaces/boost/app-config.yaml`, lines 80-250, contains the referenced AI Catalog agents and OGX configuration. The provider emits agent instruction/model/tool/handoff/RAG fields but omits provider-level root/limits from individual entities. The report uses one predefined specialist without handoffs. The pinned user link is retained; matching local configuration was inspected.
- `workspaces/boost/plugins/ogx-entity-provider/src/providers/OgxAgentEntityProvider.ts` maps configured agents to AiResource entities. It is a configuration source, not the executor.
- New LCore `POST /v1/agents/run` is the proposed agent execution controller endpoint. Dedicated request/response/event models, static snapshot/revision validation, context integration and payload bounds are implementation work. The existing workflow runner is the internal execution foundation.
- `src/models/api/requests/agents.py` documents a one-step agent/result workflow; `src/app/endpoints/workflows.py` validates definitions and starts the runner. Grouping session_id does not assemble conversational history.
- `src/workflow/executor_factory.py` embeds LocalWorkflowRunner with state/transcript stores. A supported OpenAI Agents SDK step adapter and correlated chat event/history integration must still be proved.
- `src/workflow/spawner_factory.py` and SpawnerConfiguration use the OpenShell spawner. The gateway owns its compute driver. Fixed image/network/resource/mount policy mapping and the SDK worker image remain implementation gates.
- `src/models/database/conversations.py` has user_conversation/user_turn, owner/model/provider/response metadata, but no required agent/workflow columns. completed_at is currently non-null. `src/workflow/storage.py` uses PostgreSQL for run/transcript stores. Cloud Agents `src/cloud_agents/workflow/executor/local/executor.py` passes the full workflow definition to its run store at start. This separate persistence requires an explicit retention boundary; it does not justify saving execution snapshots in conversation history.

Required proof: one configured agent, exact allowed tools, fixed sandbox policy, SDK through OGX, persistent two-turn context, correlated events, owner isolation and cancellation. No discovery, approval workflow or additional agent/backend is an MVP proof case.
