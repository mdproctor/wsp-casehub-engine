# DetectedRole → AgentRole Bridging SPI — Design Spec

**Issue:** casehubio/engine#1295
**Parent epic:** casehubio/engine#1281 (Swarm Mechanical Enforcement Layer)
**Decisions:** D1–D8 in `decisions.md`
**Date:** 2026-11-07

## Problem

The engine detects emergent behavioral roles via `RoleTracker` (behavioral fingerprint clustering) and emits `DetectedRole` records. The coordination vocabulary defines declared roles via `AgentRole` (name + structural role + platform role). Nothing connects these two layers — a `DetectedRole` whose dominant features match coordinator behavior has no path to influence the `TopologyDescriptor.agentRoles` map.

The #1282 spec identified this gap: "Whether detected patterns can influence declared roles (e.g., auto-promoting an agent whose behavior matches a coordinator pattern) is a behavioral contract for downstream SPIs." This is that SPI.

## Design Overview

A new `RoleBridgingStrategy` SPI in engine-api maps `DetectedRole` → `RoleSuggestion`. A user-configurable `RoleBridgingPolicy` in agency controls what happens with suggestions (advisory, conditional auto-apply, or automatic). The topology layer in agency orchestrates the flow, applying policy and resolving conflicts.

```
Engine (detection)                    Agency (topology)
┌──────────────┐                      ┌──────────────────────────┐
│ RoleTracker   │──detect──▶          │                          │
│  (clustering) │          │          │  CoordinationTopology    │
└──────────────┘          │          │  (HiveTopology, ...)     │
                          │          │                          │
┌──────────────┐          │          │  ┌────────────────────┐  │
│SwarmProvisioner│─notify──▶ ─bridge──▶ │ onSwarmEvent()     │  │
│  (listeners)  │                     │ │  ┌─────────────┐   │  │
└──────────────┘                      │ │  │ RoleBridging│   │  │
                                      │ │  │ Strategy    │   │  │
Engine-API (shared SPI)               │ │  │ (map roles) │   │  │
┌──────────────────┐                  │ │  └──────┬──────┘   │  │
│RoleBridgingStrategy│◀──impl─────────│ │         │          │  │
│  (DetectedRole →  │                 │ │  ┌──────▼──────┐   │  │
│   RoleSuggestion) │                 │ │  │ RoleBridging│   │  │
└──────────────────┘                  │ │  │ Policy      │   │  │
                                      │ │  │ (apply?)    │   │  │
                                      │ │  └──────┬──────┘   │  │
                                      │ │         │          │  │
                                      │ │  ┌──────▼──────┐   │  │
                                      │ │  │ Conflict    │   │  │
                                      │ │  │ Resolution  │   │  │
                                      │ │  └─────────────┘   │  │
                                      │ └────────────────────┘  │
                                      └──────────────────────────┘
```

**Layered responsibilities:**
- **Engine:** detects roles, fires events via `SwarmCoordinationListener`
- **Engine-API:** defines shared SPI types (`RoleBridgingStrategy`, `RoleSuggestion`)
- **Agency:** owns the policy (`RoleBridgingPolicy`), orchestration, conflict resolution

## 1. New Types in engine-api

### 1.1 RoleSuggestion

Intermediate type returned by the strategy. Carries the strategy's assessment without constructing a full `AgentRole` — the topology applies structural constraints before creating roles.

```java
// api, io.casehub.api.model.coordination
package io.casehub.api.model.coordination;

import io.casehub.api.model.stigmergy.DetectedRole;
import org.jspecify.annotations.Nullable;

/**
 * A suggestion from a {@link io.casehub.api.spi.coordination.RoleBridgingStrategy} mapping a
 * {@link DetectedRole} to a proposed {@link AgentRole}. The topology layer creates the
 * {@code AgentRole}, applying topology-appropriate structural constraints.
 *
 * <p>Both {@code suggestedStructuralRole} and {@code suggestedPlatformRole} are nullable: null
 * means the strategy has no assessment for that facet, and the topology assigns from its own logic.
 *
 * @param suggestedName hierarchical role name (e.g. "coordinator", "reviewer/code")
 * @param suggestedStructuralRole structural position assessment, or null if unknown
 * @param suggestedPlatformRole platform behavioral contract, or null for domain roles
 * @param confidence score in [0, 1] synthesizing detection quality signals
 * @param source the detected role this suggestion was derived from
 */
public record RoleSuggestion(
    String suggestedName,
    @Nullable StructuralRole suggestedStructuralRole,
    @Nullable PlatformRole suggestedPlatformRole,
    double confidence,
    DetectedRole source) {

  public RoleSuggestion {
    if (confidence < 0.0 || confidence > 1.0) {
      throw new IllegalArgumentException("confidence must be in [0, 1], got " + confidence);
    }
    if (suggestedPlatformRole != null && suggestedStructuralRole != null
        && suggestedStructuralRole != suggestedPlatformRole.requiredStructuralRole()) {
      throw new IllegalArgumentException(
          suggestedPlatformRole + " requires " + suggestedPlatformRole.requiredStructuralRole()
              + ", got " + suggestedStructuralRole);
    }
  }
}
```

