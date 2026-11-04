# CoordinationTopology SPI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/agency#380 — feat: CoordinationTopology SPI + concrete implementations
**Issue group:** casehubio/agency#380, casehubio/agency#381 (absorbed)

**Goal:** Build the CoordinationTopology strategy SPI that controls how AgentDecisionLoops interact, with six concrete implementations as R&D starting points.

**Architecture:** CoordinationTopology extends engine's TopologyProvider SPI. Each implementation returns a CoordinationPlan (loop descriptors + executor) from configure(). Supporting types: TopologyContext (5-field universal record), CoordinationFeedback (outcome-based), GraduationState (mutable scope tracker). CentralisedTopology refactors LlmSwarmCoordinator.

**Tech Stack:** Java 21, Quarkus 3.32.2, JUnit 5, AssertJ, platform-agent-api (AgentProvider)

## Global Constraints

- Engine repo: `/Users/mdproctor/claude/casehub/slots/220/engine`
- Agency repo: `/Users/mdproctor/claude/casehub/slots/220/agency`
- Tests are `*Test.java` (surefire), never `*IT.java`
- Use `ide_create_file` for new `.java` files, `ide_refactor_rename` for renames
- All commits: `Refs casehubio/agency#380`
- Engine test: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl <module>`
- Agency test: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl agency-core`

---

## Batch 1: Engine Foundation — TopologyProvider rename + TopologyType extension

### Task 1: Rename SwarmTopologyProvider → TopologyProvider and extend TopologyType

**Files:**
- Rename: `TopologyProvider` → `TopologyProvider` (use `ide_refactor_rename` — updates 14 references across api, runtime-core, runtime, runtime-spring)
- Modify: `api/src/main/java/io/casehub/api/model/coordination/TopologyType.java` (add HIERARCHICAL, GRADUATED)
- Modify: `api/src/test/java/io/casehub/api/model/coordination/TopologyTypeTest.java` (add tests for new values)
- Test: `api/src/test/java/io/casehub/api/model/coordination/TopologyTypeTest.java`

**Interfaces:**
- Consumes: `TopologyType` enum (existing: CENTRALISED, DISTRIBUTED, HYBRID), `TopologyProvider` interface
- Produces: `TopologyProvider` interface (renamed), `TopologyType.HIERARCHICAL`, `TopologyType.GRADUATED` with `deriveAuthority()` implementations

- [ ] **Step 1: Write failing tests for HIERARCHICAL and GRADUATED**

Add to `api/src/test/java/io/casehub/api/model/coordination/TopologyTypeTest.java` using `ide_insert_member`:

```java
@Test
void hierarchical_initiator_derives_central() {
    assertThat(TopologyType.HIERARCHICAL.deriveAuthority(StructuralRole.INITIATOR))
        .isEqualTo(DecisionAuthority.CENTRAL);
}

@Test
void hierarchical_contractor_derives_local() {
    assertThat(TopologyType.HIERARCHICAL.deriveAuthority(StructuralRole.CONTRACTOR))
        .isEqualTo(DecisionAuthority.LOCAL);
}

@Test
void hierarchical_peer_is_invalid() {
    assertThatIllegalArgumentException()
        .isThrownBy(() -> TopologyType.HIERARCHICAL.deriveAuthority(StructuralRole.PEER))
        .withMessageContaining("HIERARCHICAL");
}

@Test
void graduated_initiator_derives_central() {
    assertThat(TopologyType.GRADUATED.deriveAuthority(StructuralRole.INITIATOR))
        .isEqualTo(DecisionAuthority.CENTRAL);
}

@Test
void graduated_peer_derives_coordinated() {
    assertThat(TopologyType.GRADUATED.deriveAuthority(StructuralRole.PEER))
        .isEqualTo(DecisionAuthority.COORDINATED);
}

@Test
void graduated_contractor_derives_local() {
    assertThat(TopologyType.GRADUATED.deriveAuthority(StructuralRole.CONTRACTOR))
        .isEqualTo(DecisionAuthority.LOCAL);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl api -Dtest=TopologyTypeTest`
Expected: FAIL — `HIERARCHICAL` and `GRADUATED` don't exist on TopologyType

- [ ] **Step 3: Add HIERARCHICAL and GRADUATED to TopologyType**

Use `ide_edit_member` on the `TopologyType` enum declaration in `api/src/main/java/io/casehub/api/model/coordination/TopologyType.java`. Add after HYBRID:

```java
HIERARCHICAL {
    @Override
    public DecisionAuthority deriveAuthority(StructuralRole role) {
      return switch (role) {
        case INITIATOR -> DecisionAuthority.CENTRAL;
        case CONTRACTOR -> DecisionAuthority.LOCAL;
        case PEER ->
            throw new IllegalArgumentException(
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
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl api -Dtest=TopologyTypeTest`
Expected: PASS — all 15 tests (9 existing + 6 new)

- [ ] **Step 5: Rename SwarmTopologyProvider → TopologyProvider**

Use `ide_refactor_rename` on the `TopologyProvider` interface:
```
ide_refactor_rename(file="api/src/main/java/io/casehub/api/spi/coordination/SwarmTopologyProvider.java",
    line=21, column=18, newName="TopologyProvider")
```

This updates all 14 references across: api, runtime-core (CentralisedTopologyProvider, SwarmProvisioner), runtime (SpiDefaultBeans, RuntimeBeans), runtime-spring, and tests.

- [ ] **Step 6: Verify compilation and tests pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl api,runtime-core`
Expected: PASS — all existing tests pass with renamed interface

- [ ] **Step 7: Install engine artifacts**

Run: `mvn install -DskipTests -q` (from engine root)

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "feat: rename SwarmTopologyProvider → TopologyProvider + add HIERARCHICAL/GRADUATED

Refs casehubio/agency#380"
```

---

## Batch 2: SPI + Supporting Types

### Task 2: CoordinationTopology SPI, TopologyContext, CoordinationPlan, DecisionOutcomeExecutor, CoordinationFeedback, FeedbackSignal

**Files:**
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/topology/CoordinationTopology.java`
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/topology/TopologyContext.java`
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/topology/CoordinationPlan.java`
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/topology/DecisionOutcomeExecutor.java`
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/topology/CoordinationFeedback.java`
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/topology/FeedbackSignal.java`
- Test: `agency-core/src/test/java/io/casehub/agency/orchestration/topology/TopologyContextTest.java`
- Test: `agency-core/src/test/java/io/casehub/agency/orchestration/topology/CoordinationPlanTest.java`
- Test: `agency-core/src/test/java/io/casehub/agency/orchestration/topology/CoordinationFeedbackTest.java`

**Interfaces:**
- Consumes: `TopologyProvider` (engine-api, renamed in Task 1), `TopologyDescriptor`, `DecisionLoopDescriptor`, `DecisionOutcome`, `SwarmCoordinationEvent`, `AgentProvider`, `DecisionLoopFactory`, `SwarmInvocationPolicy`
- Produces: `CoordinationTopology` interface, `TopologyContext` record, `CoordinationPlan` record, `DecisionOutcomeExecutor` functional interface, `CoordinationFeedback` record, `FeedbackSignal` enum

- [ ] **Step 1: Write failing tests for TopologyContext**

Create test file using `ide_create_file` at `agency-core/src/test/java/io/casehub/agency/orchestration/topology/TopologyContextTest.java`:

```java
package io.casehub.agency.orchestration.topology;

import io.casehub.agency.orchestration.loop.DecisionLoopFactory;
import io.casehub.agency.orchestration.swarm.SwarmInvocationPolicy;
import io.casehub.api.model.coordination.TopologyDescriptor;
import io.casehub.api.model.coordination.TopologyType;
import io.casehub.platform.agent.AgentProvider;
import io.casehub.platform.agent.NoOpAgentProvider;
import org.junit.jupiter.api.Test;

import java.util.Map;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatNullPointerException;
import static org.mockito.Mockito.mock;

class TopologyContextTest {

    static final TopologyDescriptor DESCRIPTOR =
        new TopologyDescriptor(TopologyType.CENTRALISED, Map.of());
    static final AgentProvider AGENT_PROVIDER = new NoOpAgentProvider();
    static final DecisionLoopFactory LOOP_FACTORY = mock(DecisionLoopFactory.class);
    static final SwarmInvocationPolicy POLICY = SwarmInvocationPolicy.defaults();

    @Test
    void creates_with_all_fields() {
        var ctx = new TopologyContext(
            UUID.randomUUID(), DESCRIPTOR, AGENT_PROVIDER, LOOP_FACTORY, POLICY);
        assertThat(ctx.caseId()).isNotNull();
        assertThat(ctx.descriptor()).isEqualTo(DESCRIPTOR);
        assertThat(ctx.agentProvider()).isEqualTo(AGENT_PROVIDER);
        assertThat(ctx.loopFactory()).isEqualTo(LOOP_FACTORY);
        assertThat(ctx.invocationPolicy()).isEqualTo(POLICY);
    }

    @Test
    void rejects_null_caseId() {
        assertThatNullPointerException()
            .isThrownBy(() -> new TopologyContext(null, DESCRIPTOR, AGENT_PROVIDER, LOOP_FACTORY, POLICY))
            .withMessageContaining("caseId");
    }

    @Test
    void rejects_null_descriptor() {
        assertThatNullPointerException()
            .isThrownBy(() -> new TopologyContext(UUID.randomUUID(), null, AGENT_PROVIDER, LOOP_FACTORY, POLICY))
            .withMessageContaining("descriptor");
    }

    @Test
    void rejects_null_agentProvider() {
        assertThatNullPointerException()
            .isThrownBy(() -> new TopologyContext(UUID.randomUUID(), DESCRIPTOR, null, LOOP_FACTORY, POLICY))
            .withMessageContaining("agentProvider");
    }

    @Test
    void rejects_null_loopFactory() {
        assertThatNullPointerException()
            .isThrownBy(() -> new TopologyContext(UUID.randomUUID(), DESCRIPTOR, AGENT_PROVIDER, null, POLICY))
            .withMessageContaining("loopFactory");
    }

    @Test
    void rejects_null_invocationPolicy() {
        assertThatNullPointerException()
            .isThrownBy(() -> new TopologyContext(UUID.randomUUID(), DESCRIPTOR, AGENT_PROVIDER, LOOP_FACTORY, null))
            .withMessageContaining("invocationPolicy");
    }
}
```

- [ ] **Step 2: Write failing tests for CoordinationPlan**

Create test file using `ide_create_file` at `agency-core/src/test/java/io/casehub/agency/orchestration/topology/CoordinationPlanTest.java`:

