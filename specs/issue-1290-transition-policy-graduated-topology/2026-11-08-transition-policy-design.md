# TransitionPolicy — Graduated Topology Transition Model

**Issue:** casehubio/engine#1290
**Parent epic:** casehubio/engine#1281
**Origin:** Deferred from engine#1282 design review — Structure R1-06, Robustness R1-03

## Summary

Model for graduated topology transitions — when and how a topology transitions between modes (e.g., CENTRALISED → DISTRIBUTED). GRADUATED was removed from TopologyType during the #1282 design review because it describes *how the topology changes*, not *what the topology is*. This issue defines the transition rules as vocabulary types plus an SPI extension on TopologyProvider.

## Design Principles

1. **Transition is orthogonal to topology.** TopologyType says *what* the topology is. TransitionPolicy says *when and how* it changes. These are separate concerns — mixing them (as GRADUATED did) conflates structure with lifecycle.

2. **Vocabulary types + SPI.** Records and enums in `model.coordination` define the policy declaratively. Default methods on `TopologyProvider` give the engine a way to evaluate transition readiness. Neither half is useful alone.

3. **Typed triggers over opaque predicates.** Each trigger type carries its own parameters (metric + threshold, duration, convergence score). Sealed interface gives exhaustive pattern matching in engine while remaining extensible via new `permits`.

4. **Strategy and reversibility are orthogonal.** IMMEDIATE vs. PHASED describes *how* the transition executes. Reversible describes *whether* it can be undone. These are independent properties, not a cross-product enum.

## Package

`io.casehub.api.model.coordination` — same package as TopologyType, TopologyDescriptor, AgentRole.

SPI extension on `io.casehub.api.spi.coordination.TopologyProvider`.

## Type Definitions

### TransitionTrigger (sealed interface)

Defines when to initiate a transition. Each variant carries typed parameters.

```java
public sealed interface TransitionTrigger
    permits TransitionTrigger.Threshold,
            TransitionTrigger.Time,
            TransitionTrigger.Convergence {

  record Threshold(String metric, double value) implements TransitionTrigger {
    public Threshold {
      Objects.requireNonNull(metric, "metric must not be null");
      if (metric.isBlank())
        throw new IllegalArgumentException("metric must not be blank");
    }
  }

  record Time(Duration duration) implements TransitionTrigger {
    public Time {
      Objects.requireNonNull(duration, "duration must not be null");
      if (duration.isNegative() || duration.isZero())
        throw new IllegalArgumentException("duration must be positive");
    }
  }

  record Convergence(double threshold) implements TransitionTrigger {
    public Convergence {
      if (threshold < 0.0 || threshold > 1.0)
        throw new IllegalArgumentException("threshold must be in [0.0, 1.0]");
    }
  }
}
```

**Threshold** — fires when a named metric reaches or exceeds the value. Metric names are application-defined (e.g., "swarm.size", "coordination.efficiency"). The engine does not prescribe the metric namespace. The direction is always "at or above" — for "at or below" triggers, the TopologyProvider implementation inverts the metric (e.g., "coordination.inefficiency" rather than a directional flag on the trigger).

**Time** — fires after a duration elapses from the topology's activation. The engine tracks activation time; the TopologyProvider evaluates against it.

**Convergence** — fires when convergence reaches a level. Bridges to `BudgetConfig` convergence thresholds — both express "how converged is the swarm?" but BudgetConfig caps budget consumption while ConvergenceTrigger triggers structural change. The threshold is normalised to [0.0, 1.0].

### TransitionStrategy (enum)

How the transition executes.

| Value | Semantics |
|-------|-----------|
| `IMMEDIATE` | TopologyType changes in one step. All agents see the new topology on their next observation cycle. |
| `PHASED` | TopologyType changes gradually. The TopologyProvider implementation owns phasing mechanics (phase count, duration per phase, agent migration order). |

PHASED details are intentionally deferred to TopologyProvider implementations. The vocabulary defines the intent ("this transition should be gradual"); the SPI implementation defines the mechanics.

### TransitionPolicy (record)

Composition of all transition concerns.

```java
public record TransitionPolicy(
    TopologyType from,
    TopologyType to,
    TransitionTrigger trigger,
    TransitionStrategy strategy,
    boolean reversible,
    @Nullable TransitionTrigger rollbackTrigger) {

  public TransitionPolicy {
    Objects.requireNonNull(from, "from must not be null");
    Objects.requireNonNull(to, "to must not be null");
    Objects.requireNonNull(trigger, "trigger must not be null");
    Objects.requireNonNull(strategy, "strategy must not be null");
    if (from == to)
      throw new IllegalArgumentException(
          "from and to must differ — a transition to the same topology is a no-op");
    if (!reversible && rollbackTrigger != null)
      throw new IllegalArgumentException(
          "rollbackTrigger requires reversible=true");
  }
}
```

