# DetectedRole → AgentRole Bridging SPI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1295 — DetectedRole → AgentRole bridging SPI
**Issue group:** #1295

**Goal:** Create the SPI types and agency wiring that connect emergent role detection (DetectedRole) to declared role management (AgentRole) via a configurable bridging strategy and policy.

**Architecture:** New `RoleBridgingStrategy` SPI in engine-api maps `DetectedRole` → `RoleSuggestion`. Agency adds `RoleBridgingPolicy` (three configurable modes: ADVISORY, AUTO_ABOVE_THRESHOLD, AUTOMATIC) and wiring in `HiveTopology` to orchestrate the flow — including demotion with oscillation protection and multi-agent conflict resolution.

**Tech Stack:** Java 21, JUnit 5, AssertJ, jspecify nullability annotations

## Global Constraints

- Engine-api types must be `mvn install`ed before agency can compile against them
- All new records use compact constructor validation following existing patterns (AgentRole, TopologyContext)
- Nullable fields use `@Nullable` from `org.jspecify.annotations` (engine-api) or `jakarta.annotation` (agency)
- Tests use AssertJ, no mocking frameworks — follow existing patterns in AgentRoleTest, TopologyContextTest
- Cross-repo: engine types in `casehubio/engine`, agency types in `casehubio/agency` — both reference #1295
- No default strategy implementation (out of scope per spec §9) — tests use inline test implementations
- No YAML configuration surface (out of scope per spec §9) — policy is set programmatically via TopologyContext

---

## Batch 1: Engine-API SPI Foundation

### Task 1: RoleSuggestion record + RoleBridgingStrategy interface

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/coordination/RoleSuggestion.java`
- Create: `api/src/main/java/io/casehub/api/spi/coordination/RoleBridgingStrategy.java`
- Test: `api/src/test/java/io/casehub/api/model/coordination/RoleSuggestionTest.java`

**Interfaces:**
- Consumes: `DetectedRole` (`io.casehub.api.model.stigmergy`), `StructuralRole`, `PlatformRole` (`io.casehub.api.model.coordination`)
- Produces: `RoleSuggestion` record, `RoleBridgingStrategy` interface — consumed by agency bridging types (Task 2+) and HiveTopology wiring (Task 4)

- [ ] **Step 1: Write failing tests for RoleSuggestion**

```java
package io.casehub.api.model.coordination;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatIllegalArgumentException;
import static org.assertj.core.api.Assertions.assertThatNoException;

import io.casehub.api.model.stigmergy.BehavioralFingerprint;
import io.casehub.api.model.stigmergy.DetectedRole;
import java.util.List;
import java.util.Set;
import org.junit.jupiter.api.Test;

class RoleSuggestionTest {

  static final DetectedRole SOURCE = new DetectedRole(
      "cluster-1", Set.of("agent-a"), BehavioralFingerprint.EMPTY,
      List.of("coordination"), 5, 0.1);

  @Test
  void valid_suggestion_with_all_fields() {
    var suggestion = new RoleSuggestion(
        "coordinator", StructuralRole.INITIATOR, PlatformRole.COORDINATOR, 0.85, SOURCE);
    assertThat(suggestion.suggestedName()).isEqualTo("coordinator");
    assertThat(suggestion.suggestedStructuralRole()).isEqualTo(StructuralRole.INITIATOR);
    assertThat(suggestion.suggestedPlatformRole()).isEqualTo(PlatformRole.COORDINATOR);
    assertThat(suggestion.confidence()).isEqualTo(0.85);
    assertThat(suggestion.source()).isEqualTo(SOURCE);
  }

  @Test
  void null_structural_role_accepted() {
    assertThatNoException().isThrownBy(() ->
        new RoleSuggestion("analyst", null, null, 0.7, SOURCE));
  }

  @Test
  void null_platform_role_accepted() {
    assertThatNoException().isThrownBy(() ->
        new RoleSuggestion("worker", StructuralRole.CONTRACTOR, null, 0.6, SOURCE));
  }

  @Test
  void null_platform_with_null_structural_accepted() {
    var suggestion = new RoleSuggestion("observer", null, null, 0.5, SOURCE);
    assertThat(suggestion.suggestedStructuralRole()).isNull();
    assertThat(suggestion.suggestedPlatformRole()).isNull();
  }

  @Test
  void confidence_zero_accepted() {
    assertThatNoException().isThrownBy(() ->
        new RoleSuggestion("low", null, null, 0.0, SOURCE));
  }

  @Test
  void confidence_one_accepted() {
    assertThatNoException().isThrownBy(() ->
        new RoleSuggestion("high", null, null, 1.0, SOURCE));
  }

  @Test
  void confidence_below_zero_rejected() {
    assertThatIllegalArgumentException()
        .isThrownBy(() -> new RoleSuggestion("bad", null, null, -0.01, SOURCE))
        .withMessageContaining("confidence");
  }

  @Test
  void confidence_above_one_rejected() {
    assertThatIllegalArgumentException()
        .isThrownBy(() -> new RoleSuggestion("bad", null, null, 1.01, SOURCE))
        .withMessageContaining("confidence");
  }

  @Test
  void inconsistent_platform_structural_pairing_rejected() {
    assertThatIllegalArgumentException()
        .isThrownBy(() -> new RoleSuggestion(
            "bad", StructuralRole.CONTRACTOR, PlatformRole.COORDINATOR, 0.5, SOURCE))
        .withMessageContaining("COORDINATOR")
        .withMessageContaining("INITIATOR");
  }

  @Test
  void consistent_platform_structural_pairing_accepted() {
    assertThatNoException().isThrownBy(() ->
        new RoleSuggestion("judge", StructuralRole.PEER, PlatformRole.JUDGE, 0.9, SOURCE));
  }