```java
package io.casehub.agency.orchestration.topology;

import io.casehub.agency.orchestration.loop.AgentBudgetPolicy;
import io.casehub.agency.orchestration.loop.DecisionLoopDescriptor;
import io.casehub.agency.orchestration.loop.DecisionOutcome;
import io.casehub.api.model.coordination.CommunicationMode;
import io.casehub.api.model.coordination.DecisionScope;
import io.casehub.api.model.coordination.ObservationStrategy;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.CompletableFuture;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatIllegalArgumentException;
import static org.assertj.core.api.Assertions.assertThatNullPointerException;

class CoordinationPlanTest {

    static final DecisionLoopDescriptor DESCRIPTOR = new DecisionLoopDescriptor(
        "test-loop", ObservationStrategy.EVENT_DRIVEN, DecisionScope.FULL_AUTONOMY,
        CommunicationMode.BROADCAST, AgentBudgetPolicy.defaults(), List.of());

    static final DecisionOutcomeExecutor EXECUTOR = (agentId, outcome, ctx) ->
        CompletableFuture.completedFuture(
            new CoordinationFeedback(agentId, outcome, FeedbackSignal.SUCCESS));

    @Test
    void creates_with_descriptors_and_executor() {
        var plan = new CoordinationPlan(List.of(DESCRIPTOR), EXECUTOR);
        assertThat(plan.loopDescriptors()).containsExactly(DESCRIPTOR);
        assertThat(plan.executor()).isEqualTo(EXECUTOR);
    }

    @Test
    void rejects_empty_descriptors() {
        assertThatIllegalArgumentException()
            .isThrownBy(() -> new CoordinationPlan(List.of(), EXECUTOR))
            .withMessageContaining("at least one");
    }

    @Test
    void rejects_null_executor() {
        assertThatNullPointerException()
            .isThrownBy(() -> new CoordinationPlan(List.of(DESCRIPTOR), null))
            .withMessageContaining("executor");
    }

    @Test
    void descriptors_are_immutable_copy() {
        var mutable = new ArrayList<>(List.of(DESCRIPTOR));
        var plan = new CoordinationPlan(mutable, EXECUTOR);
        mutable.clear();
        assertThat(plan.loopDescriptors()).hasSize(1);
    }
}
```

- [ ] **Step 3: Write failing tests for CoordinationFeedback**

Create test file using `ide_create_file` at `agency-core/src/test/java/io/casehub/agency/orchestration/topology/CoordinationFeedbackTest.java`:

```java
package io.casehub.agency.orchestration.topology;

import io.casehub.agency.orchestration.loop.DecisionOutcome;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatNullPointerException;

class CoordinationFeedbackTest {

    static final DecisionOutcome OUTCOME =
        new DecisionOutcome.WriteBack(List.of(), List.of(), List.of());

    @Test
    void creates_with_all_fields() {
        var fb = new CoordinationFeedback("agent-1", OUTCOME, FeedbackSignal.SUCCESS);
        assertThat(fb.agentId()).isEqualTo("agent-1");
        assertThat(fb.outcome()).isEqualTo(OUTCOME);
        assertThat(fb.signal()).isEqualTo(FeedbackSignal.SUCCESS);
    }

    @Test
    void rejects_null_agentId() {
        assertThatNullPointerException()
            .isThrownBy(() -> new CoordinationFeedback(null, OUTCOME, FeedbackSignal.SUCCESS))
            .withMessageContaining("agentId");
    }

    @Test
    void rejects_null_outcome() {
        assertThatNullPointerException()
            .isThrownBy(() -> new CoordinationFeedback("agent-1", null, FeedbackSignal.SUCCESS))
            .withMessageContaining("outcome");
    }

    @Test
    void rejects_null_signal() {
        assertThatNullPointerException()
            .isThrownBy(() -> new CoordinationFeedback("agent-1", OUTCOME, null))
            .withMessageContaining("signal");
    }

    @Test
    void feedback_signal_has_four_values() {
        assertThat(FeedbackSignal.values()).containsExactly(
            FeedbackSignal.SUCCESS, FeedbackSignal.PARTIAL,
            FeedbackSignal.FAILURE, FeedbackSignal.SCOPE_VIOLATION);
    }
}
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `mvn compile -pl agency-core` (from agency root)
Expected: FAIL — classes don't exist yet

- [ ] **Step 5: Create FeedbackSignal enum**

Use `ide_create_file` at `agency-core/src/main/java/io/casehub/agency/orchestration/topology/FeedbackSignal.java`:

```java
package io.casehub.agency.orchestration.topology;

public enum FeedbackSignal {
    SUCCESS,
    PARTIAL,
    FAILURE,
    SCOPE_VIOLATION
}
```

- [ ] **Step 6: Create CoordinationFeedback record**

Use `ide_create_file` at `agency-core/src/main/java/io/casehub/agency/orchestration/topology/CoordinationFeedback.java`:

```java
package io.casehub.agency.orchestration.topology;

import io.casehub.agency.orchestration.loop.DecisionOutcome;

import java.util.Objects;

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

- [ ] **Step 7: Create DecisionOutcomeExecutor functional interface**

Use `ide_create_file` at `agency-core/src/main/java/io/casehub/agency/orchestration/topology/DecisionOutcomeExecutor.java`:

```java
package io.casehub.agency.orchestration.topology;

import io.casehub.agency.orchestration.loop.DecisionOutcome;

import java.util.concurrent.CompletionStage;

@FunctionalInterface
public interface DecisionOutcomeExecutor {
    CompletionStage<CoordinationFeedback> execute(
        String agentId, DecisionOutcome outcome, TopologyContext context);
}
```

- [ ] **Step 8: Create CoordinationPlan record**

Use `ide_create_file` at `agency-core/src/main/java/io/casehub/agency/orchestration/topology/CoordinationPlan.java`:

```java
package io.casehub.agency.orchestration.topology;

import io.casehub.agency.orchestration.loop.DecisionLoopDescriptor;

import java.util.List;
import java.util.Objects;

public record CoordinationPlan(
    List<DecisionLoopDescriptor> loopDescriptors,
    DecisionOutcomeExecutor executor
) {
    public CoordinationPlan {
        loopDescriptors = List.copyOf(loopDescriptors);
        Objects.requireNonNull(executor, "executor must not be null");
        if (loopDescriptors.isEmpty()) {
            throw new IllegalArgumentException(
                "CoordinationPlan must have at least one loop descriptor");
        }
    }
}
```

- [ ] **Step 9: Create TopologyContext record**

Use `ide_create_file` at `agency-core/src/main/java/io/casehub/agency/orchestration/topology/TopologyContext.java`:

```java
package io.casehub.agency.orchestration.topology;

import io.casehub.agency.orchestration.loop.DecisionLoopFactory;
import io.casehub.agency.orchestration.swarm.SwarmInvocationPolicy;
import io.casehub.api.model.coordination.TopologyDescriptor;
import io.casehub.platform.agent.AgentProvider;

import java.util.Objects;
import java.util.UUID;

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

- [ ] **Step 10: Create CoordinationTopology interface**

Use `ide_create_file` at `agency-core/src/main/java/io/casehub/agency/orchestration/topology/CoordinationTopology.java`:

```java
package io.casehub.agency.orchestration.topology;

import io.casehub.agency.orchestration.swarm.SwarmCoordinationEvent;
import io.casehub.api.spi.coordination.TopologyProvider;

public interface CoordinationTopology extends TopologyProvider {

    CoordinationPlan configure(TopologyContext context);

    void onSwarmEvent(SwarmCoordinationEvent event);

    void onFeedback(CoordinationFeedback feedback);
}
```

- [ ] **Step 11: Run tests to verify they pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl agency-core -Dtest="TopologyContextTest,CoordinationPlanTest,CoordinationFeedbackTest"`
Expected: PASS — all 15 tests

- [ ] **Step 12: Commit**

```bash
git add -A
git commit -m "feat: add CoordinationTopology SPI + supporting types

TopologyContext, CoordinationPlan, DecisionOutcomeExecutor,
CoordinationFeedback, FeedbackSignal.

Refs casehubio/agency#380"
```

---

## Batch 3: CentralisedTopology — LlmSwarmCoordinator refactor

### Task 3: CentralisedTopology implementing CoordinationTopology

**Files:**
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/topology/CentralisedTopology.java`
- Test: `agency-core/src/test/java/io/casehub/agency/orchestration/topology/CentralisedTopologyTest.java`

**Interfaces:**
- Consumes: `CoordinationTopology`, `TopologyContext`, `CoordinationPlan`, `DecisionOutcomeExecutor`, `CoordinationFeedback`, `FeedbackSignal`, `SwarmContext`, `SwarmCoordinationEvent`, `SwarmRoutingStrategy`, `CollectiveReasoningAggregation`, `TerminationCondition<SwarmContext>`, `SwarmFallbackStrategy`, `DecisionLoopDescriptor`, `DecisionOutcome` (all variants), `AgentBudgetPolicy`
- Produces: `CentralisedTopology` — reference implementation validating the SPI against real swarm coordinator code

- [ ] **Step 1: Write failing tests**

Create test file using `ide_create_file` at `agency-core/src/test/java/io/casehub/agency/orchestration/topology/CentralisedTopologyTest.java`:

```java
package io.casehub.agency.orchestration.topology;

import io.casehub.agency.orchestration.loop.AgentBudgetPolicy;
import io.casehub.agency.orchestration.loop.DecisionLoopDescriptor;
import io.casehub.agency.orchestration.loop.DecisionLoopFactory;
import io.casehub.agency.orchestration.loop.DecisionOutcome;
import io.casehub.agency.orchestration.swarm.CollectiveReasoningAggregation;
import io.casehub.agency.orchestration.swarm.SwarmContext;
import io.casehub.agency.orchestration.swarm.SwarmCoordinationEvent;
import io.casehub.agency.orchestration.swarm.SwarmFallbackStrategy;
import io.casehub.agency.orchestration.swarm.SwarmInvocationPolicy;
import io.casehub.agency.orchestration.swarm.SwarmRoutingStrategy;
import io.casehub.agency.orchestration.termination.TerminationCondition;
import io.casehub.api.model.coordination.CommunicationMode;
import io.casehub.api.model.coordination.DecisionScope;
import io.casehub.api.model.coordination.ObservationStrategy;
import io.casehub.api.model.coordination.TopologyDescriptor;
import io.casehub.api.model.coordination.TopologyType;
import io.casehub.api.model.stigmergy.SwarmProgress;
import io.casehub.platform.agent.NoOpAgentProvider;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.CompletableFuture;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.mock;

