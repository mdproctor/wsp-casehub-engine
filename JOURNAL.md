# Design Journal — issue-377-distributed-per-agent-coordination

## 2026-11-01 — Coordination Vocabulary Types (engine#1282)

**Decision: Three-faceted AgentRole model.** Agent roles carry three independent properties:
(1) hierarchical Path-based name (extensible, domain-defined), (2) structural archetype
(INITIATOR/PEER/CONTRACTOR — closed enum for mechanical enforcement), (3) platform
behavioral contract (COORDINATOR/SUPERVISOR/JUDGE/MODERATOR — closed enum, each with
requiredArchetype()). Not all facets are required — symbolic roles have name only.

**Decision: CONTRACTOR over WORKER/PARTICIPANT/RESPONDENT.** The platform already owns
Worker as a first-class concept hierarchy (Worker → FlowWorker, PlaybookWorker, CamelWorker,
AgentWorkerFunction). Using WORKER for the archetype would conflate coordination structure
with execution identity. PARTICIPANT collides with Participation.PARTICIPANT (compound
completion semantics). CONTRACTOR is the FIPA Contract Net Protocol origin term — same
register as INITIATOR, no platform collision.

**Decision: Coordination overlays the Worker layer.** An agent in a coordination topology
IS an AgentWorker. The coordination vocabulary describes how agents relate structurally,
not what they are. The Worker layer already answers "what."

**Decision: DecisionAuthority is derived, not stored.** Authority level (LOCAL/COORDINATED/CENTRAL)
is a total function of TopologyType × RoleArchetype — 6 valid combinations, 6 invalid ones
that throw. Storing it separately risks staleness.

**Decision: TopologyDescriptor deferred to #1283+.** It's runtime state (mutable agent→role
map), not vocabulary. The vocabulary issue defines enums and the AgentRole record only.

**Decision: GRADUATED removed from TopologyType.** Topology transitions describe *how the
topology changes*, not *what it is*. Transition policy is a separate concern (#1290).

**Surfaced gap: PlatformRole behavioral SPIs.** Only SUPERVISOR has engine-level behavioral
infrastructure (OversightGateService). COORDINATOR, JUDGE, MODERATOR need SPIs. Filed
epic #1296 with children #1297-#1299.

**Surfaced gap: Five-dimension model incomplete.** CommunicationMode (#1291) and
ObservationStrategy (#1292) had no engine types or issues. Now tracked.