**Compact constructor invariants:**

1. **from != to:** A transition to the same topology is a no-op. This catches configuration errors.
2. **rollbackTrigger requires reversible:** If the transition cannot be reversed, specifying a rollback trigger is incoherent.
3. **rollbackTrigger is nullable even when reversible:** A reversible transition without a rollbackTrigger means rollback is manually initiated (e.g., by the CoordinationTopology responding to degradation events), not automatically triggered.

## SPI Extension

Two default methods added to `TopologyProvider`:

```java
default Optional<TransitionPolicy> transitionPolicy(String agentId, String tenancyId) {
    return Optional.empty();
}

default Optional<TopologyType> evaluateTransition(String agentId, String tenancyId) {
    return Optional.empty();
}
```

**transitionPolicy** — returns the configured transition policy for the given agent's topology, if any. Informational — consumers can inspect what transition rules are in effect.

**evaluateTransition** — asks the provider whether a transition should fire now. Returns the target TopologyType if the trigger condition is met; empty otherwise. This is the active evaluation path.

**CentralisedTopologyProvider** — unchanged. Default methods return `Optional.empty()`, which is correct — the default provider does not support transitions.

**Integration with onTopologyChanged:** When `evaluateTransition` returns a target, the engine runtime fires `onTopologyChanged(unitId, fromTopology, toTopology, tenancyId)` on all `AgentWorkLifecycleListener` instances. The existing hook is the notification mechanism; this SPI extension is the decision mechanism.

**Agency integration:** `CoordinationTopology` (which extends `TopologyProvider`) can override both methods. `transitionPolicy` returns the configured policy. `evaluateTransition` applies swarm intelligence (convergence scores, swarm size, degradation signals) against the policy's trigger conditions. The `onSwarmEvent`, `onFeedback`, and `onDegradation` hooks on `CoordinationTopology` provide the state that informs transition evaluation.

## GRADUATED Cleanup

Remove `GRADUATED` from `TopologyType`:

- Delete the enum constant and its `deriveAuthority()` override
- GRADUATED has identical logic to HYBRID (accepts all StructuralRoles) — no unique mechanical behavior
- Only referenced in 3 test methods in `TopologyTypeTest` — no production consumers
- TransitionPolicy is the canonical replacement for the concept GRADUATED represented

## Test Coverage

- **TransitionTrigger variants:** Threshold with blank metric rejected, Time with zero/negative duration rejected, Convergence outside [0.0, 1.0] rejected. Valid construction for each variant.
- **TransitionPolicy invariants:** from == to rejected, rollbackTrigger without reversible rejected. Valid construction with each trigger type × strategy combination. Reversible with and without rollbackTrigger.
- **TopologyProvider defaults:** Default `transitionPolicy` returns empty. Default `evaluateTransition` returns empty.
- **GRADUATED removal:** TopologyTypeTest updated — 3 GRADUATED tests removed.

## What This Issue Does NOT Include

- **Transition execution lifecycle** — who calls `evaluateTransition` and when (polling interval, event-driven). Owned by the engine runtime or agency's CoordinationTopology.
- **Phased transition mechanics** — phase count, duration per phase, agent migration order. Owned by TopologyProvider implementations.
- **Metric registry** — the namespace and semantics of threshold metric names. Application-defined.
- **YAML surface** — DSL representation of TransitionPolicy. Deferred to schema/codegen work.
- **Persistence** — how transition state (current phase, elapsed time) is stored. Runtime concern.

## References

- `api/src/main/java/io/casehub/api/model/coordination/TopologyType.java` — GRADUATED enum constant to remove
- `api/src/test/java/io/casehub/api/model/coordination/TopologyTypeTest.java` — 3 GRADUATED tests to remove
- `api/src/main/java/io/casehub/api/spi/coordination/TopologyProvider.java` — SPI to extend
- `api/src/main/java/io/casehub/api/model/convergence/BudgetConfig.java` — related convergence thresholds
- `api/src/main/java/io/casehub/api/spi/stigmergy/AgentWorkLifecycleListener.java:42` — onTopologyChanged hook
- `agency-core: CoordinationTopology.java` — agency's TopologyProvider extension, integration point
- `specs/issue-1282-coordination-vocabulary-types/SPEC.md` — Resolved Question 3 (GRADUATED removal)
- casehubio/engine#1282 — coordination vocabulary types (foundation)
- casehubio/engine#1284 — SwarmTopologyProvider (transition execution owner)
