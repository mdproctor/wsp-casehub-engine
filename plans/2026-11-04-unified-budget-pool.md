# Unified Budget Pool Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/agency#382 — Per-agent invocation budget — flat per-agent trackers
**Issue group:** #382

**Goal:** Unify the two-pool budget split so all budget enforcement and token recording use a single `AutonomyBudgetPool` created by the topology.

**Architecture:** Remove pool creation from `SwarmCoordinatorBuilder`. Add `setTokenRecorder(LongConsumer)` to three direct consumers so `CentralisedTopology.configure()` can wire token recording without call gating. Decision loops' `BudgetAwareSummariser` remains the single call gate.

**Tech Stack:** Java 21, JUnit 5, AssertJ, Mockito

## Global Constraints

- All code in `agency-core` module (`io.casehub.agency.orchestration`)
- Builder test in `agency` module
- `SwarmProvisioningAdvisor` SPI interface must not be modified (lives in engine-api)
- `setBudgetPool()` API on consumers must be preserved for YAML path (PatternCompiler)

---

## Batch 1: Token recorder on consumers + builder cleanup

### Task 1: Add `setTokenRecorder` to SwarmRoutingStrategy

**Files:**
- Modify: `agency-core/src/main/java/io/casehub/agency/orchestration/swarm/SwarmRoutingStrategy.java`
- Test: `agency-core/src/test/java/io/casehub/agency/orchestration/swarm/SwarmRoutingStrategyTokenRecorderTest.java`

**Interfaces:**
- Produces: `void setTokenRecorder(@Nullable LongConsumer recorder)` — public setter on `SwarmRoutingStrategy`. When set, token recording uses the callback instead of `budgetPool.recordTokens()`.

- [ ] **Step 1: Write the failing test**

Create `SwarmRoutingStrategyTokenRecorderTest.java`:

```java
package io.casehub.agency.orchestration.swarm;

import org.junit.jupiter.api.Test;
import java.util.concurrent.atomic.AtomicLong;
import static org.assertj.core.api.Assertions.assertThat;

class SwarmRoutingStrategyTokenRecorderTest {

    @Test
    void setTokenRecorder_not_present_by_default() {
        var strategy = new SwarmRoutingStrategy(
                io.casehub.agency.orchestration.topology.TestAgentProviders.noOp(),
                new SwarmExperienceSection(ExperienceReplayConfig.defaults()), 12);
        // Just verify the method exists and accepts null
        strategy.setTokenRecorder(null);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl agency-core -Dtest=SwarmRoutingStrategyTokenRecorderTest -q`
Expected: FAIL — `setTokenRecorder` method does not exist

- [ ] **Step 3: Add `tokenRecorder` field and setter to SwarmRoutingStrategy**

Add after the `budgetPool` field (line 36):

```java
private @Nullable java.util.function.LongConsumer tokenRecorder;
```

Add after `setBudgetPool` method (after line 58):

```java
public void setTokenRecorder(@Nullable java.util.function.LongConsumer recorder) {
    this.tokenRecorder = recorder;
}
```

Add import for `java.util.function.LongConsumer` at the top.

- [ ] **Step 4: Modify token recording to use tokenRecorder when present**

Change the token recording block (lines 107-109) from:

```java
if (budgetPool != null) {
    budgetPool.recordTokens("routing", s.metadata().inputTokens() + s.metadata().outputTokens());
}
```

To:

```java
long tokens = s.metadata().inputTokens() + s.metadata().outputTokens();
if (tokenRecorder != null) {
    tokenRecorder.accept(tokens);
} else if (budgetPool != null) {
    budgetPool.recordTokens("routing", tokens);
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn test -pl agency-core -Dtest=SwarmRoutingStrategyTokenRecorderTest -q`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add agency-core/src/main/java/io/casehub/agency/orchestration/swarm/SwarmRoutingStrategy.java
git add agency-core/src/test/java/io/casehub/agency/orchestration/swarm/SwarmRoutingStrategyTokenRecorderTest.java
git commit -m "feat(#382): add setTokenRecorder to SwarmRoutingStrategy

Refs casehubio/agency#382"
```

### Task 2: Add `setTokenRecorder` to CollectiveReasoningAggregation

**Files:**
- Modify: `agency-core/src/main/java/io/casehub/agency/orchestration/swarm/CollectiveReasoningAggregation.java`
- Test: `agency-core/src/test/java/io/casehub/agency/orchestration/swarm/CollectiveReasoningAggregationTokenRecorderTest.java`

**Interfaces:**
- Produces: `void setTokenRecorder(@Nullable LongConsumer recorder)` — public setter on `CollectiveReasoningAggregation`.

- [ ] **Step 1: Write the failing test**

Create `CollectiveReasoningAggregationTokenRecorderTest.java`:

```java
package io.casehub.agency.orchestration.swarm;

import org.junit.jupiter.api.Test;

