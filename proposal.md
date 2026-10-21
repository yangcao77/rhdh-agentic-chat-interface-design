## Why

[RHDHPLAN-1644](https://redhat.atlassian.net/browse/RHDHPLAN-1644) needs a native Boost chat backend. For RHDH 2.2, a user talks to one statically configured agent with optional tool filtering and sandboxing, defaulting to all available tools and no sandbox. RHDH owns configuration; LCore owns execution and persistent conversation history.

## What Changes

- Add native `boost-chat-backend` and browser-safe `boost-chat-common` packages using Backstage services.
- Resolve one predefined AI Catalog agent and static model, tool/MCP, skill, knowledge, limit and sandbox bindings into a versioned execution snapshot. Do not discover agents or traverse handoff targets.
- Send the resolved configuration with every execution for the MVP; startup synchronization using the same contract is a stretch goal. No dynamic reload or change pickup.
- Add new LCore `POST /v1/agents/run` as the agent execution controller API, accepting typed static execution configuration and a turn invocation envelope.
- LCore hosts the execution/handoff controller and Cloud Agents integration. For 2.2, dispatch exactly one agent and reject cross-agent transitions. Use one OGX backend with the OpenAI Agents SDK integration owned by LCore.
- LCore translates the resolved RHDH profile into one Cloud Agents agent step, preserving root, limits, optional sandbox settings and optional tool restrictions. Missing settings use no sandbox and all available configured tools. Prove the SDK/executor bridge without nested agent loops.
- Stream correlated execution events through RHDH, finalize persistent LCore history, and return a truthful final result.
- Extend conversation and turn metadata with agent and execution attribution; do not save or pin execution snapshots in conversation history. Keep chat content out of the RHDH plugin database.
- Document installation, model/MCP/artifact connectivity, static sandbox provisioning settings, persistent storage and operational checks.

## Capabilities

### New Capabilities

- `boost-chat-agent-runtime`: native packages and single-agent LCore/Cloud Agents execution with static sandbox settings.
- `boost-chat-execution-configuration`: static configuration resolution, snapshots and one-step workflow translation.
- `boost-chat-allowed-tools`: optional tool restrictions enforced at execution; omitted means all available configured tools.
- `boost-chat-stream-contract`: correlated streaming, cancellation and truthful terminal results.
- `boost-chat-lcore-history`: authoritative persistent history and agent/run metadata.
- `boost-chat-access-and-privacy`: authenticated ownership and trusted service/credential boundaries.

### Modified Capabilities

None. Long-lived capability specs remain unchanged during design.

## Non-goals

Agent discovery; human-in-the-loop approval, edit/reject or interactive authorization; multi-agent execution or handoffs in 2.2; arbitrary workflow generation; user/model edits to definitions or sandbox settings; additional execution backends; live configuration editing; parallel teams; advanced history/search/export; infrastructure operator implementation.

## Canonical Touchpoints

**Change type:** architecture. No canonical documents are updated now. Implementation will add target `workspaces/boost/specifications/adr/0001-chat-cloud-agents-lcore.md`, reconcile target Boost `specifications/CURRENT.md` and frontend architecture, and promote the six delta capabilities under `openspec/specs/` on archive. No PRD change is proposed.

## Impact

Coordinated RHDH, LCore and Cloud Agents integration is required. Existing endpoints, storage and spawner adapters are foundations. This design and its unchecked tasks do not claim delivered runtime behavior.
