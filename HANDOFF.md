# HANDOFF — casehub-engine

## Last Session (2026-11-04, session 8)

### agency#381 — LlmSwarmCoordinator → CentralisedTopology refactoring (in progress)

Completed Task 4 (wire loops into CentralisedTopology). CentralisedTopology now creates three AgentDecisionLoop instances during configure() via DecisionLoopFactory, with adapter reasoners from Batch 2. onSwarmEvent() feeds observations into the appropriate loop by name. Each loop gets a distinct agentId (descriptor name) for budget isolation.

Plan deviation from written plan: used `Supplier<List<RoutingCandidate>>` instead of `AgentProvider` for the candidate supplier (AgentProvider has no `listCandidates()` method), and caseId sourced from `TopologyContext.caseId()` during `configure()` instead of a constructor parameter.

Also updated SwarmCoordinatorBuilder.buildTopology() to pass the new constructor params (SwarmProvisioningAdvisorImpl + `List::of` candidate supplier) — this will be fully reworked in Task 5.

Completed Batches 1-3 (4 of 5 tasks):
- Added `DecisionOutcome.Provision` variant with null-safety and CentralisedTopology executor case arm
- Fixed `SectionAwareContentSummariser` bug — assembled prompt context was silently discarded
- Created three `ContentSummariser` adapters: `RoutingReasoner`, `AggregationReasoner`, `ProvisioningReasoner`
- Wired three `AgentDecisionLoop` instances into CentralisedTopology with event→observation routing

Branch: `issue-381-llm-swarm-coordinator-centralised-topology` in agency repo (4 commits). Engine repo remains on main.

## Immediate Next Step

Task 5: Eliminate LlmSwarmCoordinator — delete the class (0 callers), remove `build()` from SwarmCoordinatorBuilder (dead code), update `buildTopology()` to properly wire SwarmProvisioningAdvisorImpl with the shared SwarmContext, and update tests.

## Cross-Module

- **Blocked by** — agency#391/#392 (topology test assertions) still blocked on pre-existing compilation errors in agency-core topology classes

## References

| Artifact | Path |
|----------|------|
| Design spec | `specs/issue-381-llm-swarm-coordinator-centralised-topology/2026-11-04-llm-swarm-coordinator-centralised-topology-design.md` |
| Decisions | `specs/issue-381-llm-swarm-coordinator-centralised-topology/decisions.md` |
| Implementation plan | `plans/2026-11-04-llm-swarm-coordinator-centralised-topology.md` |
| Plan | `/Users/mdproctor/claude/casehub/slots/220/.plan` |
