# Resolve Conductor → PlatformRole.SUPERVISOR Overlap — Design Spec

**Issue:** casehubio/engine#1294
**Parent epic:** casehubio/engine#1281 (Swarm Mechanical Enforcement Layer)
**Decisions:** D1–D5 in `decisions.md`
**Date:** 2026-11-07

## Problem

Three approval-gating mechanisms exist in casehub-engine:

| Mechanism | Module | Purpose | Granularity |
|-----------|--------|---------|-------------|
| `ConductorInboxManager` | runtime-core | Evolution pipeline stage gates | Per-stage (research scope, hypothesis, PR review) |
| `OversightGateService` | api (SPI) | Worker action gates | Per-action (PlannedAction approval) |
| `ActionRiskClassifier` | api (SPI) | Determines if gating needed | Classifier — precedes the gate |

`PlatformRole.SUPERVISOR` defines "oversees, approves/rejects, gates progression" as a behavioral contract. The #1282 spec identifies OversightGateService and ActionRiskClassifier as intended consumers. The conductor pattern implements the same behavioral pattern but has no reference to SUPERVISOR. The #1282 spec explicitly deferred this reconciliation to a downstream issue — this one.

## Design Overview

Formalise the relationship between the conductor pattern and PlatformRole.SUPERVISOR without merging or replacing the mechanisms (D1). Add a `SupervisorGateBehavior` marker interface (D2, D5) in `api/model/coordination` (D3) that both `ConductorInboxManager` and `OversightGateService` implement. `ActionRiskClassifier` is documented as the classification step that precedes the gate, not a gate itself (D4).

```
PlatformRole.SUPERVISOR
    "oversees, approves/rejects, gates progression"
          │
          ▼
SupervisorGateBehavior (marker interface)
          │
    ┌─────┴──────────────────┐
    │                        │
    ▼                        ▼
OversightGateService    ConductorInboxManager
(worker-action gates)   (evolution-pipeline gates)
    │                        │
    │                        │
    ▼                        ▼
ActionRiskClassifier    EscalationProvider
(classifier — decides   (evaluator — decides
 IF gating needed)       IF escalation needed)
```

**Layered model:**
- **Vocabulary:** `PlatformRole.SUPERVISOR` — the behavioral contract
- **Type marker:** `SupervisorGateBehavior` — classifies implementations
- **Generic gate:** `OversightGateService` — worker-action level
- **Domain gate:** `ConductorInboxManager` — evolution-domain level
- **Classifiers:** `ActionRiskClassifier`, `EscalationProvider` — triage steps that precede gates

## 1. SupervisorGateBehavior Marker Interface

```java
// api, io.casehub.api.model.coordination
package io.casehub.api.model.coordination;

/**
 * Marker for components that implement {@link PlatformRole#SUPERVISOR} gating
 * behavior — the ability to pause execution, wait for approval, and resume
 * with a decision.
 *
 * <p>Implementors operate at different granularities:
 * <ul>
 *   <li>{@link io.casehub.api.spi.OversightGateService} — worker-action
 *       level, gates individual {@code PlannedAction}s
 *   <li>{@code ConductorInboxManager} — evolution-pipeline level, gates
 *       research pipeline stages
 * </ul>
 *
 * <p>Classification components that decide <em>whether</em> gating is needed
 * (such as {@link io.casehub.api.spi.ActionRiskClassifier}) are not gate
 * implementations and do not carry this marker. They are the triage step
 * that precedes the gate.
 *
 * @see PlatformRole#SUPERVISOR
 */
public interface SupervisorGateBehavior {}
```

No methods. The marker is a type-level classification for CDI discoverability and documentation.

## 2. OversightGateService Changes

```java
// api, io.casehub.api.spi
package io.casehub.api.spi;

import io.casehub.api.model.coordination.SupervisorGateBehavior;

/**
 * Engine SPI for approval-gating agent actions. Implements
 * {@link SupervisorGateBehavior} — the {@link
 * io.casehub.api.model.coordination.PlatformRole#SUPERVISOR} gating
 * contract at the worker-action level.
 *
 * <p>{@link io.casehub.api.spi.ActionRiskClassifier} determines whether
 * a {@link io.casehub.worker.api.PlannedAction} requires gating. When
 * gating is required, this service manages the gate lifecycle.
 */
public interface OversightGateService extends SupervisorGateBehavior {
  GateOutcome openGate(String agentId, String commitmentId, String outcome, String tenancyId);

  void fulfill(UUID gateId, String rawOutput);
}
```

`NoOpOversightGateService` inherits `SupervisorGateBehavior` transitively — no change needed.

## 3. ConductorInboxManager Changes

