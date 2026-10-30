## Why

[RHDHPLAN-1644](https://redhat.atlassian.net/browse/RHDHPLAN-1644) needs a native RHDH 2.2 Boost backend for one static agent. Omitted allowedTools defaults to available configured tools; an omitted spawn mode uses the LCore/Cloud Agents default (currently `ephemeral`). RHDH owns configuration; LCore owns execution and persistent history.

## What Changes

- Add native `agentic-chat-backend` and browser-safe `agentic-chat-common` packages using Backstage services.
- Retain the AI Catalog OpenAI Agents SDK agent schema, adding optional per-agent `spawn` (`"none"`, `"local"` or `"ephemeral"`), passed through to LCore unchanged so omission uses the LCore/Cloud Agents default. Send one predefined agent snapshot with static model, tool/MCP, skill, knowledge, limit and sandbox configuration; do not discover agents or traverse handoff targets.
- Send the resolved configuration with every execution for the MVP; startup synchronization using the same contract is a stretch goal. No dynamic reload or change pickup.
- Add new LCore `POST /v1/agents/run` as the agent execution controller API, accepting typed static execution configuration and a turn invocation envelope.
- LCore hosts the execution/handoff controller and Cloud Agents integration. For 2.2, dispatch exactly one agent and reject cross-agent transitions.
- LCore translates the RHDH profile into one Cloud Agents agent step, preserving root, limits, optional sandbox settings and tool restrictions. Prove the SDK/executor bridge without nested loops.
- Use an explicit allowed tools list for the intended MVP profile, while accepting omission. Missing model, skill or tool bindings do not reject configuration; enabled sandbox provisioning failure stops the run without fallback.
- Stream correlated execution events through RHDH, finalize persistent LCore history, and return a truthful final result.
- Extend conversation and turn metadata with agent and execution attribution; do not save or pin execution snapshots in conversation history. Keep chat content out of the RHDH plugin database.
- Keep Cloud Agents transcripts as linked evidence outside main chat history by default; optional storage of intermediate execution output is a stretch goal.
- Document installation, model/MCP/artifact connectivity, static sandbox provisioning settings, persistent storage and operational checks.

## Capabilities

### New Capabilities

- `boost-chat-rhdh-backend`: native packages, Catalog/spawn, run/history routes, authorization and integrated LCore/Cloud Agents configuration for persistence, credentials, RBAC and optional sandbox infrastructure.
- `boost-chat-lcore-cloud-agents`: controller/workflow dispatch, SDK executor, tools/sandbox policy, streaming and authoritative history.
- `boost-chat-deployment`: packaging, installation and deployed verification using backend-owned configuration, plus consolidated documentation.

### Modified Capabilities

None. Long-lived capability specs remain unchanged during design.

## Non-goals

Agent discovery; human-in-the-loop approval, edit/reject or interactive authorization; multi-agent execution or handoffs in 2.2; arbitrary workflow generation; user/model edits to definitions or sandbox settings; additional execution backends; live configuration editing; parallel teams; advanced history/search/export; infrastructure operator implementation.

## Canonical Touchpoints

**Change type:** architecture. No canonical documents are updated now. Implementation will add target `workspaces/boost/specifications/adr/0001-chat-cloud-agents-lcore.md`, reconcile target Boost `specifications/CURRENT.md` and frontend architecture, and promote the three delta capabilities under `openspec/specs/` on archive. No PRD change is proposed.

## Impact

Requires coordinated RHDH, LCore and Cloud Agents integration. Existing endpoints, storage and spawner adapters are foundations; runtime implementation remains pending.
