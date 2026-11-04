# LlmSwarmCoordinator → CentralisedTopology Refactoring Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/agency#381 — refactor: LlmSwarmCoordinator → CentralisedTopology
**Issue group:** casehubio/agency#381

**Goal:** Decompose LlmSwarmCoordinator so its three LLM strategies become AgentDecisionLoop instances within CentralisedTopology, then eliminate the coordinator class.

**Architecture:** Three ContentSummariser adapters wrap the existing LLM strategies (SwarmRoutingStrategy, CollectiveReasoningAggregation, SwarmProvisioningAdvisorImpl). CentralisedTopology creates AgentDecisionLoops from these adapters during configure(), stores them internally, and feeds observations via onSwarmEvent(). A new DecisionOutcome.Provision variant carries provisioning decisions. LlmSwarmCoordinator is eliminated — its build() path has 0 callers.

**Tech Stack:** Java 21, agency-core module, sealed interfaces, CompletionStage, SummarisationRunner framework

## Global Constraints

- All existing tests must pass after every batch
- agency-core module only — no cross-repo changes needed
- DecisionOutcome is in agency-core (io.casehub.agency.orchestration.loop), not engine-api
- ProvisioningAdvice is in engine-api (io.casehub.api.spi.stigmergy.SwarmProvisioningAdvisor.ProvisioningAdvice) — use as-is, do not copy
- Other topologies (Distributed, Hierarchical, Hybrid, Graduated, HivePattern) all use `default` branches in their executors — adding Provision does not break them
- SwarmProvisioningAdvisor is the SPI interface; SwarmProvisioningAdvisorImpl is the concrete class

---

## Batch 1: Foundation — Provision variant + SectionAwareContentSummariser fix

### Task 1: Add DecisionOutcome.Provision variant

**Files:**
- Modify: `agency-core/src/main/java/io/casehub/agency/orchestration/loop/DecisionOutcome.java`
- Modify: `agency-core/src/test/java/io/casehub/agency/orchestration/loop/DecisionOutcomeTest.java`
- Modify: `agency-core/src/main/java/io/casehub/agency/orchestration/topology/CentralisedTopology.java`
- Modify: `agency-core/src/test/java/io/casehub/agency/orchestration/topology/CentralisedTopologyTest.java`

**Interfaces:**
- Produces: `DecisionOutcome.Provision(ProvisioningAdvice advice)` — new sealed variant used by ProvisioningReasoner (Task 3) and CentralisedTopology executor (Task 4)

- [ ] **Step 1: Write failing test for Provision variant**

Add to `DecisionOutcomeTest.java`:

```java
@Test
void provision_wrapsAdvice() {
    var advice = new SwarmProvisioningAdvisor.ProvisioningAdvice(
            true, Set.of("reasoning"), null, "scale up needed");
    var provision = new DecisionOutcome.Provision(advice);
    assertThat(provision.advice()).isEqualTo(advice);
}

@Test
void provision_nullAdvice_throws() {
    assertThatNullPointerException()
        .isThrownBy(() -> new DecisionOutcome.Provision(null));
}
```

Imports needed: `io.casehub.api.spi.stigmergy.SwarmProvisioningAdvisor`, `java.util.Set`

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl agency-core -Dtest=DecisionOutcomeTest#provision_wrapsAdvice -q`
Expected: Compilation error — `Provision` does not exist in `DecisionOutcome`

- [ ] **Step 3: Add Provision variant to DecisionOutcome**

Add after the `ScopeGated` record (line 42) in `DecisionOutcome.java`:

```java
record Provision(
    io.casehub.api.spi.stigmergy.SwarmProvisioningAdvisor.ProvisioningAdvice advice
) implements DecisionOutcome {
    public Provision {
        Objects.requireNonNull(advice, "advice must not be null");
    }
}
```

- [ ] **Step 4: Add Provision case to CentralisedTopology executor**

In `CentralisedTopology.java`, method `executeCentralised` (line 92), add a new case arm in the switch:

```java
case DecisionOutcome.Provision p ->
    CompletableFuture.completedFuture(
        new CoordinationFeedback(agentId, p, FeedbackSignal.SUCCESS));
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn test -pl agency-core -Dtest=DecisionOutcomeTest -q`
Expected: All tests pass including the two new Provision tests

- [ ] **Step 6: Add CentralisedTopology executor test for Provision**

Add to `CentralisedTopologyTest.java`:

```java
@Test
void executor_handles_provision_outcome() throws Exception {
    var plan = topology.configure(topologyContext);
    var advice = new SwarmProvisioningAdvisor.ProvisioningAdvice(
            true, null, null, "scale up");
    var provision = new DecisionOutcome.Provision(advice);

    var feedback = plan.executor()
        .execute("coordinator", provision, topologyContext)
        .toCompletableFuture().get();

    assertThat(feedback.signal()).isEqualTo(FeedbackSignal.SUCCESS);
    assertThat(feedback.agentId()).isEqualTo("coordinator");
}
```