```java
// runtime-core, io.casehub.engine.runtime.improvement
package io.casehub.engine.runtime.improvement;

import io.casehub.api.model.coordination.SupervisorGateBehavior;

/**
 * Manages the evolution conductor's approval inbox — the {@link
 * SupervisorGateBehavior} implementation for evolution-pipeline gates.
 *
 * <p>Operates at the pipeline-stage level (research scope, hypothesis
 * approval, PR review) with evolution-specific context: escalation
 * triggers, watch patterns, confidence scores, categories, and area IDs.
 *
 * <p>Complementary to {@link io.casehub.api.spi.OversightGateService},
 * which provides {@link SupervisorGateBehavior} at the worker-action
 * level for individual {@code PlannedAction}s.
 *
 * @see io.casehub.api.model.coordination.PlatformRole#SUPERVISOR
 */
public class ConductorInboxManager implements SupervisorGateBehavior {
  // ... existing implementation unchanged
}
```

## 4. ActionRiskClassifier Javadoc Update

```java
// api, io.casehub.api.spi
/**
 * Classifies a worker's planned action to determine whether it may
 * proceed autonomously or must be gated for human approval.
 *
 * <p>This is the classification step in the {@link
 * io.casehub.api.model.coordination.PlatformRole#SUPERVISOR} pipeline
 * — it decides <em>whether</em> gating is needed. The gate itself is
 * managed by {@link OversightGateService}, which implements {@link
 * io.casehub.api.model.coordination.SupervisorGateBehavior}.
 *
 * <p>Implementations must be annotated {@link RiskClassifier} and
 * {@code @ApplicationScoped}. The engine chains all registered
 * implementations — the most restrictive result wins.
 *
 * <p>If {@code classify} throws, the engine applies a fail-safe
 * {@link RiskDecision.GateRequired} requiring manual review.
 */
public interface ActionRiskClassifier {
  RiskDecision classify(PlannedAction action, ClassificationContext context);
}
```

## 5. PlatformRole Javadoc Update

```java
// api, io.casehub.api.model.coordination
/**
 * Intrinsic behavioral contracts — the platform implements specific
 * behavior for these roles.
 */
public enum PlatformRole {
  COORDINATOR(StructuralRole.INITIATOR),

  /**
   * Oversees, approves/rejects, gates progression. Implemented at two
   * granularities via {@link SupervisorGateBehavior}:
   * <ul>
   *   <li>{@link io.casehub.api.spi.OversightGateService} — worker-action
   *       gates for {@code PlannedAction} approval
   *   <li>{@code ConductorInboxManager} — evolution-pipeline gates for
   *       research stage approval
   * </ul>
   * Classification (deciding <em>whether</em> gating is needed) is
   * handled by {@link io.casehub.api.spi.ActionRiskClassifier}, which
   * precedes the gate.
   */
  SUPERVISOR(StructuralRole.INITIATOR),

  JUDGE(StructuralRole.PEER),
  MODERATOR(StructuralRole.PEER);

  // ... existing implementation unchanged
}
```

## 6. Test Strategy

### New tests

| Test | What it covers |
|------|---------------|
| `SupervisorGateBehaviorTest` | `OversightGateService` extends `SupervisorGateBehavior`; `NoOpOversightGateService` implements it transitively; `ConductorInboxManager` implements it; `ActionRiskClassifier` does NOT extend it |
| `SupervisorGateDiscoveryTest` | CDI `Instance<SupervisorGateBehavior>` discovers both `NoOpOversightGateService` and `ConductorInboxManager` in a test context |

### Modified tests

None — no behavioral changes, only type hierarchy additions.

## 7. Module Placement

| Component | Module | Package |
|-----------|--------|---------|
| `SupervisorGateBehavior` | `api` | `io.casehub.api.model.coordination` |
| `OversightGateService` (modified) | `api` | `io.casehub.api.spi` |
| `ConductorInboxManager` (modified) | `runtime-core` | `io.casehub.engine.runtime.improvement` |
| `ActionRiskClassifier` (Javadoc only) | `api` | `io.casehub.api.spi` |
| `PlatformRole` (Javadoc only) | `api` | `io.casehub.api.model.coordination` |

## 8. Scope Exclusions

- **No behavioral changes.** All existing gate lifecycle logic stays as-is.
- **No API surface changes.** No new methods on any existing class.
- **No ActionRiskClassifier refactoring.** The classifier stays separate from the gate.
- **No conductor migration to PlatformRole dispatch.** The conductor keeps its own dispatch model. The marker is a classification, not a dispatch mechanism.

## References

- `PlatformRole.java:18-33` — SUPERVISOR enum constant
- `OversightGateService.java:20-24` — generic gate SPI
- `ConductorInboxManager.java:27-90` — evolution-specific gate
- `ActionRiskClassifier.java:34-37` — risk classifier SPI
- `NoOpOversightGateService.java:24` — default gate implementation
- `AgentRole.java:29` — SUPERVISOR constant with PlatformRole.SUPERVISOR
- Issue casehubio/engine#1282 — coordination vocabulary spec, §PlatformRole §Relationship to conductor infrastructure
- Issue casehubio/engine#1148 — generalise evolution conductor (deferred the reconciliation)
- Issue casehubio/engine#1288 — agent behavioral SPI (filed as target for reconciliation)
- Decisions D1–D5 in `decisions.md`
