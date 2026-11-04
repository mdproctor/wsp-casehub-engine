# HANDOFF — casehub-engine

## Last Session (2026-11-04, session 8)

### agency#381 — LlmSwarmCoordinator → CentralisedTopology refactoring (complete)

Completed all 5 tasks across 4 batches. The LlmSwarmCoordinator class has been eliminated — CentralisedTopology is now the single coordination path.

**Batch 1 (Foundation):**
- Added `DecisionOutcome.Provision` variant with null-safety and CentralisedTopology executor case arm
- Fixed `SectionAwareContentSummariser` bug — assembled prompt context was silently discarded

**Batch 2 (Adapters):**
- Created three `ContentSummariser` adapters: `RoutingReasoner`, `AggregationReasoner`, `ProvisioningReasoner`

**Batch 3 (Wiring):**
- Wired three `AgentDecisionLoop` instances into CentralisedTopology with event→observation routing
- Constructor gained `SwarmProvisioningAdvisor` + `Supplier<List<RoutingCandidate>>` (plan deviation: `AgentProvider.listCandidates()` doesn't exist)
- Each loop gets a distinct agentId (descriptor name) for budget isolation

**Batch 4 (Cleanup):**
- Deleted `LlmSwarmCoordinator` class (0 production callers)
- Removed `build()` from `SwarmCoordinatorBuilder` (dead code) — `buildTopology()` is the single path
- Deleted `LlmSwarmCoordinatorTest`, removed 4 `build()`-related tests from `SwarmCoordinatorBuilderTest`
- Migrated `SwarmPromptCustomiserIntegrationTest` to use `buildTopology()`
- Updated CLAUDE.md module index

Branch: `issue-381-llm-swarm-coordinator-centralised-topology` in agency repo (5 commits). Engine repo remains on main.

All 1896 agency module tests pass.

## Immediate Next Step

Issue #381 is functionally complete. Ready for `work end` to close the branch, or proceed to the next issue in the queue (#382 — HierarchicalInvocationTracker budget).

## Cross-Module

- **Blocked by** — agency#391/#392 (topology test assertions) still blocked on pre-existing compilation errors in agency-core topology classes

## References

| Artifact | Path |
|----------|------|
| Design spec | `specs/issue-381-llm-swarm-coordinator-centralised-topology/2026-11-04-llm-swarm-coordinator-centralised-topology-design.md` |
| Decisions | `specs/issue-381-llm-swarm-coordinator-centralised-topology/decisions.md` |
| Implementation plan | `plans/2026-11-04-llm-swarm-coordinator-centralised-topology.md` |
| Plan | `/Users/mdproctor/claude/casehub/slots/220/.plan` |
