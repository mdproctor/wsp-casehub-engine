# Handoff — issue-377-distributed-per-agent-coordination

## This session

Designed and implemented agency#379 (AgentDecisionLoop) — the unified
observe→reason→act primitive for all LLM coordination decisions. Full
brainstorm→spec→review→plan→implementation cycle.

16 new files in `agency-core/src/main/java/io/casehub/agency/orchestration/loop/`,
57 tests passing. Key types: DecisionLoopDescriptor (playbook), AgentDecisionLoop
(runtime), DecisionLoopFactory (assembly), DecisionOutcome (sealed action type),
DecisionContext (per-agent coordination view), AgentBudgetTracker (per-agent +
shared cap), PromptSectionRegistry, SectionAwareContentSummariser.

Design review (light) caught 5 bugs before implementation: budget never acquired,
wildcard emission broken, context name collision, prompt sections dead weight,
flush silent on budget exhaustion. All fixed in spec before code was written.

## Key decisions

- **Playbook model:** DecisionLoopDescriptor (data record) + AgentDecisionLoop (runtime). YAML-aware design for #385 parity — Java types are shaped for YAML serialisation but YAML parsing itself is deferred.
- **ContentSummariser for LLM:** The loop's reasoning phase IS summarisation. Composes with existing InFlightGate, EmissionPolicy, SummarisationRunner pipeline.
- **DecisionOutcome sealed interface:** Bridges EnvironmentWriter write-back and RoutingDecision/AggregationResult return-value semantics. Variants: WriteBack, Route, Aggregate, Composite, ScopeGated.
- **DecisionContext (not ObservationContext):** Renamed to avoid collision with two existing ObservationContext types. Excludes AgentWorkState (dynamic, shouldn't freeze in immutable record).
- **Per-agent budget with shared cap:** AgentBudgetTracker wraps per-agent limits with optional SwarmInvocationTracker delegation.
- **LoopPromptSection:** New functional interface taking DecisionContext, not SwarmContext. Topology layer adapts existing SwarmPromptSection instances.
- **Scope enforcement in the loop:** DecisionScope.isPermitted() checked in the ContentSummariser, not in the pipeline. Agency-side visibility into blocked actions.
- **Loop produces, doesn't execute:** DecisionOutcome is returned to the caller. Topology layer (#380/#381) owns execution.

## Closed this session

| Issue | Title | How |
|-------|-------|-----|
| agency#379 | AgentDecisionLoop | 5 commits: foundation types, DecisionOutcome, AgentBudgetTracker, registry+sections, factory+loop |

## Next action

Agency#380 (CoordinationTopology SPI + concrete implementations) — L/High.
This builds the topology layer that USES AgentDecisionLoop. Centralised,
distributed, hybrid, hierarchical, graduated topologies. Each topology
creates loops with different descriptors (observation strategy, scope,
communication mode) and executes DecisionOutcomes differently.

Needs brainstorming. Key input: the playbook model means each topology
is primarily a factory for DecisionLoopDescriptors + a DecisionOutcome
executor.

## Queue state

Position 3/10 in .plan. agency#379 is the active queue item (just
implemented). Next implementable: agency#380.

## References

| Artifact | Path |
|----------|------|
| Spec (#379) | `specs/issue-377-distributed-per-agent-coordination/2026-11-02-agent-decision-loop-design.md` |
| Decisions (#379) | `specs/issue-377-distributed-per-agent-coordination/decisions-379.md` |
| Plan (#379) | `plans/2026-11-02-agent-decision-loop.md` |
| Spec (#1284) | `specs/issue-377-distributed-per-agent-coordination/2026-11-02-swarm-topology-provider-design.md` |
| Spec (#1285) | `specs/issue-377-distributed-per-agent-coordination/2026-11-02-agent-work-lifecycle-design.md` |
| Blog (vocabulary) | `blog/2026-11-01-mdp01-the-vocabulary-that-shapes-everything.md` |
| Blog (enforcement) | `blog/2026-11-02-mdp01-from-vocabulary-to-enforcement.md` |
| Blog (topology) | `blog/2026-11-02-mdp02-topology-aware-engine.md` |