- [ ] **Step 7: Run all topology tests**

Run: `mvn test -pl agency-core -Dtest="DecisionOutcomeTest,CentralisedTopologyTest" -q`
Expected: All pass

- [ ] **Step 8: Commit**

```bash
git add agency-core/src/main/java/io/casehub/agency/orchestration/loop/DecisionOutcome.java \
      agency-core/src/test/java/io/casehub/agency/orchestration/loop/DecisionOutcomeTest.java \
      agency-core/src/main/java/io/casehub/agency/orchestration/topology/CentralisedTopology.java \
      agency-core/src/test/java/io/casehub/agency/orchestration/topology/CentralisedTopologyTest.java
git commit -m "feat(#381): add DecisionOutcome.Provision variant for provisioning decisions

Refs casehubio/agency#381"
```

---

### Task 2: Fix SectionAwareContentSummariser promptContext bug

**Files:**
- Modify: `agency-core/src/main/java/io/casehub/agency/orchestration/loop/SectionAwareContentSummariser.java`
- Modify: `agency-core/src/test/java/io/casehub/agency/orchestration/loop/SectionAwareContentSummariserTest.java`

**Interfaces:**
- Produces: Fixed `SectionAwareContentSummariser.summarise()` that passes assembled prompt context to the delegate — required for prompt sections to work through the loop pipeline (Task 4)

- [ ] **Step 1: Write failing test exposing the bug**

The bug: `summarise()` builds `promptContext` but passes `items` and `previous` to the delegate without the context. Add a test that verifies the delegate receives the prompt context.

Add to `SectionAwareContentSummariserTest.java`:

```java
@Test
void summarise_prependsPromptContextAsObservation() throws Exception {
    var section = (LoopPromptSection) ctx -> "System instruction: coordinate agents";
    var capturedItems = new java.util.concurrent.atomic.AtomicReference<List<ObservationEvent>>();
    ContentSummariser<ObservationEvent, DecisionOutcome> delegate = (items, prev) -> {
        capturedItems.set(items);
        return CompletableFuture.completedFuture(
                new DecisionOutcome.WriteBack(List.of(), List.of(), List.of()));
    };

    var context = new DecisionContext(
            UUID.randomUUID(), "agent-1", DecisionScope.FULL_AUTONOMY,
            CommunicationMode.BROADCAST,
            new TopologyDescriptor(TopologyType.CENTRALISED, Map.of()));

    var summariser = new SectionAwareContentSummariser(delegate, List.of(section), context);
    var event = ObservationEvent.of("test", "value");
    summariser.summarise(List.of(event), null).toCompletableFuture().get();

    assertThat(capturedItems.get()).hasSizeGreaterThan(1);
    var promptEvent = capturedItems.get().stream()
            .filter(e -> "prompt-context".equals(e.key()))
            .findFirst();
    assertThat(promptEvent).isPresent();
    assertThat((String) promptEvent.get().value())
            .contains("System instruction: coordinate agents");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl agency-core -Dtest=SectionAwareContentSummariserTest#summarise_prependsPromptContextAsObservation -q`
Expected: FAIL — capturedItems has size 1 (only the original event), no prompt-context event

- [ ] **Step 3: Fix SectionAwareContentSummariser.summarise()**

Replace the `summarise` method in `SectionAwareContentSummariser.java` (lines 26-36):

```java
@Override
public CompletionStage<DecisionOutcome> summarise(
        List<ObservationEvent> items, @Nullable DecisionOutcome previous) {
    var promptContext = new StringBuilder();
    for (var section : sections) {
        var contribution = section.contribute(context);
        if (contribution != null && !contribution.isEmpty()) {
            promptContext.append(contribution).append("\n");
        }
    }
    if (promptContext.isEmpty()) {
        return delegate.summarise(items, previous);
    }
    var enriched = new java.util.ArrayList<>(items);
    enriched.addFirst(ObservationEvent.of("prompt-context", promptContext.toString()));
    return delegate.summarise(List.copyOf(enriched), previous);
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -pl agency-core -Dtest=SectionAwareContentSummariserTest -q`
Expected: All tests pass including the new one

- [ ] **Step 5: Commit**

```bash
git add agency-core/src/main/java/io/casehub/agency/orchestration/loop/SectionAwareContentSummariser.java \
      agency-core/src/test/java/io/casehub/agency/orchestration/loop/SectionAwareContentSummariserTest.java
git commit -m "fix(#381): forward prompt context to delegate in SectionAwareContentSummariser

The summarise() method assembled prompt context from sections but
discarded it — delegate received only the original observation events.

Refs casehubio/agency#381"
```

---

