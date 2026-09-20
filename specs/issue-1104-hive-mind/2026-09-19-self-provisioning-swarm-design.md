# Self-Provisioning Swarm — Design Spec

**Issue:** casehubio/engine#1113
**Epic:** casehubio/engine#1104 (Hive Mind)
**Date:** 2026-09-19
**Decisions:** D84–D91

## Problem

The swarm execution model (#1112) tracks emergent roles, teams, and collective progress — but the swarm cannot grow or shrink. Agent membership is fixed at case start. When workload exceeds capacity, or when capacity exceeds demand, the swarm has no mechanism to adapt:

| Gap | What's missing |
|-----|---------------|
| Scale-up | Swarm cannot request more agents when overwhelmed |
| Scale-down | Idle agents persist, consuming resources |
| Capacity awareness | No link between swarm observation and provisioning infrastructure |
| Integration model | No protocol for how new agents join a running swarm |
| Budget control | No provisioning-specific budget enforcement |

The existing `WorkerProvisioner` SPI handles engine-initiated provisioning (PlanItem needs a worker, no matching worker exists). Self-provisioning inverts the trigger: the swarm observes workload and requests capacity through the signal/pheromone model. The engine acts on consensus among agents, not on PlanItem matching.

**Scope boundary:** This issue covers the full provisioning mechanism including CBR-based learning. LLM-powered provisioning reasoning is blocks#285 — this issue provides clean SPI extension points for it. Self-improvement of provisioning strategy is #1114.

## Architecture Overview

Self-provisioning extends the swarm execution model — same `StigmergyConfig.swarm` configuration surface. A new `SwarmProvisioner` bean coordinates the provisioning lifecycle, called from the evaluation pipeline when signal consensus indicates capacity need.

```
┌──────────────────────────────────────────────────────────────────────┐
│                    Self-Provisioning Swarm                            │
│                                                                      │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐   │
│  │ SwarmProvisioner  │  │ ProvisionBudget  │  │ IntegrationPolicy│   │
│  │                   │  │ Enforcer         │  │                  │   │
│  │ • consensus gate  │  │ • maxSwarmSize   │  │ • bootstrap      │   │
│  │ • eidos resolve   │  │ • maxProvisions  │  │   richness       │   │
│  │ • bootstrap ctx   │  │ • cooldownCycles │  │ • integration    │   │
│  │ • CBR query/store │  │ • DispatchBudget │  │   delay          │   │
│  │ • provision call  │  │                  │  │ • self-          │   │
│  │ • join + activate │  │                  │  │   determination  │   │
│  └────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘   │
│           │                     │                      │             │
│           ▼                     ▼                      ▼             │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                 Swarm Execution Model (#1112)                 │   │
│  │  RoleTracker · TeamDetector · SwarmProgressTracker            │   │
│  │  MetricsSpace · SwarmConfig                                   │   │
│  │                                                               │   │
│  │  ┌──────────────────────────────────────────────────────┐    │   │
│  │  │         Stigmergy Execution Model (#1111)             │    │   │
│  │  │  StigmergyCoordinator · SignalRegistry                │    │   │
│  │  │  ObservationRegistry · RuleRegistry · ActivityTracker │    │   │
│  │  └──────────────────────────────────────────────────────┘    │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                   Platform SPIs                               │   │
│  │  WorkerProvisioner · CapabilityHealth · DispatchBudget        │   │
│  │  AgentDescriptor (eidos) · ModelQuery (platform)              │   │
│  │  CbrCaseMemoryStore (neocortex)                               │   │
│  └──────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────┘
```

**Data flow — provisioning:**
1. Agents deposit `swarm:need-capacity` signal when overloaded
2. Signal reaches consensus (multiple sources reinforce)
3. Handler's `convergenceDetection()` detects consensus, reads provisioning request from context
4. Handler calls `SwarmProvisioner.evaluateAndProvision()`
5. SwarmProvisioner validates budget (3 layers) → queries CBR for past outcomes → resolves capabilities via eidos → builds bootstrap context → calls `WorkerProvisioner.provision()` → `agentJoined()` + `agentActivated()` → emits audit events → records CBR outcome

**Data flow — de-provisioning:**
1. `ActivityTracker` monitors agent activity rates each evaluation cycle
2. When rate drops below `idleThreshold` for `idleGraceCycles` consecutive cycles
3. AND the agent is not in its integration delay window
4. AND the original provisioning signal for this agent has decayed (no reinforcement)
5. SwarmProvisioner calls `WorkerProvisioner.terminate()` → `agentDeparted()` → cleanup → audit

## 1. Configuration Model

### ProvisionBudget

`ProvisionBudget` (`api`, `io.casehub.api.model.stigmergy`) — provisioning-specific budget enforcement:

```java
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

| Field | Default | Purpose |
|-------|---------|---------|
| `maxProvisions` | 50 | Lifetime cap on total provisioning actions per case |
| `maxConcurrent` | 10 | Maximum simultaneously active swarm-provisioned agents |
| `cooldownCycles` | 5 | Minimum evaluation cycles between successive provisions |
| `idleThreshold` | 0.01 | Activity rate below which an agent is considered idle |
| `idleGraceCycles` | 10 | Consecutive idle cycles before de-provisioning eligible |

### IntegrationPolicy

`IntegrationPolicy` (`api`, `io.casehub.api.model.stigmergy`) — three-axis tunable model for agent integration:

```java
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

| Axis | Range | Default | Meaning |
|------|-------|---------|---------|
| `bootstrapRichness` | 0.0–1.0 | 0.7 | How much swarm state the engine provides. 0.0 = no briefing. 1.0 = full signal/role/team/progress state transfer |
| `integrationDelay` | 0–N cycles | 3 | Cycles before the agent's fingerprint influences role/team detection. During delay, agent participates but doesn't destabilize clusters |
| `selfDetermination` | 0.0–1.0 | 0.5 | How much the agent decides its own interests/signals/rules. 0.0 = engine pre-registers based on the gap analysis. 1.0 = agent discovers everything autonomously from the bootstrap context |

### SwarmConfig Extension

`SwarmConfig` gains two optional nested records:

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
    @Nullable ProvisionBudget provisionBudget,       // NEW
    @Nullable IntegrationPolicy integrationPolicy) {  // NEW

  // existing effective* methods unchanged...

  public int effectiveMaxSwarmSize() {
    return maxSwarmSize != null ? maxSwarmSize : 20;
  }
}
```

Backward compat: existing `SwarmConfig` constructors without the new fields continue to work (positional records with `null` for new fields).

## 2. Provisioning Flow

### 2.1 Signal Convention

Agents request provisioning by depositing signals with the `swarm:` prefix:

- **`swarm:need-capacity`** — generic capacity request
- **`swarm:need-capability:<tag>`** — request for specific capability (e.g., `swarm:need-capability:analyst`)

The signal carries intent via its name. Detailed provisioning request metadata is written to the case context under a reserved key:

```java
public record ProvisioningRequest(
    Set<String> requiredCapabilities,
    @Nullable String preferredModelId,
    @Nullable String reason,
    String requestingAgentId,
    Instant requestedAt) {}