  @Test
  void platform_role_with_null_structural_accepted() {
    var suggestion = new RoleSuggestion(
        "coordinator", null, PlatformRole.COORDINATOR, 0.8, SOURCE);
    assertThat(suggestion.suggestedPlatformRole()).isEqualTo(PlatformRole.COORDINATOR);
    assertThat(suggestion.suggestedStructuralRole()).isNull();
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=RoleSuggestionTest -q`
Expected: FAIL — `RoleSuggestion` class does not exist

- [ ] **Step 3: Create RoleSuggestion record**

```java
package io.casehub.api.model.coordination;

import io.casehub.api.model.stigmergy.DetectedRole;
import org.jspecify.annotations.Nullable;

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

- [ ] **Step 4: Create RoleBridgingStrategy interface**

```java
package io.casehub.api.spi.coordination;

import io.casehub.api.model.coordination.RoleSuggestion;
import io.casehub.api.model.stigmergy.DetectedRole;
import java.util.List;

public interface RoleBridgingStrategy {

  List<RoleSuggestion> map(List<DetectedRole> detectedRoles);
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=RoleSuggestionTest -q`
Expected: PASS — all 11 tests green

- [ ] **Step 6: Install engine to local Maven**

Run: `mvn install -DskipTests -q`

- [ ] **Step 7: Commit**

```bash
git add api/src/main/java/io/casehub/api/model/coordination/RoleSuggestion.java \
       api/src/main/java/io/casehub/api/spi/coordination/RoleBridgingStrategy.java \
       api/src/test/java/io/casehub/api/model/coordination/RoleSuggestionTest.java
git commit -m "feat: add RoleSuggestion record and RoleBridgingStrategy SPI

Refs #1295"
```

---

## Batch 2: Agency Bridging Types

### Task 2: RoleBridgingPolicy + BridgedRoleState records

**Files:**
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/bridging/RoleBridgingPolicy.java`
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/bridging/BridgedRoleState.java`
- Test: `agency-core/src/test/java/io/casehub/agency/orchestration/bridging/RoleBridgingPolicyTest.java`
- Test: `agency-core/src/test/java/io/casehub/agency/orchestration/bridging/BridgedRoleStateTest.java`

**Interfaces:**
- Consumes: `AgentRole` (`io.casehub.api.model.coordination`)
- Produces: `RoleBridgingPolicy` record, `BridgedRoleState` record — consumed by TopologyContext (Task 3) and HiveTopology bridging (Task 4)

**Note:** These files are in the agency repo at `/Users/mdproctor/claude/casehub/agency/`. The `bridging` package does not exist yet — create it.

- [ ] **Step 1: Write failing tests for RoleBridgingPolicy**

```java
package io.casehub.agency.orchestration.bridging;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatIllegalArgumentException;
import static org.assertj.core.api.Assertions.assertThatNoException;

import org.junit.jupiter.api.Test;

class RoleBridgingPolicyTest {

  @Test
  void advisory_constant_has_advisory_mode() {
    assertThat(RoleBridgingPolicy.ADVISORY.mode()).isEqualTo(RoleBridgingPolicy.Mode.ADVISORY);
  }

  @Test
  void disabled_constant_has_unreachable_threshold() {
    assertThat(RoleBridgingPolicy.DISABLED.minConfidence()).isGreaterThan(1.0);
  }

  @Test
  void advisory_mode_allows_equal_thresholds() {
    assertThatNoException().isThrownBy(() ->
        new RoleBridgingPolicy(RoleBridgingPolicy.Mode.ADVISORY, 0.5, 0.5, 0));
  }

  @Test
  void auto_above_threshold_requires_hysteresis() {
    assertThatIllegalArgumentException()
        .isThrownBy(() -> new RoleBridgingPolicy(
            RoleBridgingPolicy.Mode.AUTO_ABOVE_THRESHOLD, 0.8, 0.8, 5))
        .withMessageContaining("demotionConfidence")
        .withMessageContaining("minConfidence");
  }

  @Test
  void auto_above_threshold_accepts_proper_hysteresis() {
    assertThatNoException().isThrownBy(() ->
        new RoleBridgingPolicy(RoleBridgingPolicy.Mode.AUTO_ABOVE_THRESHOLD, 0.8, 0.5, 3));
  }

  @Test
  void automatic_requires_hysteresis() {
    assertThatIllegalArgumentException()
        .isThrownBy(() -> new RoleBridgingPolicy(
            RoleBridgingPolicy.Mode.AUTOMATIC, 0.6, 0.7, 2))
        .withMessageContaining("demotionConfidence")
        .withMessageContaining("minConfidence");
  }

  @Test
  void negative_hold_cycles_rejected() {
    assertThatIllegalArgumentException()
        .isThrownBy(() -> new RoleBridgingPolicy(
            RoleBridgingPolicy.Mode.ADVISORY, 0.5, 0.3, -1))
        .withMessageContaining("minHoldCycles");
  }

  @Test
  void zero_hold_cycles_accepted() {
    assertThatNoException().isThrownBy(() ->
        new RoleBridgingPolicy(RoleBridgingPolicy.Mode.AUTO_ABOVE_THRESHOLD, 0.8, 0.3, 0));
  }
}
```

- [ ] **Step 2: Write failing tests for BridgedRoleState**

```java
package io.casehub.agency.orchestration.bridging;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.api.model.coordination.AgentRole;
import io.casehub.api.model.coordination.StructuralRole;
import org.junit.jupiter.api.Test;

class BridgedRoleStateTest {

  static final AgentRole COORDINATOR_ROLE = AgentRole.of("coordinator", StructuralRole.INITIATOR);

  @Test
  void withConfidence_increments_hold_cycles() {
    var state = new BridgedRoleState("agent-1", COORDINATOR_ROLE, 0.8, 0);
    var updated = state.withConfidence(0.75);
    assertThat(updated.holdCycles()).isEqualTo(1);
    assertThat(updated.lastConfidence()).isEqualTo(0.75);
    assertThat(updated.agentId()).isEqualTo("agent-1");
    assertThat(updated.bridgedRole()).isEqualTo(COORDINATOR_ROLE);
  }

  @Test
  void withConfidence_accumulates_hold_cycles() {
    var state = new BridgedRoleState("agent-1", COORDINATOR_ROLE, 0.9, 0);
    state = state.withConfidence(0.85);
    state = state.withConfidence(0.80);
    state = state.withConfidence(0.75);
    assertThat(state.holdCycles()).isEqualTo(3);
  }

  @Test
  void demotionEligible_true_when_both_thresholds_met() {
    var policy = new RoleBridgingPolicy(
        RoleBridgingPolicy.Mode.AUTO_ABOVE_THRESHOLD, 0.8, 0.5, 3);
    var state = new BridgedRoleState("agent-1", COORDINATOR_ROLE, 0.3, 4);
    assertThat(state.demotionEligible(policy)).isTrue();
  }

  @Test
  void demotionEligible_false_when_confidence_above_demotion_threshold() {
    var policy = new RoleBridgingPolicy(
        RoleBridgingPolicy.Mode.AUTO_ABOVE_THRESHOLD, 0.8, 0.5, 3);
    var state = new BridgedRoleState("agent-1", COORDINATOR_ROLE, 0.6, 4);
    assertThat(state.demotionEligible(policy)).isFalse();
  }

  @Test
  void demotionEligible_false_when_hold_cycles_insufficient() {
    var policy = new RoleBridgingPolicy(
        RoleBridgingPolicy.Mode.AUTO_ABOVE_THRESHOLD, 0.8, 0.5, 3);
    var state = new BridgedRoleState("agent-1", COORDINATOR_ROLE, 0.3, 2);
    assertThat(state.demotionEligible(policy)).isFalse();
  }

  @Test
  void demotionEligible_true_at_exact_hold_cycle_boundary() {
    var policy = new RoleBridgingPolicy(
        RoleBridgingPolicy.Mode.AUTO_ABOVE_THRESHOLD, 0.8, 0.5, 3);
    var state = new BridgedRoleState("agent-1", COORDINATOR_ROLE, 0.3, 3);
    assertThat(state.demotionEligible(policy)).isTrue();
  }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl agency-core -Dtest="RoleBridgingPolicyTest,BridgedRoleStateTest" -q`
(from agency repo root)
Expected: FAIL — classes do not exist

- [ ] **Step 4: Create RoleBridgingPolicy record**

```java
package io.casehub.agency.orchestration.bridging;

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

- [ ] **Step 5: Create BridgedRoleState record**

```java
package io.casehub.agency.orchestration.bridging;

import io.casehub.api.model.coordination.AgentRole;

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

- [ ] **Step 6: Run tests to verify they pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl agency-core -Dtest="RoleBridgingPolicyTest,BridgedRoleStateTest" -q`
Expected: PASS — all tests green

- [ ] **Step 7: Commit (agency repo)**

```bash
git -C /Users/mdproctor/claude/casehub/agency add \
  agency-core/src/main/java/io/casehub/agency/orchestration/bridging/RoleBridgingPolicy.java \
  agency-core/src/main/java/io/casehub/agency/orchestration/bridging/BridgedRoleState.java \
  agency-core/src/test/java/io/casehub/agency/orchestration/bridging/RoleBridgingPolicyTest.java \
  agency-core/src/test/java/io/casehub/agency/orchestration/bridging/BridgedRoleStateTest.java
git -C /Users/mdproctor/claude/casehub/agency commit -m "feat: add RoleBridgingPolicy and BridgedRoleState records

Refs casehubio/engine#1295"
```

---

## Batch 3: Topology Wiring and Integration

### Task 3: TopologyContext — add bridgingPolicy field

**Files:**
- Modify: `/Users/mdproctor/claude/casehub/agency/agency-core/src/main/java/io/casehub/agency/orchestration/topology/TopologyContext.java`
- Modify: `/Users/mdproctor/claude/casehub/agency/agency-core/src/test/java/io/casehub/agency/orchestration/topology/TopologyContextTest.java`
- Modify: `/Users/mdproctor/claude/casehub/agency/agency-core/src/test/java/io/casehub/agency/orchestration/topology/HiveTopologyTest.java` (8-arg constructor calls → 9-arg)

**Interfaces:**
- Consumes: `RoleBridgingPolicy` from Task 2
- Produces: `TopologyContext.bridgingPolicy()` accessor — consumed by HiveTopology in Task 4

- [ ] **Step 1: Write failing test for bridgingPolicy on TopologyContext**

Add to `TopologyContextTest.java`:

```java
@Test
void creates_with_bridging_policy() {
    var bridgingPolicy = new RoleBridgingPolicy(
            RoleBridgingPolicy.Mode.AUTO_ABOVE_THRESHOLD, 0.8, 0.5, 3);
    var ctx = new TopologyContext(
            UUID.randomUUID(), DESCRIPTOR, AGENT_PROVIDER, LOOP_FACTORY, POLICY, DEG_POLICY,
            bridgingPolicy, "group-1", "tenant-1");
    assertThat(ctx.bridgingPolicy()).isEqualTo(bridgingPolicy);
}

@Test
void creates_with_null_bridging_policy() {
    var ctx = new TopologyContext(
            UUID.randomUUID(), DESCRIPTOR, AGENT_PROVIDER, LOOP_FACTORY, POLICY, DEG_POLICY,
            null, null, null);
    assertThat(ctx.bridgingPolicy()).isNull();
}

@Test
void convenience_constructor_has_null_bridging_policy() {
    var ctx = new TopologyContext(
            UUID.randomUUID(), DESCRIPTOR, AGENT_PROVIDER, LOOP_FACTORY, POLICY, DEG_POLICY);
    assertThat(ctx.bridgingPolicy()).isNull();
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl agency-core -Dtest=TopologyContextTest -q`
Expected: FAIL — `bridgingPolicy` field/accessor does not exist

- [ ] **Step 3: Add bridgingPolicy to TopologyContext canonical constructor**

Modify the record to add `@Nullable RoleBridgingPolicy bridgingPolicy` between `degradationPolicy` and `groupId`:

```java
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
) {
    public TopologyContext {
        Objects.requireNonNull(caseId, "caseId must not be null");
        Objects.requireNonNull(descriptor, "descriptor must not be null");
        Objects.requireNonNull(agentProvider, "agentProvider must not be null");
        Objects.requireNonNull(loopFactory, "loopFactory must not be null");
        Objects.requireNonNull(budgetPolicy, "budgetPolicy must not be null");
        Objects.requireNonNull(degradationPolicy, "degradationPolicy must not be null");
    }

    public TopologyContext(
            UUID caseId,
            TopologyDescriptor descriptor,
            AgentProvider agentProvider,
            DecisionLoopFactory loopFactory,
            AutonomyBudgetPolicy budgetPolicy,
            DegradationPolicy degradationPolicy) {
        this(caseId, descriptor, agentProvider, loopFactory, budgetPolicy, degradationPolicy,
             null, null, null);
    }
}
```

Add import: `import io.casehub.agency.orchestration.bridging.RoleBridgingPolicy;`

- [ ] **Step 4: Update existing 8-arg constructor calls to 9-arg**

**TopologyContextTest.java** — `creates_with_group_context` and `creates_with_null_group_context`: insert `null` for bridgingPolicy before groupId/tenancyId.

**HiveTopologyTest.java** — `configure_capturesGroupIdAndTenancyId`: insert `null` for bridgingPolicy before `"group-42"`.

Each call changes from:
```java
new TopologyContext(uuid, DESCRIPTOR, AGENT_PROVIDER, LOOP_FACTORY, POLICY, DEG_POLICY, "group-42", "tenant-1")
```
to:
```java
new TopologyContext(uuid, DESCRIPTOR, AGENT_PROVIDER, LOOP_FACTORY, POLICY, DEG_POLICY, null, "group-42", "tenant-1")
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl agency-core -Dtest="TopologyContextTest,HiveTopologyTest" -q`
Expected: PASS — all tests green including new and existing

- [ ] **Step 6: Commit (agency repo)**

```bash
git -C /Users/mdproctor/claude/casehub/agency add \
  agency-core/src/main/java/io/casehub/agency/orchestration/topology/TopologyContext.java \
  agency-core/src/test/java/io/casehub/agency/orchestration/topology/TopologyContextTest.java \
  agency-core/src/test/java/io/casehub/agency/orchestration/topology/HiveTopologyTest.java
git -C /Users/mdproctor/claude/casehub/agency commit -m "feat: add bridgingPolicy to TopologyContext

Refs casehubio/engine#1295"
```

---

### Task 4: HiveTopology bridging implementation + integration tests

**Files:**
- Modify: `/Users/mdproctor/claude/casehub/agency/agency-core/src/main/java/io/casehub/agency/orchestration/topology/HiveTopology.java`
- Modify: `/Users/mdproctor/claude/casehub/agency/agency-core/src/test/java/io/casehub/agency/orchestration/topology/HiveTopologyTest.java`
- Create: `/Users/mdproctor/claude/casehub/agency/agency-core/src/test/java/io/casehub/agency/orchestration/bridging/RoleBridgingIntegrationTest.java`
- Create: `/Users/mdproctor/claude/casehub/agency/agency-core/src/test/java/io/casehub/agency/orchestration/bridging/OscillationProtectionTest.java`
- Create: `/Users/mdproctor/claude/casehub/agency/agency-core/src/test/java/io/casehub/agency/orchestration/bridging/ConflictResolutionTest.java`

**Interfaces:**
- Consumes: `RoleBridgingPolicy`, `BridgedRoleState` (Task 2), `TopologyContext.bridgingPolicy()` (Task 3), `RoleBridgingStrategy`, `RoleSuggestion` (Task 1)
- Produces: Working bridging flow in `HiveTopology.onSwarmEvent()` — the deliverable for #1295

- [ ] **Step 1: Write HiveTopologyTest — onSwarmEvent no-op when strategy null**

Add to `HiveTopologyTest.java`:

```java
@Test
void onSwarmEvent_noop_when_no_strategy() {
    topology.configure(topologyContext);
    var roles = List.of(new DetectedRole(
        "cluster-1", Set.of("builder"), BehavioralFingerprint.EMPTY,
        List.of("coordination"), 5, 0.1));
    topology.onSwarmEvent(new SwarmCoordinationEvent.RoleChanged(roles,
        new SwarmProgress(1, 3, Set.of(), Set.of(), 0.5)));

    var descriptor = topology.describeTopology(UUID.randomUUID());
    assertThat(descriptor.agentRoles()).isEqualTo(agentRoles);
}
```

- [ ] **Step 2: Run test to verify it passes (already no-op)**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl agency-core -Dtest="HiveTopologyTest#onSwarmEvent_noop_when_no_strategy" -q`
Expected: PASS — `onSwarmEvent` is currently empty

- [ ] **Step 3: Write RoleBridgingIntegrationTest**

```java
package io.casehub.agency.orchestration.bridging;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.agency.orchestration.budget.AutonomyBudgetPolicy;
import io.casehub.agency.orchestration.degradation.DegradationPolicy;
import io.casehub.agency.orchestration.loop.DecisionLoopFactory;
import io.casehub.agency.orchestration.topology.HiveTopology;
import io.casehub.agency.orchestration.topology.HivePatternAdapter;
import io.casehub.agency.orchestration.topology.OrchestrationPatternType;
import io.casehub.agency.orchestration.topology.TestAgentProviders;
import io.casehub.agency.orchestration.topology.TopologyContext;
import io.casehub.agency.summarisation.EventStreamBus;
import io.casehub.api.model.coordination.AgentRole;
import io.casehub.api.model.coordination.PlatformRole;
import io.casehub.api.model.coordination.RoleSuggestion;
import io.casehub.api.model.coordination.StructuralRole;
import io.casehub.api.model.coordination.TopologyDescriptor;
import io.casehub.api.model.coordination.TopologyType;
import io.casehub.api.model.stigmergy.BehavioralFingerprint;
import io.casehub.api.model.stigmergy.DetectedRole;
import io.casehub.api.model.stigmergy.SwarmProgress;
import io.casehub.api.spi.coordination.RoleBridgingStrategy;
import io.casehub.agency.orchestration.swarm.SwarmCoordinationEvent;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class RoleBridgingIntegrationTest {

  static final SwarmProgress PROGRESS = new SwarmProgress(1, 3, Set.of(), Set.of(), 0.5);

  Map<String, AgentRole> agentRoles;
  HiveTopology topology;

  RoleBridgingStrategy testStrategy = detectedRoles -> detectedRoles.stream()
      .filter(dr -> dr.dominantFeatures().contains("coordination"))
      .map(dr -> new RoleSuggestion(
          "coordinator", StructuralRole.INITIATOR, PlatformRole.COORDINATOR,
          0.85, dr))
      .toList();

  @BeforeEach
  void setUp() {
    agentRoles = Map.of(
        "supervisor", AgentRole.of("supervisor", StructuralRole.INITIATOR),
        "builder", AgentRole.of("builder", StructuralRole.CONTRACTOR),
        "critic", AgentRole.of("critic", StructuralRole.CONTRACTOR));
  }

  @Test
  void advisory_mode_does_not_modify_roles() {
    topology = new HiveTopology(agentRoles,
        new HivePatternAdapter(TestAgentProviders.noOp()),
        OrchestrationPatternType.SUPERVISOR, testStrategy);

    var ctx = new TopologyContext(
        UUID.randomUUID(),
        new TopologyDescriptor(TopologyType.CENTRALISED, agentRoles),
        TestAgentProviders.noOp(),
        new DecisionLoopFactory(name -> c -> "", new EventStreamBus<>()),
        AutonomyBudgetPolicy.defaults(), DegradationPolicy.defaults(),
        RoleBridgingPolicy.ADVISORY, null, null);
    topology.configure(ctx);

    var detected = List.of(new DetectedRole("cluster-1", Set.of("builder"),
        BehavioralFingerprint.EMPTY, List.of("coordination"), 5, 0.1));
    topology.onSwarmEvent(new SwarmCoordinationEvent.RoleChanged(detected, PROGRESS));

    assertThat(topology.describeTopology(UUID.randomUUID()).agentRoles())
        .isEqualTo(agentRoles);
  }

  @Test
  void auto_above_threshold_applies_suggestion_above_threshold() {
    topology = new HiveTopology(agentRoles,
        new HivePatternAdapter(TestAgentProviders.noOp()),
        OrchestrationPatternType.SUPERVISOR, testStrategy);

    var policy = new RoleBridgingPolicy(
        RoleBridgingPolicy.Mode.AUTO_ABOVE_THRESHOLD, 0.8, 0.5, 3);
    var ctx = new TopologyContext(
        UUID.randomUUID(),
        new TopologyDescriptor(TopologyType.CENTRALISED, agentRoles),
        TestAgentProviders.noOp(),
        new DecisionLoopFactory(name -> c -> "", new EventStreamBus<>()),
        AutonomyBudgetPolicy.defaults(), DegradationPolicy.defaults(),
        policy, null, null);
    topology.configure(ctx);

    var detected = List.of(new DetectedRole("cluster-1", Set.of("builder"),
        BehavioralFingerprint.EMPTY, List.of("coordination"), 5, 0.1));
    topology.onSwarmEvent(new SwarmCoordinationEvent.RoleChanged(detected, PROGRESS));

    var roles = topology.describeTopology(UUID.randomUUID()).agentRoles();
    assertThat(roles.get("builder").platformRole()).isEqualTo(PlatformRole.COORDINATOR);
    assertThat(roles.get("builder").structuralRole()).isEqualTo(StructuralRole.INITIATOR);
  }

  @Test
  void auto_above_threshold_ignores_suggestion_below_threshold() {
    RoleBridgingStrategy lowConfidence = detectedRoles -> detectedRoles.stream()
        .map(dr -> new RoleSuggestion("coordinator", StructuralRole.INITIATOR,
            PlatformRole.COORDINATOR, 0.4, dr))
        .toList();

    topology = new HiveTopology(agentRoles,
        new HivePatternAdapter(TestAgentProviders.noOp()),
        OrchestrationPatternType.SUPERVISOR, lowConfidence);

    var policy = new RoleBridgingPolicy(
        RoleBridgingPolicy.Mode.AUTO_ABOVE_THRESHOLD, 0.8, 0.5, 3);
    var ctx = new TopologyContext(
        UUID.randomUUID(),
        new TopologyDescriptor(TopologyType.CENTRALISED, agentRoles),
        TestAgentProviders.noOp(),
        new DecisionLoopFactory(name -> c -> "", new EventStreamBus<>()),
        AutonomyBudgetPolicy.defaults(), DegradationPolicy.defaults(),
        policy, null, null);
    topology.configure(ctx);

    var detected = List.of(new DetectedRole("cluster-1", Set.of("builder"),
        BehavioralFingerprint.EMPTY, List.of("coordination"), 5, 0.1));
    topology.onSwarmEvent(new SwarmCoordinationEvent.RoleChanged(detected, PROGRESS));

    assertThat(topology.describeTopology(UUID.randomUUID()).agentRoles())
        .isEqualTo(agentRoles);
  }
}
```

- [ ] **Step 4: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl agency-core -Dtest=RoleBridgingIntegrationTest -q`
Expected: FAIL — HiveTopology constructor doesn't accept RoleBridgingStrategy, onSwarmEvent is empty

- [ ] **Step 5: Implement HiveTopology bridging**

Modify `HiveTopology.java`:

**Add fields:**
```java
private final @org.jspecify.annotations.Nullable RoleBridgingStrategy bridgingStrategy;
private volatile @org.jspecify.annotations.Nullable RoleBridgingPolicy bridgingPolicy;
private final Map<String, BridgedRoleState> bridgedStates = new ConcurrentHashMap<>();
```

**Add constructor overload (keeping original for backward compatibility):**
```java
public HiveTopology(Map<String, AgentRole> agentRoles,
                    HivePatternAdapter patternAdapter,
                    OrchestrationPatternType patternType,
                    @org.jspecify.annotations.Nullable RoleBridgingStrategy bridgingStrategy) {
    this.agentRoles = Map.copyOf(Objects.requireNonNull(agentRoles));
    this.patternAdapter = Objects.requireNonNull(patternAdapter);
    this.patternType = Objects.requireNonNull(patternType);
    this.bridgingStrategy = bridgingStrategy;
}

public HiveTopology(Map<String, AgentRole> agentRoles,
                    HivePatternAdapter patternAdapter,
                    OrchestrationPatternType patternType) {
    this(agentRoles, patternAdapter, patternType, null);
}
```

**Modify `configure()` — extract bridgingPolicy:**
```java
@Override
public CoordinationPlan configure(TopologyContext context) {
    this.groupId = context.groupId();
    this.tenancyId = context.tenancyId();
    this.bridgingPolicy = context.bridgingPolicy();
    // ... rest unchanged
```

**Modify `describeTopology()` — merge bridged roles:**
```java
@Override
public TopologyDescriptor describeTopology(UUID caseId) {
    if (bridgedStates.isEmpty()) {
        return new TopologyDescriptor(TopologyType.CENTRALISED, agentRoles);
    }
    var merged = new LinkedHashMap<>(agentRoles);
    bridgedStates.forEach((id, state) -> merged.put(id, state.bridgedRole()));
    return new TopologyDescriptor(TopologyType.CENTRALISED, Map.copyOf(merged));
}
```

**Implement `onSwarmEvent()`:**
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

**Implement `applyBridging()` — core orchestration logic:**
```java
private void applyBridging(List<RoleSuggestion> suggestions) {
    if (bridgingPolicy.mode() == RoleBridgingPolicy.Mode.ADVISORY) {
        return;
    }

    // Track per-agent confidence from ALL suggestions (before policy filtering)
    // Used for demotion: the spec says to use actual latest confidence, not 0.0
    Map<String, Double> latestConfidence = new LinkedHashMap<>();
    for (var suggestion : suggestions) {
        for (String agentId : suggestion.source().memberAgents()) {
            latestConfidence.merge(agentId, suggestion.confidence(), Math::max);
        }
    }

    // Filter by minConfidence for promotion
    Map<String, RoleSuggestion> candidates = new LinkedHashMap<>();
    for (var suggestion : suggestions) {
        if (suggestion.confidence() < bridgingPolicy.minConfidence()) continue;
        for (String agentId : suggestion.source().memberAgents()) {
            candidates.merge(agentId, suggestion, (existing, candidate) ->
                candidate.confidence() > existing.confidence() ? candidate : existing);
        }
    }

    resolveConflicts(candidates);

    // Apply promotions
    for (var entry : candidates.entrySet()) {
        String agentId = entry.getKey();
        RoleSuggestion suggestion = entry.getValue();
        AgentRole role = toBridgedRole(suggestion);
        bridgedStates.compute(agentId, (id, existing) ->
            existing != null
                ? existing.withConfidence(suggestion.confidence())
                : new BridgedRoleState(id, role, suggestion.confidence(), 0));
    }

    // Handle demotions for previously bridged agents not promoted this cycle
    var toRemove = new ArrayList<String>();
    for (var entry : bridgedStates.entrySet()) {
        if (!candidates.containsKey(entry.getKey())) {
            double confidence = latestConfidence.getOrDefault(entry.getKey(), 0.0);
            var updated = entry.getValue().withConfidence(confidence);
            if (updated.demotionEligible(bridgingPolicy)) {
                toRemove.add(entry.getKey());
            } else {
                bridgedStates.put(entry.getKey(), updated);
            }
        }
    }
    toRemove.forEach(bridgedStates::remove);
}
```

**Implement `resolveConflicts()` — structural exclusivity:**
```java
private void resolveConflicts(Map<String, RoleSuggestion> candidates) {
    Map<StructuralRole, List<String>> exclusiveGroups = new LinkedHashMap<>();
    for (var entry : candidates.entrySet()) {
        StructuralRole structural = effectiveStructuralRole(entry.getValue());
        if (structural == StructuralRole.INITIATOR) {
            exclusiveGroups.computeIfAbsent(structural, k -> new ArrayList<>())
                .add(entry.getKey());
        }
    }
    for (var group : exclusiveGroups.values()) {
        if (group.size() > 1) {
            group.sort((a, b) -> {
                int cmp = Double.compare(
                    candidates.get(b).confidence(), candidates.get(a).confidence());
                return cmp != 0 ? cmp : a.compareTo(b);
            });
            for (int i = 1; i < group.size(); i++) {
                candidates.remove(group.get(i));
            }
        }
    }
}
```

**Implement helper — `toBridgedRole()`:**
```java
private AgentRole toBridgedRole(RoleSuggestion suggestion) {
    var structural = suggestion.suggestedStructuralRole();
    var platform = suggestion.suggestedPlatformRole();
    if (platform != null && structural == null) {
        structural = platform.requiredStructuralRole();
    }
    return new AgentRole(
        io.casehub.platform.api.path.Path.parse(suggestion.suggestedName()),
        structural, platform);
}
```

**Implement helper — `effectiveStructuralRole()`:**
```java
private StructuralRole effectiveStructuralRole(RoleSuggestion suggestion) {
    if (suggestion.suggestedStructuralRole() != null) {
        return suggestion.suggestedStructuralRole();
    }
    if (suggestion.suggestedPlatformRole() != null) {
        return suggestion.suggestedPlatformRole().requiredStructuralRole();
    }
    return null;
}
```

**Required imports to add:**
```java
import io.casehub.agency.orchestration.bridging.BridgedRoleState;
import io.casehub.agency.orchestration.bridging.RoleBridgingPolicy;
import io.casehub.api.model.coordination.RoleSuggestion;
import io.casehub.api.spi.coordination.RoleBridgingStrategy;
import java.util.LinkedHashSet;
```

- [ ] **Step 6: Run integration tests to verify they pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl agency-core -Dtest="RoleBridgingIntegrationTest,HiveTopologyTest" -q`
Expected: PASS

- [ ] **Step 7: Write OscillationProtectionTest**

```java
package io.casehub.agency.orchestration.bridging;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.agency.orchestration.budget.AutonomyBudgetPolicy;
import io.casehub.agency.orchestration.degradation.DegradationPolicy;
import io.casehub.agency.orchestration.loop.DecisionLoopFactory;
import io.casehub.agency.orchestration.topology.HiveTopology;
import io.casehub.agency.orchestration.topology.HivePatternAdapter;
import io.casehub.agency.orchestration.topology.OrchestrationPatternType;
import io.casehub.agency.orchestration.topology.TestAgentProviders;
import io.casehub.agency.orchestration.topology.TopologyContext;
import io.casehub.agency.summarisation.EventStreamBus;
import io.casehub.api.model.coordination.AgentRole;
import io.casehub.api.model.coordination.PlatformRole;
import io.casehub.api.model.coordination.RoleSuggestion;
import io.casehub.api.model.coordination.StructuralRole;
import io.casehub.api.model.coordination.TopologyDescriptor;
import io.casehub.api.model.coordination.TopologyType;
import io.casehub.api.model.stigmergy.BehavioralFingerprint;
import io.casehub.api.model.stigmergy.DetectedRole;
import io.casehub.api.model.stigmergy.SwarmProgress;
import io.casehub.api.spi.coordination.RoleBridgingStrategy;
import io.casehub.agency.orchestration.swarm.SwarmCoordinationEvent;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class OscillationProtectionTest {

  static final SwarmProgress PROGRESS = new SwarmProgress(1, 3, Set.of(), Set.of(), 0.5);

  Map<String, AgentRole> agentRoles;
  HiveTopology topology;

  @BeforeEach
  void setUp() {
    agentRoles = Map.of(
        "supervisor", AgentRole.of("supervisor", StructuralRole.INITIATOR),
        "builder", AgentRole.of("builder", StructuralRole.CONTRACTOR));
  }

  private HiveTopology createTopology(RoleBridgingStrategy strategy) {
    return new HiveTopology(agentRoles,
        new HivePatternAdapter(TestAgentProviders.noOp()),
        OrchestrationPatternType.SUPERVISOR, strategy);
  }

  private TopologyContext createContext(RoleBridgingPolicy policy) {
    return new TopologyContext(
        UUID.randomUUID(),
        new TopologyDescriptor(TopologyType.CENTRALISED, agentRoles),
        TestAgentProviders.noOp(),
        new DecisionLoopFactory(name -> c -> "", new EventStreamBus<>()),
        AutonomyBudgetPolicy.defaults(), DegradationPolicy.defaults(),
        policy, null, null);
  }

  private SwarmCoordinationEvent.RoleChanged roleEvent(double confidence) {
    RoleBridgingStrategy strategy = dr -> dr.stream()
        .map(d -> new RoleSuggestion("coordinator", StructuralRole.INITIATOR,
            PlatformRole.COORDINATOR, confidence, d))
        .toList();
    var detected = new DetectedRole("cluster-1", Set.of("builder"),
        BehavioralFingerprint.EMPTY, List.of("coordination"), 5, 0.1);
    return new SwarmCoordinationEvent.RoleChanged(List.of(detected), PROGRESS);
  }

  @Test
  void promoted_at_high_confidence_not_demoted_within_hysteresis_band() {
    var strategy = new RoleBridgingStrategy() {
      double confidence = 0.85;
      @Override
      public List<RoleSuggestion> map(List<DetectedRole> detectedRoles) {
        return detectedRoles.stream()
            .map(d -> new RoleSuggestion("coordinator", StructuralRole.INITIATOR,
                PlatformRole.COORDINATOR, confidence, d))
            .toList();
      }
    };

    topology = createTopology(strategy);
    var policy = new RoleBridgingPolicy(
        RoleBridgingPolicy.Mode.AUTO_ABOVE_THRESHOLD, 0.8, 0.5, 3);
    topology.configure(createContext(policy));

    var detected = List.of(new DetectedRole("cluster-1", Set.of("builder"),
        BehavioralFingerprint.EMPTY, List.of("coordination"), 5, 0.1));
    var event = new SwarmCoordinationEvent.RoleChanged(detected, PROGRESS);

    // Promote at 0.85
    topology.onSwarmEvent(event);
    assertThat(topology.describeTopology(UUID.randomUUID()).agentRoles()
        .get("builder").platformRole()).isEqualTo(PlatformRole.COORDINATOR);

    // Drop confidence to 0.6 — within hysteresis band (above 0.5 demotion threshold)
    strategy.confidence = 0.6;
    topology.onSwarmEvent(event);
    topology.onSwarmEvent(event);
    topology.onSwarmEvent(event);
    topology.onSwarmEvent(event);

    // Still promoted — confidence (0.6) above demotionConfidence (0.5)
    assertThat(topology.describeTopology(UUID.randomUUID()).agentRoles()
        .get("builder").platformRole()).isEqualTo(PlatformRole.COORDINATOR);
  }

  @Test
  void not_demoted_before_min_hold_cycles() {
    var strategy = new RoleBridgingStrategy() {
      double confidence = 0.85;
      @Override
      public List<RoleSuggestion> map(List<DetectedRole> detectedRoles) {
        return detectedRoles.stream()
            .map(d -> new RoleSuggestion("coordinator", StructuralRole.INITIATOR,
                PlatformRole.COORDINATOR, confidence, d))
            .toList();
      }
    };

    topology = createTopology(strategy);
    var policy = new RoleBridgingPolicy(
        RoleBridgingPolicy.Mode.AUTO_ABOVE_THRESHOLD, 0.8, 0.5, 3);
    topology.configure(createContext(policy));

    var detected = List.of(new DetectedRole("cluster-1", Set.of("builder"),
        BehavioralFingerprint.EMPTY, List.of("coordination"), 5, 0.1));
    var event = new SwarmCoordinationEvent.RoleChanged(detected, PROGRESS);

    // Promote
    topology.onSwarmEvent(event);
    assertThat(topology.describeTopology(UUID.randomUUID()).agentRoles()
        .get("builder").platformRole()).isEqualTo(PlatformRole.COORDINATOR);

    // Drop confidence below demotion threshold but within hold cycle window
    strategy.confidence = 0.3;
    topology.onSwarmEvent(event); // holdCycles=1
    topology.onSwarmEvent(event); // holdCycles=2
    // Still promoted — hold cycles (2) < minHoldCycles (3)
    assertThat(topology.describeTopology(UUID.randomUUID()).agentRoles()
        .get("builder").platformRole()).isEqualTo(PlatformRole.COORDINATOR);

    // One more cycle — now holdCycles=3, demotion eligible (0.3 < 0.5 AND 3 >= 3)
    topology.onSwarmEvent(event);
    assertThat(topology.describeTopology(UUID.randomUUID()).agentRoles()
        .get("builder")).isEqualTo(agentRoles.get("builder"));
  }

  @Test
  void absent_from_suggestions_triggers_demotion_after_hold_cycles() {
    var strategy = new RoleBridgingStrategy() {
      boolean returnSuggestions = true;
      @Override
      public List<RoleSuggestion> map(List<DetectedRole> detectedRoles) {
        if (!returnSuggestions) return List.of();
        return detectedRoles.stream()
            .filter(d -> d.memberAgents().contains("builder"))
            .map(d -> new RoleSuggestion("coordinator", StructuralRole.INITIATOR,
                PlatformRole.COORDINATOR, 0.9, d))
            .toList();
      }
    };

    topology = createTopology(strategy);
    var policy = new RoleBridgingPolicy(
        RoleBridgingPolicy.Mode.AUTO_ABOVE_THRESHOLD, 0.8, 0.5, 2);
    topology.configure(createContext(policy));

    var detected = List.of(new DetectedRole("cluster-1", Set.of("builder"),
        BehavioralFingerprint.EMPTY, List.of("coordination"), 5, 0.1));
    topology.onSwarmEvent(new SwarmCoordinationEvent.RoleChanged(detected, PROGRESS));
    assertThat(topology.describeTopology(UUID.randomUUID()).agentRoles()
        .get("builder").platformRole()).isEqualTo(PlatformRole.COORDINATOR);

    // Agent absent from suggestions — strategy returns empty
    strategy.returnSuggestions = false;
    topology.onSwarmEvent(new SwarmCoordinationEvent.RoleChanged(detected, PROGRESS)); // holdCycles=1
    topology.onSwarmEvent(new SwarmCoordinationEvent.RoleChanged(detected, PROGRESS)); // holdCycles=2

    // After 2 cycles absent (holdCycles=2 >= minHoldCycles=2, 0.0 < 0.5), demotion eligible
    assertThat(topology.describeTopology(UUID.randomUUID()).agentRoles()
        .get("builder")).isEqualTo(agentRoles.get("builder"));
  }
}
```

- [ ] **Step 8: Run oscillation tests to verify they pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl agency-core -Dtest=OscillationProtectionTest -q`
Expected: PASS

- [ ] **Step 9: Write ConflictResolutionTest**

```java
package io.casehub.agency.orchestration.bridging;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.agency.orchestration.budget.AutonomyBudgetPolicy;
import io.casehub.agency.orchestration.degradation.DegradationPolicy;
import io.casehub.agency.orchestration.loop.DecisionLoopFactory;
import io.casehub.agency.orchestration.topology.HiveTopology;
import io.casehub.agency.orchestration.topology.HivePatternAdapter;
import io.casehub.agency.orchestration.topology.OrchestrationPatternType;
import io.casehub.agency.orchestration.topology.TestAgentProviders;
import io.casehub.agency.orchestration.topology.TopologyContext;
import io.casehub.agency.summarisation.EventStreamBus;
import io.casehub.api.model.coordination.AgentRole;
import io.casehub.api.model.coordination.PlatformRole;
import io.casehub.api.model.coordination.RoleSuggestion;
import io.casehub.api.model.coordination.StructuralRole;
import io.casehub.api.model.coordination.TopologyDescriptor;
import io.casehub.api.model.coordination.TopologyType;
import io.casehub.api.model.stigmergy.BehavioralFingerprint;
import io.casehub.api.model.stigmergy.DetectedRole;
import io.casehub.api.model.stigmergy.SwarmProgress;
import io.casehub.api.spi.coordination.RoleBridgingStrategy;
import io.casehub.agency.orchestration.swarm.SwarmCoordinationEvent;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.UUID;
import org.junit.jupiter.api.Test;

class ConflictResolutionTest {

  static final SwarmProgress PROGRESS = new SwarmProgress(1, 3, Set.of(), Set.of(), 0.5);

  @Test
  void two_agents_suggested_as_initiator_highest_confidence_wins() {
    Map<String, AgentRole> roles = Map.of(
        "supervisor", AgentRole.of("supervisor", StructuralRole.INITIATOR),
        "agent-a", AgentRole.of("worker-a", StructuralRole.CONTRACTOR),
        "agent-b", AgentRole.of("worker-b", StructuralRole.CONTRACTOR));

    var clusterA = new DetectedRole("cluster-a", Set.of("agent-a"),
        BehavioralFingerprint.EMPTY, List.of("coordination"), 5, 0.1);
    var clusterB = new DetectedRole("cluster-b", Set.of("agent-b"),
        BehavioralFingerprint.EMPTY, List.of("coordination"), 8, 0.05);

    RoleBridgingStrategy strategy = detectedRoles -> {
      var suggestions = new ArrayList<RoleSuggestion>();
      for (var dr : detectedRoles) {
        double confidence = dr.roleId().equals("cluster-a") ? 0.85 : 0.92;
        suggestions.add(new RoleSuggestion("coordinator", StructuralRole.INITIATOR,
            PlatformRole.COORDINATOR, confidence, dr));
      }
      return suggestions;
    };

    var topology = new HiveTopology(roles,
        new HivePatternAdapter(TestAgentProviders.noOp()),
        OrchestrationPatternType.SUPERVISOR, strategy);
    var policy = new RoleBridgingPolicy(
        RoleBridgingPolicy.Mode.AUTO_ABOVE_THRESHOLD, 0.8, 0.5, 3);
    var ctx = new TopologyContext(
        UUID.randomUUID(),
        new TopologyDescriptor(TopologyType.CENTRALISED, roles),
        TestAgentProviders.noOp(),
        new DecisionLoopFactory(name -> c -> "", new EventStreamBus<>()),
        AutonomyBudgetPolicy.defaults(), DegradationPolicy.defaults(),
        policy, null, null);
    topology.configure(ctx);

    topology.onSwarmEvent(new SwarmCoordinationEvent.RoleChanged(
        List.of(clusterA, clusterB), PROGRESS));

    var updatedRoles = topology.describeTopology(UUID.randomUUID()).agentRoles();
    // agent-b wins (0.92 > 0.85)
    assertThat(updatedRoles.get("agent-b").platformRole()).isEqualTo(PlatformRole.COORDINATOR);
    // agent-a keeps original role (lost conflict)
    assertThat(updatedRoles.get("agent-a")).isEqualTo(roles.get("agent-a"));
  }

  @Test
  void equal_confidence_uses_lexicographic_tiebreaker() {
    Map<String, AgentRole> roles = Map.of(
        "supervisor", AgentRole.of("supervisor", StructuralRole.INITIATOR),
        "agent-a", AgentRole.of("worker-a", StructuralRole.CONTRACTOR),
        "agent-b", AgentRole.of("worker-b", StructuralRole.CONTRACTOR));

    var clusterA = new DetectedRole("cluster-a", Set.of("agent-a"),
        BehavioralFingerprint.EMPTY, List.of("coordination"), 5, 0.1);
    var clusterB = new DetectedRole("cluster-b", Set.of("agent-b"),
        BehavioralFingerprint.EMPTY, List.of("coordination"), 5, 0.1);

    RoleBridgingStrategy strategy = detectedRoles -> detectedRoles.stream()
        .map(dr -> new RoleSuggestion("coordinator", StructuralRole.INITIATOR,
            PlatformRole.COORDINATOR, 0.9, dr))
        .toList();

    var topology = new HiveTopology(roles,
        new HivePatternAdapter(TestAgentProviders.noOp()),
        OrchestrationPatternType.SUPERVISOR, strategy);
    var policy = new RoleBridgingPolicy(
        RoleBridgingPolicy.Mode.AUTO_ABOVE_THRESHOLD, 0.8, 0.5, 3);
    var ctx = new TopologyContext(
        UUID.randomUUID(),
        new TopologyDescriptor(TopologyType.CENTRALISED, roles),
        TestAgentProviders.noOp(),
        new DecisionLoopFactory(name -> c -> "", new EventStreamBus<>()),
        AutonomyBudgetPolicy.defaults(), DegradationPolicy.defaults(),
        policy, null, null);
    topology.configure(ctx);

    topology.onSwarmEvent(new SwarmCoordinationEvent.RoleChanged(
        List.of(clusterA, clusterB), PROGRESS));

    var updatedRoles = topology.describeTopology(UUID.randomUUID()).agentRoles();
    // agent-a wins lexicographically ("agent-a" < "agent-b")
    assertThat(updatedRoles.get("agent-a").platformRole()).isEqualTo(PlatformRole.COORDINATOR);
    assertThat(updatedRoles.get("agent-b")).isEqualTo(roles.get("agent-b"));
  }

  @Test
  void peer_suggestions_allow_multiple_agents() {
    Map<String, AgentRole> roles = Map.of(
        "supervisor", AgentRole.of("supervisor", StructuralRole.INITIATOR),
        "agent-a", AgentRole.of("worker-a", StructuralRole.CONTRACTOR),
        "agent-b", AgentRole.of("worker-b", StructuralRole.CONTRACTOR));

    var clusterA = new DetectedRole("cluster-a", Set.of("agent-a"),
        BehavioralFingerprint.EMPTY, List.of("judging"), 5, 0.1);
    var clusterB = new DetectedRole("cluster-b", Set.of("agent-b"),
        BehavioralFingerprint.EMPTY, List.of("judging"), 5, 0.1);

    RoleBridgingStrategy strategy = detectedRoles -> detectedRoles.stream()
        .map(dr -> new RoleSuggestion("judge", StructuralRole.PEER,
            PlatformRole.JUDGE, 0.9, dr))
        .toList();

    var topology = new HiveTopology(roles,
        new HivePatternAdapter(TestAgentProviders.noOp()),
        OrchestrationPatternType.SUPERVISOR, strategy);
    var policy = new RoleBridgingPolicy(
        RoleBridgingPolicy.Mode.AUTO_ABOVE_THRESHOLD, 0.8, 0.5, 3);
    var ctx = new TopologyContext(
        UUID.randomUUID(),
        new TopologyDescriptor(TopologyType.CENTRALISED, roles),
        TestAgentProviders.noOp(),
        new DecisionLoopFactory(name -> c -> "", new EventStreamBus<>()),
        AutonomyBudgetPolicy.defaults(), DegradationPolicy.defaults(),
        policy, null, null);
    topology.configure(ctx);

    topology.onSwarmEvent(new SwarmCoordinationEvent.RoleChanged(
        List.of(clusterA, clusterB), PROGRESS));

    var updatedRoles = topology.describeTopology(UUID.randomUUID()).agentRoles();
    // Both agents get JUDGE — PEER is not exclusive
    assertThat(updatedRoles.get("agent-a").platformRole()).isEqualTo(PlatformRole.JUDGE);
    assertThat(updatedRoles.get("agent-b").platformRole()).isEqualTo(PlatformRole.JUDGE);
  }
}
```

- [ ] **Step 10: Run conflict resolution tests to verify they pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl agency-core -Dtest=ConflictResolutionTest -q`
Expected: PASS

- [ ] **Step 11: Run full agency-core test suite**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl agency-core -q`
Expected: PASS — all existing and new tests green, no regressions

- [ ] **Step 12: Commit (agency repo)**

```bash
git -C /Users/mdproctor/claude/casehub/agency add \
  agency-core/src/main/java/io/casehub/agency/orchestration/topology/HiveTopology.java \
  agency-core/src/test/java/io/casehub/agency/orchestration/topology/HiveTopologyTest.java \
  agency-core/src/test/java/io/casehub/agency/orchestration/bridging/RoleBridgingIntegrationTest.java \
  agency-core/src/test/java/io/casehub/agency/orchestration/bridging/OscillationProtectionTest.java \
  agency-core/src/test/java/io/casehub/agency/orchestration/bridging/ConflictResolutionTest.java
git -C /Users/mdproctor/claude/casehub/agency commit -m "feat: implement role bridging in HiveTopology

Wire RoleBridgingStrategy into HiveTopology.onSwarmEvent to
map DetectedRole suggestions through configurable policy
(ADVISORY/AUTO_ABOVE_THRESHOLD/AUTOMATIC) with oscillation
protection (hysteresis + hold cycles) and structural conflict
resolution for exclusive positions.

Refs casehubio/engine#1295"
```

---

## References

- `specs/issue-1295-detected-role-agent-role-bridging/2026-11-07-detected-role-agent-role-bridging-design.md` — design spec
- `specs/issue-1295-detected-role-agent-role-bridging/decisions.md` — D1–D8
- `api/src/main/java/io/casehub/api/model/stigmergy/DetectedRole.java:21-33` — source record
- `api/src/main/java/io/casehub/api/model/coordination/AgentRole.java:22-73` — target record with invariants
- `api/src/main/java/io/casehub/api/model/coordination/PlatformRole.java:18-33` — platform role enum
- `api/src/main/java/io/casehub/api/model/coordination/StructuralRole.java:22-27` — structural role enum
- `agency-core TopologyContext.java:13-42` — current topology context record
- `agency-core HiveTopology.java:38-173` — current hive topology (empty onSwarmEvent)
- `agency-core SwarmCoordinationEvent.java:10-17` — RoleChanged event
- GitHub casehubio/engine#1295 — focal issue
- GitHub casehubio/engine#1281 — parent epic