class CollectiveReasoningAggregationTokenRecorderTest {

    @Test
    void setTokenRecorder_not_present_by_default() {
        var aggregation = new CollectiveReasoningAggregation(
                io.casehub.agency.orchestration.topology.TestAgentProviders.noOp());
        aggregation.setTokenRecorder(null);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl agency-core -Dtest=CollectiveReasoningAggregationTokenRecorderTest -q`
Expected: FAIL — `setTokenRecorder` method does not exist

- [ ] **Step 3: Add `tokenRecorder` field and setter**

Add after the `budgetPool` field (line 28):

```java
private @Nullable java.util.function.LongConsumer tokenRecorder;
```

Add after `setBudgetPool` method (after line 44):

```java
public void setTokenRecorder(@Nullable java.util.function.LongConsumer recorder) {
    this.tokenRecorder = recorder;
}
```

- [ ] **Step 4: Modify token recording**

Change lines 81-83 from:

```java
if (budgetPool != null) {
    budgetPool.recordTokens("aggregation", s.metadata().inputTokens() + s.metadata().outputTokens());
}
```

To:

```java
long tokens = s.metadata().inputTokens() + s.metadata().outputTokens();
if (tokenRecorder != null) {
    tokenRecorder.accept(tokens);
} else if (budgetPool != null) {
    budgetPool.recordTokens("aggregation", tokens);
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn test -pl agency-core -Dtest=CollectiveReasoningAggregationTokenRecorderTest -q`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add agency-core/src/main/java/io/casehub/agency/orchestration/swarm/CollectiveReasoningAggregation.java
git add agency-core/src/test/java/io/casehub/agency/orchestration/swarm/CollectiveReasoningAggregationTokenRecorderTest.java
git commit -m "feat(#382): add setTokenRecorder to CollectiveReasoningAggregation

Refs casehubio/agency#382"
```

### Task 3: Add `setTokenRecorder` to SwarmProvisioningAdvisorImpl

**Files:**
- Modify: `agency-core/src/main/java/io/casehub/agency/orchestration/swarm/SwarmProvisioningAdvisorImpl.java`
- Test: `agency-core/src/test/java/io/casehub/agency/orchestration/swarm/SwarmProvisioningAdvisorImplTokenRecorderTest.java`

**Interfaces:**
- Produces: `void setTokenRecorder(@Nullable LongConsumer recorder)` — public setter on `SwarmProvisioningAdvisorImpl`.

- [ ] **Step 1: Write the failing test**

Create `SwarmProvisioningAdvisorImplTokenRecorderTest.java`:

```java
package io.casehub.agency.orchestration.swarm;

import org.junit.jupiter.api.Test;

class SwarmProvisioningAdvisorImplTokenRecorderTest {

    @Test
    void setTokenRecorder_not_present_by_default() {
        var advisor = new SwarmProvisioningAdvisorImpl(
                io.casehub.agency.orchestration.topology.TestAgentProviders.noOp(),
                new SwarmContext());
        advisor.setTokenRecorder(null);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl agency-core -Dtest=SwarmProvisioningAdvisorImplTokenRecorderTest -q`
Expected: FAIL — `setTokenRecorder` method does not exist

- [ ] **Step 3: Add `tokenRecorder` field and setter**

Add after the `budgetPool` field (line 23):

```java
private @Nullable java.util.function.LongConsumer tokenRecorder;
```

Add after `setBudgetPool` method (after line 41):

```java
public void setTokenRecorder(@Nullable java.util.function.LongConsumer recorder) {
    this.tokenRecorder = recorder;
}
```

- [ ] **Step 4: Modify token recording**

Change lines 71-73 from:

```java
if (budgetPool != null) {
    budgetPool.recordTokens("provisioning", s.metadata().inputTokens() + s.metadata().outputTokens());
}
```

To:

```java
long tokens = s.metadata().inputTokens() + s.metadata().outputTokens();
if (tokenRecorder != null) {
    tokenRecorder.accept(tokens);
} else if (budgetPool != null) {
    budgetPool.recordTokens("provisioning", tokens);
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn test -pl agency-core -Dtest=SwarmProvisioningAdvisorImplTokenRecorderTest -q`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add agency-core/src/main/java/io/casehub/agency/orchestration/swarm/SwarmProvisioningAdvisorImpl.java
git add agency-core/src/test/java/io/casehub/agency/orchestration/swarm/SwarmProvisioningAdvisorImplTokenRecorderTest.java
git commit -m "feat(#382): add setTokenRecorder to SwarmProvisioningAdvisorImpl

Refs casehubio/agency#382"
```

## Batch 2: Topology wiring + builder cleanup

### Task 4: Wire token recorders in CentralisedTopology and remove builder pool

**Files:**
- Modify: `agency-core/src/main/java/io/casehub/agency/orchestration/topology/CentralisedTopology.java`
- Modify: `agency-core/src/main/java/io/casehub/agency/orchestration/swarm/SwarmCoordinatorBuilder.java`
- Modify: `agency-core/src/test/java/io/casehub/agency/orchestration/topology/CentralisedTopologyTest.java`
- Modify: `agency/src/test/java/io/casehub/agency/orchestration/swarm/SwarmCoordinatorBuilderTest.java`

**Interfaces:**
- Consumes: `setTokenRecorder(LongConsumer)` from Tasks 1-3

- [ ] **Step 1: Write the failing test — topology wires token recorders**

Add test to `CentralisedTopologyTest.java`:

```java
@Test
void configure_wires_tokenRecorder_on_routing_strategy() {
    topology.configure(topologyContext);

    // Verify the routing strategy has a tokenRecorder wired
    assertThat(topology.routingStrategy())
            .extracting("tokenRecorder")
            .isNotNull();
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl agency-core -Dtest=CentralisedTopologyTest#configure_wires_tokenRecorder_on_routing_strategy -q`
Expected: FAIL — `tokenRecorder` field is null (not wired)

- [ ] **Step 3: Wire token recorders in CentralisedTopology.configure()**

In `CentralisedTopology.java`, after the pool creation (line 71), add:

```java
routingStrategy.setTokenRecorder(t -> pool.recordTokens("routing", t));
aggregation.setTokenRecorder(t -> pool.recordTokens("aggregation", t));
if (provisioningAdvisor instanceof io.casehub.agency.orchestration.swarm.SwarmProvisioningAdvisorImpl impl) {
    impl.setTokenRecorder(t -> pool.recordTokens("provisioning", t));
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl agency-core -Dtest=CentralisedTopologyTest#configure_wires_tokenRecorder_on_routing_strategy -q`
Expected: PASS

- [ ] **Step 5: Remove pool creation and setBudgetPool from builder**

In `SwarmCoordinatorBuilder.java`, remove lines 86, 90, 92, 114:

```java
// Remove this line:
var pool              = new AutonomyBudgetPool(budgetPolicy, "swarm-" + UUID.randomUUID());
// Remove this line:
routingStrategy.setBudgetPool(pool);
// Remove this line:
aggregation.setBudgetPool(pool);
// Remove this line:
provisioner.setBudgetPool(pool);
```

Also remove the now-unused import for `AutonomyBudgetPool`.

- [ ] **Step 6: Update builder test**

Replace the content of `SwarmCoordinatorBuilderTest.java`:

```java
package io.casehub.agency.orchestration.swarm;

import io.casehub.platform.agent.AgentProvider;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.mock;

class SwarmCoordinatorBuilderTest {

    @Test
    void buildTopology_does_not_inject_budgetPool_on_strategies() {
        var topology = new SwarmCoordinatorBuilder()
                               .agentProvider(mock(AgentProvider.class))
                               .buildTopology();

        assertThat(topology.routingStrategy())
                .extracting("budgetPool")
                .isNull();
        assertThat(topology.aggregation())
                .extracting("budgetPool")
                .isNull();
    }
}
```

- [ ] **Step 7: Run full test suite**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl agency-core,agency -q`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```bash
git add agency-core/src/main/java/io/casehub/agency/orchestration/topology/CentralisedTopology.java
git add agency-core/src/main/java/io/casehub/agency/orchestration/swarm/SwarmCoordinatorBuilder.java
git add agency-core/src/test/java/io/casehub/agency/orchestration/topology/CentralisedTopologyTest.java
git add agency/src/test/java/io/casehub/agency/orchestration/swarm/SwarmCoordinatorBuilderTest.java
git commit -m "feat(#382): unify to single budget pool — topology owns creation, consumers record tokens

Removes pool creation from SwarmCoordinatorBuilder. Decision loops
(BudgetAwareSummariser) are the single call gate. Token recording
flows through setTokenRecorder callbacks wired in CentralisedTopology.

Closes casehubio/agency#382"
```

## References

- [2026-11-04-per-agent-invocation-budget-design.md] — design spec this plan implements
- `SwarmCoordinatorBuilder.java:83-121` — builder pool creation (removed)
- `CentralisedTopology.java:70-124` — topology pool creation (wiring added)
- `SwarmRoutingStrategy.java:36,78-84,107-109` — budget field and token recording
- `CollectiveReasoningAggregation.java:28,59-65,81-83` — budget field and token recording
- `SwarmProvisioningAdvisorImpl.java:23,50-56,71-73` — budget field and token recording
- `SwarmCoordinatorBuilderTest.java` — builder test (assertions flipped)
- `CentralisedTopologyTest.java` — topology test (assertion added)
- GitHub casehubio/agency#382 — focal issue
- GitHub casehubio/agency#391 — downstream: priority test assertions
- GitHub casehubio/agency#392 — downstream: pool test assertions
- GitHub casehubio/agency#394 — downstream: setBudgetPool fallback tests
