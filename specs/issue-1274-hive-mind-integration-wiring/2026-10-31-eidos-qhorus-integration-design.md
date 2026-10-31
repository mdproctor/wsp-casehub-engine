# Eidos and Qhorus Integration Wiring — Engine Swarm Pipeline

**Epic:** #1274 (Hive Mind integration wiring)
**Issues:** #1279 (eidos proximity queries), #1280 (qhorus broadcast)
**Branch:** `issue-1274-hive-mind-integration-wiring`
**Date:** 2026-10-31

## Context

Cross-repo review revealed that eidos and qhorus both have complete
internal implementations, but engine's swarm coordination code doesn't
use them. Eidos provides agent graph queries (proximity, co-activity,
collaboration). Qhorus provides structured case-scoped messaging with
commitment tracking. Both APIs are already on the engine classpath
via engine-api dependencies.

Batch 1 (thread safety + SPI cleanup: #1275–#1278) landed this session.
This spec covers Batch 2: wiring eidos queries and qhorus messaging
into the swarm coordination pipeline.

## Design Principles

1. **Qhorus-first for inter-agent communication.** Qhorus channels are
   observable, participatory, and cross-process. Any agent or service
   can join the coordination channel and consume swarm events without
   the engine anticipating them. CDI events complement for in-process
   consumers.

2. **Direct injection for queries.** Eidos queries are synchronous
   lookups, not messaging. Inject via `Instance<T>` for optional
   availability — skip the integration gracefully when eidos isn't
   deployed.

3. **Engine accesses qhorus through CaseChannelProvider only.** Never
   inject `MessageDispatcher` directly. Protocol PP-20261031-060886.

4. **STATUS for agent-visible state changes, EVENT for telemetry.**
   STATUS is agent-visible and informational — agents observe and adapt
   voluntarily. Fits the stigmergy model. EVENT is agent-invisible —
   for monitoring/observability only.

## Part 1: Eidos Proximity Queries (#1279)

### Dependency Model

`Instance<AgentGraphQuery>` and `Instance<RuntimeCollaborationQuery>`
injected into swarm classes. When no eidos implementation is deployed,
`Instance.isResolvable()` returns false and the integration is skipped.
No new module dependencies — engine-api already depends on eidos-api.

### Integration Points

#### 1.1 SwarmProvisioner — capability overlap check

In `evaluateAndProvision()`, before provisioning a new agent, query
`AgentGraphQuery.coActiveAgents(caseId, tenancyId)` to check if agents
with overlapping capabilities are already active. Also use
`topAgentsByOutcome(capabilityTag, taskDomain, tenancyId, limit)` to
prefer agents with proven track records for the requested capability.

**Where:** Inside `doProvision()`, after the provisioning signal is
confirmed but before `workerProvisioner.provision()` is called.

**Behavior when unavailable:** Skip the check, provision as before.

#### 1.2 StigmergyCoordinator — neighbor awareness

In `agentActivated()`, populate the agent's neighbor awareness by
querying `AgentGraphQuery.coActiveAgents(externalRef, tenancyId)`.
This enriches the coordinator's view of which agents are working
together.

**Where:** At the end of `agentActivated()`, after the agent state
transition.

**Behavior when unavailable:** Skip — neighbor awareness is
supplementary.

#### 1.3 TeamDetector — collaboration graph enrichment

In the team detection cycle, query
`RuntimeCollaborationQuery.collaborators(agentId, tenancyId)` to
supplement behavioral fingerprint clustering with actual collaboration
graph data.

**Where:** In the clustering phase, as an additional similarity signal
alongside behavioral fingerprint distance.

**Behavior when unavailable:** Skip — team detection still works from
behavioral fingerprints alone.

### Testing Strategy

Each integration point gets two test paths:
- **With eidos:** Mock `AgentGraphQuery`/`RuntimeCollaborationQuery`,
  verify the integration enriches behavior
- **Without eidos:** Verify `Instance.isResolvable() == false` path
  degrades gracefully with no behavioral change

## Part 2: Qhorus Broadcast (#1280)

### Strategy

Support both qhorus channel posting and CDI domain events:

- **Primary:** Post swarm coordination events to the `coordination`
  channel via `CaseChannelProvider.postToChannel()`. This is observable
  and participatory — any agent or service joined to the channel can
  consume and react.
- **Complement:** Fire CDI events for in-process observers that need
  synchronous reaction within the same JVM.

Qhorus will eventually have routers and brokers for intent-based routing
("I need X" without direct RPC). The coordination channel approach is
compatible with that evolution — a qhorus router can observe the channel
and re-route as needed.

### Events to Post

All posted to the `coordination` channel as `MessageType.STATUS`:

| Event | Trigger | Content |
|-------|---------|---------|
| SWARM_PROVISION_COMPLETED | Agent successfully provisioned | agentId, bindingName, capabilities |
| SWARM_AGENT_TERMINATED | Agent deprovisioned (idle) | agentId, reason, idleCycles |
| SWARM_MEMBER_COMPLETED | Agent departed swarm | agentId |
| SIGNAL_CONSENSUS_DETECTED | Signal consensus found | signalName, sourceCount |
| COORDINATION_STORM_DETECTED | Coordination storm detected | signalRate, threshold |
| ROLE_EMERGED / ROLE_DISSOLVED | Role change detected | roleName, memberAgents |
| TEAM_EMERGED / TEAM_DISSOLVED | Team change detected | teamName, memberAgents |
| PROGRESS_UPDATED | Swarm progress changed | explorationPace, roleStability, convergence |

### Implementation Approach

Inject `Instance<CaseChannelProvider>` into `SwarmProvisioner` and
`StigmergyCoordinator`. Post STATUS messages at each event site with
structured JSON content. Wrap in try-catch consistent with the
listener exception pattern from #1277.

**Where the posts go:**

- `SwarmProvisioner.doProvision()` — after successful provision, post
  SWARM_PROVISION_COMPLETED
- `SwarmProvisioner.evaluateDeprovisioning()` — after termination, post
  SWARM_AGENT_TERMINATED; after member removal, post
  SWARM_MEMBER_COMPLETED
- `StigmergyCoordinator.detectPatterns()` — after pattern detection,
  post SIGNAL_CONSENSUS_DETECTED / COORDINATION_STORM_DETECTED
- `SwarmProvisioner.notifyRolesChanged()` — post ROLE_EMERGED /
  ROLE_DISSOLVED
- `SwarmProvisioner.notifyTeamsChanged()` — post TEAM_EMERGED /
  TEAM_DISSOLVED
- `SwarmProvisioner.notifyProgressUpdated()` — post PROGRESS_UPDATED

### Tenancy

All qhorus and eidos APIs require `tenancyId`. The swarm classes
currently operate on `caseId` only. The tenancyId must be resolved
from the case context — check if `WorkerScope` or `CaseInstance`
provides it.

### Testing Strategy

- **With CaseChannelProvider:** Mock the provider, verify STATUS
  messages are posted with correct content at each event site
- **Without CaseChannelProvider:** Verify `Instance.isResolvable()`
  path skips posting — no exceptions, existing behavior preserved
- **Integration:** Verify the coordination channel receives messages
  when both engine and qhorus are active

## References

- `engine/api/pom.xml` — existing eidos-api, qhorus-api dependencies
- `NormativeChannelLayout.java` — coordination channel definition
- `CaseChannelProvider.java` — engine's qhorus access SPI
- `MessageType.java` — STATUS (agent-visible), EVENT (agent-invisible)
- `CommitmentState.java` — qhorus obligation lifecycle
- `parent/docs/platform/messaging-vs-notifications.md` — positioning guide
- PP-20261031-060886 — messaging channel selection protocol
- `SwarmProvisioner.java`, `StigmergyCoordinator.java`, `TeamDetector.java` — integration targets
- `AgentGraphQuery.java`, `RuntimeCollaborationQuery.java` — eidos APIs
