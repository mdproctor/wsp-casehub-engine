# CoordinationTopology SPI — Topology Strategy with Concrete Implementations

**Issue:** agency#380
**Parent epic:** agency#378 (Agency Intelligence Layer — Decision Loops, Topologies, YAML Parity)
**Cross-repo parent:** agency#377 (Distributed Per-Agent LLM Coordination)
**Depends on:** agency#379 (AgentDecisionLoop — done), engine#1284 (TopologyProvider — done)
**Absorbs:** agency#381 (LlmSwarmCoordinator → CentralisedTopology refactor)

## Summary

CoordinationTopology is the strategy SPI that controls how AgentDecisionLoops interact within a coordination topology. It extends engine's TopologyProvider (renamed from SwarmTopologyProvider) to bridge the engine's topology-aware provisioning with agency's decision loop infrastructure.

Six concrete implementations serve as R&D starting points — each configures loops with different scopes, observation strategies, and communication modes, and provides its own DecisionOutcome execution strategy.

**Scope:**
1. Engine: rename SwarmTopologyProvider → TopologyProvider, extend TopologyType with HIERARCHICAL/GRADUATED
2. Agency: CoordinationTopology SPI + supporting types (TopologyContext, CoordinationPlan, DecisionOutcomeExecutor, CoordinationFeedback, FeedbackSignal, GraduationState)
3. Agency: 6 concrete implementations with full configure() + event handling
4. Agency: HivePatternAdapter bridging existing orchestration patterns to the CoordinationTopology SPI
5. Agency: Full refactor of LlmSwarmCoordinator into CentralisedTopology (closes #381)

---

## 1. Engine-Side Changes

### 1.1 Rename SwarmTopologyProvider → TopologyProvider

**Via IntelliJ refactoring.** Three files affected:

- `api/spi/coordination/SwarmTopologyProvider.java` → `TopologyProvider.java`
- `runtime-core/runtime/coordination/CentralisedTopologyProvider.java` — `implements TopologyProvider`
- `runtime/quarkus/SpiDefaultBeans.java` — return type + method name

```java
// api/spi/coordination/TopologyProvider.java
public interface TopologyProvider {
    TopologyDescriptor describeTopology(UUID caseId);
}
```

### 1.2 Extend TopologyType

Add HIERARCHICAL and GRADUATED to the existing enum:

```java
public enum TopologyType {
    CENTRALISED { /* existing — INITIATOR→CENTRAL, CONTRACTOR→LOCAL, PEER→throws */ },
    DISTRIBUTED { /* existing — PEER→COORDINATED, INITIATOR→throws, CONTRACTOR→throws */ },
    HYBRID      { /* existing — INITIATOR→CENTRAL, PEER→COORDINATED, CONTRACTOR→LOCAL */ },

    HIERARCHICAL {
        @Override
        public DecisionAuthority deriveAuthority(StructuralRole role) {
            return switch (role) {
                case INITIATOR -> DecisionAuthority.CENTRAL;
                case CONTRACTOR -> DecisionAuthority.LOCAL;
                case PEER -> throw new IllegalArgumentException(
                    "HIERARCHICAL topology does not support PEER"
                    + " — tree structure requires parent/child relationships");
            };
        }
    },

    GRADUATED {
        @Override
        public DecisionAuthority deriveAuthority(StructuralRole role) {
            return switch (role) {
                case INITIATOR -> DecisionAuthority.CENTRAL;
                case PEER -> DecisionAuthority.COORDINATED;
                case CONTRACTOR -> DecisionAuthority.LOCAL;
            };
        }
    };

    public abstract DecisionAuthority deriveAuthority(StructuralRole role);
}
```

**HIERARCHICAL** mirrors CENTRALISED's authority model (INITIATOR→CENTRAL, CONTRACTOR→LOCAL) but disallows PEER — tree structures have parent/child, not peer relationships.

**GRADUATED** mirrors HYBRID — all three roles are valid. The static deriveAuthority provides a baseline; the actual effective authority is determined at runtime by GraduationState, which adjusts the DecisionScope per agent based on feedback signals.

---

## 2. CoordinationTopology SPI

**Package:** `io.casehub.agency.orchestration.topology`

```java
public interface CoordinationTopology extends TopologyProvider {

    CoordinationPlan configure(TopologyContext context);

    void onSwarmEvent(SwarmCoordinationEvent event);

    void onFeedback(CoordinationFeedback feedback);
}
```

Three methods beyond the inherited `describeTopology(UUID)`:

| Method | Purpose |
|--------|---------|
| `configure` | Given a TopologyContext, return a CoordinationPlan describing what loops to create and how to execute their outcomes |
| `onSwarmEvent` | React to coordination events (role changes, progress, member completion) |
| `onFeedback` | Receive outcome-based feedback for graduated autonomy and metrics |

The SPI is stateful — implementations hold per-instance state (SwarmContext, GraduationState, etc.) via constructor injection. `configure()` may be called multiple times as the topology reconfigures in response to events.

---

## 3. Supporting Types

All in `io.casehub.agency.orchestration.topology`.

### 3.1 TopologyContext

Per-invocation context passed to `configure()`. Carries 5 universal fields — everything any topology needs to set up its loops.

```java
public record TopologyContext(
    UUID caseId,
    TopologyDescriptor descriptor,
    AgentProvider agentProvider,
    DecisionLoopFactory loopFactory,
    SwarmInvocationPolicy invocationPolicy
) {
    public TopologyContext {
        Objects.requireNonNull(caseId, "caseId must not be null");
        Objects.requireNonNull(descriptor, "descriptor must not be null");
        Objects.requireNonNull(agentProvider, "agentProvider must not be null");
        Objects.requireNonNull(loopFactory, "loopFactory must not be null");
        Objects.requireNonNull(invocationPolicy, "invocationPolicy must not be null");
    }
}
```

**Design principle:** TopologyContext is the per-case situation. Topology-specific dependencies (SwarmContext, GraduationState, GroupHierarchy) are constructor-injected into each topology implementation — clean per-instance vs per-invocation separation with full compile-time type safety.

### 3.2 CoordinationPlan

The return type of `configure()`. A complete recipe: what loops to create and how to execute their outcomes.

```java
public record CoordinationPlan(
    List<DecisionLoopDescriptor> loopDescriptors,
    DecisionOutcomeExecutor executor
) {
    public CoordinationPlan {
        loopDescriptors = List.copyOf(loopDescriptors);
        Objects.requireNonNull(executor, "executor must not be null");
        if (loopDescriptors.isEmpty()) {
            throw new IllegalArgumentException("CoordinationPlan must have at least one loop descriptor");
        }
    }
}
```

### 3.3 DecisionOutcomeExecutor

Functional interface — topology-provided execution strategy for DecisionOutcomes.

```java
@FunctionalInterface
public interface DecisionOutcomeExecutor {
    CompletionStage<CoordinationFeedback> execute(
        String agentId, DecisionOutcome outcome, TopologyContext context);
}
```

Returns a `CompletionStage<CoordinationFeedback>` — the executor applies the outcome and reports back whether it succeeded. This feedback is routed to `onFeedback()` to close the observe→reason→act→feedback loop.

### 3.4 CoordinationFeedback

Outcome-based feedback generated by the executor after applying a decision.

```java
public record CoordinationFeedback(
    String agentId,
    DecisionOutcome outcome,
    FeedbackSignal signal
) {
    public CoordinationFeedback {
        Objects.requireNonNull(agentId, "agentId must not be null");
        Objects.requireNonNull(outcome, "outcome must not be null");
        Objects.requireNonNull(signal, "signal must not be null");
    }
}
```

### 3.5 FeedbackSignal

```java
public enum FeedbackSignal {
    SUCCESS,
    PARTIAL,
    FAILURE,
    SCOPE_VIOLATION
}
```

| Signal | Meaning | Graduation effect |
|--------|---------|-------------------|
| SUCCESS | Decision applied cleanly | Consider scope upgrade |
| PARTIAL | Decision partially applied | No scope change |
| FAILURE | Decision failed to apply | Consider scope downgrade |
| SCOPE_VIOLATION | Decision attempted action outside permitted scope | Hard scope downgrade |

### 3.6 GraduationState

Mutable per-agent scope tracker for GraduatedTopology. Wraps immutable DecisionScope enum values.

```java
public class GraduationState {

    private final Map<String, DecisionScope> agentScopes;
    private final Map<String, List<FeedbackSignal>> recentSignals;
    private final int windowSize;

    public GraduationState(Map<String, DecisionScope> initialScopes, int windowSize) {
        this.agentScopes = new ConcurrentHashMap<>(initialScopes);
        this.recentSignals = new ConcurrentHashMap<>();
        this.windowSize = windowSize;
    }

    public DecisionScope scopeFor(String agentId) {
        return agentScopes.getOrDefault(agentId, DecisionScope.STIMULUS_DRIVEN);
    }

    public void recordFeedback(String agentId, FeedbackSignal signal) {
        recentSignals.computeIfAbsent(agentId, k -> new CopyOnWriteArrayList<>()).add(signal);
        adjustScope(agentId);
    }

    private void adjustScope(String agentId) {
        var signals = recentSignals.getOrDefault(agentId, List.of());
        var window = signals.subList(
            Math.max(0, signals.size() - windowSize), signals.size());

        if (window.stream().anyMatch(s -> s == FeedbackSignal.SCOPE_VIOLATION)) {
            downgrade(agentId);
            return;
        }

        long successes = window.stream().filter(s -> s == FeedbackSignal.SUCCESS).count();
        long failures = window.stream().filter(s -> s == FeedbackSignal.FAILURE).count();

        if (successes >= windowSize * 0.8) {
            upgrade(agentId);
        } else if (failures >= windowSize * 0.5) {
            downgrade(agentId);
        }
    }

    public void upgrade(String agentId) {
        agentScopes.compute(agentId, (id, scope) -> switch (scope != null ? scope : DecisionScope.STIMULUS_DRIVEN) {
            case STIMULUS_DRIVEN -> DecisionScope.TASK_ADAPTATION;
            case TASK_ADAPTATION -> DecisionScope.FULL_AUTONOMY;
            case FULL_AUTONOMY -> DecisionScope.FULL_AUTONOMY;
        });
    }

    public void downgrade(String agentId) {
        agentScopes.compute(agentId, (id, scope) -> switch (scope != null ? scope : DecisionScope.STIMULUS_DRIVEN) {
            case FULL_AUTONOMY -> DecisionScope.TASK_ADAPTATION;
            case TASK_ADAPTATION -> DecisionScope.STIMULUS_DRIVEN;
            case STIMULUS_DRIVEN -> DecisionScope.STIMULUS_DRIVEN;
        });
    }
}
```

**Design:** GraduationState is a windowed feedback accumulator. Scope transitions follow the DecisionScope ordering: STIMULUS_DRIVEN ↔ TASK_ADAPTATION ↔ FULL_AUTONOMY. SCOPE_VIOLATION causes an immediate hard downgrade. The window size and thresholds (80% success for upgrade, 50% failure for downgrade) are construction parameters.

---

## 4. Concrete Implementations

All in `io.casehub.agency.orchestration.topology`. Each topology implements the full CoordinationTopology interface with working configure(), onSwarmEvent(), and onFeedback().

### 4.1 CentralisedTopology

**Refactored from LlmSwarmCoordinator.** One global-scope loop. The coordinator agent makes all routing and aggregation decisions.

**Constructor dependencies:** SwarmContext, SwarmRoutingStrategy, CollectiveReasoningAggregation, TerminationCondition\<SwarmContext\>, SwarmFallbackStrategy

**Loop configuration:**

| Property | Value |
|----------|-------|
| Count | 1 |
| Name | "centralised-coordinator" |
| Scope | FULL_AUTONOMY |
| ObservationStrategy | EVENT_DRIVEN |
| CommunicationMode | BROADCAST |
| Budget | AgentBudgetPolicy.defaults() |

```java
public class CentralisedTopology implements CoordinationTopology {

    private final SwarmContext swarmContext;
    private final SwarmRoutingStrategy routingStrategy;
    private final CollectiveReasoningAggregation aggregation;
    private final TerminationCondition<SwarmContext> terminationCondition;
    private final SwarmFallbackStrategy fallbackStrategy;
    private final List<String> promptSectionNames;

    // constructor omitted for brevity — takes all fields

    @Override
    public TopologyDescriptor describeTopology(UUID caseId) {
        return new TopologyDescriptor(TopologyType.CENTRALISED, Map.of());
    }

    @Override
    public CoordinationPlan configure(TopologyContext context) {
        var descriptor = new DecisionLoopDescriptor(
            "centralised-coordinator",
            ObservationStrategy.EVENT_DRIVEN,
            DecisionScope.FULL_AUTONOMY,
            CommunicationMode.BROADCAST,
            AgentBudgetPolicy.defaults(),
            promptSectionNames
        );

        DecisionOutcomeExecutor executor = (agentId, outcome, ctx) ->
            executeCentralised(outcome, ctx);

        return new CoordinationPlan(List.of(descriptor), executor);
    }

    private CompletionStage<CoordinationFeedback> executeCentralised(
            DecisionOutcome outcome, TopologyContext ctx) {
        return switch (outcome) {
            case DecisionOutcome.Route r ->
                executeRouting(r.decision(), ctx);
            case DecisionOutcome.Aggregate a ->
                executeAggregation(a.result(), ctx);
            case DecisionOutcome.WriteBack w ->
                executeWriteBack(w, ctx);
            case DecisionOutcome.Composite c ->
                executeComposite(c.outcomes(), ctx);
            case DecisionOutcome.ScopeGated sg ->
                CompletableFuture.completedFuture(
                    new CoordinationFeedback("coordinator", sg,
                        FeedbackSignal.SCOPE_VIOLATION));
        };
    }

    // executeRouting delegates to routingStrategy
    // executeAggregation delegates to aggregation
    // executeWriteBack applies observations/signals/context writes
    // executeComposite sequences sub-outcomes

    @Override
    public void onSwarmEvent(SwarmCoordinationEvent event) {
        switch (event) {
            case SwarmCoordinationEvent.SwarmInitialised e ->
                swarmContext.onSwarmInitialised(e.experiences(), e.progress());
            case SwarmCoordinationEvent.RoleChanged e ->
                swarmContext.onRoleChanged(e.roles(), e.progress());
            case SwarmCoordinationEvent.TeamChanged e ->
                swarmContext.onTeamChanged(e.teams(), e.progress());
            case SwarmCoordinationEvent.ProgressUpdated e ->
                swarmContext.onProgressUpdated(e.progress());
            case SwarmCoordinationEvent.MemberCompleted e ->
                swarmContext.onMemberCompleted(e.agentId());
            case SwarmCoordinationEvent.ProvisioningChanged e ->
                swarmContext.onProvisioningChanged(e.currentSize(), e.maxSize());
        }
    }

    @Override
    public void onFeedback(CoordinationFeedback feedback) {
        // CentralisedTopology uses feedback for metrics only — no scope adjustment
    }
}
```

**Refactoring from LlmSwarmCoordinator:**

| LlmSwarmCoordinator field | CentralisedTopology mapping |
|---------------------------|---------------------------|
| agentProvider | TopologyContext.agentProvider() |
| context (SwarmContext) | Constructor-injected |
| routingStrategy | Constructor-injected, used by executor |
| aggregation | Constructor-injected, used by executor |
| terminationCondition | Constructor-injected, used by executor |
| eventSource | Replaced by onSwarmEvent() — direct SwarmContext updates |
| invocationTracker | TopologyContext.invocationPolicy() → new SwarmInvocationTracker |
| fallbackStrategy | Constructor-injected, used by executor |
| promptSections | promptSectionNames in DecisionLoopDescriptor |

### 4.2 DistributedTopology

N independent loops with full-autonomy scope. Each agent decides independently via peer-to-peer coordination through stigmergy.

**Constructor dependencies:** `Map<String, AgentRole> agentRoles` — the agent roster for this topology. Stored at construction time for use by `describeTopology()`. The same roles are available at `configure()` time via `TopologyContext.descriptor().agentRoles()`, but `describeTopology()` is called independently of `configure()`.

**Loop configuration:**

| Property | Value |
|----------|-------|
| Count | N (one per agent in descriptor.agentRoles()) |
| Name | "distributed-{agentId}" |
| Scope | FULL_AUTONOMY |
| ObservationStrategy | EVENT_DRIVEN |
| CommunicationMode | STIGMERGY |
| Budget | AgentBudgetPolicy.defaults() |

```java
public class DistributedTopology implements CoordinationTopology {

    @Override
    public TopologyDescriptor describeTopology(UUID caseId) {
        return new TopologyDescriptor(TopologyType.DISTRIBUTED, agentRoles);
    }

    @Override
    public CoordinationPlan configure(TopologyContext context) {
        var descriptors = context.descriptor().agentRoles().keySet().stream()
            .map(agentId -> new DecisionLoopDescriptor(
                "distributed-" + agentId,
                ObservationStrategy.EVENT_DRIVEN,
                DecisionScope.FULL_AUTONOMY,
                CommunicationMode.STIGMERGY,
                AgentBudgetPolicy.defaults(),
                List.of("environment-observer", "peer-state")
            ))
            .toList();

        DecisionOutcomeExecutor executor = (agentId, outcome, ctx) ->
            executeLocal(agentId, outcome, ctx);

        return new CoordinationPlan(descriptors, executor);
    }

    private CompletionStage<CoordinationFeedback> executeLocal(
            String agentId, DecisionOutcome outcome, TopologyContext ctx) {
        return switch (outcome) {
            case DecisionOutcome.WriteBack w -> applyLocalWrites(agentId, w, ctx);
            case DecisionOutcome.Route r -> applyLocalRoute(agentId, r, ctx);
            case DecisionOutcome.ScopeGated sg ->
                CompletableFuture.completedFuture(
                    new CoordinationFeedback(agentId, sg, FeedbackSignal.SCOPE_VIOLATION));
            default -> applyGeneric(agentId, outcome, ctx);
        };
    }

    @Override
    public void onSwarmEvent(SwarmCoordinationEvent event) {
        // Distributed topology observes events but does not centrally coordinate —
        // events are forwarded to individual agent loops as observations
    }

    @Override
    public void onFeedback(CoordinationFeedback feedback) {
        // Distributed topology records feedback for metrics only
    }
}
```

### 4.3 HybridTopology

One global coordinator loop + N local agent loops with constrained scope. The global loop handles coordination decisions; local loops handle agent-specific task adaptation.

**Constructor dependencies:** `Map<String, AgentRole> agentRoles` — the agent roster. Same pattern as DistributedTopology — stored for `describeTopology()` independence.

**Loop configuration:**

| Loop | Scope | ObservationStrategy | CommunicationMode |
|------|-------|--------------------|--------------------|
| Global coordinator | FULL_AUTONOMY | EVENT_DRIVEN | BROADCAST |
| Per-agent local | TASK_ADAPTATION | TICK_DRIVEN | STIGMERGY |

```java
public class HybridTopology implements CoordinationTopology {

    @Override
    public TopologyDescriptor describeTopology(UUID caseId) {
        return new TopologyDescriptor(TopologyType.HYBRID, agentRoles);
    }

    @Override
    public CoordinationPlan configure(TopologyContext context) {
        var globalDescriptor = new DecisionLoopDescriptor(
            "hybrid-coordinator",
            ObservationStrategy.EVENT_DRIVEN,
            DecisionScope.FULL_AUTONOMY,
            CommunicationMode.BROADCAST,
            AgentBudgetPolicy.defaults(),
            List.of("global-state", "coordination-strategy")
        );

        var localDescriptors = context.descriptor().agentRoles().entrySet().stream()
            .filter(e -> e.getValue().structuralRole() != StructuralRole.INITIATOR)
            .map(e -> new DecisionLoopDescriptor(
                "hybrid-local-" + e.getKey(),
                ObservationStrategy.TICK_DRIVEN,
                DecisionScope.TASK_ADAPTATION,
                CommunicationMode.STIGMERGY,
                AgentBudgetPolicy.defaults(),
                List.of("local-task-context")
            ))
            .toList();

        var allDescriptors = new ArrayList<DecisionLoopDescriptor>();
        allDescriptors.add(globalDescriptor);
        allDescriptors.addAll(localDescriptors);

        DecisionOutcomeExecutor executor = (agentId, outcome, ctx) -> {
            if ("hybrid-coordinator".equals(agentId)) {
                return executeCentrally(outcome, ctx);
            }
            return executeLocally(agentId, outcome, ctx);
        };

        return new CoordinationPlan(allDescriptors, executor);
    }

    @Override
    public void onSwarmEvent(SwarmCoordinationEvent event) {
        // Global coordinator loop observes all events
        // Local loops observe events relevant to their agent only
    }

    @Override
    public void onFeedback(CoordinationFeedback feedback) {
        // Hybrid uses feedback to rebalance global/local authority
    }
}
```

### 4.4 HierarchicalTopology

Tree of loops at different scopes. Parent nodes have broader scope; leaf nodes have narrower scope. Delegation flows down the tree; aggregation flows up.

**Constructor dependencies:** `Map<String, AgentRole> agentRoles` (roster), `Map<String, String> parentMap` — maps each agent to its parent in the hierarchy. Root agent has no parent.

**Loop configuration:**

| Node level | Scope | ObservationStrategy | CommunicationMode |
|------------|-------|--------------------|--------------------|
| Root | FULL_AUTONOMY | EVENT_DRIVEN | BROADCAST |
| Intermediate | TASK_ADAPTATION | TICK_DRIVEN | DIRECT_MESSAGING |
| Leaf | STIMULUS_DRIVEN | EVENT_DRIVEN | STIGMERGY |

```java
public class HierarchicalTopology implements CoordinationTopology {

    private final Map<String, String> parentMap;

    public HierarchicalTopology(Map<String, String> parentMap) {
        this.parentMap = Map.copyOf(parentMap);
    }

    @Override
    public TopologyDescriptor describeTopology(UUID caseId) {
        return new TopologyDescriptor(TopologyType.HIERARCHICAL, agentRoles);
    }

    @Override
    public CoordinationPlan configure(TopologyContext context) {
        var descriptors = new ArrayList<DecisionLoopDescriptor>();
        var roles = context.descriptor().agentRoles();

        for (var entry : roles.entrySet()) {
            String agentId = entry.getKey();
            int depth = computeDepth(agentId);
            boolean isLeaf = isLeaf(agentId, roles.keySet());

            DecisionScope scope;
            ObservationStrategy strategy;
            CommunicationMode comms;

            if (depth == 0) {
                scope = DecisionScope.FULL_AUTONOMY;
                strategy = ObservationStrategy.EVENT_DRIVEN;
                comms = CommunicationMode.BROADCAST;
            } else if (isLeaf) {
                scope = DecisionScope.STIMULUS_DRIVEN;
                strategy = ObservationStrategy.EVENT_DRIVEN;
                comms = CommunicationMode.STIGMERGY;
            } else {
                scope = DecisionScope.TASK_ADAPTATION;
                strategy = ObservationStrategy.TICK_DRIVEN;
                comms = CommunicationMode.DIRECT_MESSAGING;
            }

            descriptors.add(new DecisionLoopDescriptor(
                "hierarchical-" + agentId,
                strategy, scope, comms,
                AgentBudgetPolicy.defaults(),
                List.of("hierarchy-context")
            ));
        }

        DecisionOutcomeExecutor executor = (agentId, outcome, ctx) ->
            executeDelegated(agentId, outcome, ctx);

        return new CoordinationPlan(descriptors, executor);
    }

    private CompletionStage<CoordinationFeedback> executeDelegated(
            String agentId, DecisionOutcome outcome, TopologyContext ctx) {
        // Route outcomes map to delegations down the tree
        // WriteBack outcomes apply locally and propagate results up to parent
        // ...
    }

    @Override
    public void onSwarmEvent(SwarmCoordinationEvent event) {
        // Events flow down the tree — root receives all, intermediate nodes filter
    }

    @Override
    public void onFeedback(CoordinationFeedback feedback) {
        // Feedback propagates up the tree — leaf feedback affects parent's view
    }
}
```

### 4.5 GraduatedTopology

N per-agent loops with mutable scope. Dynamic scope adjustment at runtime based on feedback signals via GraduationState.

**Constructor dependencies:** `Map<String, AgentRole> agentRoles` (roster), `GraduationState graduationState`

**Loop configuration:**

| Property | Value |
|----------|-------|
| Count | N (one per agent) |
| Name | "graduated-{agentId}" |
| Scope | Determined by GraduationState (starts at STIMULUS_DRIVEN) |
| ObservationStrategy | EVENT_DRIVEN |
| CommunicationMode | STIGMERGY |
| Budget | AgentBudgetPolicy.defaults() |

```java
public class GraduatedTopology implements CoordinationTopology {

    private final GraduationState graduationState;

    public GraduatedTopology(GraduationState graduationState) {
        this.graduationState = Objects.requireNonNull(graduationState);
    }

    @Override
    public TopologyDescriptor describeTopology(UUID caseId) {
        return new TopologyDescriptor(TopologyType.GRADUATED, agentRoles);
    }

    @Override
    public CoordinationPlan configure(TopologyContext context) {
        var descriptors = context.descriptor().agentRoles().keySet().stream()
            .map(agentId -> new DecisionLoopDescriptor(
                "graduated-" + agentId,
                ObservationStrategy.EVENT_DRIVEN,
                graduationState.scopeFor(agentId),
                CommunicationMode.STIGMERGY,
                AgentBudgetPolicy.defaults(),
                List.of("graduated-context")
            ))
            .toList();

        DecisionOutcomeExecutor executor = (agentId, outcome, ctx) ->
            executeGraduated(agentId, outcome, ctx);

        return new CoordinationPlan(descriptors, executor);
    }

    private CompletionStage<CoordinationFeedback> executeGraduated(
            String agentId, DecisionOutcome outcome, TopologyContext ctx) {
        // Execute the outcome, then return feedback based on success/failure
        // The feedback flows to onFeedback() which adjusts scope
        return applyOutcome(agentId, outcome, ctx)
            .thenApply(signal -> new CoordinationFeedback(agentId, outcome, signal));
    }

    @Override
    public void onSwarmEvent(SwarmCoordinationEvent event) {
        // Events are delivered to individual agent loops as observations
    }

    @Override
    public void onFeedback(CoordinationFeedback feedback) {
        graduationState.recordFeedback(feedback.agentId(), feedback.signal());
        // After scope adjustment, the loop's DecisionContext should be rebuilt
        // with the new scope on the next tick. This triggers a reconfigure
        // if the scope has changed.
    }
}
```

**Reconfiguration on scope change:** When `onFeedback()` adjusts a scope, the next `configure()` call returns a CoordinationPlan with the updated scope in the DecisionLoopDescriptor. The orchestration layer detects the changed plan and rebuilds the affected loop's DecisionContext.

### 4.6 HiveTopology

Role-scoped loops with supervised orchestration. Composes existing orchestration patterns (supervisor, debate, voting) via HivePatternAdapter.

**Constructor dependencies:** `Map<String, AgentRole> agentRoles` (roster), `HivePatternAdapter patternAdapter`, `OrchestrationPatternType patternType`

**Loop configuration:**

| Role | Scope | ObservationStrategy | CommunicationMode |
|------|-------|--------------------|--------------------|
| Coordinator | FULL_AUTONOMY | EVENT_DRIVEN | BROADCAST |
| Worker roles | TASK_ADAPTATION | DEMAND_DRIVEN | DIRECT_MESSAGING |

```java
public class HiveTopology implements CoordinationTopology {

    private final HivePatternAdapter patternAdapter;
    private final OrchestrationPatternType patternType;

    public HiveTopology(HivePatternAdapter patternAdapter,
                        OrchestrationPatternType patternType) {
        this.patternAdapter = Objects.requireNonNull(patternAdapter);
        this.patternType = Objects.requireNonNull(patternType);
    }

    @Override
    public TopologyDescriptor describeTopology(UUID caseId) {
        return new TopologyDescriptor(TopologyType.CENTRALISED, agentRoles);
    }

    @Override
    public CoordinationPlan configure(TopologyContext context) {
        var coordinatorId = context.descriptor().coordinatorAgentId()
            .orElseThrow(() -> new IllegalStateException(
                "HiveTopology requires a coordinator agent (INITIATOR role)"));

        var coordinatorDescriptor = new DecisionLoopDescriptor(
            "hive-coordinator",
            ObservationStrategy.EVENT_DRIVEN,
            DecisionScope.FULL_AUTONOMY,
            CommunicationMode.BROADCAST,
            AgentBudgetPolicy.defaults(),
            List.of("hive-coordination-strategy", "role-assignments")
        );

        var workerDescriptors = context.descriptor().agentRoles().entrySet().stream()
            .filter(e -> e.getValue().structuralRole() != StructuralRole.INITIATOR)
            .map(e -> new DecisionLoopDescriptor(
                "hive-worker-" + e.getKey(),
                ObservationStrategy.DEMAND_DRIVEN,
                DecisionScope.TASK_ADAPTATION,
                CommunicationMode.DIRECT_MESSAGING,
                AgentBudgetPolicy.defaults(),
                List.of("role-context-" + e.getValue().name().value())
            ))
            .toList();

        var allDescriptors = new ArrayList<DecisionLoopDescriptor>();
        allDescriptors.add(coordinatorDescriptor);
        allDescriptors.addAll(workerDescriptors);

        DecisionOutcomeExecutor executor = (agentId, outcome, ctx) ->
            patternAdapter.execute(patternType, agentId, outcome, ctx);

        return new CoordinationPlan(allDescriptors, executor);
    }

    @Override
    public void onSwarmEvent(SwarmCoordinationEvent event) {
        // Coordinator receives all events; workers receive role-scoped events only
    }

    @Override
    public void onFeedback(CoordinationFeedback feedback) {
        // Coordinator aggregates feedback from workers for pattern-level decisions
        patternAdapter.onFeedback(patternType, feedback);
    }
}
```

### 4.7 HivePatternAdapter

Bridges existing SupervisorBuilder/DebateBuilder/VotingBuilder to the CoordinationTopology SPI.

```java
public enum OrchestrationPatternType {
    SUPERVISOR,
    DEBATE,
    VOTING
}
```

```java
public class HivePatternAdapter {

    private final AgentProvider agentProvider;

    public HivePatternAdapter(AgentProvider agentProvider) {
        this.agentProvider = Objects.requireNonNull(agentProvider);
    }

    public CompletionStage<CoordinationFeedback> execute(
            OrchestrationPatternType patternType,
            String agentId,
            DecisionOutcome outcome,
            TopologyContext context) {
        return switch (patternType) {
            case SUPERVISOR -> executeSupervisor(agentId, outcome, context);
            case DEBATE -> executeDebate(agentId, outcome, context);
            case VOTING -> executeVoting(agentId, outcome, context);
        };
    }

    private CompletionStage<CoordinationFeedback> executeSupervisor(
            String agentId, DecisionOutcome outcome, TopologyContext ctx) {
        // Translate DecisionOutcome.Route → supervisor routing decision
        // Translate DecisionOutcome.WriteBack → supervisor state update
        // Use SupervisorBuilder infrastructure for execution
        // Return feedback based on execution result
    }

    private CompletionStage<CoordinationFeedback> executeDebate(
            String agentId, DecisionOutcome outcome, TopologyContext ctx) {
        // Translate DecisionOutcome.Aggregate → debate round aggregation
        // Use DebateBuilder infrastructure for execution
    }

    private CompletionStage<CoordinationFeedback> executeVoting(
            String agentId, DecisionOutcome outcome, TopologyContext ctx) {
        // Translate DecisionOutcome.Aggregate → voting round
        // Use VotingBuilder infrastructure for execution
    }

    public void onFeedback(OrchestrationPatternType patternType,
                           CoordinationFeedback feedback) {
        // Pattern-specific feedback handling
    }
}
```

---

## 5. Topology-to-Configuration Summary

| Topology | TopologyType | Loops | Scope | ObservationStrategy | CommunicationMode | Executor |
|----------|-------------|-------|-------|--------------------|--------------------|----------|
| CentralisedTopology | CENTRALISED | 1 global | FULL_AUTONOMY | EVENT_DRIVEN | BROADCAST | Central routing + aggregation |
| DistributedTopology | DISTRIBUTED | N per-agent | FULL_AUTONOMY | EVENT_DRIVEN | STIGMERGY | Local write-back |
| HybridTopology | HYBRID | 1 global + N local | FULL_AUTONOMY / TASK_ADAPTATION | EVENT_DRIVEN / TICK_DRIVEN | BROADCAST / STIGMERGY | Central for global, local for agents |
| HierarchicalTopology | HIERARCHICAL | Tree (root + intermediate + leaf) | FULL → TASK → STIMULUS | EVENT / TICK / EVENT | BROADCAST / DIRECT / STIGMERGY | Delegated down, aggregated up |
| GraduatedTopology | GRADUATED | N per-agent (mutable scope) | Dynamic via GraduationState | EVENT_DRIVEN | STIGMERGY | Graduated with feedback loop |
| HiveTopology | CENTRALISED | 1 coordinator + N workers | FULL_AUTONOMY / TASK_ADAPTATION | EVENT_DRIVEN / DEMAND_DRIVEN | BROADCAST / DIRECT_MESSAGING | Delegated to pattern adapter |

---

## 6. Integration Points

### 6.1 Orchestration Layer

The coordination topology is consumed by the orchestration layer (above the individual decision loops). The orchestration layer:

1. Creates the topology instance with constructor-injected dependencies
2. Calls `configure(TopologyContext)` to get the CoordinationPlan
3. Creates AgentDecisionLoops from the plan's loop descriptors via DecisionLoopFactory
4. Subscribes to loop outputs and feeds them to the plan's executor
5. Routes executor feedback to `topology.onFeedback()`
6. Routes SwarmCoordinationEvents to `topology.onSwarmEvent()`

### 6.2 Engine SwarmProvisioner

CentralisedTopologyProvider (engine default) is replaced by the active CoordinationTopology instance. The topology provides `describeTopology(UUID)` — SwarmProvisioner's existing topology-type gate works unchanged.

### 6.3 DecisionLoopFactory

DecisionLoopFactory is passed in TopologyContext. The orchestration layer calls it with each descriptor from the CoordinationPlan:

```java
for (var descriptor : plan.loopDescriptors()) {
    var context = new DecisionContext(
        topologyContext.caseId(), extractAgentId(descriptor),
        descriptor.scope(), descriptor.communicationMode(),
        topologyContext.descriptor());
    var loop = topologyContext.loopFactory().create(
        descriptor, context, reasoner, tracker);
    // subscribe to loop output → executor
}
```

### 6.4 StimergyCoordinator / AgentWorkStateNotifier

The existing StigmergyCoordinator implements AgentWorkStateNotifier (#1285). CoordinationTopology implementations that need lifecycle events receive them through the onSwarmEvent() method — SwarmCoordinationEvent already carries role changes, progress updates, and member completions.

---

## 7. Testing Strategy

### 7.1 Engine-side tests

1. **TopologyType extension tests:**
   - HIERARCHICAL: INITIATOR→CENTRAL, CONTRACTOR→LOCAL, PEER→throws
   - GRADUATED: INITIATOR→CENTRAL, PEER→COORDINATED, CONTRACTOR→LOCAL
   - All existing TopologyType tests still pass

2. **TopologyProvider rename verification:**
   - CentralisedTopologyProvider implements TopologyProvider
   - SpiDefaultBeans produces TopologyProvider

### 7.2 SPI and supporting type tests

3. **TopologyContext validation:**
   - Null field rejection
   - All 5 fields accessible

4. **CoordinationPlan validation:**
   - Empty descriptor list throws
   - Immutable copy of descriptors
   - Executor required

5. **CoordinationFeedback construction:**
   - Null rejection for all fields

6. **FeedbackSignal enum completeness**

7. **GraduationState:**
   - Initial scope assignment
   - Upgrade path: STIMULUS → TASK_ADAPTATION → FULL_AUTONOMY
   - Downgrade path: FULL_AUTONOMY → TASK_ADAPTATION → STIMULUS
   - SCOPE_VIOLATION → hard downgrade
   - Window-based signal accumulation
   - Concurrent access safety

### 7.3 Concrete topology tests

For each of the 6 topologies:

8. **configure() tests:**
   - Returns correct number of loop descriptors
   - Correct scope/strategy/communication per descriptor
   - Executor is non-null

9. **describeTopology() tests:**
   - Returns correct TopologyType
   - Agent roles map is consistent

10. **onSwarmEvent() tests:**
    - CentralisedTopology: updates SwarmContext state
    - DistributedTopology: events forwarded to agent loops
    - HybridTopology: global loop receives all, local loops filter
    - HierarchicalTopology: events flow down tree
    - GraduatedTopology: events delivered to agent loops
    - HiveTopology: coordinator receives all, workers receive scoped

11. **onFeedback() tests:**
    - GraduatedTopology: scope changes on feedback signals
    - Other topologies: feedback recorded for metrics

12. **DecisionOutcomeExecutor tests per topology:**
    - CentralisedTopology: Route → routing strategy, Aggregate → aggregation
    - DistributedTopology: WriteBack → local application
    - HybridTopology: global vs local execution paths
    - HierarchicalTopology: delegation down tree
    - GraduatedTopology: execution + feedback generation
    - HiveTopology: delegation to pattern adapter

### 7.4 HivePatternAdapter tests

13. **Pattern translation:**
    - Supervisor: Route outcome → supervisor routing
    - Debate: Aggregate outcome → debate round
    - Voting: Aggregate outcome → voting round
    - Feedback routing per pattern type

### 7.5 CentralisedTopology refactoring verification

14. **Behavioral parity with LlmSwarmCoordinator:**
    - Routing decisions produce same results
    - Aggregation produces same results
    - Fallback strategy invoked under same conditions
    - Event handling updates SwarmContext identically

---

## 8. What This Issue Does NOT Include

- **YAML surface for topologies** — YAML parity is agency#385
- **TopologyProvider rename consumers outside this slot** — other slots pick up the rename on next rebase
- **Orchestration layer that consumes CoordinationTopology** — the orchestration loop that calls configure(), creates loops, and routes outcomes is a follow-on concern
- **Topology transitions** — dynamically switching from one topology type to another at runtime
- **Group/MOISE+ integration** — HiveTopology receives role assignments but the group policy layer (eidos#216) is separate

---

## References

- `api/spi/coordination/SwarmTopologyProvider.java` — SPI being renamed
- `api/model/coordination/TopologyType.java` — enum being extended
- `api/model/coordination/TopologyDescriptor.java` — topology descriptor record
- `api/model/coordination/DecisionScope.java` — scope enum used by GraduationState
- `api/model/coordination/CommunicationMode.java` — communication mode enum
- `api/model/coordination/ObservationStrategy.java` — observation strategy enum
- `api/model/coordination/AgentRole.java` — role record with structural/platform facets
- `agency-core/orchestration/loop/AgentDecisionLoop.java` — loop runtime
- `agency-core/orchestration/loop/DecisionLoopDescriptor.java` — loop playbook record
- `agency-core/orchestration/loop/DecisionLoopFactory.java` — loop factory
- `agency-core/orchestration/loop/DecisionOutcome.java` — sealed outcome type
- `agency-core/orchestration/loop/DecisionContext.java` — per-agent context record
- `agency-core/orchestration/swarm/LlmSwarmCoordinator.java` — refactoring source
- `agency-core/orchestration/swarm/SwarmCoordinatorBuilder.java` — refactoring source
- `agency-core/orchestration/swarm/SwarmContext.java` — mutable swarm state
- `agency-core/orchestration/swarm/SwarmCoordinationEvent.java` — sealed event types
- `agency-core/orchestration/swarm/SwarmRoutingStrategy.java` — routing used by CentralisedTopology
- `agency-core/orchestration/swarm/CollectiveReasoningAggregation.java` — aggregation used by CentralisedTopology
- `agency-core/orchestration/swarm/SwarmFallbackStrategy.java` — fallback used by CentralisedTopology
- `agency-core/orchestration/swarm/SwarmInvocationTracker.java` — budget tracking
- `agency-core/orchestration/swarm/SwarmInvocationPolicy.java` — budget policy
- `agency-core/orchestration/pattern/SupervisorBuilder.java` — pattern for HiveTopology
- `agency-core/orchestration/pattern/DebateBuilder.java` — pattern for HiveTopology
- `agency-core/orchestration/pattern/VotingBuilder.java` — pattern for HiveTopology
- `specs/issue-377-distributed-per-agent-coordination/2026-11-02-agent-decision-loop-design.md` — #379 spec
- `specs/issue-377-distributed-per-agent-coordination/2026-11-02-swarm-topology-provider-design.md` — #1284 spec
- `specs/issue-377-distributed-per-agent-coordination/decisions-380.md` — design decisions
- agency#380 issue — original requirements
- agency#381 issue — absorbed (LlmSwarmCoordinator refactor)
- engine#1284 issue — TopologyProvider rename dependency
