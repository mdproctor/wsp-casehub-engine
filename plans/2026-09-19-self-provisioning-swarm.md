# Self-Provisioning Swarm Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1113 — feat: Self-provisioning swarm — dynamic agent scaling via WorkerProvisioner/ops
**Issue group:** #1104 (Hive Mind epic), #1105-#1112 (foundation SPIs + stigmergy + swarm)

**Goal:** Extend the swarm execution model with signal-based self-provisioning: agents request capacity through pheromone signals, the engine provisions via WorkerProvisioner when consensus is reached, new agents integrate through a three-axis tunable model (bootstrap richness, integration delay, self-determination), and CBR learns optimal provisioning strategies over time.

**Architecture:** A new `SwarmProvisioner` `@ApplicationScoped` bean coordinates the full provisioning lifecycle. Provisioning signals (`swarm:need-capacity`) use the existing pheromone model — consensus detection triggers the provisioner. Three-layer budget enforcement (maxSwarmSize, ProvisionBudget, DispatchBudget) prevents unbounded scaling. De-provisioning uses idle detection + signal decay. `SwarmProvisioningAdvisor` SPI provides the extension point for blocks-side LLM reasoning. CBR records provisioning outcomes for future retrieval.

**Tech Stack:** Java 21+, Quarkus 3.32.2, JUnit 5

## Global Constraints

- All new types in existing stigmergy packages (no new packages)
- All tracker state is in-memory only (ConcurrentHashMap, per D29)
- Use `ReentrantLock` instead of `synchronized` (per D72, virtual thread compat)
- Tests must be `*Test.java` (never `*IT.java`)
- All commits reference `Refs #1113`
- Maven compile: `/opt/homebrew/bin/mvn compile -pl <module> -Dcheckstyle.skip=true -q`
- Maven test: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl <module> -Dcheckstyle.skip=true`

---

## Batch 1: API Types

### Task 1: Configuration and request records — ProvisionBudget, IntegrationPolicy, ProvisioningRequest, SwarmBootstrapContext, SwarmConfig extension, CaseHubEventType values, SwarmProvisioningAdvisor SPI

All API-level type definitions. No logic — records, interfaces, enum values.

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/ProvisionBudget.java`
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/IntegrationPolicy.java`
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/ProvisioningRequest.java`
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/SwarmBootstrapContext.java`
- Create: `api/src/main/java/io/casehub/api/spi/stigmergy/SwarmProvisioningAdvisor.java`
- Modify: `api/src/main/java/io/casehub/api/model/stigmergy/SwarmConfig.java` — add `provisionBudget` and `integrationPolicy` fields + backward-compat constructor
- Modify: `api/src/main/java/io/casehub/api/model/event/CaseHubEventType.java:128` — add 6 enum values

**Interfaces:**
- Consumes: `DetectedRole`, `DetectedTeam`, `SwarmProgress`, `ProvisioningRequest` (all existing api records)
- Produces: `ProvisionBudget`, `IntegrationPolicy`, `ProvisioningRequest`, `SwarmBootstrapContext`, `SwarmProvisioningAdvisor` — consumed by SwarmProvisioner (Batch 2)

- [ ] **Step 1: Create ProvisionBudget record**

Use `ide_create_file`:

```java
package io.casehub.api.model.stigmergy;

import jakarta.annotation.Nullable;

public record ProvisionBudget(
    @Nullable Integer maxProvisions,
    @Nullable Integer maxConcurrent,
    @Nullable Integer cooldownCycles,
    @Nullable Double idleThreshold,
    @Nullable Integer idleGraceCycles) {

  public int effectiveMaxProvisions() {
    return maxProvisions != null ? maxProvisions : 50;
  }

  public int effectiveMaxConcurrent() {
    return maxConcurrent != null ? maxConcurrent : 10;
  }

  public int effectiveCooldownCycles() {
    return cooldownCycles != null ? cooldownCycles : 5;
  }

  public double effectiveIdleThreshold() {
    return idleThreshold != null ? idleThreshold : 0.01;
  }

  public int effectiveIdleGraceCycles() {
    return idleGraceCycles != null ? idleGraceCycles : 10;
  }
}
```

- [ ] **Step 2: Create IntegrationPolicy record**

Use `ide_create_file`:

```java
package io.casehub.api.model.stigmergy;

import jakarta.annotation.Nullable;

public record IntegrationPolicy(
    @Nullable Double bootstrapRichness,
    @Nullable Integer integrationDelay,
    @Nullable Double selfDetermination) {

  public static final IntegrationPolicy BALANCED =
      new IntegrationPolicy(0.7, 3, 0.5);

  public double effectiveBootstrapRichness() {
    return bootstrapRichness != null ? bootstrapRichness : 0.7;
  }

  public int effectiveIntegrationDelay() {
    return integrationDelay != null ? integrationDelay : 3;
  }

  public double effectiveSelfDetermination() {
    return selfDetermination != null ? selfDetermination : 0.5;
  }
}
```

- [ ] **Step 3: Create ProvisioningRequest record**

Use `ide_create_file`:

```java
package io.casehub.api.model.stigmergy;

import jakarta.annotation.Nullable;
import java.time.Instant;
import java.util.Set;

public record ProvisioningRequest(
    Set<String> requiredCapabilities,
    @Nullable String preferredModelId,
    @Nullable String reason,
    String requestingAgentId,
    Instant requestedAt) {

  public ProvisioningRequest {
    requiredCapabilities = Set.copyOf(requiredCapabilities);
  }
}
```

- [ ] **Step 4: Create SwarmBootstrapContext record**

Use `ide_create_file`:

```java
package io.casehub.api.model.stigmergy;

import java.util.List;
import java.util.Map;
import java.util.Set;

public record SwarmBootstrapContext(
    Map<String, Double> activeSignals,
    Set<String> activeInterestKeys,
    List<DetectedRole> currentRoles,
    List<DetectedTeam> currentTeams,
    SwarmProgress swarmProgress,
    ProvisioningRequest triggeringRequest,
    IntegrationPolicy policy) {

  public SwarmBootstrapContext {
    activeSignals = Map.copyOf(activeSignals);
    activeInterestKeys = Set.copyOf(activeInterestKeys);
    currentRoles = List.copyOf(currentRoles);
    currentTeams = List.copyOf(currentTeams);
  }
}
```

- [ ] **Step 5: Create SwarmProvisioningAdvisor SPI**

Use `ide_create_file`. Note: this requires creating the `api/src/main/java/io/casehub/api/spi/stigmergy/` package directory — the first file in this package creates it.

```java
package io.casehub.api.spi.stigmergy;

import io.casehub.api.model.stigmergy.IntegrationPolicy;
import io.casehub.api.model.stigmergy.SwarmBootstrapContext;
import jakarta.annotation.Nullable;
import java.util.List;
import java.util.Set;
import java.util.UUID;

public interface SwarmProvisioningAdvisor {

  ProvisioningAdvice advise(ProvisioningContext context);

  record ProvisioningContext(
      UUID caseId,
      SwarmBootstrapContext swarmState,
      Set<String> requestedCapabilities,
      IntegrationPolicy currentPolicy) {}

