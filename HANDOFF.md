# HANDOFF — casehub-engine

## Last Session (2026-11-04, session 7)

### agency#381 — LlmSwarmCoordinator → CentralisedTopology refactoring (in progress)

Designed and partially implemented the decomposition of LlmSwarmCoordinator into CentralisedTopology. Full brainstorming with 7 validated decisions (standard decision review, 3 rounds). Key design change from original plan: provisioning maps to a new `DecisionOutcome.Provision` sealed variant (not WriteBack) — the decision review proved loop-identity dispatching is architecturally impossible since the executor receives no loop name.

Completed Batches 1-2 (3 of 5 tasks):
- Added `DecisionOutcome.Provision` variant with null-safety and CentralisedTopology executor case arm
- Fixed `SectionAwareContentSummariser` bug — assembled prompt context was silently discarded
- Created three `ContentSummariser` adapters: `RoutingReasoner`, `AggregationReasoner`, `ProvisioningReasoner`

Branch: `issue-381-llm-swarm-coordinator-centralised-topology` in agency repo (3 commits). Engine repo remains on main.

## Immediate Next Step

Task 4: Wire loops into CentralisedTopology — update constructor to accept `SwarmProvisioningAdvisor` + `AgentProvider` + `UUID caseId`, create three `AgentDecisionLoop` instances in `configure()` using `TopologyContext.loopFactory()`, add `loops()` accessor, wire event→observation routing in `onSwarmEvent()`. Then Task 5: eliminate `LlmSwarmCoordinator` and remove `build()` from `SwarmCoordinatorBuilder`.

## Cross-Module

- **Blocked by** — agency#391/#392 (topology test assertions) still blocked on pre-existing compilation errors in agency-core topology classes

## References

| Artifact | Path |
|----------|------|
| Design spec | `specs/issue-381-llm-swarm-coordinator-centralised-topology/2026-11-04-llm-swarm-coordinator-centralised-topology-design.md` |
| Decisions | `specs/issue-381-llm-swarm-coordinator-centralised-topology/decisions.md` |
| Implementation plan | `plans/2026-11-04-llm-swarm-coordinator-centralised-topology.md` |
| Plan | `/Users/mdproctor/claude/casehub/slots/220/.plan` |