```

The agent writes this to `context.layer(WORKING)` at key `_swarm:provision-request:<agentId>`. When multiple agents deposit the same signal, multiple requests exist — the SwarmProvisioner reads all matching requests when consensus fires.

**Why signals + context, not signal metadata:** The existing `Signal` record has no metadata field. Extending it would change the shape of an established type used across all signal operations. Storing structured details in context and using signals purely for consensus (pheromone semantics) preserves the signal model's simplicity.

### 2.2 Consensus Detection

The evaluation pipeline's `convergenceDetection()` method already calls `signalRegistry.consensusSignals()`. Swarm provisioning adds a check for `swarm:need-capacity` signals in the consensus set:

```
if (stigmergyCoordinator.isStigmergyCase(caseId)) {
    // existing pattern detection...
    // existing swarm detection (roleTracker, teamDetector, progressTracker)...

    // NEW: provisioning consensus check
    if (swarmProvisioner.isResolvable()) {
        var consensusSignals = signalRegistry.consensusSignals(caseId, minSources, threshold);
        boolean hasProvisionConsensus = consensusSignals.keySet().stream()
            .anyMatch(name -> name.startsWith("swarm:need-capacity"));
        if (hasProvisionConsensus) {
            swarmProvisioner.get().evaluateAndProvision(caseId, caseDefinition);
        }
    }
}
```

The handler detects consensus; SwarmProvisioner orchestrates the response. SwarmProvisioner is NOT a polling agent — it is called when consensus is detected.

### 2.3 SwarmProvisioner

`SwarmProvisioner` (`runtime-core`, `io.casehub.engine.internal.stigmergy`, `@ApplicationScoped`, `Resettable`) — coordinates the full provisioning lifecycle:

```java
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
  private final Instance<CapabilityHealth> capabilityHealth;
  private final Instance<CbrRetrievalService> cbrService;
  private final Instance<SwarmProvisioningAdvisor> advisor;

  // Per-case state: lock, provision count, last provision cycle, provisioned agent metadata
  private final ConcurrentHashMap<UUID, CaseProvisionState> cases = new ConcurrentHashMap<>();
}
```

**Per-case lock:** `CaseProvisionState` holds a `ReentrantLock` to prevent concurrent evaluation cycles from double-provisioning (R1-04):

```java
private static class CaseProvisionState {
  final ReentrantLock provisionLock = new ReentrantLock();
  int totalProvisions;
  int lastProvisionCycle;
  final ConcurrentHashMap<String, ProvisionedAgentMeta> provisionedAgents = new ConcurrentHashMap<>();
}
```

```java
record ProvisionedAgentMeta(
    String agentId,
    String bindingName,
    int provisionedAtCycle,
    int integrationDelayRemaining,
    Set<String> requestedCapabilities,
    Instant provisionedAt) {}