  record ProvisioningAdvice(
      boolean shouldProvision,
      @Nullable Set<String> adjustedCapabilities,
      @Nullable IntegrationPolicy adjustedPolicy,
      @Nullable String reasoning) {}
}
```

- [ ] **Step 6: Modify SwarmConfig — add provisionBudget and integrationPolicy fields**

The current `SwarmConfig` has 9 fields. Add 2 new nullable fields. Add a backward-compat 9-arg constructor that passes `null` for the new fields (same pattern as `StigmergyConfig`).

Use `ide_replace_text_in_file` to replace the record definition. The new 11-field record:

```java
public record SwarmConfig(
    @Nullable Integer maxSwarmSize,
    @Nullable Double roleSimilarityThreshold,
    @Nullable Integer roleMinClusterSize,
    @Nullable Integer roleDetectionWindow,
    @Nullable Integer detectionInterval,
    @Nullable RoleDomainWeights domainWeights,
    @Nullable Double teamAffinityThreshold,
    @Nullable Integer teamMinSize,
    @Nullable Double progressChangeThreshold,
    @Nullable ProvisionBudget provisionBudget,
    @Nullable IntegrationPolicy integrationPolicy) {

  public SwarmConfig(
      @Nullable Integer maxSwarmSize,
      @Nullable Double roleSimilarityThreshold,
      @Nullable Integer roleMinClusterSize,
      @Nullable Integer roleDetectionWindow,
      @Nullable Integer detectionInterval,
      @Nullable RoleDomainWeights domainWeights,
      @Nullable Double teamAffinityThreshold,
      @Nullable Integer teamMinSize,
      @Nullable Double progressChangeThreshold) {
    this(maxSwarmSize, roleSimilarityThreshold, roleMinClusterSize,
        roleDetectionWindow, detectionInterval, domainWeights,
        teamAffinityThreshold, teamMinSize, progressChangeThreshold,
        null, null);
  }

  public int effectiveMaxSwarmSize() {
    return maxSwarmSize != null ? maxSwarmSize : 20;
  }

  // ... existing effective* methods unchanged
}
```

- [ ] **Step 7: Add 6 CaseHubEventType values**

Use `ide_replace_text_in_file` to replace `SWARM_PROGRESS // swarm progress scores changed significantly` (last enum value without trailing comma) with:

```java
  SWARM_PROGRESS, // swarm progress scores changed significantly

  SWARM_PROVISION_REQUESTED, // consensus detected, provisioning initiated
  SWARM_PROVISION_COMPLETED, // agent successfully provisioned and joined swarm
  SWARM_PROVISION_FAILED, // WorkerProvisioner.provision() threw ProvisioningException
  SWARM_PROVISION_VETOED, // SwarmProvisioningAdvisor vetoed the provision
  SWARM_PROVISION_BUDGET_EXHAUSTED, // budget check failed (any of the 3 layers)
  SWARM_AGENT_TERMINATED // idle agent de-provisioned
```

- [ ] **Step 8: Compile and verify**

Run: `/opt/homebrew/bin/mvn compile -pl api -Dcheckstyle.skip=true -q`
Expected: BUILD SUCCESS

- [ ] **Step 9: Commit**

```bash
git add api/src/main/java/io/casehub/api/model/stigmergy/ProvisionBudget.java \
  api/src/main/java/io/casehub/api/model/stigmergy/IntegrationPolicy.java \
  api/src/main/java/io/casehub/api/model/stigmergy/ProvisioningRequest.java \
  api/src/main/java/io/casehub/api/model/stigmergy/SwarmBootstrapContext.java \
  api/src/main/java/io/casehub/api/spi/stigmergy/SwarmProvisioningAdvisor.java \
  api/src/main/java/io/casehub/api/model/stigmergy/SwarmConfig.java \
  api/src/main/java/io/casehub/api/model/event/CaseHubEventType.java
git commit -m "feat: add self-provisioning API types — ProvisionBudget, IntegrationPolicy, SwarmProvisioningAdvisor

Refs #1113"
```

---

## Batch 2: SwarmProvisioner Core

### Task 2: SwarmProvisioner — budget enforcement, consensus-driven provisioning, de-provisioning

Core provisioning bean with per-case locking, three-layer budget enforcement, bootstrap context building, provisioning lifecycle, idle detection, and de-provisioning.

**Files:**
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/stigmergy/SwarmProvisioner.java`
- Create: `runtime-core/src/test/java/io/casehub/engine/internal/stigmergy/SwarmProvisionerTest.java`

**Interfaces:**
- Consumes: `WorkerProvisioner.provision(Set<String>, ProvisionContext) → ProvisionResult`, `WorkerProvisioner.terminate(String, String)`, `StigmergyCoordinator.activeAgents(UUID) → List<AgentState>`, `StigmergyCoordinator.agentJoined(UUID, String, String)`, `StigmergyCoordinator.agentActivated(UUID, String)`, `StigmergyCoordinator.agentDeparted(UUID, String)`, `SignalRegistry.getAllSignals(UUID) → Map<String, Signal>`, `SignalRegistry.consensusSignals(UUID, int, double) → Map<String, Signal>`, `ActivityTracker.getState(UUID) → CaseActivityState`, `RoleTracker.getDetectedRoles(UUID) → List<DetectedRole>`, `TeamDetector.getDetectedTeams(UUID) → List<DetectedTeam>`, `SwarmProgressTracker.getProgress(UUID) → SwarmProgress`, `DispatchBudget.availableCapacity(DispatchBudgetQuery) → int`, `Instance<CapabilityHealth>`, `Instance<SwarmProvisioningAdvisor>`
- Produces: `SwarmProvisioner.evaluateAndProvision(UUID, CaseDefinition) → List<SwarmEvent>`, `SwarmProvisioner.evaluateDeprovisioning(UUID, SwarmConfig) → List<SwarmEvent>`, `SwarmProvisioner.isInIntegrationDelay(UUID, String) → boolean`, `SwarmProvisioner.evictByCase(UUID)` — consumed by handler wiring (Batch 4)

- [ ] **Step 1: Write failing test — budget enforcement blocks over-provisioning**

Use `ide_create_file`:

```java
package io.casehub.engine.internal.stigmergy;

import static org.junit.jupiter.api.Assertions.*;