**Compact constructor invariants:**
1. Confidence range [0, 1]
2. If both `suggestedPlatformRole` and `suggestedStructuralRole` are non-null, the pairing must be consistent (same invariant as `AgentRole`)

### 1.2 RoleBridgingStrategy

SPI interface mapping detected roles to suggestions. Lives in engine-api because its input (`DetectedRole`) and output (`RoleSuggestion`) are engine-api vocabulary. Agency provides implementations.

```java
// api, io.casehub.api.spi.coordination
package io.casehub.api.spi.coordination;

import io.casehub.api.model.coordination.RoleSuggestion;
import io.casehub.api.model.stigmergy.DetectedRole;
import java.util.List;

/**
 * Maps emergent {@link DetectedRole}s to declared role suggestions. Implementations synthesize
 * detection quality signals (stability count, centroid drift, cluster quality) into a single
 * confidence score on each {@link RoleSuggestion}.
 *
 * <p>The strategy is the expert on what a behavioral pattern MEANS. The topology is the expert
 * on what structural positions are VALID. The strategy produces suggestions; the topology filters
 * structurally invalid ones and resolves conflicts.
 *
 * @see RoleSuggestion
 */
public interface RoleBridgingStrategy {

  List<RoleSuggestion> map(List<DetectedRole> detectedRoles);
}
```

The strategy receives the full list of detected roles (not one at a time) so it can make cross-role decisions — e.g., if two clusters both look like coordinators, the strategy can assign higher confidence to the more stable one.

## 2. New Types in agency

### 2.1 RoleBridgingPolicy

Agency-side policy controlling what happens with role suggestions. Lives alongside peer policies `AutonomyBudgetPolicy` and `DegradationPolicy`.

```java
// agency-core, io.casehub.agency.orchestration.bridging
package io.casehub.agency.orchestration.bridging;

/**
 * Controls how the topology acts on {@link io.casehub.api.model.coordination.RoleSuggestion}s
 * from the {@link io.casehub.api.spi.coordination.RoleBridgingStrategy}.
 *
 * @param mode bridging mode
 * @param minConfidence promotion threshold — suggestion must meet or exceed this
 * @param demotionConfidence demotion threshold — bridged role revoked when confidence drops below;
 *     must be lower than minConfidence for hysteresis
 * @param minHoldCycles minimum detection cycles a bridged role is maintained before demotion
 *     is eligible
 */
public record RoleBridgingPolicy(
    Mode mode,
    double minConfidence,
    double demotionConfidence,
    int minHoldCycles) {

  public static final RoleBridgingPolicy ADVISORY =
      new RoleBridgingPolicy(Mode.ADVISORY, 0.0, 0.0, 0);

  public static final RoleBridgingPolicy DISABLED =
      new RoleBridgingPolicy(Mode.ADVISORY, 1.1, 0.0, 0);

  public RoleBridgingPolicy {
    if (mode != Mode.ADVISORY && demotionConfidence >= minConfidence) {
      throw new IllegalArgumentException(
          "demotionConfidence (" + demotionConfidence
              + ") must be lower than minConfidence (" + minConfidence
              + ") for hysteresis");
    }
    if (minHoldCycles < 0) {
      throw new IllegalArgumentException("minHoldCycles must be >= 0");
    }
  }

  public enum Mode {
    ADVISORY,
    AUTO_ABOVE_THRESHOLD,
    AUTOMATIC
  }
}
```

**Compact constructor invariants:**
1. `demotionConfidence < minConfidence` for non-ADVISORY modes (hysteresis requirement from D7)
2. `minHoldCycles >= 0`

**Convenience constants:**
- `ADVISORY` — all suggestions surfaced, none auto-applied (thresholds irrelevant)
- `DISABLED` — minConfidence 1.1 makes no suggestion achievable

### 2.2 BridgedRoleState