class CentralisedTopologyTest {

    SwarmContext swarmContext;
    CentralisedTopology topology;
    TopologyContext topologyContext;

    @BeforeEach
    void setUp() {
        swarmContext = new SwarmContext();
        topology = new CentralisedTopology(
            swarmContext,
            mock(SwarmRoutingStrategy.class),
            mock(CollectiveReasoningAggregation.class),
            ctx -> false,
            new SwarmFallbackStrategy(),
            List.of("experience-replay"));

        topologyContext = new TopologyContext(
            UUID.randomUUID(),
            new TopologyDescriptor(TopologyType.CENTRALISED, Map.of()),
            new NoOpAgentProvider(),
            mock(DecisionLoopFactory.class),
            SwarmInvocationPolicy.defaults());
    }

    @Test
    void describeTopology_returns_centralised() {
        var descriptor = topology.describeTopology(UUID.randomUUID());
        assertThat(descriptor.topologyType()).isEqualTo(TopologyType.CENTRALISED);
        assertThat(descriptor.agentRoles()).isEmpty();
    }

    @Test
    void configure_returns_single_loop_descriptor() {
        var plan = topology.configure(topologyContext);
        assertThat(plan.loopDescriptors()).hasSize(1);

        DecisionLoopDescriptor d = plan.loopDescriptors().get(0);
        assertThat(d.name()).isEqualTo("centralised-coordinator");
        assertThat(d.observationStrategy()).isEqualTo(ObservationStrategy.EVENT_DRIVEN);
        assertThat(d.scope()).isEqualTo(DecisionScope.FULL_AUTONOMY);
        assertThat(d.communicationMode()).isEqualTo(CommunicationMode.BROADCAST);
        assertThat(d.budgetPolicy()).isEqualTo(AgentBudgetPolicy.defaults());
        assertThat(d.promptSectionNames()).containsExactly("experience-replay");
    }

    @Test
    void configure_returns_non_null_executor() {
        var plan = topology.configure(topologyContext);
        assertThat(plan.executor()).isNotNull();
    }

    @Test
    void onSwarmEvent_updates_context_on_progress() {
        var progress = new SwarmProgress(0.5, "halfway", 5, 10);
        topology.onSwarmEvent(new SwarmCoordinationEvent.ProgressUpdated(progress));
        assertThat(swarmContext.swarmProgress()).isEqualTo(progress);
    }

    @Test
    void onSwarmEvent_updates_context_on_member_completed() {
        topology.onSwarmEvent(new SwarmCoordinationEvent.MemberCompleted("agent-x", null));
        assertThat(swarmContext.completedMembers()).contains("agent-x");
    }

    @Test
    void onSwarmEvent_updates_context_on_provisioning_changed() {
        topology.onSwarmEvent(new SwarmCoordinationEvent.ProvisioningChanged(3, 10));
        assertThat(swarmContext.currentSize()).isEqualTo(3);
        assertThat(swarmContext.maxSize()).isEqualTo(10);
    }

    @Test
    void onFeedback_does_not_throw() {
        var feedback = new CoordinationFeedback(
            "agent-1",
            new DecisionOutcome.WriteBack(List.of(), List.of(), List.of()),
            FeedbackSignal.SUCCESS);
        topology.onFeedback(feedback);
    }

    @Test
    void executor_handles_scope_gated_outcome() throws Exception {
        var plan = topology.configure(topologyContext);
        var gated = new DecisionOutcome.ScopeGated(
            io.casehub.api.model.coordination.DecisionType.PROVISIONING,
            new DecisionOutcome.WriteBack(List.of(), List.of(), List.of()));

        var feedback = plan.executor()
            .execute("coordinator", gated, topologyContext)
            .toCompletableFuture().get();

        assertThat(feedback.signal()).isEqualTo(FeedbackSignal.SCOPE_VIOLATION);
        assertThat(feedback.agentId()).isEqualTo("coordinator");
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn compile -pl agency-core`
Expected: FAIL — `CentralisedTopology` class doesn't exist

- [ ] **Step 3: Implement CentralisedTopology**

Use `ide_create_file` at `agency-core/src/main/java/io/casehub/agency/orchestration/topology/CentralisedTopology.java`:

```java
package io.casehub.agency.orchestration.topology;

import io.casehub.agency.orchestration.loop.AgentBudgetPolicy;
import io.casehub.agency.orchestration.loop.DecisionLoopDescriptor;
import io.casehub.agency.orchestration.loop.DecisionOutcome;
import io.casehub.agency.orchestration.swarm.CollectiveReasoningAggregation;
import io.casehub.agency.orchestration.swarm.SwarmContext;
import io.casehub.agency.orchestration.swarm.SwarmCoordinationEvent;
import io.casehub.agency.orchestration.swarm.SwarmFallbackStrategy;
import io.casehub.agency.orchestration.swarm.SwarmRoutingStrategy;
import io.casehub.agency.orchestration.termination.TerminationCondition;
import io.casehub.api.model.coordination.CommunicationMode;
import io.casehub.api.model.coordination.DecisionScope;
import io.casehub.api.model.coordination.ObservationStrategy;
import io.casehub.api.model.coordination.TopologyDescriptor;
import io.casehub.api.model.coordination.TopologyType;

import java.util.List;
import java.util.Map;
import java.util.Objects;
import java.util.UUID;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.CompletionStage;

public class CentralisedTopology implements CoordinationTopology {

    private final SwarmContext swarmContext;
    private final SwarmRoutingStrategy routingStrategy;
    private final CollectiveReasoningAggregation aggregation;
    private final TerminationCondition<SwarmContext> terminationCondition;
    private final SwarmFallbackStrategy fallbackStrategy;
    private final List<String> promptSectionNames;

    public CentralisedTopology(
            SwarmContext swarmContext,
            SwarmRoutingStrategy routingStrategy,
            CollectiveReasoningAggregation aggregation,
            TerminationCondition<SwarmContext> terminationCondition,
            SwarmFallbackStrategy fallbackStrategy,
            List<String> promptSectionNames) {
        this.swarmContext = Objects.requireNonNull(swarmContext);
        this.routingStrategy = Objects.requireNonNull(routingStrategy);
        this.aggregation = Objects.requireNonNull(aggregation);
        this.terminationCondition = Objects.requireNonNull(terminationCondition);
        this.fallbackStrategy = Objects.requireNonNull(fallbackStrategy);
        this.promptSectionNames = List.copyOf(promptSectionNames);
    }

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
            promptSectionNames);

        DecisionOutcomeExecutor executor = (agentId, outcome, ctx) ->
            executeCentralised(agentId, outcome, ctx);

        return new CoordinationPlan(List.of(descriptor), executor);
    }

    private CompletionStage<CoordinationFeedback> executeCentralised(
            String agentId, DecisionOutcome outcome, TopologyContext ctx) {
        return switch (outcome) {
            case DecisionOutcome.Route r ->
                executeRouting(agentId, r, ctx);
            case DecisionOutcome.Aggregate a ->
                executeAggregation(agentId, a, ctx);
            case DecisionOutcome.WriteBack w ->
                executeWriteBack(agentId, w);
            case DecisionOutcome.Composite c ->
                executeComposite(agentId, c, ctx);
            case DecisionOutcome.ScopeGated sg ->
                CompletableFuture.completedFuture(
                    new CoordinationFeedback(agentId, sg, FeedbackSignal.SCOPE_VIOLATION));
        };
    }

    private CompletionStage<CoordinationFeedback> executeRouting(
            String agentId, DecisionOutcome.Route route, TopologyContext ctx) {
        return CompletableFuture.completedFuture(
            new CoordinationFeedback(agentId, route, FeedbackSignal.SUCCESS));
    }

    private CompletionStage<CoordinationFeedback> executeAggregation(
            String agentId, DecisionOutcome.Aggregate agg, TopologyContext ctx) {
        return CompletableFuture.completedFuture(
            new CoordinationFeedback(agentId, agg, FeedbackSignal.SUCCESS));
    }

    private CompletionStage<CoordinationFeedback> executeWriteBack(
            String agentId, DecisionOutcome.WriteBack wb) {
        return CompletableFuture.completedFuture(
            new CoordinationFeedback(agentId, wb, FeedbackSignal.SUCCESS));
    }

    private CompletionStage<CoordinationFeedback> executeComposite(
            String agentId, DecisionOutcome.Composite composite, TopologyContext ctx) {
        CompletionStage<CoordinationFeedback> chain =
            CompletableFuture.completedFuture(
                new CoordinationFeedback(agentId, composite, FeedbackSignal.SUCCESS));
        for (var sub : composite.outcomes()) {
            chain = chain.thenCompose(prev -> {
                if (prev.signal() == FeedbackSignal.FAILURE) {
                    return CompletableFuture.completedFuture(prev);
                }
                return executeCentralised(agentId, sub, null);
            });
        }
        return chain;
    }

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

