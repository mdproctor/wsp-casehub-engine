# HANDOFF — casehub-engine

## Last Session (2026-11-03)

### Branch `issue-377-distributed-per-agent-coordination` — #380 + #381 COMPLETE

Designed and implemented agency#380 (CoordinationTopology SPI + concrete
implementations) — L/High issue. Full brainstorm→spec→plan→implementation
cycle. Also absorbed and closed agency#381 (LlmSwarmCoordinator refactor).

**Engine changes (1 commit on branch, rebased onto latest main):**
- Renamed SwarmTopologyProvider → TopologyProvider (IntelliJ refactoring, 12 files)
- Extended TopologyType with HIERARCHICAL and GRADUATED

**Agency changes (6 commits):**
- CoordinationTopology SPI (extends TopologyProvider) with 3 methods: configure, onSwarmEvent, onFeedback
- Supporting types: TopologyContext (5-field universal record), CoordinationPlan (descriptors + executor), DecisionOutcomeExecutor, CoordinationFeedback, FeedbackSignal
- 6 concrete topologies, all with full configure() + event handling:
  - CentralisedTopology — 3 loops (team-formation, collective-reasoning, provisioning), refactored from LlmSwarmCoordinator
  - DistributedTopology — N per-agent loops, FULL_AUTONOMY, STIGMERGY
  - HybridTopology — 1 global + N local loops
  - HierarchicalTopology — tree of loops (root/intermediate/leaf scopes)
  - GraduatedTopology — dynamic scope via GraduationState (windowed feedback accumulator)
  - HiveTopology — composes existing orchestration patterns via HivePatternAdapter
- SwarmCoordinatorBuilder.buildTopology() bridges to the new SPI
- 25 new files in agency-core, all tests passing

**Rebase resolved 4 conflicts** with 9 new commits on main (swarm telemetry,
gallery API, org-aware escalation). Conflicts were in RuntimeBeans.java
(SwarmProvisioner producer), StigmergyCoordinator.java (agent counters vs
work state notifier), and AgentWorkState.java (merged both sets of enum values).

## Key decisions (#380)

- TopologyContext: 5 universal fields (caseId, descriptor, agentProvider, loopFactory, invocationPolicy). Topology-specific deps constructor-injected.
- CoordinationPlan: List<DecisionLoopDescriptor> + DecisionOutcomeExecutor. Factory + executor pattern.
- GraduationState: mutable wrapper over immutable DecisionScope. Windowed feedback (80% success → upgrade, 50% failure → downgrade, SCOPE_VIOLATION → hard downgrade).
- No Mockito in agency-core — used TestAgentProviders.noOp() helper and real DecisionLoopFactory instances.

## Next action

**agency#382** — feat: HierarchicalInvocationTracker — LLM budget management.
M/Med complexity. Needs brainstorming. Related to the topology work (budget
tracking is used by topologies) but a distinct concern — new SPI, new tracker,
new budget model.

## Queue state

Branch `issue-377-distributed-per-agent-coordination` is on main (work-end completed).
The .plan queue has 4 remaining issues: #382, #383, #384, #385.
#381 was absorbed into #380 and should be skipped/closed on next advance.

## Artifacts

| Artifact | Path |
|----------|------|
| Spec (#380) | `specs/issue-377-distributed-per-agent-coordination/2026-11-03-coordination-topology-design.md` |
| Decisions (#380) | `specs/issue-377-distributed-per-agent-coordination/decisions-380.md` |
| Plan (#380) | `plans/2026-11-03-coordination-topology.md` |
| Blog | `blog/2026-11-03-mdp01-six-topologies-one-primitive.md` |