```

### 2.4 `evaluateAndProvision()` Method

```
evaluateAndProvision(UUID caseId, CaseDefinition caseDefinition):
  1. Acquire per-case lock (tryLock, non-blocking — skip if contended)
  2. Read SwarmConfig from caseDefinition.getStigmergyConfig().swarm()
  3. Budget check — 3 layers (see §5 Budget Enforcement)
  4. Cooldown check — cycles since last provision >= cooldownCycles
  5. Read provisioning requests from context (_swarm:provision-request:*)
  6. Merge capabilities across all requesting agents
  7. CBR query — retrieve past provisioning outcomes for similar swarm state (see §4)
  8. Blocks advisor hook — if SwarmProvisioningAdvisor is resolvable, consult it (see §6)
  9. Resolve capabilities via eidos CapabilityHealth
 10. Generate binding name: "swarm-<caseId-short>-<counter>"
 11. Build ProvisionContext
 12. Build SwarmBootstrapContext (per IntegrationPolicy — see §3)
 13. Call workerProvisioner.provision(capabilities, provisionContext)
 14. On success:
     - coordinator.agentJoined(caseId, resolvedWorkerId, bindingName)
     - coordinator.agentActivated(caseId, resolvedWorkerId)
     - Register ProvisionedAgentMeta with integration delay
     - Emit SWARM_PROVISION_COMPLETED event
     - Record CBR outcome (success)
     - Increment provision count, update last provision cycle
     - Clear consumed provisioning request keys from context
 15. On ProvisioningException:
     - Emit SWARM_PROVISION_FAILED event
     - Record CBR outcome (failure, with exception message)
     - Do NOT retry — let the next consensus cycle re-evaluate
 16. Release lock
```

**Binding name generation (R2-01):** Swarm-provisioned agents get dynamically generated binding names: `"swarm-" + caseId.toString().substring(0, 8) + "-" + provisionCounter`. This distinguishes them from definition-declared agents and ensures uniqueness within the case.

## 3. Integration Model — Informed Emergence

The integration model is the bridge between engine-provided context and agent autonomy. It uses three independently tunable axes (D86) to control how new agents join the swarm.

### SwarmBootstrapContext

`SwarmBootstrapContext` (`api`, `io.casehub.api.model.stigmergy`) — informational briefing for newly provisioned agents:

```java
public record SwarmBootstrapContext(
    Map<String, Double> activeSignals,
    Set<String> activeInterestKeys,
    List<DetectedRole> currentRoles,
    List<DetectedTeam> currentTeams,
    SwarmProgress swarmProgress,
    ProvisioningRequest triggeringRequest,
    IntegrationPolicy policy) {}