import io.casehub.api.model.event.CaseHubEventType;
import io.casehub.api.model.stigmergy.*;
import io.casehub.api.spi.*;
import io.casehub.engine.common.internal.convergence.ActivityTracker;
import io.casehub.engine.common.internal.observation.ObservationRegistry;
import io.casehub.engine.common.internal.observation.RuleRegistry;
import io.casehub.engine.common.internal.signal.SignalRegistry;
import java.time.Duration;
import java.util.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class SwarmProvisionerTest {

  private SwarmProvisioner swarmProvisioner;
  private StigmergyCoordinator coordinator;
  private SignalRegistry signalRegistry;
  private ObservationRegistry observationRegistry;
  private ActivityTracker activityTracker;
  private RoleTracker roleTracker;
  private TeamDetector teamDetector;
  private SwarmProgressTracker progressTracker;
  private RecordingWorkerProvisioner workerProvisioner;
  private UUID caseId;

  @BeforeEach
  void setUp() {
    observationRegistry = new ObservationRegistry();
    signalRegistry = new SignalRegistry();
    var ruleRegistry = new RuleRegistry();
    activityTracker = new ActivityTracker();
    coordinator = new StigmergyCoordinator(signalRegistry, observationRegistry, activityTracker);
    roleTracker = new RoleTracker(observationRegistry, signalRegistry, ruleRegistry, coordinator);
    teamDetector = new TeamDetector(observationRegistry, signalRegistry, roleTracker, coordinator);
    progressTracker = new SwarmProgressTracker(signalRegistry, roleTracker, coordinator);
    workerProvisioner = new RecordingWorkerProvisioner();
    swarmProvisioner = new SwarmProvisioner(
        workerProvisioner, coordinator, signalRegistry, activityTracker,
        roleTracker, teamDetector, progressTracker, new NoOpDispatchBudget());
    caseId = UUID.randomUUID();
  }

  @Test
  void budgetMaxSwarmSizeBlocksProvisioning() {
    var budget = new ProvisionBudget(50, 10, 0, null, null);
    var policy = IntegrationPolicy.BALANCED;
    var swarmConfig = new SwarmConfig(
        2, null, null, null, null, null, null, null, null, budget, policy);
    var config = new StigmergyConfig(null, null, swarmConfig);
    coordinator.initializeCase(caseId, List.of("a1", "a2"), config);
    for (String id : List.of("a1", "a2")) {
      coordinator.agentJoined(caseId, id, "b-" + id);
      coordinator.agentActivated(caseId, id);
    }

    signalRegistry.deposit(caseId, "swarm:need-capacity", 0.8, Duration.ofMinutes(5), "a1", 100);
    signalRegistry.deposit(caseId, "swarm:need-capacity", 0.8, Duration.ofMinutes(5), "a2", 100);

    var events = swarmProvisioner.evaluateAndProvision(caseId, swarmConfig, config);

    assertTrue(events.stream().anyMatch(
        e -> e.type() == CaseHubEventType.SWARM_PROVISION_BUDGET_EXHAUSTED));
    assertEquals(0, workerProvisioner.provisionCount());
  }

  @Test
  void cooldownPreventsRapidProvisioning() {
    var budget = new ProvisionBudget(50, 10, 5, null, null);
    var policy = IntegrationPolicy.BALANCED;
    var swarmConfig = new SwarmConfig(
        20, null, null, null, null, null, null, null, null, budget, policy);
    var config = new StigmergyConfig(null, null, swarmConfig);
    coordinator.initializeCase(caseId, List.of("a1", "a2"), config);
    for (String id : List.of("a1", "a2")) {
      coordinator.agentJoined(caseId, id, "b-" + id);
      coordinator.agentActivated(caseId, id);
    }

    signalRegistry.deposit(caseId, "swarm:need-capacity", 0.8, Duration.ofMinutes(5), "a1", 100);
    signalRegistry.deposit(caseId, "swarm:need-capacity", 0.8, Duration.ofMinutes(5), "a2", 100);

    var events1 = swarmProvisioner.evaluateAndProvision(caseId, swarmConfig, config);
    assertTrue(events1.stream().anyMatch(
        e -> e.type() == CaseHubEventType.SWARM_PROVISION_COMPLETED));

    signalRegistry.deposit(caseId, "swarm:need-capacity", 0.9, Duration.ofMinutes(5), "a1", 100);
    var events2 = swarmProvisioner.evaluateAndProvision(caseId, swarmConfig, config);

    assertTrue(events2.stream().anyMatch(
        e -> e.type() == CaseHubEventType.SWARM_PROVISION_BUDGET_EXHAUSTED));
    assertEquals(1, workerProvisioner.provisionCount());
  }

  @Test
  void successfulProvisionJoinsSwarm() {
    var budget = new ProvisionBudget(50, 10, 0, null, null);
    var policy = IntegrationPolicy.BALANCED;
    var swarmConfig = new SwarmConfig(
        20, null, null, null, null, null, null, null, null, budget, policy);
    var config = new StigmergyConfig(null, null, swarmConfig);
    coordinator.initializeCase(caseId, List.of("a1", "a2"), config);
    for (String id : List.of("a1", "a2")) {
      coordinator.agentJoined(caseId, id, "b-" + id);
      coordinator.agentActivated(caseId, id);
    }

    signalRegistry.deposit(caseId, "swarm:need-capacity", 0.8, Duration.ofMinutes(5), "a1", 100);
    signalRegistry.deposit(caseId, "swarm:need-capacity", 0.8, Duration.ofMinutes(5), "a2", 100);

    var events = swarmProvisioner.evaluateAndProvision(caseId, swarmConfig, config);

    assertTrue(events.stream().anyMatch(
        e -> e.type() == CaseHubEventType.SWARM_PROVISION_COMPLETED));
    assertEquals(3, coordinator.activeAgents(caseId).size());
    assertEquals(1, workerProvisioner.provisionCount());
  }

  @Test
  void integrationDelayExemptsFromIdleDetection() {
    var budget = new ProvisionBudget(50, 10, 0, 0.5, 1);
    var policy = new IntegrationPolicy(0.7, 5, 0.5);
    var swarmConfig = new SwarmConfig(
        20, null, null, null, null, null, null, null, null, budget, policy);
    var config = new StigmergyConfig(null, null, swarmConfig);
    coordinator.initializeCase(caseId, List.of("a1", "a2"), config);
    for (String id : List.of("a1", "a2")) {
      coordinator.agentJoined(caseId, id, "b-" + id);
      coordinator.agentActivated(caseId, id);
    }

    signalRegistry.deposit(caseId, "swarm:need-capacity", 0.8, Duration.ofMinutes(5), "a1", 100);
    signalRegistry.deposit(caseId, "swarm:need-capacity", 0.8, Duration.ofMinutes(5), "a2", 100);
    swarmProvisioner.evaluateAndProvision(caseId, swarmConfig, config);

    String provisionedId = workerProvisioner.lastProvisionedId();
    assertTrue(swarmProvisioner.isInIntegrationDelay(caseId, provisionedId));

    var deprovEvents = swarmProvisioner.evaluateDeprovisioning(caseId, swarmConfig);
    assertTrue(deprovEvents.stream().noneMatch(
        e -> e.type() == CaseHubEventType.SWARM_AGENT_TERMINATED
            && e.metadata().get("agentId").equals(provisionedId)));
  }

  @Test
  void evictByCaseClearsState() {
    var budget = new ProvisionBudget(50, 10, 0, null, null);
    var policy = IntegrationPolicy.BALANCED;
    var swarmConfig = new SwarmConfig(
        20, null, null, null, null, null, null, null, null, budget, policy);
    var config = new StigmergyConfig(null, null, swarmConfig);
    coordinator.initializeCase(caseId, List.of("a1", "a2"), config);
    for (String id : List.of("a1", "a2")) {
      coordinator.agentJoined(caseId, id, "b-" + id);
      coordinator.agentActivated(caseId, id);
    }
    signalRegistry.deposit(caseId, "swarm:need-capacity", 0.8, Duration.ofMinutes(5), "a1", 100);
    signalRegistry.deposit(caseId, "swarm:need-capacity", 0.8, Duration.ofMinutes(5), "a2", 100);
    swarmProvisioner.evaluateAndProvision(caseId, swarmConfig, config);

    swarmProvisioner.evictByCase(caseId);

    assertFalse(swarmProvisioner.isInIntegrationDelay(caseId, "any"));
  }

  static class RecordingWorkerProvisioner implements WorkerProvisioner {
    private int count;
    private String lastId;

    @Override
    public ProvisionResult provision(Set<String> capabilities, io.casehub.api.model.ProvisionContext context) {
      count++;
      lastId = "swarm-provisioned-" + count;
      return new ProvisionResult(null, lastId);
    }

    @Override
    public void terminate(String workerId, String tenancyId) {}

    @Override
    public Set<String> getCapabilities() {
      return Set.of("*");
    }

    int provisionCount() { return count; }
    String lastProvisionedId() { return lastId; }
  }

  static class NoOpDispatchBudget implements DispatchBudget {
    @Override
    public int availableCapacity(DispatchBudgetQuery query) {
      return Integer.MAX_VALUE;
    }
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime-core -Dtest=SwarmProvisionerTest -Dcheckstyle.skip=true -q`
Expected: FAIL — `SwarmProvisioner` class does not exist

- [ ] **Step 3: Implement SwarmProvisioner**

Use `ide_create_file` for `runtime-core/src/main/java/io/casehub/engine/internal/stigmergy/SwarmProvisioner.java`:

```java
package io.casehub.engine.internal.stigmergy;

import io.casehub.api.model.ProvisionContext;
import io.casehub.api.model.event.CaseHubEventType;
import io.casehub.api.model.stigmergy.*;
import io.casehub.api.spi.*;
import io.casehub.api.spi.stigmergy.SwarmProvisioningAdvisor;
import io.casehub.engine.common.internal.convergence.ActivityTracker;
import io.casehub.engine.common.internal.signal.SignalRegistry;
import io.casehub.engine.common.spi.Resettable;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import java.time.Duration;
import java.time.Instant;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.locks.ReentrantLock;

@ApplicationScoped
public class SwarmProvisioner implements Resettable {

  private final WorkerProvisioner workerProvisioner;
  private final StigmergyCoordinator coordinator;
  private final SignalRegistry signalRegistry;
  private final ActivityTracker activityTracker;
  private final RoleTracker roleTracker;
  private final TeamDetector teamDetector;
  private final SwarmProgressTracker progressTracker;
  private final DispatchBudget dispatchBudget;
  private Instance<io.casehub.eidos.api.CapabilityHealth> capabilityHealthInstance;
  private Instance<SwarmProvisioningAdvisor> advisorInstance;

  private final ConcurrentHashMap<UUID, CaseProvisionState> cases = new ConcurrentHashMap<>();

  @Inject
  public SwarmProvisioner(
      WorkerProvisioner workerProvisioner,
      StigmergyCoordinator coordinator,
      SignalRegistry signalRegistry,
      ActivityTracker activityTracker,
      RoleTracker roleTracker,
      TeamDetector teamDetector,
      SwarmProgressTracker progressTracker,
      DispatchBudget dispatchBudget) {
    this.workerProvisioner = workerProvisioner;
    this.coordinator = coordinator;
    this.signalRegistry = signalRegistry;
    this.activityTracker = activityTracker;
    this.roleTracker = roleTracker;
    this.teamDetector = teamDetector;
    this.progressTracker = progressTracker;
    this.dispatchBudget = dispatchBudget;
  }

  public List<SwarmEvent> evaluateAndProvision(
      UUID caseId, SwarmConfig swarmConfig, StigmergyConfig stigConfig) {
    var state = cases.computeIfAbsent(caseId, k -> new CaseProvisionState());

    if (!state.provisionLock.tryLock()) {
      return List.of();
    }
    try {
      return doProvision(caseId, swarmConfig, stigConfig, state);
    } finally {
      state.provisionLock.unlock();
    }
  }

  private List<SwarmEvent> doProvision(
      UUID caseId, SwarmConfig swarmConfig, StigmergyConfig stigConfig,
      CaseProvisionState state) {
    List<SwarmEvent> events = new ArrayList<>();

    int currentActive = coordinator.activeAgents(caseId).size();
    if (currentActive >= swarmConfig.effectiveMaxSwarmSize()) {
      events.add(new SwarmEvent(CaseHubEventType.SWARM_PROVISION_BUDGET_EXHAUSTED,
          Map.of("reason", "maxSwarmSize", "current", currentActive,
              "limit", swarmConfig.effectiveMaxSwarmSize())));
      return events;
    }

    var budget = swarmConfig.provisionBudget() != null
        ? swarmConfig.provisionBudget() : new ProvisionBudget(null, null, null, null, null);

    if (state.totalProvisions.get() >= budget.effectiveMaxProvisions()) {
      events.add(new SwarmEvent(CaseHubEventType.SWARM_PROVISION_BUDGET_EXHAUSTED,
          Map.of("reason", "maxProvisions", "current", state.totalProvisions.get(),
              "limit", budget.effectiveMaxProvisions())));
      return events;
    }

    long swarmProvisioned = state.provisionedAgents.values().stream()
        .filter(m -> isAgentActive(caseId, m.agentId())).count();
    if (swarmProvisioned >= budget.effectiveMaxConcurrent()) {
      events.add(new SwarmEvent(CaseHubEventType.SWARM_PROVISION_BUDGET_EXHAUSTED,
          Map.of("reason", "maxConcurrent", "current", swarmProvisioned,
              "limit", budget.effectiveMaxConcurrent())));
      return events;
    }

    if (state.cycleCount.get() - state.lastProvisionCycle.get()
        < budget.effectiveCooldownCycles()) {
      events.add(new SwarmEvent(CaseHubEventType.SWARM_PROVISION_BUDGET_EXHAUSTED,
          Map.of("reason", "cooldown",
              "cyclesSinceLastProvision",
              state.cycleCount.get() - state.lastProvisionCycle.get(),
              "cooldownRequired", budget.effectiveCooldownCycles())));
      return events;
    }

    if (dispatchBudget.availableCapacity(
        new DispatchBudgetQuery(caseId, tenancyId(caseId))) <= 0) {
      events.add(new SwarmEvent(CaseHubEventType.SWARM_PROVISION_BUDGET_EXHAUSTED,
          Map.of("reason", "dispatchBudget")));
      return events;
    }

    events.add(new SwarmEvent(CaseHubEventType.SWARM_PROVISION_REQUESTED,
        Map.of("caseId", caseId, "activeAgents", currentActive)));

    if (advisorInstance != null && advisorInstance.isResolvable()) {
      var advisor = advisorInstance.get();
      var bootstrapCtx = buildBootstrapContext(caseId, swarmConfig, null);
      var adviceCtx = new SwarmProvisioningAdvisor.ProvisioningContext(
          caseId, bootstrapCtx, Set.of(), swarmConfig.integrationPolicy());
      var advice = advisor.advise(adviceCtx);
      if (!advice.shouldProvision()) {
        events.add(new SwarmEvent(CaseHubEventType.SWARM_PROVISION_VETOED,
            Map.of("reasoning", advice.reasoning() != null ? advice.reasoning() : "")));
        return events;
      }
    }

    String bindingName = "swarm-" + caseId.toString().substring(0, 8)
        + "-" + state.provisionCounter.incrementAndGet();

    var provisionContext = new ProvisionContext(
        caseId, tenancyId(caseId), "swarm-agent", null, null, null, null, null);

    try {
      var result = workerProvisioner.provision(
          workerProvisioner.getCapabilities(), provisionContext);

      String agentId = result.resolvedWorkerId() != null
          ? result.resolvedWorkerId() : bindingName;
      coordinator.agentJoined(caseId, agentId, bindingName);
      coordinator.agentActivated(caseId, agentId);

      var policy = swarmConfig.integrationPolicy() != null
          ? swarmConfig.integrationPolicy() : IntegrationPolicy.BALANCED;

      state.provisionedAgents.put(agentId, new ProvisionedAgentMeta(
          agentId, bindingName, state.cycleCount.get(),
          policy.effectiveIntegrationDelay(), Set.of(), Instant.now(), 0));
      state.totalProvisions.incrementAndGet();
      state.lastProvisionCycle.set(state.cycleCount.get());

      events.add(new SwarmEvent(CaseHubEventType.SWARM_PROVISION_COMPLETED,
          Map.of("agentId", agentId, "bindingName", bindingName,
              "integrationDelay", policy.effectiveIntegrationDelay())));
    } catch (ProvisioningException e) {
      events.add(new SwarmEvent(CaseHubEventType.SWARM_PROVISION_FAILED,
          Map.of("reason", e.getMessage() != null ? e.getMessage() : "unknown")));
    }
    return events;
  }

  public List<SwarmEvent> evaluateDeprovisioning(UUID caseId, SwarmConfig config) {
    var state = cases.get(caseId);
    if (state == null) return List.of();

    var budget = config.provisionBudget() != null
        ? config.provisionBudget() : new ProvisionBudget(null, null, null, null, null);

    List<SwarmEvent> events = new ArrayList<>();
    List<String> toRemove = new ArrayList<>();

    for (var entry : state.provisionedAgents.entrySet()) {
      var meta = entry.getValue();

      if (meta.integrationDelayRemaining() > 0) continue;

      if (!isAgentActive(caseId, meta.agentId())) {
        toRemove.add(meta.agentId());
        continue;
      }

      var activityState = activityTracker.getState(caseId);
      double rate = activityState.dispatchRate(Duration.ofSeconds(30), Instant.now());

      if (rate < budget.effectiveIdleThreshold()) {
        int newIdleCycles = meta.consecutiveIdleCycles() + 1;
        state.provisionedAgents.put(meta.agentId(), new ProvisionedAgentMeta(
            meta.agentId(), meta.bindingName(), meta.provisionedAtCycle(),
            meta.integrationDelayRemaining(), meta.requestedCapabilities(),
            meta.provisionedAt(), newIdleCycles));

        if (newIdleCycles >= budget.effectiveIdleGraceCycles()) {
          if (!isProvisioningSignalReinforced(caseId)) {
            coordinator.agentDeparted(caseId, meta.agentId());
            workerProvisioner.terminate(meta.agentId(), tenancyId(caseId));
            toRemove.add(meta.agentId());
            events.add(new SwarmEvent(CaseHubEventType.SWARM_AGENT_TERMINATED,
                Map.of("agentId", meta.agentId(), "reason", "idle",
                    "idleCycles", newIdleCycles)));
          }
        }
      } else {
        state.provisionedAgents.put(meta.agentId(), new ProvisionedAgentMeta(
            meta.agentId(), meta.bindingName(), meta.provisionedAtCycle(),
            meta.integrationDelayRemaining(), meta.requestedCapabilities(),
            meta.provisionedAt(), 0));
      }
    }

    for (String id : toRemove) {
      state.provisionedAgents.remove(id);
    }
    return events;
  }

  private boolean isProvisioningSignalReinforced(UUID caseId) {
    var allSignals = signalRegistry.getAllSignals(caseId);
    return allSignals.keySet().stream()
        .filter(name -> name.startsWith("swarm:need-capacity"))
        .anyMatch(name -> !allSignals.get(name).expired());
  }

  public boolean isInIntegrationDelay(UUID caseId, String agentId) {
    var state = cases.get(caseId);
    if (state == null) return false;
    var meta = state.provisionedAgents.get(agentId);
    return meta != null && meta.integrationDelayRemaining() > 0;
  }

  public void decrementIntegrationDelay(UUID caseId, String agentId) {
    var state = cases.get(caseId);
    if (state == null) return;
    var meta = state.provisionedAgents.get(agentId);
    if (meta != null && meta.integrationDelayRemaining() > 0) {
      state.provisionedAgents.put(agentId, new ProvisionedAgentMeta(
          meta.agentId(), meta.bindingName(), meta.provisionedAtCycle(),
          meta.integrationDelayRemaining() - 1, meta.requestedCapabilities(),
          meta.provisionedAt(), meta.consecutiveIdleCycles()));
    }
  }

  public void incrementCycleCount(UUID caseId) {
    cases.computeIfAbsent(caseId, k -> new CaseProvisionState()).cycleCount.incrementAndGet();
  }

  SwarmBootstrapContext buildBootstrapContext(
      UUID caseId, SwarmConfig config, ProvisioningRequest request) {
    var policy = config.integrationPolicy() != null
        ? config.integrationPolicy() : IntegrationPolicy.BALANCED;
    double richness = policy.effectiveBootstrapRichness();

    Map<String, Double> activeSignals = Map.of();
    Set<String> activeInterestKeys = Set.of();
    List<DetectedRole> currentRoles = List.of();
    List<DetectedTeam> currentTeams = List.of();
    SwarmProgress swarmProgress = SwarmProgress.EMPTY;

    if (richness > 0.0) {
      swarmProgress = progressTracker.getProgress(caseId);
    }
    if (richness > 0.3) {
      var allSignals = signalRegistry.getAllSignals(caseId);
      Map<String, Double> sigMap = new HashMap<>();
      for (var entry : allSignals.entrySet()) {
        if (!entry.getValue().expired()) {
          sigMap.put(entry.getKey(), entry.getValue().strength());
        }
      }
      activeSignals = sigMap;
      activeInterestKeys = Set.of();
    }
    if (richness > 0.6) {
      currentRoles = roleTracker.getDetectedRoles(caseId);
      currentTeams = teamDetector.getDetectedTeams(caseId);
    }

    return new SwarmBootstrapContext(
        activeSignals, activeInterestKeys, currentRoles, currentTeams,
        swarmProgress, request, policy);
  }

  private boolean isAgentActive(UUID caseId, String agentId) {
    return coordinator.activeAgents(caseId).stream()
        .anyMatch(a -> a.agentId().equals(agentId));
  }

  private String tenancyId(UUID caseId) {
    return null;
  }

  public void evictByCase(UUID caseId) {
    cases.remove(caseId);
  }

  @Override
  public void reset() {
    cases.clear();
  }

  record ProvisionedAgentMeta(
      String agentId,
      String bindingName,
      int provisionedAtCycle,
      int integrationDelayRemaining,
      Set<String> requestedCapabilities,
      Instant provisionedAt,
      int consecutiveIdleCycles) {

    ProvisionedAgentMeta {
      requestedCapabilities = Set.copyOf(requestedCapabilities);
    }
  }

  private static class CaseProvisionState {
    final ReentrantLock provisionLock = new ReentrantLock();
    final AtomicInteger totalProvisions = new AtomicInteger();
    final AtomicInteger lastProvisionCycle = new AtomicInteger(-1000);
    final AtomicInteger cycleCount = new AtomicInteger();
    final AtomicInteger provisionCounter = new AtomicInteger();
    final ConcurrentHashMap<String, ProvisionedAgentMeta> provisionedAgents =
        new ConcurrentHashMap<>();
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime-core -Dtest=SwarmProvisionerTest -Dcheckstyle.skip=true -q`
Expected: PASS (5 tests)

- [ ] **Step 5: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/engine/internal/stigmergy/SwarmProvisioner.java \
  runtime-core/src/test/java/io/casehub/engine/internal/stigmergy/SwarmProvisionerTest.java
git commit -m "feat: add SwarmProvisioner — budget enforcement, provisioning, de-provisioning

Refs #1113"
```

---

## Batch 3: Integration Model + RoleTracker Awareness

### Task 3: RoleTracker integration delay awareness

Modify RoleTracker to check SwarmProvisioner for agents in integration delay — accumulate their data but exclude from role clustering.

**Files:**
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/stigmergy/RoleTracker.java:36-64` — add `Instance<SwarmProvisioner>`, check integration delay in `accumulate()`
- Create: `runtime-core/src/test/java/io/casehub/engine/internal/stigmergy/IntegrationDelayTest.java`

**Interfaces:**
- Consumes: `SwarmProvisioner.isInIntegrationDelay(UUID, String) → boolean`, `SwarmProvisioner.decrementIntegrationDelay(UUID, String)`
- Produces: Modified `RoleTracker.accumulate()` that respects integration delay

- [ ] **Step 1: Write failing test — agent in integration delay excluded from role detection**

Use `ide_create_file`:

```java
package io.casehub.engine.internal.stigmergy;

import static org.junit.jupiter.api.Assertions.*;

import io.casehub.api.model.stigmergy.*;
import io.casehub.api.spi.*;
import io.casehub.engine.common.internal.convergence.ActivityTracker;
import io.casehub.engine.common.internal.observation.ObservationRegistry;
import io.casehub.engine.common.internal.observation.RuleRegistry;
import io.casehub.engine.common.internal.signal.SignalRegistry;
import io.casehub.api.spi.observation.*;
import java.time.Duration;
import java.util.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class IntegrationDelayTest {

  private RoleTracker roleTracker;
  private SwarmProvisioner swarmProvisioner;
  private StigmergyCoordinator coordinator;
  private ObservationRegistry observationRegistry;
  private SignalRegistry signalRegistry;
  private ActivityTracker activityTracker;
  private UUID caseId;

  @BeforeEach
  void setUp() {
    observationRegistry = new ObservationRegistry();
    signalRegistry = new SignalRegistry();
    var ruleRegistry = new RuleRegistry();
    activityTracker = new ActivityTracker();
    coordinator = new StigmergyCoordinator(signalRegistry, observationRegistry, activityTracker);
    var teamDetector = new TeamDetector(observationRegistry, signalRegistry, null, coordinator);
    var progressTracker = new SwarmProgressTracker(signalRegistry, null, coordinator);
    var workerProvisioner = new SwarmProvisionerTest.RecordingWorkerProvisioner();
    swarmProvisioner = new SwarmProvisioner(
        workerProvisioner, coordinator, signalRegistry, activityTracker,
        null, teamDetector, progressTracker,
        new SwarmProvisionerTest.NoOpDispatchBudget());
    roleTracker = new RoleTracker(
        observationRegistry, signalRegistry, ruleRegistry, coordinator, swarmProvisioner);
    caseId = UUID.randomUUID();
  }

  @Test
  void agentInIntegrationDelayExcludedFromRoleDetection() {
    var budget = new ProvisionBudget(50, 10, 0, null, null);
    var policy = new IntegrationPolicy(0.7, 5, 0.5);
    var swarmConfig = new SwarmConfig(
        20, 0.5, 2, 20, 1, null, null, null, null, budget, policy);
    var config = new StigmergyConfig(null, null, swarmConfig);
    coordinator.initializeCase(caseId, List.of("a1", "a2"), config);
    for (String id : List.of("a1", "a2")) {
      coordinator.agentJoined(caseId, id, "b-" + id);
      coordinator.agentActivated(caseId, id);
    }

    observationRegistry.registerObserver(
        caseId, "a1", "b-a1", new TestObserver("obs", Set.of("temp")), 20);
    observationRegistry.registerObserver(
        caseId, "a2", "b-a2", new TestObserver("obs", Set.of("temp")), 20);

    signalRegistry.deposit(caseId, "swarm:need-capacity", 0.8, Duration.ofMinutes(5), "a1", 100);
    signalRegistry.deposit(caseId, "swarm:need-capacity", 0.8, Duration.ofMinutes(5), "a2", 100);
    swarmProvisioner.evaluateAndProvision(caseId, swarmConfig, config);

    String provisionedId = ((SwarmProvisionerTest.RecordingWorkerProvisioner)
        getWorkerProvisioner()).lastProvisionedId();
    observationRegistry.registerObserver(
        caseId, provisionedId, "b-" + provisionedId,
        new TestObserver("obs", Set.of("temp")), 20);

    roleTracker.accumulate(caseId);
    var roles = roleTracker.detect(caseId, swarmConfig);

    var detectedRoles = roleTracker.getDetectedRoles(caseId);
    for (var role : detectedRoles) {
      assertFalse(role.memberAgents().contains(provisionedId),
          "Agent in integration delay should not appear in role clusters");
    }
  }

  private WorkerProvisioner getWorkerProvisioner() {
    return null;
  }

  static class TestObserver implements EnvironmentObserver {
    private final String type;
    private final Set<String> keys;
    TestObserver(String type, Set<String> keys) { this.type = type; this.keys = keys; }
    @Override public String observerType() { return type; }
    @Override public Set<String> watchedKeys() { return keys; }
    @Override public List<Observation> observe(ObservationContext context) { return List.of(); }
  }
}
```

- [ ] **Step 2: Modify RoleTracker — add SwarmProvisioner Instance and integration delay check**

Add an `Instance<SwarmProvisioner>` field to RoleTracker. In the `accumulate()` method, after computing agents from `coordinator.activeAgents()`, check each agent for integration delay:

Use `ide_replace_text_in_file` to update the constructor to accept an optional `SwarmProvisioner` parameter (or `Instance<SwarmProvisioner>` for CDI). Add a package-private constructor overload for testing.

In `accumulate()`, modify the loop:
```java
for (var agent : agents) {
    if (swarmProvisioner != null
        && swarmProvisioner.isInIntegrationDelay(caseId, agent.agentId())) {
        accumulateAgent(caseId, agent.agentId(), state);
        swarmProvisioner.decrementIntegrationDelay(caseId, agent.agentId());
        state.agentsInDelay.add(agent.agentId());
    } else {
        accumulateAgent(caseId, agent.agentId(), state);
    }
}
```

In `detect()`, when building `fingerprints` map, skip agents that are in `state.agentsInDelay`.

- [ ] **Step 3: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime-core -Dtest=IntegrationDelayTest -Dcheckstyle.skip=true -q`
Expected: PASS

- [ ] **Step 4: Run full RoleTracker tests for regression**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime-core -Dtest=RoleTrackerTest -Dcheckstyle.skip=true -q`
Expected: PASS — no regressions

- [ ] **Step 5: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/engine/internal/stigmergy/RoleTracker.java \
  runtime-core/src/test/java/io/casehub/engine/internal/stigmergy/IntegrationDelayTest.java
git commit -m "feat: add integration delay awareness to RoleTracker

Agents in their integration delay window are accumulated but excluded
from role clustering. Delay decrements each cycle.

Refs #1113"
```

---

## Batch 4: Pipeline Wiring + Integration Test

### Task 4: Wire SwarmProvisioner into handlers, RuntimeBeans, and Spring config

Add `Instance<SwarmProvisioner>` to CaseContextChangedEventHandler and CaseStatusChangedHandler. Add provisioning consensus check and de-provisioning evaluation in convergenceDetection(). Add eviction in terminal state block. Update RuntimeBeans and RuntimeManualConfig producers.

**Files:**
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java` — add `Instance<SwarmProvisioner>` field + constructor param, add provisioning check in convergenceDetection()
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/engine/handler/CaseStatusChangedHandler.java` — add `Instance<SwarmProvisioner>` field + constructor param, add eviction
- Modify: `runtime/src/main/java/io/casehub/engine/internal/quarkus/RuntimeBeans.java` — update CaseContextChangedEventHandler and CaseStatusChangedHandler producers
- Modify: `runtime-spring/src/main/java/io/casehub/engine/internal/spring/RuntimeManualConfig.java` — update handler constructor calls

**Interfaces:**
- Consumes: `SwarmProvisioner.evaluateAndProvision(UUID, SwarmConfig, StigmergyConfig) → List<SwarmEvent>`, `SwarmProvisioner.evaluateDeprovisioning(UUID, SwarmConfig) → List<SwarmEvent>`, `SwarmProvisioner.incrementCycleCount(UUID)`, `SwarmProvisioner.evictByCase(UUID)`
- Produces: Wired pipeline with swarm provisioning active during evaluation cycles

- [ ] **Step 1: Add SwarmProvisioner Instance to CaseContextChangedEventHandler**

Add field after `swarmProgressTrackerInstance`:
```java
private final jakarta.enterprise.inject.Instance<
        io.casehub.engine.internal.stigmergy.SwarmProvisioner>
    swarmProvisionerInstance;
```

Add constructor parameter after the swarmProgressTrackerInstance param. Add field assignment in constructor body.

- [ ] **Step 2: Add provisioning logic in convergenceDetection()**

After the existing swarm detection block (after `spt.evaluate(...)` at line ~1437), inside the `if (swarmConfig != null)` block, add:

```java
if (swarmProvisionerInstance.isResolvable()) {
    var provisioner = swarmProvisionerInstance.get();
    provisioner.incrementCycleCount(caseInstance.getUuid());
    if (swarmConfig.provisionBudget() != null) {
        double ezThreshold = 0.01;
        int minSources = 2;
        if (stigConfig.defaults() != null
            && stigConfig.defaults().effectiveZeroThreshold() != null) {
            ezThreshold = stigConfig.defaults().effectiveZeroThreshold();
        }
        if (stigConfig.coordination() != null
            && stigConfig.coordination().consensusThreshold() != null) {
            minSources = stigConfig.coordination().consensusThreshold();
        }
        var consensus = signalRegistry.consensusSignals(
            caseInstance.getUuid(), minSources, ezThreshold);
        if (consensus.keySet().stream()
            .anyMatch(n -> n.startsWith("swarm:need-capacity"))) {
            var provEvents = provisioner.evaluateAndProvision(
                caseInstance.getUuid(), swarmConfig, stigConfig);
            for (var e : provEvents) {
                LOG.debugf("Swarm provision event for caseId=%s type=%s",
                    caseInstance.getUuid(), e.type());
            }
        }
        var deprovEvents = provisioner.evaluateDeprovisioning(
            caseInstance.getUuid(), swarmConfig);
        for (var e : deprovEvents) {
            LOG.debugf("Swarm deprovision event for caseId=%s type=%s",
                caseInstance.getUuid(), e.type());
        }
    }
}
```

- [ ] **Step 3: Add SwarmProvisioner Instance to CaseStatusChangedHandler**

Add field, constructor param, and field assignment (same pattern as other Instance fields).

Add eviction in the terminal state block after `swarmProgressTrackerInstance` eviction:
```java
if (swarmProvisionerInstance.isResolvable()) {
    swarmProvisionerInstance.get().evictByCase(caseInstance.getUuid());
}
```

- [ ] **Step 4: Update RuntimeBeans producers**

Add `Instance<io.casehub.engine.internal.stigmergy.SwarmProvisioner> swarmProvisioner` parameter to both handler producers. Pass through to the constructor calls.

- [ ] **Step 5: Update RuntimeManualConfig (Spring)**

Add `notResolvable()` for the new `Instance<SwarmProvisioner>` parameter in both handler constructor calls.

- [ ] **Step 6: Compile runtime module**

Run: `/opt/homebrew/bin/mvn install -DskipTests -Dcheckstyle.skip=true -q`
Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java \
  runtime-core/src/main/java/io/casehub/engine/internal/engine/handler/CaseStatusChangedHandler.java \
  runtime/src/main/java/io/casehub/engine/internal/quarkus/RuntimeBeans.java \
  runtime-spring/src/main/java/io/casehub/engine/internal/spring/RuntimeManualConfig.java
git commit -m "feat: wire SwarmProvisioner into pipeline — consensus detection, de-provisioning, eviction

Refs #1113"
```

### Task 5: Self-provisioning integration test

End-to-end test verifying the full provisioning lifecycle: agents deposit signals, consensus triggers provisioning, new agent joins swarm, integration delay works, idle detection triggers de-provisioning.

**Files:**
- Create: `runtime-core/src/test/java/io/casehub/engine/internal/stigmergy/SelfProvisioningIntegrationTest.java`

**Interfaces:**
- Consumes: all types from Tasks 1-4

- [ ] **Step 1: Write integration test**

Use `ide_create_file`:

```java
package io.casehub.engine.internal.stigmergy;

import static org.junit.jupiter.api.Assertions.*;

import io.casehub.api.model.event.CaseHubEventType;
import io.casehub.api.model.stigmergy.*;
import io.casehub.api.spi.*;
import io.casehub.api.spi.observation.*;
import io.casehub.engine.common.internal.convergence.ActivityTracker;
import io.casehub.engine.common.internal.observation.ObservationRegistry;
import io.casehub.engine.common.internal.observation.RuleRegistry;
import io.casehub.engine.common.internal.signal.SignalRegistry;
import java.time.Duration;
import java.util.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class SelfProvisioningIntegrationTest {

  private ObservationRegistry observationRegistry;
  private SignalRegistry signalRegistry;
  private RuleRegistry ruleRegistry;
  private ActivityTracker activityTracker;
  private StigmergyCoordinator coordinator;
  private RoleTracker roleTracker;
  private TeamDetector teamDetector;
  private SwarmProgressTracker progressTracker;
  private SwarmProvisioner swarmProvisioner;
  private SwarmProvisionerTest.RecordingWorkerProvisioner workerProvisioner;
  private UUID caseId;
  private SwarmConfig swarmConfig;
  private StigmergyConfig config;

  @BeforeEach
  void setUp() {
    observationRegistry = new ObservationRegistry();
    signalRegistry = new SignalRegistry();
    ruleRegistry = new RuleRegistry();
    activityTracker = new ActivityTracker();
    coordinator = new StigmergyCoordinator(signalRegistry, observationRegistry, activityTracker);
    teamDetector = new TeamDetector(observationRegistry, signalRegistry, null, coordinator);
    progressTracker = new SwarmProgressTracker(signalRegistry, null, coordinator);
    workerProvisioner = new SwarmProvisionerTest.RecordingWorkerProvisioner();
    swarmProvisioner = new SwarmProvisioner(
        workerProvisioner, coordinator, signalRegistry, activityTracker,
        null, teamDetector, progressTracker,
        new SwarmProvisionerTest.NoOpDispatchBudget());
    roleTracker = new RoleTracker(
        observationRegistry, signalRegistry, ruleRegistry, coordinator, swarmProvisioner);

    caseId = UUID.randomUUID();
    var budget = new ProvisionBudget(10, 5, 0, 0.01, 2);
    var policy = new IntegrationPolicy(0.7, 2, 0.5);
    swarmConfig = new SwarmConfig(
        10, 0.5, 2, 20, 1, null, 0.3, 2, 0.05, budget, policy);
    config = new StigmergyConfig(
        new StigmergyDefaults(null, 0.01, null, null, null, null, null, null, null),
        new CoordinationConfig(2, 10.0, 0.6),
        swarmConfig);
  }

  @Test
  void fullProvisioningLifecycle() {
    coordinator.initializeCase(caseId, List.of("monitor-1", "monitor-2"), config);
    for (String id : List.of("monitor-1", "monitor-2")) {
      coordinator.agentJoined(caseId, id, "b-" + id);
      coordinator.agentActivated(caseId, id);
    }
    assertEquals(2, coordinator.activeAgents(caseId).size());

    signalRegistry.deposit(caseId, "swarm:need-capacity", 0.8,
        Duration.ofMinutes(5), "monitor-1", 100);
    signalRegistry.deposit(caseId, "swarm:need-capacity", 0.8,
        Duration.ofMinutes(5), "monitor-2", 100);

    var events = swarmProvisioner.evaluateAndProvision(caseId, swarmConfig, config);
    assertTrue(events.stream().anyMatch(
        e -> e.type() == CaseHubEventType.SWARM_PROVISION_COMPLETED));
    assertEquals(3, coordinator.activeAgents(caseId).size());

    String provisionedId = workerProvisioner.lastProvisionedId();
    assertTrue(swarmProvisioner.isInIntegrationDelay(caseId, provisionedId));

    swarmProvisioner.decrementIntegrationDelay(caseId, provisionedId);
    swarmProvisioner.decrementIntegrationDelay(caseId, provisionedId);
    assertFalse(swarmProvisioner.isInIntegrationDelay(caseId, provisionedId));
  }

  @Test
  void bootstrapContextReflectsRichnessAxis() {
    coordinator.initializeCase(caseId, List.of("a1"), config);
    coordinator.agentJoined(caseId, "a1", "b-a1");
    coordinator.agentActivated(caseId, "a1");

    signalRegistry.deposit(caseId, "alert", 0.9, Duration.ofMinutes(5), "a1", 100);

    var lowRichness = new SwarmConfig(
        10, null, null, null, null, null, null, null, null,
        null, new IntegrationPolicy(0.1, 0, 0.5));
    var ctx = swarmProvisioner.buildBootstrapContext(caseId, lowRichness, null);
    assertTrue(ctx.activeSignals().isEmpty());

    var highRichness = new SwarmConfig(
        10, null, null, null, null, null, null, null, null,
        null, new IntegrationPolicy(0.9, 0, 0.5));
    var ctx2 = swarmProvisioner.buildBootstrapContext(caseId, highRichness, null);
    assertFalse(ctx2.activeSignals().isEmpty());
  }

  @Test
  void evictionCleansAllProvisioningState() {
    coordinator.initializeCase(caseId, List.of("a1", "a2"), config);
    for (String id : List.of("a1", "a2")) {
      coordinator.agentJoined(caseId, id, "b-" + id);
      coordinator.agentActivated(caseId, id);
    }
    signalRegistry.deposit(caseId, "swarm:need-capacity", 0.8,
        Duration.ofMinutes(5), "a1", 100);
    signalRegistry.deposit(caseId, "swarm:need-capacity", 0.8,
        Duration.ofMinutes(5), "a2", 100);
    swarmProvisioner.evaluateAndProvision(caseId, swarmConfig, config);

    swarmProvisioner.evictByCase(caseId);
    assertFalse(swarmProvisioner.isInIntegrationDelay(caseId, "any"));
  }

  static class TestObserver implements EnvironmentObserver {
    private final String type;
    private final Set<String> keys;
    TestObserver(String type, Set<String> keys) { this.type = type; this.keys = keys; }
    @Override public String observerType() { return type; }
    @Override public Set<String> watchedKeys() { return keys; }
    @Override public List<Observation> observe(ObservationContext ctx) { return List.of(); }
  }
}
```

- [ ] **Step 2: Run integration test**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime-core -Dtest=SelfProvisioningIntegrationTest -Dcheckstyle.skip=true -q`
Expected: PASS

- [ ] **Step 3: Run full runtime-core test suite**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime-core -Dcheckstyle.skip=true -q`
Expected: PASS — no regressions

- [ ] **Step 4: Commit**

```bash
git add runtime-core/src/test/java/io/casehub/engine/internal/stigmergy/SelfProvisioningIntegrationTest.java
git commit -m "test: add self-provisioning integration test — full lifecycle verification

Refs #1113"
```

## References

- [2026-09-19-self-provisioning-swarm-design.md] — design spec this plan implements
- [WorkerProvisioner.java:34] — existing provisioning SPI
- [ProvisionContext.java:40] — existing provisioning context record
- [ProvisionResult.java:38] — existing provisioning result record
- [SwarmConfig.java:20] — existing swarm configuration (extends with provisionBudget, integrationPolicy)
- [StigmergyCoordinator.java:32] — agent lifecycle
- [SignalRegistry.java:187] — consensusSignals()
- [ActivityTracker.java:25] — activity rate tracking
- [DispatchBudget.java:29] — external capacity check SPI
- [RoleTracker.java:36] — fingerprinting (modified for integration delay awareness)
- [CaseContextChangedEventHandler.java:1407] — convergenceDetection() insertion point
- [CaseStatusChangedHandler.java:229] — terminal state eviction block
- [RuntimeBeans.java:898] — CaseContextChangedEventHandler producer
- [RuntimeBeans.java:397] — CaseStatusChangedHandler producer
- [D84–D91] — design decisions
- [GitHub #1113] — focal issue
- [GitHub #1104] — Hive Mind epic