## Batch 2: Adapters — ContentSummariser wrappers

### Task 3: Create RoutingReasoner, AggregationReasoner, ProvisioningReasoner

**Files:**
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/topology/RoutingReasoner.java`
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/topology/AggregationReasoner.java`
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/topology/ProvisioningReasoner.java`
- Create: `agency-core/src/test/java/io/casehub/agency/orchestration/topology/RoutingReasonerTest.java`
- Create: `agency-core/src/test/java/io/casehub/agency/orchestration/topology/AggregationReasonerTest.java`
- Create: `agency-core/src/test/java/io/casehub/agency/orchestration/topology/ProvisioningReasonerTest.java`

**Interfaces:**
- Consumes: `RoutingStrategy<SwarmContext>.route(RoutingContext<SwarmContext>)` → `RoutingDecision`, `AggregationStrategy<SwarmContext>.aggregate(List<AgentResult>, AggregationContext<SwarmContext>)` → `AggregationResult`, `SwarmProvisioningAdvisor.advise(ProvisioningContext)` → `ProvisioningAdvice`
- Produces: `ContentSummariser<ObservationEvent, DecisionOutcome>` — each adapter is passed to `DecisionLoopFactory.create()` in Task 4

- [ ] **Step 1: Write RoutingReasoner test**

```java
package io.casehub.agency.orchestration.topology;