```

**Bootstrap richness axis** controls what fields are populated:

| Richness | Populated fields |
|----------|-----------------|
| 0.0 | `triggeringRequest` + `policy` only (minimal — why you were provisioned) |
| 0.0–0.3 | + `swarmProgress` (aggregate metrics only) |
| 0.3–0.6 | + `activeSignals` + `activeInterestKeys` (environment state) |
| 0.6–1.0 | + `currentRoles` + `currentTeams` (full swarm topology) |

SwarmProvisioner builds the context by reading from existing registries based on the richness threshold. The context is placed into the `WorkerContext` that the provisioned agent receives on its first dispatch.

### Self-determination axis

Controls whether the engine pre-registers interests/rules for the new agent:

| Self-determination | Engine behavior |
|-------------------|----------------|
| 0.0 | Engine registers interests matching the capability gap, deposits an announcement signal, declares standard monitoring rules |
| 0.0–0.5 | Engine registers interests only (agent handles signals and rules) |
| 0.5–1.0 | Engine provides bootstrap context only — agent self-registers everything on first dispatch |
| 1.0 | No engine-side registration. Agent receives bootstrap context and acts entirely autonomously |

### Integration delay axis

Controls when the new agent's behavioral fingerprint influences swarm detection:

```java
// In RoleTracker.accumulate():
var meta = swarmProvisioner.getProvisionedAgentMeta(caseId, agentId);
if (meta != null && meta.integrationDelayRemaining() > 0) {
    // Accumulate data (agent IS active) but skip role clustering for this agent
    accumulateAgent(caseId, agentId, state);
    swarmProvisioner.decrementIntegrationDelay(caseId, agentId);
    return; // Don't include in fingerprint comparison this cycle
}
```

**Integration delay exempts from idle detection (R1-03):** An agent in its integration delay window is learning, not idle. The idle detection logic in de-provisioning explicitly checks `integrationDelayRemaining > 0` and skips the agent.

### Agent Self-Tuning (feed-forward for #1114)

Agents can propose tuning adjustments by writing to a reserved context key:

```
_swarm:tuning-proposal:<agentId> → {
    "bootstrapRichness": 0.9,
    "integrationDelay": 1,
    "selfDetermination": 0.8,
    "reason": "High bootstrap was essential — without role map I would have duplicated monitor-1's work"
}
```

SwarmProvisioner reads proposals during CBR outcome recording. Proposals are stored as CBR features for future retrieval — they don't change the current case's policy, but inform future provisioning decisions in similar contexts.

## 4. CBR Integration

### Query: Past Provisioning Outcomes

Before each provisioning decision, SwarmProvisioner queries CBR for past outcomes in similar swarm states:

```java
record SwarmProvisioningFeatures(
    int activeAgentCount,
    int detectedRoleCount,
    int detectedTeamCount,
    double explorationPace,
    double consensusScore,
    double stabilityScore,
    Set<String> requestedCapabilities,
    String caseType) {}
```

The query uses `CbrCaseMemoryStore` with a new case type registration `"swarm-provisioning"` backed by `FeatureVectorCbrCase`. Feature extraction converts swarm state into a vector for similarity search.

If matching past outcomes exist, SwarmProvisioner reads:
- Which `IntegrationPolicy` settings produced good outcomes (fast stabilization, minimal role disruption)
- Which capability resolutions matched well (eidos descriptor → actual agent behavior correlation)
- Whether provisioning in similar states was worthwhile (some states resolve without new agents)

**CBR-adjusted tuning:** If past outcomes suggest different tuning than the configured defaults, SwarmProvisioner adjusts the `IntegrationPolicy` for this provision. The adjustment is bounded — never more than ±0.3 from the configured value — to prevent CBR from completely overriding author intent.

### Record: Provisioning Outcome

After each provisioning action (success or failure), SwarmProvisioner records:

```java
record ProvisioningOutcome(
    SwarmProvisioningFeatures stateAtDecision,
    IntegrationPolicy policyUsed,
    IntegrationPolicy policyAdjusted,
    Set<String> resolvedCapabilities,
    boolean success,
    @Nullable String failureReason,
    @Nullable ProvisioningImpact impact) {}
```

```java
record ProvisioningImpact(
    int cyclesUntilStable,
    double explorationPaceChange,
    double stabilityScoreChange,
    int rolesAdded,
    int rolesDisrupted,
    @Nullable Map<String, Double> agentTuningProposal) {}