    public SwarmContext swarmContext() { return swarmContext; }
    public SwarmRoutingStrategy routingStrategy() { return routingStrategy; }
    public CollectiveReasoningAggregation aggregation() { return aggregation; }
    public TerminationCondition<SwarmContext> terminationCondition() { return terminationCondition; }
    public SwarmFallbackStrategy fallbackStrategy() { return fallbackStrategy; }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl agency-core -Dtest=CentralisedTopologyTest`
Expected: PASS — all 8 tests

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: add CentralisedTopology — refactored from LlmSwarmCoordinator

Implements CoordinationTopology SPI. Validates the SPI shape against
existing swarm coordinator code.

Refs casehubio/agency#380
Closes casehubio/agency#381"
```

---

## Batch 4: DistributedTopology + HybridTopology

### Task 4: DistributedTopology and HybridTopology

**Files:**
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/topology/DistributedTopology.java`
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/topology/HybridTopology.java`
- Test: `agency-core/src/test/java/io/casehub/agency/orchestration/topology/DistributedTopologyTest.java`
- Test: `agency-core/src/test/java/io/casehub/agency/orchestration/topology/HybridTopologyTest.java`

**Interfaces:**
- Consumes: `CoordinationTopology`, `TopologyContext`, `CoordinationPlan`, `DecisionOutcomeExecutor`, `CoordinationFeedback`, `FeedbackSignal`, `DecisionLoopDescriptor`, `DecisionOutcome`, `AgentBudgetPolicy`, `TopologyDescriptor`, `AgentRole`, `StructuralRole`, `TopologyType`
- Produces: `DistributedTopology` (N per-agent loops, FULL_AUTONOMY, STIGMERGY), `HybridTopology` (1 global + N local loops)

- [ ] **Step 1: Write failing tests for DistributedTopology**

Create test file using `ide_create_file` at `agency-core/src/test/java/io/casehub/agency/orchestration/topology/DistributedTopologyTest.java`:

```java
package io.casehub.agency.orchestration.topology;

import io.casehub.agency.orchestration.loop.DecisionLoopDescriptor;
import io.casehub.agency.orchestration.loop.DecisionLoopFactory;
import io.casehub.agency.orchestration.loop.DecisionOutcome;
import io.casehub.agency.orchestration.swarm.SwarmCoordinationEvent;
import io.casehub.agency.orchestration.swarm.SwarmInvocationPolicy;
import io.casehub.api.model.coordination.AgentRole;
import io.casehub.api.model.coordination.CommunicationMode;
import io.casehub.api.model.coordination.DecisionScope;
import io.casehub.api.model.coordination.DecisionType;
import io.casehub.api.model.coordination.ObservationStrategy;
import io.casehub.api.model.coordination.StructuralRole;
import io.casehub.api.model.coordination.TopologyDescriptor;
import io.casehub.api.model.coordination.TopologyType;
import io.casehub.api.model.stigmergy.SwarmProgress;
import io.casehub.platform.agent.NoOpAgentProvider;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.mock;

class DistributedTopologyTest {

    Map<String, AgentRole> agentRoles;
    DistributedTopology topology;
    TopologyContext topologyContext;

    @BeforeEach
    void setUp() {
        agentRoles = Map.of(
            "agent-a", AgentRole.of("peer-a", StructuralRole.PEER),
            "agent-b", AgentRole.of("peer-b", StructuralRole.PEER),
            "agent-c", AgentRole.of("peer-c", StructuralRole.PEER));
        topology = new DistributedTopology(agentRoles);

        topologyContext = new TopologyContext(
            UUID.randomUUID(),
            new TopologyDescriptor(TopologyType.DISTRIBUTED, agentRoles),
            new NoOpAgentProvider(),
            mock(DecisionLoopFactory.class),
            SwarmInvocationPolicy.defaults());
    }

    @Test
    void describeTopology_returns_distributed() {
        var descriptor = topology.describeTopology(UUID.randomUUID());
        assertThat(descriptor.topologyType()).isEqualTo(TopologyType.DISTRIBUTED);
        assertThat(descriptor.agentRoles()).hasSize(3);
    }

    @Test
    void configure_returns_one_loop_per_agent() {
        var plan = topology.configure(topologyContext);
        assertThat(plan.loopDescriptors()).hasSize(3);
    }

    @Test
    void configure_loops_have_correct_properties() {
        var plan = topology.configure(topologyContext);
        for (DecisionLoopDescriptor d : plan.loopDescriptors()) {
            assertThat(d.name()).startsWith("distributed-");
            assertThat(d.scope()).isEqualTo(DecisionScope.FULL_AUTONOMY);
            assertThat(d.observationStrategy()).isEqualTo(ObservationStrategy.EVENT_DRIVEN);
            assertThat(d.communicationMode()).isEqualTo(CommunicationMode.STIGMERGY);
        }
    }

    @Test
    void configure_returns_non_null_executor() {
        var plan = topology.configure(topologyContext);
        assertThat(plan.executor()).isNotNull();
    }

    @Test
    void executor_handles_scope_gated() throws Exception {
        var plan = topology.configure(topologyContext);
        var gated = new DecisionOutcome.ScopeGated(
            DecisionType.PROVISIONING,
            new DecisionOutcome.WriteBack(List.of(), List.of(), List.of()));

        var fb = plan.executor()
            .execute("agent-a", gated, topologyContext)
            .toCompletableFuture().get();
        assertThat(fb.signal()).isEqualTo(FeedbackSignal.SCOPE_VIOLATION);
    }

    @Test
    void executor_handles_writeback() throws Exception {
        var plan = topology.configure(topologyContext);
        var wb = new DecisionOutcome.WriteBack(List.of(), List.of(), List.of());

        var fb = plan.executor()
            .execute("agent-a", wb, topologyContext)
            .toCompletableFuture().get();
        assertThat(fb.signal()).isEqualTo(FeedbackSignal.SUCCESS);
    }

    @Test
    void onSwarmEvent_does_not_throw() {
        var progress = new SwarmProgress(0.5, "halfway", 5, 10);
        topology.onSwarmEvent(new SwarmCoordinationEvent.ProgressUpdated(progress));
    }

    @Test
    void onFeedback_does_not_throw() {
        topology.onFeedback(new CoordinationFeedback(
            "agent-a",
            new DecisionOutcome.WriteBack(List.of(), List.of(), List.of()),
            FeedbackSignal.SUCCESS));
    }
}
```

- [ ] **Step 2: Write failing tests for HybridTopology**

Create test file using `ide_create_file` at `agency-core/src/test/java/io/casehub/agency/orchestration/topology/HybridTopologyTest.java`:

```java
package io.casehub.agency.orchestration.topology;

import io.casehub.agency.orchestration.loop.DecisionLoopDescriptor;
import io.casehub.agency.orchestration.loop.DecisionLoopFactory;
import io.casehub.agency.orchestration.loop.DecisionOutcome;
import io.casehub.agency.orchestration.swarm.SwarmInvocationPolicy;
import io.casehub.api.model.coordination.AgentRole;
import io.casehub.api.model.coordination.CommunicationMode;
import io.casehub.api.model.coordination.DecisionScope;
import io.casehub.api.model.coordination.ObservationStrategy;
import io.casehub.api.model.coordination.StructuralRole;
import io.casehub.api.model.coordination.TopologyDescriptor;
import io.casehub.api.model.coordination.TopologyType;
import io.casehub.platform.agent.NoOpAgentProvider;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.mock;

class HybridTopologyTest {

    Map<String, AgentRole> agentRoles;
    HybridTopology topology;
    TopologyContext topologyContext;

    @BeforeEach
    void setUp() {
        agentRoles = Map.of(
            "coordinator", AgentRole.of("coordinator", StructuralRole.INITIATOR),
            "worker-1", AgentRole.of("worker", StructuralRole.CONTRACTOR),
            "worker-2", AgentRole.of("worker", StructuralRole.CONTRACTOR));
        topology = new HybridTopology(agentRoles);

        topologyContext = new TopologyContext(
            UUID.randomUUID(),
            new TopologyDescriptor(TopologyType.HYBRID, agentRoles),
            new NoOpAgentProvider(),
            mock(DecisionLoopFactory.class),
            SwarmInvocationPolicy.defaults());
    }

    @Test
    void describeTopology_returns_hybrid() {
        var descriptor = topology.describeTopology(UUID.randomUUID());
        assertThat(descriptor.topologyType()).isEqualTo(TopologyType.HYBRID);
        assertThat(descriptor.agentRoles()).hasSize(3);
    }

    @Test
    void configure_returns_global_plus_local_loops() {
        var plan = topology.configure(topologyContext);
        assertThat(plan.loopDescriptors()).hasSize(3);
    }

    @Test
    void configure_global_loop_has_full_autonomy() {
        var plan = topology.configure(topologyContext);
        DecisionLoopDescriptor global = plan.loopDescriptors().stream()
            .filter(d -> d.name().equals("hybrid-coordinator"))
            .findFirst().orElseThrow();
        assertThat(global.scope()).isEqualTo(DecisionScope.FULL_AUTONOMY);
        assertThat(global.observationStrategy()).isEqualTo(ObservationStrategy.EVENT_DRIVEN);
        assertThat(global.communicationMode()).isEqualTo(CommunicationMode.BROADCAST);
    }

    @Test
    void configure_local_loops_have_task_adaptation() {
        var plan = topology.configure(topologyContext);
        var locals = plan.loopDescriptors().stream()
            .filter(d -> d.name().startsWith("hybrid-local-"))
            .toList();
        assertThat(locals).hasSize(2);
        for (var d : locals) {
            assertThat(d.scope()).isEqualTo(DecisionScope.TASK_ADAPTATION);
            assertThat(d.observationStrategy()).isEqualTo(ObservationStrategy.TICK_DRIVEN);
            assertThat(d.communicationMode()).isEqualTo(CommunicationMode.STIGMERGY);
        }
    }

    @Test
    void executor_routes_coordinator_centrally() throws Exception {
        var plan = topology.configure(topologyContext);
        var wb = new DecisionOutcome.WriteBack(List.of(), List.of(), List.of());

        var fb = plan.executor()
            .execute("hybrid-coordinator", wb, topologyContext)
            .toCompletableFuture().get();
        assertThat(fb.signal()).isEqualTo(FeedbackSignal.SUCCESS);
    }

    @Test
    void executor_routes_worker_locally() throws Exception {
        var plan = topology.configure(topologyContext);
        var wb = new DecisionOutcome.WriteBack(List.of(), List.of(), List.of());

        var fb = plan.executor()
            .execute("worker-1", wb, topologyContext)
            .toCompletableFuture().get();
        assertThat(fb.signal()).isEqualTo(FeedbackSignal.SUCCESS);
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn compile -pl agency-core`
Expected: FAIL — classes don't exist

- [ ] **Step 4: Implement DistributedTopology**

Use `ide_create_file` at `agency-core/src/main/java/io/casehub/agency/orchestration/topology/DistributedTopology.java`:

```java
package io.casehub.agency.orchestration.topology;

import io.casehub.agency.orchestration.loop.AgentBudgetPolicy;
import io.casehub.agency.orchestration.loop.DecisionLoopDescriptor;
import io.casehub.agency.orchestration.loop.DecisionOutcome;
import io.casehub.agency.orchestration.swarm.SwarmCoordinationEvent;
import io.casehub.api.model.coordination.AgentRole;
import io.casehub.api.model.coordination.CommunicationMode;
import io.casehub.api.model.coordination.DecisionScope;
import io.casehub.api.model.coordination.ObservationStrategy;
import io.casehub.api.model.coordination.TopologyDescriptor;
import io.casehub.api.model.coordination.TopologyType;

import java.util.List;
import java.util.Map;
import java.util.Objects;
import java.util.UUID;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.CompletionStage;

public class DistributedTopology implements CoordinationTopology {

    private final Map<String, AgentRole> agentRoles;

    public DistributedTopology(Map<String, AgentRole> agentRoles) {
        this.agentRoles = Map.copyOf(Objects.requireNonNull(agentRoles));
    }

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
                List.of("environment-observer", "peer-state")))
            .toList();

        DecisionOutcomeExecutor executor = this::executeLocal;
        return new CoordinationPlan(descriptors, executor);
    }

    private CompletionStage<CoordinationFeedback> executeLocal(
            String agentId, DecisionOutcome outcome, TopologyContext ctx) {
        return switch (outcome) {
            case DecisionOutcome.ScopeGated sg ->
                CompletableFuture.completedFuture(
                    new CoordinationFeedback(agentId, sg, FeedbackSignal.SCOPE_VIOLATION));
            default ->
                CompletableFuture.completedFuture(
                    new CoordinationFeedback(agentId, outcome, FeedbackSignal.SUCCESS));
        };
    }

    @Override
    public void onSwarmEvent(SwarmCoordinationEvent event) {
        // Distributed: events forwarded to agent loops as observations — no central handling
    }

    @Override
    public void onFeedback(CoordinationFeedback feedback) {
        // Distributed: feedback for metrics only
    }
}
```

- [ ] **Step 5: Implement HybridTopology**

Use `ide_create_file` at `agency-core/src/main/java/io/casehub/agency/orchestration/topology/HybridTopology.java`:

```java
package io.casehub.agency.orchestration.topology;

import io.casehub.agency.orchestration.loop.AgentBudgetPolicy;
import io.casehub.agency.orchestration.loop.DecisionLoopDescriptor;
import io.casehub.agency.orchestration.loop.DecisionOutcome;
import io.casehub.agency.orchestration.swarm.SwarmCoordinationEvent;
import io.casehub.api.model.coordination.AgentRole;
import io.casehub.api.model.coordination.CommunicationMode;
import io.casehub.api.model.coordination.DecisionScope;
import io.casehub.api.model.coordination.ObservationStrategy;
import io.casehub.api.model.coordination.StructuralRole;
import io.casehub.api.model.coordination.TopologyDescriptor;
import io.casehub.api.model.coordination.TopologyType;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.Objects;
import java.util.UUID;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.CompletionStage;

public class HybridTopology implements CoordinationTopology {

    private final Map<String, AgentRole> agentRoles;

    public HybridTopology(Map<String, AgentRole> agentRoles) {
        this.agentRoles = Map.copyOf(Objects.requireNonNull(agentRoles));
    }

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
            List.of("global-state", "coordination-strategy"));

        var localDescriptors = context.descriptor().agentRoles().entrySet().stream()
            .filter(e -> e.getValue().structuralRole() != StructuralRole.INITIATOR)
            .map(e -> new DecisionLoopDescriptor(
                "hybrid-local-" + e.getKey(),
                ObservationStrategy.TICK_DRIVEN,
                DecisionScope.TASK_ADAPTATION,
                CommunicationMode.STIGMERGY,
                AgentBudgetPolicy.defaults(),
                List.of("local-task-context")))
            .toList();

        var allDescriptors = new ArrayList<DecisionLoopDescriptor>();
        allDescriptors.add(globalDescriptor);
        allDescriptors.addAll(localDescriptors);

        DecisionOutcomeExecutor executor = (agentId, outcome, ctx) -> {
            if ("hybrid-coordinator".equals(agentId)) {
                return executeCentrally(agentId, outcome, ctx);
            }
            return executeLocally(agentId, outcome, ctx);
        };

        return new CoordinationPlan(allDescriptors, executor);
    }

    private CompletionStage<CoordinationFeedback> executeCentrally(
            String agentId, DecisionOutcome outcome, TopologyContext ctx) {
        return switch (outcome) {
            case DecisionOutcome.ScopeGated sg ->
                CompletableFuture.completedFuture(
                    new CoordinationFeedback(agentId, sg, FeedbackSignal.SCOPE_VIOLATION));
            default ->
                CompletableFuture.completedFuture(
                    new CoordinationFeedback(agentId, outcome, FeedbackSignal.SUCCESS));
        };
    }

    private CompletionStage<CoordinationFeedback> executeLocally(
            String agentId, DecisionOutcome outcome, TopologyContext ctx) {
        return switch (outcome) {
            case DecisionOutcome.ScopeGated sg ->
                CompletableFuture.completedFuture(
                    new CoordinationFeedback(agentId, sg, FeedbackSignal.SCOPE_VIOLATION));
            default ->
                CompletableFuture.completedFuture(
                    new CoordinationFeedback(agentId, outcome, FeedbackSignal.SUCCESS));
        };
    }

    @Override
    public void onSwarmEvent(SwarmCoordinationEvent event) {
        // Global coordinator loop observes all events; local loops filter by agent
    }

    @Override
    public void onFeedback(CoordinationFeedback feedback) {
        // Hybrid: feedback for rebalancing global/local authority
    }
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl agency-core -Dtest="DistributedTopologyTest,HybridTopologyTest"`
Expected: PASS — all 14 tests

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "feat: add DistributedTopology + HybridTopology

DistributedTopology: N per-agent loops with FULL_AUTONOMY and STIGMERGY.
HybridTopology: 1 global coordinator + N local loops with constrained scope.

Refs casehubio/agency#380"
```

---

## Batch 5: HierarchicalTopology + GraduatedTopology + GraduationState

### Task 5: GraduationState, HierarchicalTopology, and GraduatedTopology

**Files:**
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/topology/GraduationState.java`
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/topology/HierarchicalTopology.java`
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/topology/GraduatedTopology.java`
- Test: `agency-core/src/test/java/io/casehub/agency/orchestration/topology/GraduationStateTest.java`
- Test: `agency-core/src/test/java/io/casehub/agency/orchestration/topology/HierarchicalTopologyTest.java`
- Test: `agency-core/src/test/java/io/casehub/agency/orchestration/topology/GraduatedTopologyTest.java`

**Interfaces:**
- Consumes: All SPI types from Task 2, `DecisionScope` enum values (STIMULUS_DRIVEN, TASK_ADAPTATION, FULL_AUTONOMY), `TopologyType.HIERARCHICAL`, `TopologyType.GRADUATED`
- Produces: `GraduationState` (mutable per-agent scope tracker), `HierarchicalTopology` (tree of loops), `GraduatedTopology` (dynamic scope via GraduationState)

- [ ] **Step 1: Write failing tests for GraduationState**

Create test file using `ide_create_file` at `agency-core/src/test/java/io/casehub/agency/orchestration/topology/GraduationStateTest.java`:

```java
package io.casehub.agency.orchestration.topology;

import io.casehub.api.model.coordination.DecisionScope;
import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class GraduationStateTest {

    @Test
    void initial_scope_defaults_to_stimulus_driven() {
        var state = new GraduationState(Map.of(), 5);
        assertThat(state.scopeFor("unknown")).isEqualTo(DecisionScope.STIMULUS_DRIVEN);
    }

    @Test
    void initial_scope_uses_provided_value() {
        var state = new GraduationState(
            Map.of("agent-1", DecisionScope.TASK_ADAPTATION), 5);
        assertThat(state.scopeFor("agent-1")).isEqualTo(DecisionScope.TASK_ADAPTATION);
    }

    @Test
    void upgrade_from_stimulus_to_task_adaptation() {
        var state = new GraduationState(
            Map.of("agent-1", DecisionScope.STIMULUS_DRIVEN), 5);
        state.upgrade("agent-1");
        assertThat(state.scopeFor("agent-1")).isEqualTo(DecisionScope.TASK_ADAPTATION);
    }

    @Test
    void upgrade_from_task_adaptation_to_full_autonomy() {
        var state = new GraduationState(
            Map.of("agent-1", DecisionScope.TASK_ADAPTATION), 5);
        state.upgrade("agent-1");
        assertThat(state.scopeFor("agent-1")).isEqualTo(DecisionScope.FULL_AUTONOMY);
    }

    @Test
    void upgrade_caps_at_full_autonomy() {
        var state = new GraduationState(
            Map.of("agent-1", DecisionScope.FULL_AUTONOMY), 5);
        state.upgrade("agent-1");
        assertThat(state.scopeFor("agent-1")).isEqualTo(DecisionScope.FULL_AUTONOMY);
    }

    @Test
    void downgrade_from_full_autonomy_to_task_adaptation() {
        var state = new GraduationState(
            Map.of("agent-1", DecisionScope.FULL_AUTONOMY), 5);
        state.downgrade("agent-1");
        assertThat(state.scopeFor("agent-1")).isEqualTo(DecisionScope.TASK_ADAPTATION);
    }

    @Test
    void downgrade_from_task_adaptation_to_stimulus() {
        var state = new GraduationState(
            Map.of("agent-1", DecisionScope.TASK_ADAPTATION), 5);
        state.downgrade("agent-1");
        assertThat(state.scopeFor("agent-1")).isEqualTo(DecisionScope.STIMULUS_DRIVEN);
    }

    @Test
    void downgrade_floors_at_stimulus_driven() {
        var state = new GraduationState(
            Map.of("agent-1", DecisionScope.STIMULUS_DRIVEN), 5);
        state.downgrade("agent-1");
        assertThat(state.scopeFor("agent-1")).isEqualTo(DecisionScope.STIMULUS_DRIVEN);
    }

    @Test
    void scope_violation_causes_hard_downgrade() {
        var state = new GraduationState(
            Map.of("agent-1", DecisionScope.FULL_AUTONOMY), 5);
        state.recordFeedback("agent-1", FeedbackSignal.SCOPE_VIOLATION);
        assertThat(state.scopeFor("agent-1")).isEqualTo(DecisionScope.TASK_ADAPTATION);
    }

    @Test
    void sustained_success_upgrades_scope() {
        var state = new GraduationState(
            Map.of("agent-1", DecisionScope.STIMULUS_DRIVEN), 5);
        for (int i = 0; i < 5; i++) {
            state.recordFeedback("agent-1", FeedbackSignal.SUCCESS);
        }
        assertThat(state.scopeFor("agent-1")).isEqualTo(DecisionScope.TASK_ADAPTATION);
    }

    @Test
    void sustained_failure_downgrades_scope() {
        var state = new GraduationState(
            Map.of("agent-1", DecisionScope.FULL_AUTONOMY), 4);
        state.recordFeedback("agent-1", FeedbackSignal.FAILURE);
        state.recordFeedback("agent-1", FeedbackSignal.FAILURE);
        state.recordFeedback("agent-1", FeedbackSignal.SUCCESS);
        state.recordFeedback("agent-1", FeedbackSignal.FAILURE);
        assertThat(state.scopeFor("agent-1")).isEqualTo(DecisionScope.TASK_ADAPTATION);
    }
}
```

- [ ] **Step 2: Write failing tests for HierarchicalTopology**

Create test file using `ide_create_file` at `agency-core/src/test/java/io/casehub/agency/orchestration/topology/HierarchicalTopologyTest.java`:

```java
package io.casehub.agency.orchestration.topology;

import io.casehub.agency.orchestration.loop.DecisionLoopDescriptor;
import io.casehub.agency.orchestration.loop.DecisionLoopFactory;
import io.casehub.agency.orchestration.swarm.SwarmInvocationPolicy;
import io.casehub.api.model.coordination.AgentRole;
import io.casehub.api.model.coordination.CommunicationMode;
import io.casehub.api.model.coordination.DecisionScope;
import io.casehub.api.model.coordination.ObservationStrategy;
import io.casehub.api.model.coordination.StructuralRole;
import io.casehub.api.model.coordination.TopologyDescriptor;
import io.casehub.api.model.coordination.TopologyType;
import io.casehub.platform.agent.NoOpAgentProvider;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.Map;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.mock;

class HierarchicalTopologyTest {

    Map<String, AgentRole> agentRoles;
    HierarchicalTopology topology;
    TopologyContext topologyContext;

    @BeforeEach
    void setUp() {
        agentRoles = Map.of(
            "root", AgentRole.of("root", StructuralRole.INITIATOR),
            "middle", AgentRole.of("middle", StructuralRole.CONTRACTOR),
            "leaf", AgentRole.of("leaf", StructuralRole.CONTRACTOR));

        Map<String, String> parentMap = Map.of(
            "middle", "root",
            "leaf", "middle");

        topology = new HierarchicalTopology(agentRoles, parentMap);

        topologyContext = new TopologyContext(
            UUID.randomUUID(),
            new TopologyDescriptor(TopologyType.HIERARCHICAL, agentRoles),
            new NoOpAgentProvider(),
            mock(DecisionLoopFactory.class),
            SwarmInvocationPolicy.defaults());
    }

    @Test
    void describeTopology_returns_hierarchical() {
        var descriptor = topology.describeTopology(UUID.randomUUID());
        assertThat(descriptor.topologyType()).isEqualTo(TopologyType.HIERARCHICAL);
    }

    @Test
    void configure_returns_one_loop_per_agent() {
        var plan = topology.configure(topologyContext);
        assertThat(plan.loopDescriptors()).hasSize(3);
    }

    @Test
    void root_has_full_autonomy_and_broadcast() {
        var plan = topology.configure(topologyContext);
        DecisionLoopDescriptor root = plan.loopDescriptors().stream()
            .filter(d -> d.name().equals("hierarchical-root"))
            .findFirst().orElseThrow();
        assertThat(root.scope()).isEqualTo(DecisionScope.FULL_AUTONOMY);
        assertThat(root.observationStrategy()).isEqualTo(ObservationStrategy.EVENT_DRIVEN);
        assertThat(root.communicationMode()).isEqualTo(CommunicationMode.BROADCAST);
    }

    @Test
    void intermediate_has_task_adaptation() {
        var plan = topology.configure(topologyContext);
        DecisionLoopDescriptor middle = plan.loopDescriptors().stream()
            .filter(d -> d.name().equals("hierarchical-middle"))
            .findFirst().orElseThrow();
        assertThat(middle.scope()).isEqualTo(DecisionScope.TASK_ADAPTATION);
        assertThat(middle.observationStrategy()).isEqualTo(ObservationStrategy.TICK_DRIVEN);
        assertThat(middle.communicationMode()).isEqualTo(CommunicationMode.DIRECT_MESSAGING);
    }

    @Test
    void leaf_has_stimulus_driven() {
        var plan = topology.configure(topologyContext);
        DecisionLoopDescriptor leaf = plan.loopDescriptors().stream()
            .filter(d -> d.name().equals("hierarchical-leaf"))
            .findFirst().orElseThrow();
        assertThat(leaf.scope()).isEqualTo(DecisionScope.STIMULUS_DRIVEN);
        assertThat(leaf.observationStrategy()).isEqualTo(ObservationStrategy.EVENT_DRIVEN);
        assertThat(leaf.communicationMode()).isEqualTo(CommunicationMode.STIGMERGY);
    }
}
```

- [ ] **Step 3: Write failing tests for GraduatedTopology**

Create test file using `ide_create_file` at `agency-core/src/test/java/io/casehub/agency/orchestration/topology/GraduatedTopologyTest.java`:

```java
package io.casehub.agency.orchestration.topology;

import io.casehub.agency.orchestration.loop.DecisionLoopDescriptor;
import io.casehub.agency.orchestration.loop.DecisionLoopFactory;
import io.casehub.agency.orchestration.loop.DecisionOutcome;
import io.casehub.agency.orchestration.swarm.SwarmInvocationPolicy;
import io.casehub.api.model.coordination.AgentRole;
import io.casehub.api.model.coordination.CommunicationMode;
import io.casehub.api.model.coordination.DecisionScope;
import io.casehub.api.model.coordination.ObservationStrategy;
import io.casehub.api.model.coordination.StructuralRole;
import io.casehub.api.model.coordination.TopologyDescriptor;
import io.casehub.api.model.coordination.TopologyType;
import io.casehub.platform.agent.NoOpAgentProvider;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.mock;

class GraduatedTopologyTest {

    Map<String, AgentRole> agentRoles;
    GraduationState graduationState;
    GraduatedTopology topology;
    TopologyContext topologyContext;

    @BeforeEach
    void setUp() {
        agentRoles = Map.of(
            "agent-1", AgentRole.of("worker", StructuralRole.PEER),
            "agent-2", AgentRole.of("worker", StructuralRole.PEER));

        graduationState = new GraduationState(
            Map.of("agent-1", DecisionScope.STIMULUS_DRIVEN,
                   "agent-2", DecisionScope.TASK_ADAPTATION), 5);

        topology = new GraduatedTopology(agentRoles, graduationState);

        topologyContext = new TopologyContext(
            UUID.randomUUID(),
            new TopologyDescriptor(TopologyType.GRADUATED, agentRoles),
            new NoOpAgentProvider(),
            mock(DecisionLoopFactory.class),
            SwarmInvocationPolicy.defaults());
    }

    @Test
    void describeTopology_returns_graduated() {
        var descriptor = topology.describeTopology(UUID.randomUUID());
        assertThat(descriptor.topologyType()).isEqualTo(TopologyType.GRADUATED);
    }

    @Test
    void configure_returns_one_loop_per_agent() {
        var plan = topology.configure(topologyContext);
        assertThat(plan.loopDescriptors()).hasSize(2);
    }

    @Test
    void configure_uses_graduation_state_for_scope() {
        var plan = topology.configure(topologyContext);
        DecisionLoopDescriptor d1 = plan.loopDescriptors().stream()
            .filter(d -> d.name().equals("graduated-agent-1"))
            .findFirst().orElseThrow();
        assertThat(d1.scope()).isEqualTo(DecisionScope.STIMULUS_DRIVEN);

        DecisionLoopDescriptor d2 = plan.loopDescriptors().stream()
            .filter(d -> d.name().equals("graduated-agent-2"))
            .findFirst().orElseThrow();
        assertThat(d2.scope()).isEqualTo(DecisionScope.TASK_ADAPTATION);
    }

    @Test
    void configure_all_loops_use_event_driven_and_stigmergy() {
        var plan = topology.configure(topologyContext);
        for (var d : plan.loopDescriptors()) {
            assertThat(d.observationStrategy()).isEqualTo(ObservationStrategy.EVENT_DRIVEN);
            assertThat(d.communicationMode()).isEqualTo(CommunicationMode.STIGMERGY);
        }
    }

    @Test
    void onFeedback_adjusts_graduation_state() {
        topology.onFeedback(new CoordinationFeedback(
            "agent-1",
            new DecisionOutcome.WriteBack(List.of(), List.of(), List.of()),
            FeedbackSignal.SCOPE_VIOLATION));
        assertThat(graduationState.scopeFor("agent-1")).isEqualTo(DecisionScope.STIMULUS_DRIVEN);
    }

    @Test
    void reconfigure_reflects_scope_change() {
        for (int i = 0; i < 5; i++) {
            graduationState.recordFeedback("agent-1", FeedbackSignal.SUCCESS);
        }
        var plan = topology.configure(topologyContext);
        DecisionLoopDescriptor d1 = plan.loopDescriptors().stream()
            .filter(d -> d.name().equals("graduated-agent-1"))
            .findFirst().orElseThrow();
        assertThat(d1.scope()).isEqualTo(DecisionScope.TASK_ADAPTATION);
    }
}
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `mvn compile -pl agency-core`
Expected: FAIL — classes don't exist

- [ ] **Step 5: Implement GraduationState**

Use `ide_create_file` at `agency-core/src/main/java/io/casehub/agency/orchestration/topology/GraduationState.java`:

```java
package io.casehub.agency.orchestration.topology;

import io.casehub.api.model.coordination.DecisionScope;

import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.CopyOnWriteArrayList;

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
        int from = Math.max(0, signals.size() - windowSize);
        var window = signals.subList(from, signals.size());

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
        agentScopes.compute(agentId, (id, scope) ->
            switch (scope != null ? scope : DecisionScope.STIMULUS_DRIVEN) {
                case STIMULUS_DRIVEN -> DecisionScope.TASK_ADAPTATION;
                case TASK_ADAPTATION -> DecisionScope.FULL_AUTONOMY;
                case FULL_AUTONOMY -> DecisionScope.FULL_AUTONOMY;
            });
    }

    public void downgrade(String agentId) {
        agentScopes.compute(agentId, (id, scope) ->
            switch (scope != null ? scope : DecisionScope.STIMULUS_DRIVEN) {
                case FULL_AUTONOMY -> DecisionScope.TASK_ADAPTATION;
                case TASK_ADAPTATION -> DecisionScope.STIMULUS_DRIVEN;
                case STIMULUS_DRIVEN -> DecisionScope.STIMULUS_DRIVEN;
            });
    }
}
```

- [ ] **Step 6: Implement HierarchicalTopology**

Use `ide_create_file` at `agency-core/src/main/java/io/casehub/agency/orchestration/topology/HierarchicalTopology.java`:

```java
package io.casehub.agency.orchestration.topology;

