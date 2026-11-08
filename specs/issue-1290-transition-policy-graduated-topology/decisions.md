# Decisions — TransitionPolicy (#1290)

## D1: Vocabulary types + SPI (both)

**Choice:** Both vocabulary types in `model.coordination` and SPI extension on `TopologyProvider`
**Alternatives:**
- Vocabulary types only — inert data with no engine evaluation path; agency must interpret
- Engine SPI only — no declarative model for transition rules; all logic in implementations
**Rationale:** TopologyProvider SPI already exists and `onTopologyChanged` hook is in place. Vocabulary types define the policy declaratively; the SPI gives engine a way to evaluate transition readiness. The missing piece between "what is the topology" and "should the topology change."
**Trade-offs:** Slightly more surface area than vocabulary-only, but both halves are small and natural extensions of existing infrastructure
**Sources:** TopologyProvider.java (api/spi/coordination), AgentWorkLifecycleListener.java (onTopologyChanged hook), #1282 SPEC.md Resolved Question 3
**Exploration:** quick
**Status:** captured

## D2: Extend TopologyProvider rather than new SPI

**Choice:** Add default methods to `TopologyProvider` for transition evaluation
**Alternatives:**
- New `TransitionEvaluator` SPI — cleaner separation but adds another interface to wire
- Callback on agency's `CoordinationTopology` — keeps engine-api lightweight but wrong layer for engine-level concern
**Rationale:** All topology concerns in one SPI. Implementors override default methods when they support transitions. CentralisedTopologyProvider (the default) returns empty optionals. CoordinationTopology in agency can override with swarm intelligence.
**Trade-offs:** Mixes "what is the topology" with "should the topology change" on one interface — acceptable because these are closely related concerns and both are per-agent/per-tenant scoped
**Sources:** TopologyProvider.java (2 methods today), CentralisedTopologyProvider.java, CoordinationTopology.java (agency)
**Exploration:** quick
**Status:** captured

## D3: Sealed interface for transition triggers

**Choice:** `TransitionTrigger` as a sealed interface with typed record variants (`ThresholdTrigger`, `TimeTrigger`, `ConvergenceTrigger`)
**Alternatives:**
- Closed enum (`THRESHOLD`, `TIME_BASED`, `CONVERGENCE`) — simple but no typed parameters; trigger configuration becomes unstructured
- Predicate-based functional interface — maximum flexibility but no introspectable structure for logging/auditing
**Rationale:** Each trigger type carries its own parameters (metric name + threshold, duration, convergence score). Sealed interface gives exhaustive pattern matching in engine while allowing new variants via `permits`. Bridges naturally to BudgetConfig for convergence thresholds.
**Trade-offs:** Heavier than an enum; worth it because trigger parameters are inherently typed
**Sources:** BudgetConfig.java (convergence thresholds), issue #1290 body (trigger condition examples)
**Exploration:** quick
**Status:** captured

## D4: Strategy and reversibility as separate concerns

**Choice:** `TransitionStrategy` enum (`IMMEDIATE`, `PHASED`) + `boolean reversible` on `TransitionPolicy` + optional `rollbackTrigger`
**Alternatives:**
- Single enum with all variants (`IMMEDIATE`, `PHASED`, `REVERSIBLE_IMMEDIATE`, `REVERSIBLE_PHASED`) — combinatorial explosion
- Strategy as sealed interface with typed parameters — more expressive but heavier than needed at this stage
**Rationale:** "Reversible" is a property of the transition (can it be undone?), not a strategy for how it executes. Orthogonal dimensions stay orthogonal. Rollback trigger reuses the same `TransitionTrigger` sealed interface.
**Trade-offs:** Phased strategy details (phase count, duration per phase) are deferred to TopologyProvider implementations — the vocabulary doesn't prescribe phasing mechanics
**Sources:** Issue #1290 body (transition strategy and rollback criteria as separate concepts)
**Exploration:** quick
**Status:** captured

## D5: Remove GRADUATED from TopologyType

**Choice:** Remove `GRADUATED` enum value and its 3 test methods from `TopologyTypeTest`
**Alternatives:**
- Keep and `@Deprecated` — adds noise; no production consumers to migrate
**Rationale:** #1282 spec Resolved Question 3 explicitly says "GRADUATED is a transition policy, not a topology value. Removed from TopologyType." It was never removed. GRADUATED has identical `deriveAuthority()` logic to HYBRID — no unique mechanical behavior. Only 3 test references, no production consumers. TransitionPolicy is the canonical replacement.
**Trade-offs:** None — no production code references GRADUATED
**Sources:** TopologyType.java (GRADUATED identical to HYBRID), TopologyTypeTest.java (3 tests), #1282 SPEC.md Resolved Question 3
**Exploration:** quick
**Status:** captured