```

`ProvisioningImpact` is recorded asynchronously — it requires observing the swarm's state N cycles after provisioning to assess actual impact. SwarmProvisioner schedules a deferred impact assessment (default: 20 cycles after provisioning) that reads the swarm state and writes the impact to the CBR case.

## 5. Budget Enforcement

Three enforcement layers, all checked in `evaluateAndProvision()`:

### Layer 1: SwarmConfig.maxSwarmSize

Hard cap on total active agents (existing field, now enforced):

```java
int currentActive = coordinator.activeAgents(caseId).size();
if (currentActive >= swarmConfig.effectiveMaxSwarmSize()) {
    // emit SWARM_PROVISION_BUDGET_EXHAUSTED event
    return;
}
```

### Layer 2: ProvisionBudget

Rate limiting and lifetime caps:

```java
var budget = swarmConfig.provisionBudget();
var state = cases.get(caseId);

// Lifetime cap
if (state.totalProvisions >= budget.effectiveMaxProvisions()) return;

// Concurrent cap
long swarmProvisioned = state.provisionedAgents.values().stream()
    .filter(m -> isStillActive(caseId, m.agentId())).count();
if (swarmProvisioned >= budget.effectiveMaxConcurrent()) return;

// Cooldown
int currentCycle = getCycleCount(caseId);
if (currentCycle - state.lastProvisionCycle < budget.effectiveCooldownCycles()) return;
```

### Layer 3: DispatchBudget SPI

Cross-case capacity check via the existing external budget:

```java
if (dispatchBudget.availableCapacity(
        new DispatchBudgetQuery(caseId, tenancyId)) <= 0) {
    return;
}
```

## 6. De-Provisioning Flow

### Idle Detection

Each evaluation cycle, SwarmProvisioner checks all swarm-provisioned agents for idle status:

```java
public List<SwarmEvent> evaluateDeprovisioning(UUID caseId, SwarmConfig config) {
    var budget = config.provisionBudget();
    var state = cases.get(caseId);
    if (state == null) return List.of();

    List<SwarmEvent> events = new ArrayList<>();
    var activityState = activityTracker.getState(caseId);

    for (var entry : state.provisionedAgents.entrySet()) {
        var meta = entry.getValue();

        // Skip agents still in integration delay (R1-03)
        if (meta.integrationDelayRemaining() > 0) continue;

        double rate = computeAgentActivityRate(caseId, meta.agentId());
        if (rate < budget.effectiveIdleThreshold()) {
            meta.incrementIdleCycles();
            if (meta.consecutiveIdleCycles() >= budget.effectiveIdleGraceCycles()) {
                // Check: has the original provisioning signal decayed?
                if (!isProvisioningSignalReinforced(caseId, meta.requestedCapabilities())) {
                    terminate(caseId, meta, events);
                }
            }
        } else {
            meta.resetIdleCycles();
        }
    }
    return events;
}
```

### Signal Decay Check

The provisioning signal that originally justified this agent's existence must still be reinforced. If the swarm is no longer depositing `swarm:need-capacity` signals (the signal has decayed below the effective zero threshold), the justification for this agent has expired:

```java
private boolean isProvisioningSignalReinforced(UUID caseId, Set<String> capabilities) {
    var allSignals = signalRegistry.getAllSignals(caseId);
    return allSignals.keySet().stream()
        .filter(name -> name.startsWith("swarm:need-capacity"))
        .anyMatch(name -> !allSignals.get(name).expired());
}
```

### Termination

```java
private void terminate(UUID caseId, ProvisionedAgentMeta meta, List<SwarmEvent> events) {
    coordinator.agentDeparted(caseId, meta.agentId());
    workerProvisioner.terminate(meta.agentId(), tenancyId(caseId));
    cases.get(caseId).provisionedAgents.remove(meta.agentId());

    events.add(new SwarmEvent(CaseHubEventType.SWARM_AGENT_TERMINATED, Map.of(
        "agentId", meta.agentId(),
        "reason", "idle",
        "activeDuration", Duration.between(meta.provisionedAt(), Instant.now()),
        "idleCycles", meta.consecutiveIdleCycles())));

    recordDeProvisioningOutcome(caseId, meta);
}
```

`agentDeparted()` triggers `observationRegistry.unregisterByAgent()`, cleaning up all interests, signals, and rules the agent registered. The existing cleanup path handles this without new code.

## 7. Blocks Extension Points

### SwarmProvisioningAdvisor SPI

`SwarmProvisioningAdvisor` (`api`, `io.casehub.api.spi.stigmergy`) — SPI for blocks-side LLM reasoning about provisioning decisions:

```java
public interface SwarmProvisioningAdvisor {

