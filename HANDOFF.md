# Handoff — issue-377-distributed-per-agent-coordination

## This session

Completed engine#1284 (SwarmTopologyProvider), engine#1291 (CommunicationMode),
engine#1292 (ObservationStrategy). Also closed engine#1282, #1283, #1289, #1293
as already implemented by prior session work.

Key design: TopologyDescriptor is the single source of truth for authority.
`authorityFor()` lives on the descriptor (derivation from own data), not on the
SPI (consistency hazard). SwarmTopologyProvider has one method: `describeTopology()`.
SwarmProvisioner gates on topology type — DISTRIBUTED skips central provisioning.

Wrote blog entry: "The Single Source of Truth Problem in Multi-Agent Authority"
with two SVG diagrams.

## Downstream context — eidos#216

AgentMailbox will be implemented as specialised private channels in Qhorus (not a
standalone abstraction). Hive topology support is being built in a parallel session.
All engine types designed with these directions in mind — TopologyDescriptor is
snapshot-based (composes with mutable group topology), agent identity is String
(composes with group coordinator as AgentRef).

## Key decisions

- **authorityFor on TopologyDescriptor, not SPI:** Single source of truth eliminates consistency hazard where descriptor and authority function disagree. Pre-release — can add SPI override later if per-decision-type authority models emerge.
- **No DecisionType on authorityFor:** Doesn't participate in computation for existing topologies. YAGNI.
- **TopologyDescriptor is immutable snapshot:** Map.copyOf in constructor, fail-fast validation of role/topology coherence. No version/epoch — YAGNI.
- **CentralisedTopologyProvider default:** Returns CENTRALISED + empty role map. Backward compatible.
- **SwarmProvisioner topology gate:** DISTRIBUTED → skip. CENTRALISED/HYBRID → proceed. Same gate on deprovisioning.

## Closed this session

| Issue | Title | How |
|-------|-------|-----|
| engine#1284 | SwarmTopologyProvider | 3 commits: TopologyDescriptor, SPI+default, provisioner gate |
| engine#1291 | CommunicationMode | Vocabulary enum |
| engine#1292 | ObservationStrategy | Vocabulary enum |
| engine#1282 | Vocabulary types | Already implemented (prior session) |
| engine#1283 | AgentCapabilityGuard | Already implemented (prior session) |
| engine#1289 | TopologyDescriptor (standalone) | Subsumed by #1284 |
| engine#1293 | DecisionAuthority derivation | Subsumed by #1282 |

## Next action

Engine#1285 (AgentWorkLifecycleListener + role lifecycle events) — M/Med.
Needs brainstorming. Key input from eidos#216: role transitions (adopt/release)
must be first-class lifecycle events, not just state changes. A listener should
observe "agent X adopted role Coordinator in group Y."

Read the updated spec at `specs/issue-1282-coordination-vocabulary-types/SPEC.md`
for the full coordination model. The five-dimension vocabulary is now complete
(topology, scope, communication, observation, budget).

## Queue state

Position 1/9 in .plan. agency#378 (sub-epic) is the active queue item but
the next implementable issue is engine#1285.

## References

| Artifact | Path |
|----------|------|
| Spec (#1284) | `specs/issue-377-distributed-per-agent-coordination/2026-11-02-swarm-topology-provider-design.md` |
| Decisions (#1284) | `specs/issue-377-distributed-per-agent-coordination/decisions.md` |
| Plan (#1284) | `plans/2026-11-02-swarm-topology-provider.md` |
| Blog (vocabulary) | `blog/2026-11-01-mdp01-the-vocabulary-that-shapes-everything.md` |
| Blog (enforcement) | `blog/2026-11-02-mdp01-from-vocabulary-to-enforcement.md` |
| Blog (topology) | `blog/2026-11-02-mdp02-topology-aware-engine.md` |