Internal tracking record for bridged role assignments. Not an SPI type — used only within the topology implementation.

```java
// agency-core, io.casehub.agency.orchestration.bridging
package io.casehub.agency.orchestration.bridging;

import io.casehub.api.model.coordination.AgentRole;

/**
 * Tracks a bridged role assignment for oscillation protection (D7).
 */
record BridgedRoleState(
    String agentId,
    AgentRole bridgedRole,
    double lastConfidence,
    int holdCycles) {

  BridgedRoleState withConfidence(double confidence) {
    return new BridgedRoleState(agentId, bridgedRole, confidence, holdCycles + 1);
  }

  boolean demotionEligible(RoleBridgingPolicy policy) {
    return holdCycles >= policy.minHoldCycles()
        && lastConfidence < policy.demotionConfidence();
  }
}
```

## 3. TopologyContext Changes

Add `RoleBridgingPolicy` as a nullable field on `TopologyContext` (agency).

```java
// agency-core, io.casehub.agency.orchestration.topology
public record TopologyContext(
    UUID caseId,
    TopologyDescriptor descriptor,
    AgentProvider agentProvider,
    DecisionLoopFactory loopFactory,
    AutonomyBudgetPolicy budgetPolicy,
    DegradationPolicy degradationPolicy,
    @Nullable RoleBridgingPolicy bridgingPolicy,
    @Nullable String groupId,
    @Nullable String tenancyId
) { ... }
```

Nullable — topologies without stigmergy detection leave it null. The existing two-arg convenience constructor passes `null` for `bridgingPolicy`.

## 4. Orchestration Flow (D6)

The existing engine→agency event path delivers detected roles to the topology:

1. `RoleTracker.detect()` produces `List<DetectedRole>`
2. `SwarmProvisioner.notifyRolesChanged()` fires `SwarmCoordinationListener.onRoleChanged()`
3. `SwarmCoordinationListenerBridge` forwards as `SwarmCoordinationEvent.RoleChanged`
4. `CoordinationTopology.onSwarmEvent()` receives it

**New behavior in `HiveTopology.onSwarmEvent()`:**

```java
@Override
public void onSwarmEvent(SwarmCoordinationEvent event) {
  if (event instanceof SwarmCoordinationEvent.RoleChanged changed
      && bridgingStrategy != null && bridgingPolicy != null) {
    var suggestions = bridgingStrategy.map(changed.roles());
    applyBridging(suggestions);
  }
}
```

`applyBridging()` implements the policy evaluation:
- **ADVISORY:** log suggestions, emit advisory events for external consumption
- **AUTO_ABOVE_THRESHOLD:** apply suggestions where `confidence >= minConfidence`; check demotion for previously bridged roles where confidence dropped below `demotionConfidence` and `holdCycles >= minHoldCycles`
- **AUTOMATIC:** same as AUTO_ABOVE_THRESHOLD but with lower implicit thresholds

## 5. Demotion Semantics and Oscillation Protection (D7)

Bridged role assignments are reversible. Three mechanisms prevent oscillation:

1. **Asymmetric thresholds:** `minConfidence > demotionConfidence` creates a hysteresis band. An agent promoted at confidence 0.8 is not demoted until confidence drops below 0.5 (example values). This prevents flip-flopping when confidence fluctuates near a single threshold.

2. **Minimum hold cycles:** `minHoldCycles` prevents demotion during transient confidence drops. Even if confidence drops below `demotionConfidence`, the role is maintained until `holdCycles >= minHoldCycles`.

3. **Per-agent tracking:** `BridgedRoleState` tracks each bridged assignment with its last confidence and accumulated hold cycles. On each detection cycle, hold cycles increment and confidence updates.

**Demotion flow:**
```
For each previously bridged agent:
  1. Find the agent in the latest suggestions
  2. If absent or confidence dropped:
     - Update BridgedRoleState with new confidence
     - If demotionEligible(policy): remove bridged role, revert to original
  3. If confidence still above demotionConfidence:
     - Update BridgedRoleState, maintain role
```

In ADVISORY mode, demotion semantics don't apply — the operator manages roles manually.

## 6. Multi-Agent Conflict Resolution (D8)

When multiple agents are simultaneously suggested for structurally-exclusive positions:

1. **Single-occupancy positions** (e.g., one INITIATOR in CENTRALISED): highest-confidence suggestion wins. Equal-confidence ties use lexicographic agent ID as a stable tiebreaker.

2. **Non-exclusive positions** (PEER, CONTRACTOR): multiple simultaneous bridged roles allowed. No conflict resolution needed.