  ProvisioningAdvice advise(ProvisioningContext context);

  record ProvisioningContext(
      UUID caseId,
      SwarmBootstrapContext swarmState,
      Set<String> requestedCapabilities,
      List<ProvisioningOutcome> pastOutcomes,
      IntegrationPolicy currentPolicy) {}

  record ProvisioningAdvice(
      boolean shouldProvision,
      @Nullable Set<String> adjustedCapabilities,
      @Nullable IntegrationPolicy adjustedPolicy,
      @Nullable String reasoning) {}
}
```

When `SwarmProvisioningAdvisor` is resolvable (blocks is on the classpath), SwarmProvisioner consults it before provisioning. The advisor can:
- Veto a provision (`shouldProvision = false`)
- Adjust the requested capabilities
- Adjust the integration policy for this specific provision
- Provide reasoning (logged to audit trail)

Default: no advisor. Engine proceeds with signal consensus + CBR + configured policy.

## 8. Event Types

New `CaseHubEventType` values:

```java
SWARM_PROVISION_REQUESTED,    // consensus detected, provisioning initiated
SWARM_PROVISION_COMPLETED,    // agent successfully provisioned and joined swarm
SWARM_PROVISION_FAILED,       // WorkerProvisioner.provision() threw ProvisioningException
SWARM_PROVISION_VETOED,       // SwarmProvisioningAdvisor vetoed the provision
SWARM_PROVISION_BUDGET_EXHAUSTED, // budget check failed (any of the 3 layers)
SWARM_AGENT_TERMINATED        // idle agent de-provisioned
```

All events are emitted as `SwarmEvent` records (existing type from #1112) and flow through the `LedgerTraceIdProvider` chain for audit.

## 9. Module Placement

| Type | Module | Package |
|------|--------|---------|
| `ProvisionBudget` | api | `io.casehub.api.model.stigmergy` |
| `IntegrationPolicy` | api | `io.casehub.api.model.stigmergy` |
| `SwarmBootstrapContext` | api | `io.casehub.api.model.stigmergy` |
| `ProvisioningRequest` | api | `io.casehub.api.model.stigmergy` |
| `SwarmProvisioningAdvisor` | api | `io.casehub.api.spi.stigmergy` |
| `SwarmProvisioner` | runtime-core | `io.casehub.engine.internal.stigmergy` |
| `CaseProvisionState` | runtime-core | `io.casehub.engine.internal.stigmergy` (inner class) |
| `ProvisionedAgentMeta` | runtime-core | `io.casehub.engine.internal.stigmergy` (inner class) |
| `SwarmProvisioningFeatures` | runtime-core | `io.casehub.engine.internal.stigmergy` |
| `ProvisioningOutcome` | runtime-core | `io.casehub.engine.internal.stigmergy` |
| `ProvisioningImpact` | runtime-core | `io.casehub.engine.internal.stigmergy` |
| 6 new `CaseHubEventType` values | api | `io.casehub.api.model.event` |

`SwarmConfig` gains 2 new nullable fields (same module, same package, backward-compatible).

## 10. Pipeline Integration

### CaseContextChangedEventHandler

In `convergenceDetection()`, after the existing swarm detection block, add:

```java
if (swarmProvisionerInstance.isResolvable()) {
    var provisioner = swarmProvisionerInstance.get();
    var stigConfig = caseDefinition.getStigmergyConfig();
    var swarmConfig = stigConfig != null ? stigConfig.swarm() : null;
    if (swarmConfig != null && swarmConfig.provisionBudget() != null) {
        // Check for provisioning consensus
        double ezThreshold = stigConfig.defaults() != null
            && stigConfig.defaults().effectiveZeroThreshold() != null
            ? stigConfig.defaults().effectiveZeroThreshold() : 0.01;
        int minSources = stigConfig.coordination() != null
            && stigConfig.coordination().consensusThreshold() != null
            ? stigConfig.coordination().consensusThreshold() : 2;
        var consensus = signalRegistry.consensusSignals(caseId, minSources, ezThreshold);
        if (consensus.keySet().stream().anyMatch(n -> n.startsWith("swarm:need-capacity"))) {
            provisioner.evaluateAndProvision(caseId, caseDefinition);
        }
        // Check for de-provisioning
        var deprovEvents = provisioner.evaluateDeprovisioning(caseId, swarmConfig);
        for (var e : deprovEvents) {
            LOG.debugf("Swarm de-provision for caseId=%s type=%s", caseId, e.type());
        }
    }
}
```

### CaseStatusChangedHandler

In the terminal state eviction block, add:

```java
if (swarmProvisionerInstance.isResolvable()) {
    swarmProvisionerInstance.get().evictByCase(caseInstance.getUuid());
}
```

### RuntimeBeans

- Add `Instance<SwarmProvisioner>` to handler producers (same pattern as other Instance params)
- No explicit `@Produces` for SwarmProvisioner — it is `@ApplicationScoped` with `@Inject` constructor, auto-discovered by CDI

### RoleTracker Integration

RoleTracker needs to check integration delay before including an agent in clustering:

```java
// SwarmProvisioner exposes:
public boolean isInIntegrationDelay(UUID caseId, String agentId)