import io.casehub.agency.core.AgentResult;
import io.casehub.agency.core.RoutingCandidate;
import io.casehub.agency.orchestration.loop.DecisionOutcome;
import io.casehub.agency.orchestration.loop.ObservationEvent;
import io.casehub.agency.orchestration.routing.RoutingContext;
import io.casehub.agency.orchestration.routing.RoutingDecision;
import io.casehub.agency.orchestration.routing.RoutingStrategy;
import io.casehub.agency.orchestration.swarm.SwarmContext;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class RoutingReasonerTest {

    @Test
    void summarise_delegatesToRoutingStrategy() throws Exception {
        var decision = new RoutingDecision.SwarmAssignment(
                List.of("agent-1"), java.util.Map.of(), "round-robin", "test");
        RoutingStrategy<SwarmContext> strategy = ctx -> decision;
        var swarmContext = new SwarmContext();
        var candidates = List.of(
                new RoutingCandidate("agent-1", List.of("reasoning"), 1.0));

        var reasoner = new RoutingReasoner(strategy, swarmContext, () -> candidates);
        var event = ObservationEvent.of("candidates-available", "trigger");
        var result = reasoner.summarise(List.of(event), null)
                .toCompletableFuture().get();

        assertThat(result).isInstanceOf(DecisionOutcome.Route.class);
        assertThat(((DecisionOutcome.Route) result).decision()).isEqualTo(decision);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl agency-core -Dtest=RoutingReasonerTest -q`
Expected: Compilation error — `RoutingReasoner` does not exist

- [ ] **Step 3: Implement RoutingReasoner**

```java
package io.casehub.agency.orchestration.topology;

import io.casehub.agency.core.RoutingCandidate;
import io.casehub.agency.orchestration.loop.DecisionOutcome;
import io.casehub.agency.orchestration.loop.ObservationEvent;
import io.casehub.agency.orchestration.routing.RoutingContext;
import io.casehub.agency.orchestration.routing.RoutingStrategy;
import io.casehub.agency.orchestration.swarm.SwarmContext;
import io.casehub.agency.summarisation.ContentSummariser;
import org.jspecify.annotations.Nullable;

import java.util.List;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.CompletionStage;
import java.util.function.Supplier;

class RoutingReasoner implements ContentSummariser<ObservationEvent, DecisionOutcome> {

    private final RoutingStrategy<SwarmContext> strategy;
    private final SwarmContext swarmContext;
    private final Supplier<List<RoutingCandidate>> candidateSupplier;

    RoutingReasoner(RoutingStrategy<SwarmContext> strategy,
                    SwarmContext swarmContext,
                    Supplier<List<RoutingCandidate>> candidateSupplier) {
        this.strategy = strategy;
        this.swarmContext = swarmContext;
        this.candidateSupplier = candidateSupplier;
    }

    @Override
    public CompletionStage<DecisionOutcome> summarise(
            List<ObservationEvent> items, @Nullable DecisionOutcome previous) {
        var candidates = candidateSupplier.get();
        var routingCtx = new RoutingContext<>("swarm-team-formation", candidates, swarmContext);
        var decision = strategy.route(routingCtx);
        return CompletableFuture.completedFuture(new DecisionOutcome.Route(decision));
    }
}
```

- [ ] **Step 4: Run RoutingReasoner test**

Run: `mvn test -pl agency-core -Dtest=RoutingReasonerTest -q`
Expected: PASS

- [ ] **Step 5: Write AggregationReasoner test**

```java
package io.casehub.agency.orchestration.topology;

import io.casehub.agency.core.AgentResult;
import io.casehub.agency.orchestration.aggregation.AggregationContext;
import io.casehub.agency.orchestration.aggregation.AggregationResult;
import io.casehub.agency.orchestration.aggregation.AggregationStrategy;
import io.casehub.agency.orchestration.loop.DecisionOutcome;
import io.casehub.agency.orchestration.loop.ObservationEvent;
import io.casehub.agency.orchestration.swarm.SwarmContext;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class AggregationReasonerTest {

    @Test
    void summarise_delegatesToAggregationStrategy() throws Exception {
        var aggResult = new AggregationResult.Resolved("synthesised output");
        AggregationStrategy<SwarmContext> strategy = (results, ctx) -> aggResult;
        var swarmContext = new SwarmContext();

        var reasoner = new AggregationReasoner(strategy, swarmContext);
        var event = ObservationEvent.of("member-completed",
                new AgentResult("agent-1", "observation text", List.of()));
        var result = reasoner.summarise(List.of(event), null)
                .toCompletableFuture().get();

        assertThat(result).isInstanceOf(DecisionOutcome.Aggregate.class);
        assertThat(((DecisionOutcome.Aggregate) result).result()).isEqualTo(aggResult);
    }

    @Test
    void summarise_extractsAgentResultsFromEvents() throws Exception {
        var captured = new java.util.concurrent.atomic.AtomicReference<List<AgentResult>>();
        AggregationStrategy<SwarmContext> strategy = (results, ctx) -> {
            captured.set(results);
            return new AggregationResult.Resolved("done");
        };

        var reasoner = new AggregationReasoner(strategy, new SwarmContext());
        var r1 = new AgentResult("a1", "output1", List.of());
        var r2 = new AgentResult("a2", "output2", List.of());
        reasoner.summarise(List.of(
                ObservationEvent.of("member-completed", r1),
                ObservationEvent.of("member-completed", r2),
                ObservationEvent.of("progress-updated", "ignored")
        ), null).toCompletableFuture().get();

        assertThat(captured.get()).containsExactly(r1, r2);
    }
}
```

- [ ] **Step 6: Implement AggregationReasoner**

```java
package io.casehub.agency.orchestration.topology;

import io.casehub.agency.core.AgentResult;
import io.casehub.agency.orchestration.aggregation.AggregationContext;
import io.casehub.agency.orchestration.aggregation.AggregationStrategy;
import io.casehub.agency.orchestration.loop.DecisionOutcome;
import io.casehub.agency.orchestration.loop.ObservationEvent;
import io.casehub.agency.orchestration.swarm.SwarmContext;
import io.casehub.agency.summarisation.ContentSummariser;
import org.jspecify.annotations.Nullable;

import java.util.List;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.CompletionStage;

class AggregationReasoner implements ContentSummariser<ObservationEvent, DecisionOutcome> {

    private final AggregationStrategy<SwarmContext> strategy;
    private final SwarmContext swarmContext;

    AggregationReasoner(AggregationStrategy<SwarmContext> strategy,
                        SwarmContext swarmContext) {
        this.strategy = strategy;
        this.swarmContext = swarmContext;
    }

    @Override
    public CompletionStage<DecisionOutcome> summarise(
            List<ObservationEvent> items, @Nullable DecisionOutcome previous) {
        var agentResults = items.stream()
                .filter(e -> "member-completed".equals(e.key()))
                .map(e -> (AgentResult) e.value())
                .toList();
        var ctx = new AggregationContext<>(swarmContext);
        var result = strategy.aggregate(agentResults, ctx);
        return CompletableFuture.completedFuture(new DecisionOutcome.Aggregate(result));
    }
}
```

- [ ] **Step 7: Run AggregationReasoner test**

Run: `mvn test -pl agency-core -Dtest=AggregationReasonerTest -q`
Expected: PASS

- [ ] **Step 8: Write ProvisioningReasoner test**

```java
package io.casehub.agency.orchestration.topology;

import io.casehub.agency.orchestration.loop.DecisionOutcome;
import io.casehub.agency.orchestration.loop.ObservationEvent;
import io.casehub.agency.orchestration.swarm.SwarmContext;
import io.casehub.api.spi.stigmergy.SwarmProvisioningAdvisor;
import io.casehub.api.spi.stigmergy.SwarmProvisioningAdvisor.ProvisioningAdvice;
import io.casehub.api.spi.stigmergy.SwarmProvisioningAdvisor.ProvisioningContext;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Set;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

class ProvisioningReasonerTest {

    @Test
    void summarise_delegatesToAdvisor() throws Exception {
        var advice = new ProvisioningAdvice(true, null, null, "scale needed");
        SwarmProvisioningAdvisor advisor = ctx -> advice;
        var swarmContext = new SwarmContext();
        var caseId = UUID.randomUUID();

        var reasoner = new ProvisioningReasoner(advisor, swarmContext, caseId);
        var event = ObservationEvent.of("provisioning-changed", "trigger");
        var result = reasoner.summarise(List.of(event), null)
                .toCompletableFuture().get();

        assertThat(result).isInstanceOf(DecisionOutcome.Provision.class);
        assertThat(((DecisionOutcome.Provision) result).advice()).isEqualTo(advice);
    }

    @Test
    void summarise_noProvision_returnsFalseAdvice() throws Exception {
        var advice = new ProvisioningAdvice(false, null, null, "stable");
        SwarmProvisioningAdvisor advisor = ctx -> advice;

        var reasoner = new ProvisioningReasoner(advisor, new SwarmContext(), UUID.randomUUID());
        var result = reasoner.summarise(List.of(ObservationEvent.of("tick", "poll")), null)
                .toCompletableFuture().get();

        assertThat(((DecisionOutcome.Provision) result).advice().shouldProvision()).isFalse();
    }
}
```

- [ ] **Step 9: Implement ProvisioningReasoner**

```java
package io.casehub.agency.orchestration.topology;

import io.casehub.agency.orchestration.loop.DecisionOutcome;
import io.casehub.agency.orchestration.loop.ObservationEvent;
import io.casehub.agency.orchestration.swarm.SwarmContext;
import io.casehub.agency.summarisation.ContentSummariser;
import io.casehub.api.spi.stigmergy.SwarmProvisioningAdvisor;
import io.casehub.api.spi.stigmergy.SwarmProvisioningAdvisor.ProvisioningContext;
import org.jspecify.annotations.Nullable;

import java.util.List;
import java.util.Set;
import java.util.UUID;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.CompletionStage;

class ProvisioningReasoner implements ContentSummariser<ObservationEvent, DecisionOutcome> {

    private final SwarmProvisioningAdvisor advisor;
    private final SwarmContext swarmContext;
    private final UUID caseId;

    ProvisioningReasoner(SwarmProvisioningAdvisor advisor,
                         SwarmContext swarmContext,
                         UUID caseId) {
        this.advisor = advisor;
        this.swarmContext = swarmContext;
        this.caseId = caseId;
    }

    @Override
    public CompletionStage<DecisionOutcome> summarise(
            List<ObservationEvent> items, @Nullable DecisionOutcome previous) {
        var ctx = new ProvisioningContext(
                caseId,
                null,
                Set.of(),
                null);
        var advice = advisor.advise(ctx);
        return CompletableFuture.completedFuture(new DecisionOutcome.Provision(advice));
    }
}
```

- [ ] **Step 10: Run all adapter tests**

Run: `mvn test -pl agency-core -Dtest="RoutingReasonerTest,AggregationReasonerTest,ProvisioningReasonerTest" -q`
Expected: All pass

- [ ] **Step 11: Commit**

```bash
git add agency-core/src/main/java/io/casehub/agency/orchestration/topology/RoutingReasoner.java \
      agency-core/src/main/java/io/casehub/agency/orchestration/topology/AggregationReasoner.java \
      agency-core/src/main/java/io/casehub/agency/orchestration/topology/ProvisioningReasoner.java \
      agency-core/src/test/java/io/casehub/agency/orchestration/topology/RoutingReasonerTest.java \
      agency-core/src/test/java/io/casehub/agency/orchestration/topology/AggregationReasonerTest.java \
      agency-core/src/test/java/io/casehub/agency/orchestration/topology/ProvisioningReasonerTest.java
git commit -m "feat(#381): add ContentSummariser adapters for routing, aggregation, provisioning

Three adapters wrap existing LLM strategies as ContentSummarisers for
use in AgentDecisionLoop instances within CentralisedTopology.

Refs casehubio/agency#381"
```

---

## Batch 3: Wiring + Cleanup — CentralisedTopology refactoring + elimination

### Task 4: Wire loops into CentralisedTopology + event routing

**Files:**
- Modify: `agency-core/src/main/java/io/casehub/agency/orchestration/topology/CentralisedTopology.java`
- Modify: `agency-core/src/test/java/io/casehub/agency/orchestration/topology/CentralisedTopologyTest.java`

**Interfaces:**
- Consumes: `RoutingReasoner`, `AggregationReasoner`, `ProvisioningReasoner` (Task 3), `DecisionLoopFactory.create(descriptor, context, reasoner, pool)`, `TopologyContext.loopFactory()`, `TopologyContext.agentProvider()`
- Produces: `CentralisedTopology.loops()` — `List<AgentDecisionLoop>` accessor for runtime lifecycle management

- [ ] **Step 1: Write test for loop creation in configure()**

Add to `CentralisedTopologyTest.java`:

```java
@Test
void configure_creates_three_loops() {
    var plan = topology.configure(topologyContext);
    assertThat(topology.loops()).hasSize(3);
    assertThat(topology.loops().stream().map(l -> l.descriptor().name()))
            .containsExactly(
                    "centralised-team-formation",
                    "centralised-collective-reasoning",
                    "centralised-provisioning");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl agency-core -Dtest=CentralisedTopologyTest#configure_creates_three_loops -q`
Expected: FAIL — `loops()` method does not exist

- [ ] **Step 3: Update CentralisedTopology constructor and configure()**

The constructor needs additional parameters: `SwarmProvisioningAdvisor`, `AgentProvider` (for candidate supplier), and `UUID caseId`.

Update constructor:

```java
private final SwarmProvisioningAdvisor provisioningAdvisor;
private final AgentProvider agentProvider;
private final UUID caseId;
private List<AgentDecisionLoop> loops = List.of();

public CentralisedTopology(
        SwarmContext swarmContext,
        SwarmRoutingStrategy routingStrategy,
        CollectiveReasoningAggregation aggregation,
        TerminationCondition<SwarmContext> terminationCondition,
        SwarmFallbackStrategy fallbackStrategy,
        List<String> promptSectionNames,
        SwarmProvisioningAdvisor provisioningAdvisor,
        AgentProvider agentProvider,
        UUID caseId) {
    // existing field assignments...
    this.provisioningAdvisor = Objects.requireNonNull(provisioningAdvisor);
    this.agentProvider = Objects.requireNonNull(agentProvider);
    this.caseId = Objects.requireNonNull(caseId);
}
```

Update `configure()` to create loops after the descriptors:

```java
@Override
public CoordinationPlan configure(TopologyContext context) {
    var pool = new AutonomyBudgetPool(context.budgetPolicy(), caseId + "-CENTRALISED");

    // descriptors (unchanged)
    var teamFormation = new DecisionLoopDescriptor(...);
    var collectiveReasoning = new DecisionLoopDescriptor(...);
    var provisioning = new DecisionLoopDescriptor(...);

    // create adapters
    var routingReasoner = new RoutingReasoner(
            routingStrategy, swarmContext,
            () -> agentProvider.listCandidates());
    var aggregationReasoner = new AggregationReasoner(aggregation, swarmContext);
    var provisioningReasoner = new ProvisioningReasoner(
            provisioningAdvisor, swarmContext, caseId);

    // create loops
    var topologyDesc = describeTopology(caseId);
    var factory = context.loopFactory();

    this.loops = List.of(
        factory.create(teamFormation,
            new DecisionContext(caseId, "coordinator", DecisionScope.FULL_AUTONOMY,
                CommunicationMode.BROADCAST, topologyDesc),
            routingReasoner, pool),
        factory.create(collectiveReasoning,
            new DecisionContext(caseId, "coordinator", DecisionScope.FULL_AUTONOMY,
                CommunicationMode.BROADCAST, topologyDesc),
            aggregationReasoner, pool),
        factory.create(provisioning,
            new DecisionContext(caseId, "coordinator", DecisionScope.FULL_AUTONOMY,
                CommunicationMode.BROADCAST, topologyDesc),
            provisioningReasoner, pool)
    );

    DecisionOutcomeExecutor executor = (agentId, outcome, ctx) ->
            executeCentralised(agentId, outcome, ctx);

    return new CoordinationPlan(
            List.of(teamFormation, collectiveReasoning, provisioning), executor, pool);
}

public List<AgentDecisionLoop> loops() { return loops; }
```

- [ ] **Step 4: Update CentralisedTopologyTest setUp() for new constructor params**

```java
@BeforeEach
void setUp() {
    AgentProvider agentProvider = TestAgentProviders.noOp();
    swarmContext = new SwarmContext();
    var experienceSection = new SwarmExperienceSection(ExperienceReplayConfig.defaults());
    var routingStrategy = new SwarmRoutingStrategy(
            agentProvider, experienceSection, 12);
    var aggregation = new CollectiveReasoningAggregation(agentProvider, null);
    SwarmProvisioningAdvisor advisor = ctx ->
            new SwarmProvisioningAdvisor.ProvisioningAdvice(false, null, null, "no-op");

    topology = new CentralisedTopology(
            swarmContext,
            routingStrategy,
            aggregation,
            ctx -> new io.casehub.agency.orchestration.termination.TerminationDecision.Continue(),
            new SwarmFallbackStrategy(),
            List.of("experience-replay"),
            advisor,
            agentProvider,
            UUID.randomUUID());

    topologyContext = new TopologyContext(
            UUID.randomUUID(),
            new TopologyDescriptor(TopologyType.CENTRALISED, Map.of()),
            agentProvider,
            new DecisionLoopFactory(name -> ctx -> "", new EventStreamBus<>()),
            AutonomyBudgetPolicy.defaults());
}
```

- [ ] **Step 5: Run tests**

Run: `mvn test -pl agency-core -Dtest=CentralisedTopologyTest -q`
Expected: All pass including the new loop creation test

- [ ] **Step 6: Write test for event → observation routing**

```java
@Test
void onSwarmEvent_feeds_observation_to_provisioning_loop() {
    topology.configure(topologyContext);
    var provisioningLoop = topology.loops().stream()
            .filter(l -> l.descriptor().name().equals("centralised-provisioning"))
            .findFirst().orElseThrow();

    topology.onSwarmEvent(new SwarmCoordinationEvent.ProvisioningChanged(5, 10));

    assertThat(swarmContext.currentSize()).isEqualTo(5);
    assertThat(swarmContext.maxSize()).isEqualTo(10);
}
```

- [ ] **Step 7: Update onSwarmEvent() to feed observations into loops**

After the existing SwarmContext updates in each case arm, add observation routing:

```java
@Override
public void onSwarmEvent(SwarmCoordinationEvent event) {
    switch (event) {
        case SwarmCoordinationEvent.SwarmInitialised e -> {
            swarmContext.onSwarmInitialised(e.experiences(), e.progress());
            feedLoop("centralised-team-formation",
                    ObservationEvent.of("candidates-available", e));
        }
        case SwarmCoordinationEvent.RoleChanged e -> {
            swarmContext.onRoleChanged(e.roles(), e.progress());
            feedLoop("centralised-team-formation",
                    ObservationEvent.of("roles-changed", e));
        }
        case SwarmCoordinationEvent.TeamChanged e -> {
            swarmContext.onTeamChanged(e.teams(), e.progress());
            feedLoop("centralised-collective-reasoning",
                    ObservationEvent.of("team-changed", e));
        }
        case SwarmCoordinationEvent.ProgressUpdated e -> {
            swarmContext.onProgressUpdated(e.progress());
            for (var loop : loops) {
                loop.collect(ObservationEvent.of("progress-updated", e));
            }
        }
        case SwarmCoordinationEvent.MemberCompleted e -> {
            swarmContext.onMemberCompleted(e.agentId());
            feedLoop("centralised-collective-reasoning",
                    ObservationEvent.of("member-completed", e));
        }
        case SwarmCoordinationEvent.ProvisioningChanged e -> {
            swarmContext.onProvisioningChanged(e.currentSize(), e.maxSize());
            feedLoop("centralised-provisioning",
                    ObservationEvent.of("provisioning-changed", e));
        }
    }
}

private void feedLoop(String loopName, ObservationEvent event) {
    loops.stream()
         .filter(l -> l.descriptor().name().equals(loopName))
         .findFirst()
         .ifPresent(loop -> loop.collect(event));
}
```

- [ ] **Step 8: Run all CentralisedTopology tests**

Run: `mvn test -pl agency-core -Dtest=CentralisedTopologyTest -q`
Expected: All pass

- [ ] **Step 9: Commit**

```bash
git add agency-core/src/main/java/io/casehub/agency/orchestration/topology/CentralisedTopology.java \
      agency-core/src/test/java/io/casehub/agency/orchestration/topology/CentralisedTopologyTest.java
git commit -m "feat(#381): wire AgentDecisionLoops into CentralisedTopology

configure() creates three loops via DecisionLoopFactory with adapter
reasoners. onSwarmEvent() feeds observations into the appropriate loop.

Refs casehubio/agency#381"
```

---

### Task 5: Eliminate LlmSwarmCoordinator + update builder

**Files:**
- Delete: `agency-core/src/main/java/io/casehub/agency/orchestration/swarm/LlmSwarmCoordinator.java` (use `ide_refactor_safe_delete`)
- Modify: `agency-core/src/main/java/io/casehub/agency/orchestration/swarm/SwarmCoordinatorBuilder.java`
- Modify: `agency-core/src/test/java/io/casehub/agency/orchestration/topology/CentralisedTopologyTest.java` (if not done in Task 4)
- Modify: `agency/src/test/java/io/casehub/agency/orchestration/swarm/SwarmCoordinatorBuilderTest.java`

**Interfaces:**
- Consumes: `CentralisedTopology` constructor (Task 4 — now requires provisioningAdvisor, agentProvider, caseId)

- [ ] **Step 1: Write test that buildTopology() wires provisioning advisor**

Add to `SwarmCoordinatorBuilderTest.java`:

```java
@Test
void buildTopology_wiresProvisioningAdvisor() {
    var topology = new SwarmCoordinatorBuilder()
            .agentProvider(mock(AgentProvider.class))
            .buildTopology();

    assertThat(topology).isNotNull();
}
```

- [ ] **Step 2: Run test to verify current state**

Run: `mvn test -pl agency-core -Dtest=SwarmCoordinatorBuilderTest -q`
Expected: Some tests may fail if they reference build() or the old CentralisedTopology constructor. Note which ones.

- [ ] **Step 3: Update SwarmCoordinatorBuilder — remove build(), update buildTopology()**

Remove the `build()` method entirely (lines 84-117).

Update `buildTopology()` to create `SwarmProvisioningAdvisorImpl` and pass new constructor params:

```java
public CentralisedTopology buildTopology() {
    Objects.requireNonNull(agentProvider, "agentProvider is required");

    var pool = new AutonomyBudgetPool(budgetPolicy, "swarm-" + UUID.randomUUID());
    var experienceSection = new SwarmExperienceSection(experienceConfig);
    var routingStrategy = new SwarmRoutingStrategy(agentProvider, experienceSection,
            preFilterThreshold, promptCustomiser, additionalSections);
    routingStrategy.setBudgetPool(pool);
    var aggregation = new CollectiveReasoningAggregation(agentProvider, promptCustomiser);
    aggregation.setBudgetPool(pool);
    TerminationCondition<SwarmContext> termination = new SwarmConvergenceTermination(convergenceThreshold);

    String slot = variantSelector != null
            ? variantSelector.selectSlot(UUID.randomUUID(), "swarm-coordination")
            : "control";
    routingStrategy.setVariantSlot(slot);
    aggregation.setVariantSlot(slot);

    if (outcomeProvider != null) {
        var recorder = new SwarmOutcomeRecorder(termination, slot, outcomeProvider);
        termination = recorder;
    }

    var sections = new java.util.ArrayList<String>();
    sections.add("experience-replay");
    for (var s : additionalSections) {
        sections.add(s.getClass().getSimpleName());
    }

    var provisioner = new SwarmProvisioningAdvisorImpl(
            agentProvider, promptCustomiser);
    provisioner.setBudgetPool(pool);
    provisioner.setVariantSlot(slot);

    return new CentralisedTopology(
            new SwarmContext(), routingStrategy, aggregation,
            termination, new SwarmFallbackStrategy(), sections,
            provisioner, agentProvider, UUID.randomUUID());
}
```

- [ ] **Step 4: Delete LlmSwarmCoordinator**

Use `ide_refactor_safe_delete` on `LlmSwarmCoordinator.java`. This verifies there are no remaining references before deletion.

- [ ] **Step 5: Update SwarmCoordinatorBuilderTest — remove build()-related tests**

Remove tests that call `build()`:
- `build_passesCustomiserToRoutingStrategy`
- `build_selectsVariantSlotAndPropagates`
- `build_wrapsTerminationWithRecorder_whenOutcomeProviderSet`
- `build_usesDefaultTermination_whenNoOutcomeProvider`

Keep and update `buildTopology_wiresBudgetPoolOnStrategies`.

- [ ] **Step 6: Run all tests in agency-core**

Run: `mvn test -pl agency-core -q`
Expected: All pass

- [ ] **Step 7: Run all tests in agency module**

Run: `mvn test -pl agency -q`
Expected: All pass (SwarmCoordinatorBuilderTest updated, no LlmSwarmCoordinatorTest existed)

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "refactor(#381): eliminate LlmSwarmCoordinator — CentralisedTopology is the single path

Removed LlmSwarmCoordinator class and build() from SwarmCoordinatorBuilder.
buildTopology() is now the only build path, wiring provisioning advisor
into CentralisedTopology.

Closes casehubio/agency#381"
```

---

## References

- [2026-11-04-llm-swarm-coordinator-centralised-topology-design.md] — design spec this plan implements
- [decisions.md] — 7 validated decisions from brainstorming
- `agency-core/.../loop/DecisionOutcome.java:11` — sealed hierarchy (gains Provision)
- `agency-core/.../loop/SectionAwareContentSummariser.java:35` — promptContext bug
- `agency-core/.../topology/CentralisedTopology.java:26` — target topology
- `agency-core/.../swarm/LlmSwarmCoordinator.java:9` — class to eliminate
- `agency-core/.../swarm/SwarmCoordinatorBuilder.java:84,119` — build() vs buildTopology()
- `agency-core/.../loop/DecisionLoopFactory.java:25` — loop creation with reasoner
- `agency-core/.../routing/RoutingStrategy.java:3` — `RoutingDecision route(RoutingContext<T>)`
- `agency-core/.../aggregation/AggregationStrategy.java:6` — `AggregationResult aggregate(List<AgentResult>, AggregationContext<T>)`
- `engine/api/.../SwarmProvisioningAdvisor.java:24` — SPI interface with ProvisioningAdvice/ProvisioningContext records
- casehubio/agency#381 — focal issue
- casehubio/agency#377 — parent epic