The topology resolves conflicts — not the strategy. The strategy maps individual `DetectedRole`s without cross-agent visibility for structural constraints; the topology knows its own constraints and all current role assignments.

## 7. Module Placement

| Component | Module | Package |
|-----------|--------|---------|
| `RoleSuggestion` | engine `api` | `io.casehub.api.model.coordination` |
| `RoleBridgingStrategy` | engine `api` | `io.casehub.api.spi.coordination` |
| `RoleBridgingPolicy` | agency `agency-core` | `io.casehub.agency.orchestration.bridging` |
| `BridgedRoleState` | agency `agency-core` | `io.casehub.agency.orchestration.bridging` |
| `TopologyContext` (modified) | agency `agency-core` | `io.casehub.agency.orchestration.topology` |
| `HiveTopology` (modified) | agency `agency-core` | `io.casehub.agency.orchestration.topology` |

## 8. Test Strategy

### New unit tests (engine)

| Test class | What it covers |
|------------|---------------|
| `RoleSuggestionTest` | Confidence range [0, 1] validation; platform/structural role consistency invariant; null fields accepted |

### New unit tests (agency)

| Test class | What it covers |
|------------|---------------|
| `RoleBridgingPolicyTest` | Hysteresis invariant (demotionConfidence < minConfidence); ADVISORY convenience constant; DISABLED constant; minHoldCycles >= 0 |
| `BridgedRoleStateTest` | withConfidence increments holdCycles; demotionEligible returns true only when both thresholds met |
| `RoleBridgingIntegrationTest` | Full flow: DetectedRole list → strategy → suggestions → policy evaluation → role assignment. Covers ADVISORY (no change), AUTO_ABOVE_THRESHOLD (above/below threshold), AUTOMATIC modes |
| `OscillationProtectionTest` | Hysteresis: promote at 0.8, don't demote at 0.6 (within band); demote at 0.3 (below demotionConfidence). MinHoldCycles: don't demote before holdCycles reached even when confidence low |
| `ConflictResolutionTest` | Two agents suggested as INITIATOR in CENTRALISED: highest confidence wins; equal confidence: lexicographic tiebreaker; PEER suggestions: both accepted |

### Modified existing tests

| Test class | Change |
|------------|--------|
| `TopologyContextTest` | Add bridgingPolicy field; verify nullable; existing constructors still work |
| `HiveTopologyTest` | Verify onSwarmEvent with RoleChanged dispatches to bridging; verify no-op when strategy/policy null |

## 9. Scope Exclusions

- **No default strategy implementation.** This spec defines the SPI. A default `BehavioralFingerprintStrategy` (mapping fingerprint features to platform roles) is a natural follow-up but out of scope.
- **No TransitionPolicy integration.** #1290 (TransitionPolicy) may consume role bridging signals, but that coupling is deferred to #1290's spec. The interaction point is through the topology's decision logic.
- **No YAML configuration surface.** The policy is set programmatically via `TopologyContext`. YAML configuration for bridging policy is deferred until the YAML coordination config model is designed.

## References

- `DetectedRole.java:21-33` — emergent role record with stabilityCount, centroidDrift
- `AgentRole.java:22-73` — declared role record with invariants
- `BehavioralFingerprint.java:20-58` — four-domain fingerprint with cosine similarity
- `RoleTracker.java:42-477` — behavioral clustering and role detection
- `TopologyProvider.java:21-27` — topology SPI (produces TopologyDescriptor)
- `TopologyDescriptor.java:22-68` — immutable topology state
- `TopologyContext.java:13-41` — agency topology context (peer policies: AutonomyBudgetPolicy, DegradationPolicy)
- `HiveTopology.java:38-138` — CoordinationTopology implementation with no-op onSwarmEvent
- `SwarmProvisioner.java:546-569` — notifyRolesChanged fires SwarmCoordinationListener
- `SwarmCoordinationListenerBridge.java:15-49` — engine→agency event bridge
- `SwarmCoordinationEvent.java:10-17` — RoleChanged event record
- `PlatformRole.java:18-33` — COORDINATOR, SUPERVISOR, JUDGE, MODERATOR
- `StructuralRole.java` — INITIATOR, PEER, CONTRACTOR
- Issue casehubio/engine#1282 — coordination vocabulary spec §AgentRole §Relationship to DetectedRole
- Issue casehubio/engine#1290 — TransitionPolicy (deferred interaction)
- Decisions D1–D8 in `decisions.md`