// RoleTracker.accumulate() checks:
if (swarmProvisionerInstance.isResolvable()
    && swarmProvisionerInstance.get().isInIntegrationDelay(caseId, agentId)) {
    // accumulate data but exclude from clustering this cycle
}
```

## 11. Testing Strategy

### Unit Tests

1. **SwarmProvisionerTest** — budget enforcement (all 3 layers), cooldown, lock contention, consensus detection, binding name generation, integration delay tracking
2. **ProvisionBudgetTest** — effective defaults, boundary conditions
3. **IntegrationPolicyTest** — effective defaults, axis independence

### Integration Tests

1. **SwarmProvisioningIntegrationTest** — full lifecycle:
   - Set up 2 agents, deposit provisioning signals, verify consensus triggers provisioning
   - Verify budget enforcement blocks over-provisioning
   - Verify cooldown prevents burst provisioning
   - Verify new agent appears in `coordinator.activeAgents()`
   - Verify integration delay exempts from role detection for N cycles
   - Verify idle detection + signal decay triggers de-provisioning

2. **SwarmProvisioningCbrTest** — CBR loop:
   - Record a provisioning outcome
   - Query for similar state and verify retrieval
   - Verify CBR-adjusted tuning stays within bounds

### What's NOT Tested Here

- Actual `WorkerProvisioner` implementations (claudony, nono) — these are SPI implementations tested in their own repos
- LLM-based provisioning reasoning — blocks#285
- Agent self-tuning effectiveness — #1114

## References

- [WorkerProvisioner.java:34] — existing provisioning SPI
- [ProvisionContext.java:40] — existing provisioning context record
- [ProvisionResult.java:38] — existing provisioning result record
- [SwarmConfig.java:20] — existing swarm configuration (extends with provisionBudget, integrationPolicy)
- [StigmergyCoordinator.java:32] — agent lifecycle (agentJoined, agentActivated, agentDeparted)
- [SignalRegistry.java:187] — consensusSignals() for multi-source detection
- [ActivityTracker.java:25] — activity rate tracking
- [DispatchBudget.java:29] — external capacity check SPI
- [BudgetConfig.java:20] — existing budget record pattern
- [CbrRetrievalService.java:74] — CBR query infrastructure
- [RoleTracker.java:36] — fingerprinting (needs integration delay awareness)
- [TeamDetector.java:34] — team affinity (needs integration delay awareness)
- [DefaultMetricsSpace.java:30] — agent-visible metrics facet
- [CaseContextChangedEventHandler.java:1390] — convergenceDetection() insertion point
- [CaseStatusChangedHandler.java:203] — terminal state eviction block
- [RuntimeBeans.java:898] — CaseContextChangedEventHandler producer
- [2026-09-18-swarm-execution-model-design.md] — #1112 design spec (extends this)
- [2026-09-18-stigmergy-execution-model-design.md] — #1111 design spec (foundation)
- [D84–D91] — design decisions for this issue
- [GitHub #1113] — focal issue
- [GitHub #1104] — Hive Mind epic
- [blocks#285] — LLM swarm coordination (future)
- [arXiv:2603.28990] — "Drop the Hierarchy and Roles" — role autonomy over assignment