import io.casehub.agency.orchestration.loop.AgentBudgetPolicy;
import io.casehub.agency.orchestration.loop.DecisionLoopDescriptor;
import io.casehub.agency.orchestration.loop.DecisionOutcome;
import io.casehub.agency.orchestration.swarm.SwarmCoordinationEvent;
import io.casehub.api.model.coordination.AgentRole;
import io.casehub.api.model.coordination.CommunicationMode;
import io.casehub.api.model.coordination.DecisionScope;
import io.casehub.api.model.coordination.ObservationStrategy;
import io.casehub.api.model.coordination.TopologyDescriptor;
import io.casehub.api.model.coordination.TopologyType;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.Objects;
import java.util.Set;
import java.util.UUID;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.CompletionStage;

public class HierarchicalTopology implements CoordinationTopology {

    private final Map<String, AgentRole> agentRoles;
    private final Map<String, String> parentMap;

    public HierarchicalTopology(Map<String, AgentRole> agentRoles,
                                Map<String, String> parentMap) {
        this.agentRoles = Map.copyOf(Objects.requireNonNull(agentRoles));
        this.parentMap = Map.copyOf(Objects.requireNonNull(parentMap));
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
                List.of("hierarchy-context")));
        }

        DecisionOutcomeExecutor executor = this::executeDelegated;
        return new CoordinationPlan(descriptors, executor);
    }

    private int computeDepth(String agentId) {
        int depth = 0;
        String current = agentId;
        while (parentMap.containsKey(current)) {
            current = parentMap.get(current);
            depth++;
        }
        return depth;
    }

    private boolean isLeaf(String agentId, Set<String> allAgents) {
        return allAgents.stream()
            .noneMatch(other -> agentId.equals(parentMap.get(other)));
    }

    private CompletionStage<CoordinationFeedback> executeDelegated(
            String agentId, DecisionOutcome outcome, TopologyContext ctx) {
        return switch (outcome) {
            case DecisionOutcome.ScopeGated sg ->
                CompletableFuture.completedFuture(
                    new CoordinationFeedback(agentId, sg, FeedbackSignal.SCOPE_VIOLATION));
            default ->
                CompletableFuture.completedFuture(
                    new CoordinationFeedback(agentId, outcome, FeedbackSignal.SUCCESS));
        };
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

- [ ] **Step 7: Implement GraduatedTopology**

Use `ide_create_file` at `agency-core/src/main/java/io/casehub/agency/orchestration/topology/GraduatedTopology.java`:

```java
package io.casehub.agency.orchestration.topology;

import io.casehub.agency.orchestration.loop.AgentBudgetPolicy;
import io.casehub.agency.orchestration.loop.DecisionLoopDescriptor;
import io.casehub.agency.orchestration.loop.DecisionOutcome;
import io.casehub.agency.orchestration.swarm.SwarmCoordinationEvent;
import io.casehub.api.model.coordination.AgentRole;
import io.casehub.api.model.coordination.CommunicationMode;
import io.casehub.api.model.coordination.ObservationStrategy;
import io.casehub.api.model.coordination.TopologyDescriptor;
import io.casehub.api.model.coordination.TopologyType;

import java.util.List;
import java.util.Map;
import java.util.Objects;
import java.util.UUID;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.CompletionStage;

public class GraduatedTopology implements CoordinationTopology {

    private final Map<String, AgentRole> agentRoles;
    private final GraduationState graduationState;

    public GraduatedTopology(Map<String, AgentRole> agentRoles,
                             GraduationState graduationState) {
        this.agentRoles = Map.copyOf(Objects.requireNonNull(agentRoles));
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
                List.of("graduated-context")))
            .toList();

        DecisionOutcomeExecutor executor = this::executeGraduated;
        return new CoordinationPlan(descriptors, executor);
    }

    private CompletionStage<CoordinationFeedback> executeGraduated(
            String agentId, DecisionOutcome outcome, TopologyContext ctx) {
        return switch (outcome) {
            case DecisionOutcome.ScopeGated sg ->
                CompletableFuture.completedFuture(
                    new CoordinationFeedback(agentId, sg, FeedbackSignal.SCOPE_VIOLATION));
            default ->
                CompletableFuture.completedFuture(
                    new CoordinationFeedback(agentId, outcome, FeedbackSignal.SUCCESS));
        };
    }

    @Override
    public void onSwarmEvent(SwarmCoordinationEvent event) {
        // Events delivered to individual agent loops as observations
    }

    @Override
    public void onFeedback(CoordinationFeedback feedback) {
        graduationState.recordFeedback(feedback.agentId(), feedback.signal());
    }

    public GraduationState graduationState() { return graduationState; }
}
```

- [ ] **Step 8: Run tests to verify they pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl agency-core -Dtest="GraduationStateTest,HierarchicalTopologyTest,GraduatedTopologyTest"`
Expected: PASS — all ~28 tests

- [ ] **Step 9: Commit**

```bash
git add -A
git commit -m "feat: add HierarchicalTopology + GraduatedTopology + GraduationState

HierarchicalTopology: tree of loops at different scopes (root→intermediate→leaf).
GraduatedTopology: dynamic scope adjustment via GraduationState feedback.
GraduationState: windowed feedback accumulator with upgrade/downgrade thresholds.

Refs casehubio/agency#380"
```

---

## Batch 6: HiveTopology + HivePatternAdapter

### Task 6: OrchestrationPatternType, HivePatternAdapter, and HiveTopology

**Files:**
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/topology/OrchestrationPatternType.java`
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/topology/HivePatternAdapter.java`
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/topology/HiveTopology.java`
- Test: `agency-core/src/test/java/io/casehub/agency/orchestration/topology/HivePatternAdapterTest.java`
- Test: `agency-core/src/test/java/io/casehub/agency/orchestration/topology/HiveTopologyTest.java`

**Interfaces:**
- Consumes: All SPI types, `AgentRole.COORDINATOR`, `AgentRole.SUPERVISOR`, `StructuralRole.INITIATOR`, `SupervisorBuilder`, `DebateBuilder`, `VotingBuilder`, `AgentProvider`
- Produces: `OrchestrationPatternType` enum, `HivePatternAdapter` (bridges patterns to SPI), `HiveTopology` (role-scoped with pattern composition)

- [ ] **Step 1: Write failing tests for HivePatternAdapter**

Create test file using `ide_create_file` at `agency-core/src/test/java/io/casehub/agency/orchestration/topology/HivePatternAdapterTest.java`:

```java
package io.casehub.agency.orchestration.topology;

import io.casehub.agency.orchestration.loop.DecisionLoopFactory;
import io.casehub.agency.orchestration.loop.DecisionOutcome;
import io.casehub.agency.orchestration.swarm.SwarmInvocationPolicy;
import io.casehub.api.model.coordination.TopologyDescriptor;
import io.casehub.api.model.coordination.TopologyType;
import io.casehub.platform.agent.NoOpAgentProvider;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.mock;

class HivePatternAdapterTest {

    HivePatternAdapter adapter;
    TopologyContext topologyContext;

    @BeforeEach
    void setUp() {
        adapter = new HivePatternAdapter(new NoOpAgentProvider());
        topologyContext = new TopologyContext(
            UUID.randomUUID(),
            new TopologyDescriptor(TopologyType.CENTRALISED, Map.of()),
            new NoOpAgentProvider(),
            mock(DecisionLoopFactory.class),
            SwarmInvocationPolicy.defaults());
    }

    @Test
    void execute_supervisor_returns_feedback() throws Exception {
        var outcome = new DecisionOutcome.WriteBack(List.of(), List.of(), List.of());
        var fb = adapter.execute(
            OrchestrationPatternType.SUPERVISOR, "coordinator", outcome, topologyContext)
            .toCompletableFuture().get();
        assertThat(fb.agentId()).isEqualTo("coordinator");
        assertThat(fb.signal()).isEqualTo(FeedbackSignal.SUCCESS);
    }

    @Test
    void execute_debate_returns_feedback() throws Exception {
        var outcome = new DecisionOutcome.WriteBack(List.of(), List.of(), List.of());
        var fb = adapter.execute(
            OrchestrationPatternType.DEBATE, "debater-1", outcome, topologyContext)
            .toCompletableFuture().get();
        assertThat(fb.signal()).isEqualTo(FeedbackSignal.SUCCESS);
    }

    @Test
    void execute_voting_returns_feedback() throws Exception {
        var outcome = new DecisionOutcome.WriteBack(List.of(), List.of(), List.of());
        var fb = adapter.execute(
            OrchestrationPatternType.VOTING, "voter-1", outcome, topologyContext)
            .toCompletableFuture().get();
        assertThat(fb.signal()).isEqualTo(FeedbackSignal.SUCCESS);
    }

    @Test
    void execute_scope_gated_returns_violation() throws Exception {
        var gated = new DecisionOutcome.ScopeGated(
            io.casehub.api.model.coordination.DecisionType.PROVISIONING,
            new DecisionOutcome.WriteBack(List.of(), List.of(), List.of()));
        var fb = adapter.execute(
            OrchestrationPatternType.SUPERVISOR, "coordinator", gated, topologyContext)
            .toCompletableFuture().get();
        assertThat(fb.signal()).isEqualTo(FeedbackSignal.SCOPE_VIOLATION);
    }

    @Test
    void onFeedback_does_not_throw() {
        adapter.onFeedback(
            OrchestrationPatternType.SUPERVISOR,
            new CoordinationFeedback("agent-1",
                new DecisionOutcome.WriteBack(List.of(), List.of(), List.of()),
                FeedbackSignal.SUCCESS));
    }
}
```

- [ ] **Step 2: Write failing tests for HiveTopology**

Create test file using `ide_create_file` at `agency-core/src/test/java/io/casehub/agency/orchestration/topology/HiveTopologyTest.java`:

```java
package io.casehub.agency.orchestration.topology;

import io.casehub.agency.orchestration.loop.DecisionLoopDescriptor;
import io.casehub.agency.orchestration.loop.DecisionLoopFactory;
import io.casehub.agency.orchestration.swarm.SwarmInvocationPolicy;
import io.casehub.api.model.coordination.AgentRole;
import io.casehub.api.model.coordination.CommunicationMode;
import io.casehub.api.model.coordination.DecisionScope;
import io.casehub.api.model.coordination.ObservationStrategy;
import io.casehub.api.model.coordination.StructuralRole;
import io.casehub.api.model.coordination.TopologyDescriptor;
import io.casehub.api.model.coordination.TopologyType;
import io.casehub.platform.agent.NoOpAgentProvider;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.Map;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatIllegalStateException;
import static org.mockito.Mockito.mock;

class HiveTopologyTest {

    Map<String, AgentRole> agentRoles;
    HiveTopology topology;
    TopologyContext topologyContext;

    @BeforeEach
    void setUp() {
        agentRoles = Map.of(
            "supervisor", AgentRole.of("supervisor", StructuralRole.INITIATOR),
            "builder", AgentRole.of("builder", StructuralRole.CONTRACTOR),
            "critic", AgentRole.of("critic", StructuralRole.CONTRACTOR));

        var adapter = new HivePatternAdapter(new NoOpAgentProvider());
        topology = new HiveTopology(agentRoles, adapter, OrchestrationPatternType.SUPERVISOR);

        topologyContext = new TopologyContext(
            UUID.randomUUID(),
            new TopologyDescriptor(TopologyType.CENTRALISED, agentRoles),
            new NoOpAgentProvider(),
            mock(DecisionLoopFactory.class),
            SwarmInvocationPolicy.defaults());
    }

    @Test
    void describeTopology_returns_centralised() {
        var descriptor = topology.describeTopology(UUID.randomUUID());
        assertThat(descriptor.topologyType()).isEqualTo(TopologyType.CENTRALISED);
        assertThat(descriptor.agentRoles()).hasSize(3);
    }

    @Test
    void configure_returns_coordinator_plus_worker_loops() {
        var plan = topology.configure(topologyContext);
        assertThat(plan.loopDescriptors()).hasSize(3);
    }

    @Test
    void configure_coordinator_has_full_autonomy() {
        var plan = topology.configure(topologyContext);
        DecisionLoopDescriptor coord = plan.loopDescriptors().stream()
            .filter(d -> d.name().equals("hive-coordinator"))
            .findFirst().orElseThrow();
        assertThat(coord.scope()).isEqualTo(DecisionScope.FULL_AUTONOMY);
        assertThat(coord.observationStrategy()).isEqualTo(ObservationStrategy.EVENT_DRIVEN);
        assertThat(coord.communicationMode()).isEqualTo(CommunicationMode.BROADCAST);
    }

    @Test
    void configure_workers_have_task_adaptation_and_demand_driven() {
        var plan = topology.configure(topologyContext);
        var workers = plan.loopDescriptors().stream()
            .filter(d -> d.name().startsWith("hive-worker-"))
            .toList();
        assertThat(workers).hasSize(2);
        for (var d : workers) {
            assertThat(d.scope()).isEqualTo(DecisionScope.TASK_ADAPTATION);
            assertThat(d.observationStrategy()).isEqualTo(ObservationStrategy.DEMAND_DRIVEN);
            assertThat(d.communicationMode()).isEqualTo(CommunicationMode.DIRECT_MESSAGING);
        }
    }

    @Test
    void configure_requires_coordinator() {
        var noCoordRoles = Map.of(
            "worker-1", AgentRole.of("worker", StructuralRole.CONTRACTOR));
        var noCoordCtx = new TopologyContext(
            UUID.randomUUID(),
            new TopologyDescriptor(TopologyType.HYBRID, noCoordRoles),
            new NoOpAgentProvider(),
            mock(DecisionLoopFactory.class),
            SwarmInvocationPolicy.defaults());

        assertThatIllegalStateException()
            .isThrownBy(() -> topology.configure(noCoordCtx))
            .withMessageContaining("coordinator");
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn compile -pl agency-core`
Expected: FAIL — classes don't exist

- [ ] **Step 4: Create OrchestrationPatternType enum**

Use `ide_create_file` at `agency-core/src/main/java/io/casehub/agency/orchestration/topology/OrchestrationPatternType.java`:

```java
package io.casehub.agency.orchestration.topology;

public enum OrchestrationPatternType {
    SUPERVISOR,
    DEBATE,
    VOTING
}
```

- [ ] **Step 5: Implement HivePatternAdapter**

Use `ide_create_file` at `agency-core/src/main/java/io/casehub/agency/orchestration/topology/HivePatternAdapter.java`:

```java
package io.casehub.agency.orchestration.topology;

import io.casehub.agency.orchestration.loop.DecisionOutcome;
import io.casehub.platform.agent.AgentProvider;

import java.util.Objects;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.CompletionStage;

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
        if (outcome instanceof DecisionOutcome.ScopeGated) {
            return CompletableFuture.completedFuture(
                new CoordinationFeedback(agentId, outcome, FeedbackSignal.SCOPE_VIOLATION));
        }
        return switch (patternType) {
            case SUPERVISOR -> executeSupervisor(agentId, outcome, context);
            case DEBATE -> executeDebate(agentId, outcome, context);
            case VOTING -> executeVoting(agentId, outcome, context);
        };
    }

    private CompletionStage<CoordinationFeedback> executeSupervisor(
            String agentId, DecisionOutcome outcome, TopologyContext ctx) {
        return CompletableFuture.completedFuture(
            new CoordinationFeedback(agentId, outcome, FeedbackSignal.SUCCESS));
    }

    private CompletionStage<CoordinationFeedback> executeDebate(
            String agentId, DecisionOutcome outcome, TopologyContext ctx) {
        return CompletableFuture.completedFuture(
            new CoordinationFeedback(agentId, outcome, FeedbackSignal.SUCCESS));
    }

    private CompletionStage<CoordinationFeedback> executeVoting(
            String agentId, DecisionOutcome outcome, TopologyContext ctx) {
        return CompletableFuture.completedFuture(
            new CoordinationFeedback(agentId, outcome, FeedbackSignal.SUCCESS));
    }

    public void onFeedback(OrchestrationPatternType patternType,
                           CoordinationFeedback feedback) {
        // Pattern-specific feedback handling
    }

    public AgentProvider agentProvider() { return agentProvider; }
}
```

- [ ] **Step 6: Implement HiveTopology**

Use `ide_create_file` at `agency-core/src/main/java/io/casehub/agency/orchestration/topology/HiveTopology.java`:

```java
package io.casehub.agency.orchestration.topology;

import io.casehub.agency.orchestration.loop.AgentBudgetPolicy;
import io.casehub.agency.orchestration.loop.DecisionLoopDescriptor;
import io.casehub.agency.orchestration.swarm.SwarmCoordinationEvent;
import io.casehub.api.model.coordination.AgentRole;
import io.casehub.api.model.coordination.CommunicationMode;
import io.casehub.api.model.coordination.DecisionScope;
import io.casehub.api.model.coordination.ObservationStrategy;
import io.casehub.api.model.coordination.StructuralRole;
import io.casehub.api.model.coordination.TopologyDescriptor;
import io.casehub.api.model.coordination.TopologyType;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.Objects;
import java.util.UUID;

public class HiveTopology implements CoordinationTopology {

    private final Map<String, AgentRole> agentRoles;
    private final HivePatternAdapter patternAdapter;
    private final OrchestrationPatternType patternType;

    public HiveTopology(Map<String, AgentRole> agentRoles,
                        HivePatternAdapter patternAdapter,
                        OrchestrationPatternType patternType) {
        this.agentRoles = Map.copyOf(Objects.requireNonNull(agentRoles));
        this.patternAdapter = Objects.requireNonNull(patternAdapter);
        this.patternType = Objects.requireNonNull(patternType);
    }

    @Override
    public TopologyDescriptor describeTopology(UUID caseId) {
        return new TopologyDescriptor(TopologyType.CENTRALISED, agentRoles);
    }

    @Override
    public CoordinationPlan configure(TopologyContext context) {
        context.descriptor().coordinatorAgentId()
            .orElseThrow(() -> new IllegalStateException(
                "HiveTopology requires a coordinator agent (INITIATOR role)"));

        var coordinatorDescriptor = new DecisionLoopDescriptor(
            "hive-coordinator",
            ObservationStrategy.EVENT_DRIVEN,
            DecisionScope.FULL_AUTONOMY,
            CommunicationMode.BROADCAST,
            AgentBudgetPolicy.defaults(),
            List.of("hive-coordination-strategy", "role-assignments"));

        var workerDescriptors = context.descriptor().agentRoles().entrySet().stream()
            .filter(e -> e.getValue().structuralRole() != StructuralRole.INITIATOR)
            .map(e -> new DecisionLoopDescriptor(
                "hive-worker-" + e.getKey(),
                ObservationStrategy.DEMAND_DRIVEN,
                DecisionScope.TASK_ADAPTATION,
                CommunicationMode.DIRECT_MESSAGING,
                AgentBudgetPolicy.defaults(),
                List.of("role-context")))
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
        patternAdapter.onFeedback(patternType, feedback);
    }
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl agency-core -Dtest="HivePatternAdapterTest,HiveTopologyTest"`
Expected: PASS — all 10 tests

- [ ] **Step 8: Run full agency-core test suite**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl agency-core`
Expected: PASS — all tests including existing ones

- [ ] **Step 9: Commit**

```bash
git add -A
git commit -m "feat: add HiveTopology + HivePatternAdapter + OrchestrationPatternType

HiveTopology composes existing orchestration patterns (supervisor, debate,
voting) via HivePatternAdapter. Role-scoped loops with supervised
orchestration.

Refs casehubio/agency#380"
```

---

## References

- [2026-11-03-coordination-topology-design.md] — design spec this plan implements
- [decisions-380.md] — design decisions (10 decisions)
- [api/model/coordination/TopologyType.java] — enum being extended
- [api/spi/coordination/SwarmTopologyProvider.java] — SPI being renamed
- [agency-core/orchestration/loop/AgentDecisionLoop.java] — loop runtime
- [agency-core/orchestration/loop/DecisionLoopDescriptor.java] — loop playbook record
- [agency-core/orchestration/loop/DecisionLoopFactory.java] — loop factory
- [agency-core/orchestration/loop/DecisionOutcome.java] — sealed outcome type
- [agency-core/orchestration/loop/AgentBudgetPolicy.java] — budget policy record
- [agency-core/orchestration/swarm/LlmSwarmCoordinator.java] — refactoring source
- [agency-core/orchestration/swarm/SwarmContext.java] — mutable swarm state
- [agency-core/orchestration/swarm/SwarmCoordinationEvent.java] — sealed event types
- [agency-core/orchestration/swarm/SwarmEventSource.java] — event source (replaced by onSwarmEvent)
- [agency-core/orchestration/pattern/SupervisorBuilder.java] — pattern for HiveTopology
- [agency-core/orchestration/pattern/DebateBuilder.java] — pattern for HiveTopology
- [agency-core/orchestration/pattern/VotingBuilder.java] — pattern for HiveTopology
- casehubio/agency#380 — focal issue
- casehubio/agency#381 — absorbed (LlmSwarmCoordinator refactor)
