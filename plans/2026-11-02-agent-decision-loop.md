# AgentDecisionLoop Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/agency#379 — feat: AgentDecisionLoop — unified observe→reason→act primitive
**Issue group:** casehubio/agency#378

**Goal:** Build the unified observe→reason→act primitive that all LLM coordination decisions compose from — a playbook-oriented decision loop with descriptor/runtime split.

**Architecture:** Descriptor + Runtime split. `DecisionLoopDescriptor` (YAML-aware data record) describes the playbook. `AgentDecisionLoop` (runtime) executes it by composing a `SummarisationRunner` pipeline. `DecisionLoopFactory` assembles the runtime from a descriptor, wiring budget tracking, prompt sections, emission policies, and concurrency guards. `DecisionOutcome` sealed interface bridges write-back and return-value action semantics.

**Tech Stack:** Java 21, JUnit 5, AssertJ, casehub-engine-api (coordination vocabulary), casehub-agency-summarisation-api (ContentSummariser, SummarisationRunner, EmissionPolicy, InFlightGate)

## Global Constraints

- All new files in `agency-core/src/main/java/io/casehub/agency/orchestration/loop/`
- All test files in `agency-core/src/test/java/io/casehub/agency/orchestration/loop/`
- No CDI/Spring annotations — agency-core is framework-neutral POJOs
- Records must be YAML-friendly: no lambdas, no callbacks in constructors
- Immutable collections via `List.copyOf` / `Map.copyOf`
- `@Nullable` from `org.jspecify.annotations`
- Test framework: JUnit 5 + AssertJ (`assertThat`, not `assertEquals`)
- Build: `mvn test -pl agency-core` from the agency repo root
- Commit messages: `feat(#379): <description>` with `Refs casehubio/agency#379`
- Use IntelliJ MCP for all code navigation and structural editing

---

## Batch 1: Value Objects and Sealed Types

After this batch: all data types compile, are tested, and have no dependencies on each other. The foundation for Batch 2's infrastructure.

### Task 1: Configuration and Foundation Types

Create the simple records and constants that other types depend on.

**Files:**
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/loop/AgentBudgetPolicy.java`
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/loop/DecisionContext.java`
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/loop/ObservationEvent.java`
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/loop/DecisionLoopLevels.java`
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/loop/BudgetExhaustedException.java`
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/loop/LoopPromptSection.java`
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/loop/DecisionLoopDescriptor.java`
- Test: `agency-core/src/test/java/io/casehub/agency/orchestration/loop/AgentBudgetPolicyTest.java`
- Test: `agency-core/src/test/java/io/casehub/agency/orchestration/loop/DecisionContextTest.java`
- Test: `agency-core/src/test/java/io/casehub/agency/orchestration/loop/ObservationEventTest.java`
- Test: `agency-core/src/test/java/io/casehub/agency/orchestration/loop/DecisionLoopDescriptorTest.java`

**Interfaces:**
- Consumes: `io.casehub.api.model.coordination.ObservationStrategy`, `DecisionScope`, `CommunicationMode`, `TopologyDescriptor` (engine-api — already exist)
- Produces: `AgentBudgetPolicy` (record with defaults()/unlimited() factory methods), `DecisionContext` (record: caseId, agentId, scope, communicationMode, topology), `ObservationEvent` (record: key, value, timestamp + of() factory), `DecisionLoopLevels` (OBSERVATION=50, DECISION=60), `BudgetExhaustedException` (extends RuntimeException), `LoopPromptSection` (functional interface: String contribute(DecisionContext)), `DecisionLoopDescriptor` (record: name, observationStrategy, scope, communicationMode, budgetPolicy, promptSectionNames)

- [ ] **Step 1: Write AgentBudgetPolicy test**

```java
package io.casehub.agency.orchestration.loop;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class AgentBudgetPolicyTest {

    @Test
    void defaults_returnsExpectedValues() {
        var policy = AgentBudgetPolicy.defaults();
        assertThat(policy.maxCallsPerAgent()).isEqualTo(10);
        assertThat(policy.maxTokensPerAgent()).isEqualTo(50_000L);
    }

    @Test
    void unlimited_returnsMaxValues() {
        var policy = AgentBudgetPolicy.unlimited();
        assertThat(policy.maxCallsPerAgent()).isEqualTo(Integer.MAX_VALUE);
        assertThat(policy.maxTokensPerAgent()).isEqualTo(Long.MAX_VALUE);
    }

    @Test
    void negativeCalls_throws() {
        assertThatIllegalArgumentException()
            .isThrownBy(() -> new AgentBudgetPolicy(-1, 100));
    }

    @Test
    void negativeTokens_throws() {
        assertThatIllegalArgumentException()
            .isThrownBy(() -> new AgentBudgetPolicy(10, -1));
    }

    @Test
    void zeroBudget_isValid() {
        var policy = new AgentBudgetPolicy(0, 0);
        assertThat(policy.maxCallsPerAgent()).isZero();
        assertThat(policy.maxTokensPerAgent()).isZero();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl agency-core -Dtest=AgentBudgetPolicyTest -q` from `/Users/mdproctor/claude/casehub/slots/220/agency`
Expected: compilation failure — `AgentBudgetPolicy` does not exist

- [ ] **Step 3: Implement AgentBudgetPolicy**

```java
package io.casehub.agency.orchestration.loop;

public record AgentBudgetPolicy(int maxCallsPerAgent, long maxTokensPerAgent) {

    public AgentBudgetPolicy {
        if (maxCallsPerAgent < 0)
            throw new IllegalArgumentException("maxCallsPerAgent must be non-negative");
        if (maxTokensPerAgent < 0)
            throw new IllegalArgumentException("maxTokensPerAgent must be non-negative");
    }

    public static AgentBudgetPolicy defaults() {
        return new AgentBudgetPolicy(10, 50_000);
    }

    public static AgentBudgetPolicy unlimited() {
        return new AgentBudgetPolicy(Integer.MAX_VALUE, Long.MAX_VALUE);
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl agency-core -Dtest=AgentBudgetPolicyTest -q`
Expected: 5 tests pass

- [ ] **Step 5: Write DecisionContext test**

```java
package io.casehub.agency.orchestration.loop;

import io.casehub.api.model.coordination.*;
import org.junit.jupiter.api.Test;
import java.util.Map;
import java.util.UUID;
import static org.assertj.core.api.Assertions.*;

class DecisionContextTest {

    @Test
    void validConstruction() {
        var topology = new TopologyDescriptor(TopologyType.CENTRALISED, Map.of());
        var ctx = new DecisionContext(
            UUID.randomUUID(), "agent-1",
            DecisionScope.FULL_AUTONOMY,
            CommunicationMode.STIGMERGY,
            topology);
        assertThat(ctx.agentId()).isEqualTo("agent-1");
        assertThat(ctx.scope()).isEqualTo(DecisionScope.FULL_AUTONOMY);
    }

    @Test
    void nullCaseId_throws() {
        var topology = new TopologyDescriptor(TopologyType.CENTRALISED, Map.of());
        assertThatNullPointerException()
            .isThrownBy(() -> new DecisionContext(
                null, "agent-1", DecisionScope.FULL_AUTONOMY,
                CommunicationMode.STIGMERGY, topology));
    }

    @Test
    void nullAgentId_throws() {
        var topology = new TopologyDescriptor(TopologyType.CENTRALISED, Map.of());
        assertThatNullPointerException()
            .isThrownBy(() -> new DecisionContext(
                UUID.randomUUID(), null, DecisionScope.FULL_AUTONOMY,
                CommunicationMode.STIGMERGY, topology));
    }

    @Test
    void nullScope_throws() {
        var topology = new TopologyDescriptor(TopologyType.CENTRALISED, Map.of());
        assertThatNullPointerException()
            .isThrownBy(() -> new DecisionContext(
                UUID.randomUUID(), "agent-1", null,
                CommunicationMode.STIGMERGY, topology));
    }

    @Test
    void nullTopology_throws() {
        assertThatNullPointerException()
            .isThrownBy(() -> new DecisionContext(
                UUID.randomUUID(), "agent-1", DecisionScope.FULL_AUTONOMY,
                CommunicationMode.STIGMERGY, null));
    }
}
```

- [ ] **Step 6: Implement DecisionContext, ObservationEvent, DecisionLoopLevels, BudgetExhaustedException, LoopPromptSection**

`DecisionContext.java`:
```java
package io.casehub.agency.orchestration.loop;

import io.casehub.api.model.coordination.*;
import java.util.Objects;
import java.util.UUID;

public record DecisionContext(
    UUID caseId,
    String agentId,
    DecisionScope scope,
    CommunicationMode communicationMode,
    TopologyDescriptor topology
) {
    public DecisionContext {
        Objects.requireNonNull(caseId, "caseId must not be null");
        Objects.requireNonNull(agentId, "agentId must not be null");
        Objects.requireNonNull(scope, "scope must not be null");
        Objects.requireNonNull(communicationMode, "communicationMode must not be null");
        Objects.requireNonNull(topology, "topology must not be null");
    }
}
```

`ObservationEvent.java`:
```java
package io.casehub.agency.orchestration.loop;

import java.time.Instant;
import java.util.Objects;

public record ObservationEvent(String key, Object value, Instant timestamp) {

    public ObservationEvent {
        Objects.requireNonNull(key, "key must not be null");
        Objects.requireNonNull(timestamp, "timestamp must not be null");
    }

    public static ObservationEvent of(String key, Object value) {
        return new ObservationEvent(key, value, Instant.now());
    }
}
```

`DecisionLoopLevels.java`:
```java
package io.casehub.agency.orchestration.loop;

import io.casehub.agency.summarisation.EventLevel;

public final class DecisionLoopLevels {
    public static final EventLevel OBSERVATION = new EventLevel("OBSERVATION", 50);
    public static final EventLevel DECISION = new EventLevel("DECISION", 60);
    private DecisionLoopLevels() {}
}
```

`BudgetExhaustedException.java`:
```java
package io.casehub.agency.orchestration.loop;

public class BudgetExhaustedException extends RuntimeException {

    private final String agentId;

    public BudgetExhaustedException(String agentId) {
        super("Budget exhausted for agent: " + agentId);
        this.agentId = agentId;
    }

    public String agentId() { return agentId; }
}
```

`LoopPromptSection.java`:
```java
package io.casehub.agency.orchestration.loop;

@FunctionalInterface
public interface LoopPromptSection {
    String contribute(DecisionContext context);
}
```

- [ ] **Step 7: Write ObservationEvent test and run all tests**

```java
package io.casehub.agency.orchestration.loop;

import org.junit.jupiter.api.Test;
import java.time.Instant;
import static org.assertj.core.api.Assertions.*;

class ObservationEventTest {

    @Test
    void validConstruction() {
        var ts = Instant.now();
        var event = new ObservationEvent("key", "value", ts);
        assertThat(event.key()).isEqualTo("key");
        assertThat(event.value()).isEqualTo("value");
        assertThat(event.timestamp()).isEqualTo(ts);
    }

    @Test
    void nullKey_throws() {
        assertThatNullPointerException()
            .isThrownBy(() -> new ObservationEvent(null, "v", Instant.now()));
    }

    @Test
    void nullValue_allowed() {
        var event = new ObservationEvent("key", null, Instant.now());
        assertThat(event.value()).isNull();
    }

    @Test
    void factoryMethod_setsTimestamp() {
        var before = Instant.now();
        var event = ObservationEvent.of("key", 42);
        var after = Instant.now();
        assertThat(event.timestamp()).isBetween(before, after);
        assertThat(event.key()).isEqualTo("key");
        assertThat(event.value()).isEqualTo(42);
    }
}
```

Run: `mvn test -pl agency-core -Dtest="AgentBudgetPolicyTest,DecisionContextTest,ObservationEventTest" -q`
Expected: all tests pass

- [ ] **Step 8: Write DecisionLoopDescriptor test**

```java
package io.casehub.agency.orchestration.loop;

import io.casehub.api.model.coordination.*;
import org.junit.jupiter.api.Test;
import java.util.ArrayList;
import java.util.List;
import static org.assertj.core.api.Assertions.*;

class DecisionLoopDescriptorTest {

    @Test
    void validConstruction() {
        var descriptor = new DecisionLoopDescriptor(
            "test-loop",
            ObservationStrategy.EVENT_DRIVEN,
            DecisionScope.FULL_AUTONOMY,
            CommunicationMode.STIGMERGY,
            AgentBudgetPolicy.defaults(),
            List.of("section-a", "section-b"));
        assertThat(descriptor.name()).isEqualTo("test-loop");
        assertThat(descriptor.promptSectionNames()).containsExactly("section-a", "section-b");
    }

    @Test
    void nullSectionNames_defaultsToEmpty() {
        var descriptor = new DecisionLoopDescriptor(
            "test", ObservationStrategy.EVENT_DRIVEN,
            DecisionScope.FULL_AUTONOMY, CommunicationMode.STIGMERGY,
            AgentBudgetPolicy.defaults(), null);
        assertThat(descriptor.promptSectionNames()).isEmpty();
    }

    @Test
    void sectionNames_areImmutable() {
        var mutable = new ArrayList<>(List.of("a", "b"));
        var descriptor = new DecisionLoopDescriptor(
            "test", ObservationStrategy.EVENT_DRIVEN,
            DecisionScope.FULL_AUTONOMY, CommunicationMode.STIGMERGY,
            AgentBudgetPolicy.defaults(), mutable);
        mutable.add("c");
        assertThat(descriptor.promptSectionNames()).containsExactly("a", "b");
    }

    @Test
    void nullName_throws() {
        assertThatNullPointerException()
            .isThrownBy(() -> new DecisionLoopDescriptor(
                null, ObservationStrategy.EVENT_DRIVEN,
                DecisionScope.FULL_AUTONOMY, CommunicationMode.STIGMERGY,
                AgentBudgetPolicy.defaults(), List.of()));
    }

    @Test
    void nullObservationStrategy_throws() {
        assertThatNullPointerException()
            .isThrownBy(() -> new DecisionLoopDescriptor(
                "test", null,
                DecisionScope.FULL_AUTONOMY, CommunicationMode.STIGMERGY,
                AgentBudgetPolicy.defaults(), List.of()));
    }
}
```

- [ ] **Step 9: Implement DecisionLoopDescriptor**

```java
package io.casehub.agency.orchestration.loop;

import io.casehub.api.model.coordination.*;
import java.util.List;
import java.util.Objects;

public record DecisionLoopDescriptor(
    String name,
    ObservationStrategy observationStrategy,
    DecisionScope scope,
    CommunicationMode communicationMode,
    AgentBudgetPolicy budgetPolicy,
    List<String> promptSectionNames
) {
    public DecisionLoopDescriptor {
        Objects.requireNonNull(name, "name must not be null");
        Objects.requireNonNull(observationStrategy, "observationStrategy must not be null");
        Objects.requireNonNull(scope, "scope must not be null");
        Objects.requireNonNull(communicationMode, "communicationMode must not be null");
        Objects.requireNonNull(budgetPolicy, "budgetPolicy must not be null");
        promptSectionNames = promptSectionNames != null
            ? List.copyOf(promptSectionNames) : List.of();
    }
}
```

- [ ] **Step 10: Run all Task 1 tests**

Run: `mvn test -pl agency-core -Dtest="AgentBudgetPolicyTest,DecisionContextTest,ObservationEventTest,DecisionLoopDescriptorTest" -q`
Expected: all tests pass

- [ ] **Step 11: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/220/agency add agency-core/src/main/java/io/casehub/agency/orchestration/loop/ agency-core/src/test/java/io/casehub/agency/orchestration/loop/
git -C /Users/mdproctor/claude/casehub/slots/220/agency commit -m "$(cat <<'EOF'
feat(#379): add foundation types — AgentBudgetPolicy, DecisionContext, ObservationEvent, DecisionLoopDescriptor

Add configuration records, event types, and constants for the
AgentDecisionLoop playbook model.

Refs casehubio/agency#379

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

### Task 2: DecisionOutcome Sealed Interface

Create the unified action output type with all five variants.

**Files:**
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/loop/SignalDeposit.java`
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/loop/ContextWrite.java`
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/loop/DecisionOutcome.java`
- Test: `agency-core/src/test/java/io/casehub/agency/orchestration/loop/DecisionOutcomeTest.java`

**Interfaces:**
- Consumes: `io.casehub.api.spi.observation.Observation` (engine-api), `io.casehub.agency.orchestration.routing.RoutingDecision` (agency-core), `io.casehub.agency.orchestration.aggregation.AggregationResult` (agency-core), `io.casehub.api.model.coordination.DecisionType` (engine-api)
- Produces: `SignalDeposit` (record: name, strength, halfLife), `ContextWrite` (record: key, value), `DecisionOutcome` (sealed: WriteBack, Route, Aggregate, Composite, ScopeGated)

- [ ] **Step 1: Write DecisionOutcome test**

```java
package io.casehub.agency.orchestration.loop;

import com.fasterxml.jackson.databind.node.TextNode;
import io.casehub.api.model.coordination.DecisionType;
import io.casehub.api.spi.observation.Observation;
import io.casehub.agency.orchestration.aggregation.AggregationResult;
import io.casehub.agency.orchestration.routing.RoutingDecision;
import org.junit.jupiter.api.Test;
import java.time.Duration;
import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import static org.assertj.core.api.Assertions.*;

class DecisionOutcomeTest {

    @Test
    void writeBack_validConstruction() {
        var obs = new Observation("p1", 0.9, Map.of(), Instant.now());
        var signal = new SignalDeposit("quality", 0.8, Duration.ofMinutes(5));
        var write = new ContextWrite("key", new TextNode("value"));
        var wb = new DecisionOutcome.WriteBack(List.of(obs), List.of(signal), List.of(write));
        assertThat(wb.observations()).hasSize(1);
        assertThat(wb.signals()).hasSize(1);
        assertThat(wb.contextWrites()).hasSize(1);
    }

    @Test
    void writeBack_nullLists_defaultToEmpty() {
        var wb = new DecisionOutcome.WriteBack(null, null, null);
        assertThat(wb.observations()).isEmpty();
        assertThat(wb.signals()).isEmpty();
        assertThat(wb.contextWrites()).isEmpty();
    }

    @Test
    void writeBack_listsAreImmutable() {
        var mutable = new ArrayList<>(List.of(
            new SignalDeposit("s", 1.0, null)));
        var wb = new DecisionOutcome.WriteBack(List.of(), mutable, List.of());
        mutable.add(new SignalDeposit("s2", 0.5, null));
        assertThat(wb.signals()).hasSize(1);
    }

    @Test
    void route_nullDecision_throws() {
        assertThatNullPointerException()
            .isThrownBy(() -> new DecisionOutcome.Route(null));
    }

    @Test
    void route_wrapsDecision() {
        var decision = new RoutingDecision.Unresolvable("no agents");
        var route = new DecisionOutcome.Route(decision);
        assertThat(route.decision()).isEqualTo(decision);
    }

    @Test
    void aggregate_wrapsResult() {
        var result = new AggregationResult.Resolved("synthesised");
        var agg = new DecisionOutcome.Aggregate(result);
        assertThat(agg.result()).isEqualTo(result);
    }

    @Test
    void composite_immutable() {
        var inner = new DecisionOutcome.WriteBack(List.of(), List.of(), List.of());
        var mutable = new ArrayList<DecisionOutcome>(List.of(inner));
        var composite = new DecisionOutcome.Composite(mutable);
        mutable.add(inner);
        assertThat(composite.outcomes()).hasSize(1);
    }

    @Test
    void scopeGated_carriesAttemptedAndRejected() {
        var rejected = new DecisionOutcome.WriteBack(List.of(), List.of(), List.of());
        var gated = new DecisionOutcome.ScopeGated(DecisionType.PROVISIONING, rejected);
        assertThat(gated.attempted()).isEqualTo(DecisionType.PROVISIONING);
        assertThat(gated.rejected()).isEqualTo(rejected);
    }

    @Test
    void scopeGated_nullAttempted_throws() {
        var rejected = new DecisionOutcome.WriteBack(List.of(), List.of(), List.of());
        assertThatNullPointerException()
            .isThrownBy(() -> new DecisionOutcome.ScopeGated(null, rejected));
    }

    @Test
    void signalDeposit_nullHalfLife_allowed() {
        var signal = new SignalDeposit("name", 0.5, null);
        assertThat(signal.halfLife()).isNull();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl agency-core -Dtest=DecisionOutcomeTest -q`
Expected: compilation failure

- [ ] **Step 3: Implement SignalDeposit, ContextWrite, DecisionOutcome**

`SignalDeposit.java`:
```java
package io.casehub.agency.orchestration.loop;

import org.jspecify.annotations.Nullable;
import java.time.Duration;

public record SignalDeposit(String name, double strength, @Nullable Duration halfLife) {}
```

`ContextWrite.java`:
```java
package io.casehub.agency.orchestration.loop;

import com.fasterxml.jackson.databind.JsonNode;

public record ContextWrite(String key, JsonNode value) {}
```

`DecisionOutcome.java`:
```java
package io.casehub.agency.orchestration.loop;

import io.casehub.api.model.coordination.DecisionType;
import io.casehub.api.spi.observation.Observation;
import io.casehub.agency.orchestration.aggregation.AggregationResult;
import io.casehub.agency.orchestration.routing.RoutingDecision;
import java.util.List;
import java.util.Objects;

public sealed interface DecisionOutcome {

    record WriteBack(
        List<Observation> observations,
        List<SignalDeposit> signals,
        List<ContextWrite> contextWrites
    ) implements DecisionOutcome {
        public WriteBack {
            observations = observations != null ? List.copyOf(observations) : List.of();
            signals = signals != null ? List.copyOf(signals) : List.of();
            contextWrites = contextWrites != null ? List.copyOf(contextWrites) : List.of();
        }
    }

    record Route(RoutingDecision decision) implements DecisionOutcome {
        public Route { Objects.requireNonNull(decision, "decision must not be null"); }
    }

    record Aggregate(AggregationResult result) implements DecisionOutcome {
        public Aggregate { Objects.requireNonNull(result, "result must not be null"); }
    }

    record Composite(List<DecisionOutcome> outcomes) implements DecisionOutcome {
        public Composite { outcomes = List.copyOf(outcomes); }
    }

    record ScopeGated(DecisionType attempted, DecisionOutcome rejected) implements DecisionOutcome {
        public ScopeGated {
            Objects.requireNonNull(attempted, "attempted must not be null");
            Objects.requireNonNull(rejected, "rejected must not be null");
        }
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl agency-core -Dtest=DecisionOutcomeTest -q`
Expected: all tests pass

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/220/agency add agency-core/src/main/java/io/casehub/agency/orchestration/loop/ agency-core/src/test/java/io/casehub/agency/orchestration/loop/
git -C /Users/mdproctor/claude/casehub/slots/220/agency commit -m "$(cat <<'EOF'
feat(#379): add DecisionOutcome sealed interface + WriteBack supporting records

Unified action output bridging write-back (EnvironmentWriter) and
return-value (RoutingDecision/AggregationResult) semantics.

Refs casehubio/agency#379

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Batch 2: Budget Tracking and Prompt Section Infrastructure

After this batch: AgentBudgetTracker handles per-agent + shared cap budget governance. PromptSectionRegistry resolves named sections. SectionAwareContentSummariser wires sections into the LLM invocation pipeline.

### Task 3: AgentBudgetTracker

**Files:**
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/loop/AgentBudgetTracker.java`
- Test: `agency-core/src/test/java/io/casehub/agency/orchestration/loop/AgentBudgetTrackerTest.java`

**Interfaces:**
- Consumes: `AgentBudgetPolicy` (Task 1), `io.casehub.agency.orchestration.swarm.SwarmInvocationTracker` (existing), `io.casehub.agency.orchestration.swarm.SwarmInvocationPolicy` (existing)
- Produces: `AgentBudgetTracker` — `boolean tryAcquire()`, `void recordTokens(long)`, `boolean isBudgetExhausted()`, `int callsUsed()`, `long tokensUsed()`

- [ ] **Step 1: Write AgentBudgetTracker test**

```java
package io.casehub.agency.orchestration.loop;

import io.casehub.agency.orchestration.swarm.SwarmInvocationPolicy;
import io.casehub.agency.orchestration.swarm.SwarmInvocationTracker;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class AgentBudgetTrackerTest {

    @Test
    void tryAcquire_succeedsUpToLimit() {
        var tracker = new AgentBudgetTracker(new AgentBudgetPolicy(3, Long.MAX_VALUE));
        assertThat(tracker.tryAcquire()).isTrue();
        assertThat(tracker.tryAcquire()).isTrue();
        assertThat(tracker.tryAcquire()).isTrue();
        assertThat(tracker.tryAcquire()).isFalse();
        assertThat(tracker.callsUsed()).isEqualTo(3);
    }

    @Test
    void tryAcquire_failedDoesNotIncrementCounter() {
        var tracker = new AgentBudgetTracker(new AgentBudgetPolicy(1, Long.MAX_VALUE));
        assertThat(tracker.tryAcquire()).isTrue();
        assertThat(tracker.tryAcquire()).isFalse();
        assertThat(tracker.callsUsed()).isEqualTo(1);
    }

    @Test
    void recordTokens_accumulates() {
        var tracker = new AgentBudgetTracker(new AgentBudgetPolicy(10, 1000));
        tracker.recordTokens(400);
        tracker.recordTokens(300);
        assertThat(tracker.tokensUsed()).isEqualTo(700);
        assertThat(tracker.isBudgetExhausted()).isFalse();
    }

    @Test
    void isBudgetExhausted_whenTokensExceeded() {
        var tracker = new AgentBudgetTracker(new AgentBudgetPolicy(10, 500));
        tracker.recordTokens(600);
        assertThat(tracker.isBudgetExhausted()).isTrue();
    }

    @Test
    void isBudgetExhausted_whenCallsExhausted() {
        var tracker = new AgentBudgetTracker(new AgentBudgetPolicy(1, Long.MAX_VALUE));
        tracker.tryAcquire();
        assertThat(tracker.isBudgetExhausted()).isTrue();
    }

    @Test
    void sharedCap_failsWhenSharedExhausted() {
        var sharedPolicy = new SwarmInvocationPolicy(1, 0, Long.MAX_VALUE);
        var shared = new SwarmInvocationTracker(sharedPolicy);
        shared.tryAcquire("other");

        var tracker = new AgentBudgetTracker(
            new AgentBudgetPolicy(10, Long.MAX_VALUE), shared);
        assertThat(tracker.tryAcquire()).isFalse();
        assertThat(tracker.callsUsed()).isEqualTo(0);
    }

    @Test
    void sharedCap_propagatesTokens() {
        var sharedPolicy = new SwarmInvocationPolicy(100, 0, Long.MAX_VALUE);
        var shared = new SwarmInvocationTracker(sharedPolicy);

        var tracker = new AgentBudgetTracker(
            new AgentBudgetPolicy(10, Long.MAX_VALUE), shared);
        tracker.recordTokens(250);
        assertThat(shared.tokensUsed()).isEqualTo(250);
    }

    @Test
    void noSharedCap_standaloneOperation() {
        var tracker = new AgentBudgetTracker(new AgentBudgetPolicy(5, 10_000));
        assertThat(tracker.tryAcquire()).isTrue();
        tracker.recordTokens(100);
        assertThat(tracker.callsUsed()).isEqualTo(1);
        assertThat(tracker.tokensUsed()).isEqualTo(100);
        assertThat(tracker.isBudgetExhausted()).isFalse();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl agency-core -Dtest=AgentBudgetTrackerTest -q`
Expected: compilation failure

- [ ] **Step 3: Implement AgentBudgetTracker**

```java
package io.casehub.agency.orchestration.loop;

import io.casehub.agency.orchestration.swarm.SwarmInvocationTracker;
import org.jspecify.annotations.Nullable;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.atomic.AtomicLong;

public class AgentBudgetTracker {

    private final AgentBudgetPolicy agentPolicy;
    private final @Nullable SwarmInvocationTracker sharedCap;
    private final AtomicInteger callCount = new AtomicInteger(0);
    private final AtomicLong tokenCount = new AtomicLong(0);

    public AgentBudgetTracker(AgentBudgetPolicy agentPolicy,
                              @Nullable SwarmInvocationTracker sharedCap) {
        this.agentPolicy = agentPolicy;
        this.sharedCap = sharedCap;
    }

    public AgentBudgetTracker(AgentBudgetPolicy agentPolicy) {
        this(agentPolicy, null);
    }

    public boolean tryAcquire() {
        int acquired = callCount.getAndIncrement();
        if (acquired >= agentPolicy.maxCallsPerAgent()) {
            callCount.decrementAndGet();
            return false;
        }
        if (sharedCap != null && !sharedCap.tryAcquire("agent-loop")) {
            callCount.decrementAndGet();
            return false;
        }
        return true;
    }

    public void recordTokens(long tokens) {
        tokenCount.addAndGet(tokens);
        if (sharedCap != null) {
            sharedCap.recordTokens(tokens);
        }
    }

    public boolean isBudgetExhausted() {
        return callCount.get() >= agentPolicy.maxCallsPerAgent()
            || tokenCount.get() >= agentPolicy.maxTokensPerAgent()
            || (sharedCap != null && sharedCap.isTokenBudgetExhausted());
    }

    public int callsUsed() { return callCount.get(); }
    public long tokensUsed() { return tokenCount.get(); }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl agency-core -Dtest=AgentBudgetTrackerTest -q`
Expected: all tests pass

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/220/agency add agency-core/src/main/java/io/casehub/agency/orchestration/loop/AgentBudgetTracker.java agency-core/src/test/java/io/casehub/agency/orchestration/loop/AgentBudgetTrackerTest.java
git -C /Users/mdproctor/claude/casehub/slots/220/agency commit -m "$(cat <<'EOF'
feat(#379): add AgentBudgetTracker — per-agent budget with shared cap

Composite tracker: per-agent call/token limits with optional delegation
to a shared SwarmInvocationTracker for global governance.

Refs casehubio/agency#379

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

### Task 4: PromptSectionRegistry + SectionAwareContentSummariser

**Files:**
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/loop/PromptSectionRegistry.java`
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/loop/MapPromptSectionRegistry.java`
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/loop/SectionAwareContentSummariser.java`
- Test: `agency-core/src/test/java/io/casehub/agency/orchestration/loop/MapPromptSectionRegistryTest.java`
- Test: `agency-core/src/test/java/io/casehub/agency/orchestration/loop/SectionAwareContentSummariserTest.java`

**Interfaces:**
- Consumes: `LoopPromptSection` (Task 1), `DecisionContext` (Task 1), `DecisionOutcome` (Task 2), `ObservationEvent` (Task 1), `io.casehub.agency.summarisation.ContentSummariser` (summarisation-api)
- Produces: `PromptSectionRegistry` (interface: resolve(name), resolveAll(names)), `MapPromptSectionRegistry` (impl: Map-backed), `SectionAwareContentSummariser` (decorator: prepends section contributions to delegate)

- [ ] **Step 1: Write MapPromptSectionRegistry test**

```java
package io.casehub.agency.orchestration.loop;

import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import static org.assertj.core.api.Assertions.*;

class MapPromptSectionRegistryTest {

    @Test
    void resolve_knownName() {
        LoopPromptSection section = ctx -> "hello";
        var registry = new MapPromptSectionRegistry(Map.of("greeting", section));
        assertThat(registry.resolve("greeting")).isSameAs(section);
    }

    @Test
    void resolve_unknownName_throws() {
        var registry = new MapPromptSectionRegistry(Map.of());
        assertThatIllegalArgumentException()
            .isThrownBy(() -> registry.resolve("missing"))
            .withMessageContaining("Unknown prompt section: missing");
    }

    @Test
    void resolveAll_preservesOrder() {
        LoopPromptSection a = ctx -> "a";
        LoopPromptSection b = ctx -> "b";
        var registry = new MapPromptSectionRegistry(Map.of("a", a, "b", b));
        var resolved = registry.resolveAll(List.of("b", "a"));
        assertThat(resolved).containsExactly(b, a);
    }

    @Test
    void resolve_unknownName_showsRegistered() {
        LoopPromptSection s = ctx -> "x";
        var registry = new MapPromptSectionRegistry(Map.of("known", s));
        assertThatIllegalArgumentException()
            .isThrownBy(() -> registry.resolve("unknown"))
            .withMessageContaining("known");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl agency-core -Dtest=MapPromptSectionRegistryTest -q`
Expected: compilation failure

- [ ] **Step 3: Implement PromptSectionRegistry and MapPromptSectionRegistry**

`PromptSectionRegistry.java`:
```java
package io.casehub.agency.orchestration.loop;

import java.util.List;

public interface PromptSectionRegistry {

    LoopPromptSection resolve(String name);

    default List<LoopPromptSection> resolveAll(List<String> names) {
        return names.stream().map(this::resolve).toList();
    }
}
```

`MapPromptSectionRegistry.java`:
```java
package io.casehub.agency.orchestration.loop;

import java.util.Map;

public class MapPromptSectionRegistry implements PromptSectionRegistry {

    private final Map<String, LoopPromptSection> sections;

    public MapPromptSectionRegistry(Map<String, LoopPromptSection> sections) {
        this.sections = Map.copyOf(sections);
    }

    @Override
    public LoopPromptSection resolve(String name) {
        var section = sections.get(name);
        if (section == null) {
            throw new IllegalArgumentException(
                "Unknown prompt section: " + name
                + " — registered: " + sections.keySet());
        }
        return section;
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl agency-core -Dtest=MapPromptSectionRegistryTest -q`
Expected: all tests pass

- [ ] **Step 5: Write SectionAwareContentSummariser test**

```java
package io.casehub.agency.orchestration.loop;

import io.casehub.agency.summarisation.ContentSummariser;
import io.casehub.api.model.coordination.*;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.atomic.AtomicReference;
import static org.assertj.core.api.Assertions.*;

class SectionAwareContentSummariserTest {

    static final DecisionContext CTX = new DecisionContext(
        UUID.randomUUID(), "agent-1", DecisionScope.FULL_AUTONOMY,
        CommunicationMode.STIGMERGY,
        new TopologyDescriptor(TopologyType.CENTRALISED, Map.of()));

    @Test
    void delegatesToWrappedSummariser() {
        var outcome = new DecisionOutcome.WriteBack(List.of(), List.of(), List.of());
        ContentSummariser<ObservationEvent, DecisionOutcome> delegate =
            (items, prev) -> CompletableFuture.completedFuture(outcome);

        var summariser = new SectionAwareContentSummariser(
            delegate, List.of(), CTX);

        var events = List.of(ObservationEvent.of("key", "value"));
        var result = summariser.summarise(events, null)
            .toCompletableFuture().join();
        assertThat(result).isSameAs(outcome);
    }

    @Test
    void callsSectionsWithContext() {
        var captured = new AtomicReference<DecisionContext>();
        LoopPromptSection section = ctx -> {
            captured.set(ctx);
            return "section-output";
        };

        ContentSummariser<ObservationEvent, DecisionOutcome> delegate =
            (items, prev) -> CompletableFuture.completedFuture(
                new DecisionOutcome.WriteBack(List.of(), List.of(), List.of()));

        var summariser = new SectionAwareContentSummariser(
            delegate, List.of(section), CTX);

        summariser.summarise(List.of(ObservationEvent.of("k", "v")), null)
            .toCompletableFuture().join();
        assertThat(captured.get()).isSameAs(CTX);
    }

    @Test
    void skipsEmptySectionContributions() {
        LoopPromptSection empty = ctx -> "";
        LoopPromptSection nullSection = ctx -> null;
        LoopPromptSection real = ctx -> "content";

        ContentSummariser<ObservationEvent, DecisionOutcome> delegate =
            (items, prev) -> CompletableFuture.completedFuture(
                new DecisionOutcome.WriteBack(List.of(), List.of(), List.of()));

        var summariser = new SectionAwareContentSummariser(
            delegate, List.of(empty, nullSection, real), CTX);

        var result = summariser.summarise(
            List.of(ObservationEvent.of("k", "v")), null)
            .toCompletableFuture().join();
        assertThat(result).isNotNull();
    }
}
```

- [ ] **Step 6: Implement SectionAwareContentSummariser**

```java
package io.casehub.agency.orchestration.loop;

import io.casehub.agency.summarisation.ContentSummariser;
import org.jspecify.annotations.Nullable;
import java.util.List;
import java.util.concurrent.CompletionStage;

class SectionAwareContentSummariser
        implements ContentSummariser<ObservationEvent, DecisionOutcome> {

    private final ContentSummariser<ObservationEvent, DecisionOutcome> delegate;
    private final List<LoopPromptSection> sections;
    private final DecisionContext context;

    SectionAwareContentSummariser(
            ContentSummariser<ObservationEvent, DecisionOutcome> delegate,
            List<LoopPromptSection> sections,
            DecisionContext context) {
        this.delegate = delegate;
        this.sections = List.copyOf(sections);
        this.context = context;
    }

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
        return delegate.summarise(items, previous);
    }

    String assembledPromptContext() {
        var sb = new StringBuilder();
        for (var section : sections) {
            var contribution = section.contribute(context);
            if (contribution != null && !contribution.isEmpty()) {
                sb.append(contribution).append("\n");
            }
        }
        return sb.toString();
    }
}
```

- [ ] **Step 7: Run tests and commit**

Run: `mvn test -pl agency-core -Dtest="MapPromptSectionRegistryTest,SectionAwareContentSummariserTest" -q`
Expected: all tests pass

```bash
git -C /Users/mdproctor/claude/casehub/slots/220/agency add agency-core/src/main/java/io/casehub/agency/orchestration/loop/ agency-core/src/test/java/io/casehub/agency/orchestration/loop/
git -C /Users/mdproctor/claude/casehub/slots/220/agency commit -m "$(cat <<'EOF'
feat(#379): add PromptSectionRegistry + SectionAwareContentSummariser

Named prompt section resolution and decorator that wires sections into
the ContentSummariser pipeline.

Refs casehubio/agency#379

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Batch 3: Factory, Runtime, and Integration

After this batch: full pipeline working end-to-end. DecisionLoopFactory assembles runtimes from descriptors. AgentDecisionLoop drives the observe→reason→decide cycle.

### Task 5: DecisionLoopFactory + AgentDecisionLoop + Integration Tests

**Files:**
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/loop/DecisionLoopFactory.java`
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/loop/AgentDecisionLoop.java`
- Test: `agency-core/src/test/java/io/casehub/agency/orchestration/loop/DecisionLoopFactoryTest.java`
- Test: `agency-core/src/test/java/io/casehub/agency/orchestration/loop/AgentDecisionLoopTest.java`

**Interfaces:**
- Consumes: All types from Tasks 1-4. `io.casehub.agency.summarisation.SummarisationRunner` (summarisation-api), `io.casehub.agency.summarisation.EventStreamBus` (summarisation-api), `io.casehub.agency.summarisation.EmissionPolicy` (summarisation-api), `io.casehub.agency.summarisation.WindowPolicy` (summarisation-api), `io.casehub.agency.stigmergy.llm.InFlightGate` (agency-core), `io.casehub.agency.summarisation.ContentSummariser` (summarisation-api)
- Produces: `DecisionLoopFactory` (create method), `AgentDecisionLoop` (collect, tick, flush, close)

- [ ] **Step 1: Write DecisionLoopFactory test**

```java
package io.casehub.agency.orchestration.loop;

import io.casehub.agency.summarisation.ContentSummariser;
import io.casehub.agency.summarisation.EventStreamBus;
import io.casehub.api.model.coordination.*;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.CompletableFuture;
import static org.assertj.core.api.Assertions.*;

class DecisionLoopFactoryTest {

    static final DecisionContext CTX = new DecisionContext(
        UUID.randomUUID(), "agent-1", DecisionScope.FULL_AUTONOMY,
        CommunicationMode.STIGMERGY,
        new TopologyDescriptor(TopologyType.CENTRALISED, Map.of()));

    static final ContentSummariser<ObservationEvent, DecisionOutcome> NOOP_REASONER =
        (items, prev) -> CompletableFuture.completedFuture(
            new DecisionOutcome.WriteBack(List.of(), List.of(), List.of()));

    @Test
    void create_eventDriven() {
        var registry = new MapPromptSectionRegistry(Map.of());
        var bus = new EventStreamBus<DecisionOutcome>();
        var factory = new DecisionLoopFactory(registry, bus);
        var descriptor = new DecisionLoopDescriptor(
            "test", ObservationStrategy.EVENT_DRIVEN,
            DecisionScope.FULL_AUTONOMY, CommunicationMode.STIGMERGY,
            AgentBudgetPolicy.defaults(), List.of());

        var loop = factory.create(descriptor, CTX, NOOP_REASONER, null);
        assertThat(loop).isNotNull();
        assertThat(loop.descriptor()).isEqualTo(descriptor);
        assertThat(loop.context()).isEqualTo(CTX);
    }

    @Test
    void create_tickDriven() {
        var registry = new MapPromptSectionRegistry(Map.of());
        var bus = new EventStreamBus<DecisionOutcome>();
        var factory = new DecisionLoopFactory(registry, bus);
        var descriptor = new DecisionLoopDescriptor(
            "test", ObservationStrategy.TICK_DRIVEN,
            DecisionScope.FULL_AUTONOMY, CommunicationMode.STIGMERGY,
            AgentBudgetPolicy.defaults(), List.of());

        var loop = factory.create(descriptor, CTX, NOOP_REASONER, null);
        assertThat(loop).isNotNull();
    }

    @Test
    void create_demandDriven() {
        var registry = new MapPromptSectionRegistry(Map.of());
        var bus = new EventStreamBus<DecisionOutcome>();
        var factory = new DecisionLoopFactory(registry, bus);
        var descriptor = new DecisionLoopDescriptor(
            "test", ObservationStrategy.DEMAND_DRIVEN,
            DecisionScope.FULL_AUTONOMY, CommunicationMode.STIGMERGY,
            AgentBudgetPolicy.defaults(), List.of());

        var loop = factory.create(descriptor, CTX, NOOP_REASONER, null);
        assertThat(loop).isNotNull();
    }

    @Test
    void create_resolvesPromptSections() {
        LoopPromptSection section = ctx -> "section-output";
        var registry = new MapPromptSectionRegistry(Map.of("test-section", section));
        var bus = new EventStreamBus<DecisionOutcome>();
        var factory = new DecisionLoopFactory(registry, bus);
        var descriptor = new DecisionLoopDescriptor(
            "test", ObservationStrategy.EVENT_DRIVEN,
            DecisionScope.FULL_AUTONOMY, CommunicationMode.STIGMERGY,
            AgentBudgetPolicy.defaults(), List.of("test-section"));

        var loop = factory.create(descriptor, CTX, NOOP_REASONER, null);
        assertThat(loop).isNotNull();
    }

    @Test
    void create_failsFastOnUnknownSection() {
        var registry = new MapPromptSectionRegistry(Map.of());
        var bus = new EventStreamBus<DecisionOutcome>();
        var factory = new DecisionLoopFactory(registry, bus);
        var descriptor = new DecisionLoopDescriptor(
            "test", ObservationStrategy.EVENT_DRIVEN,
            DecisionScope.FULL_AUTONOMY, CommunicationMode.STIGMERGY,
            AgentBudgetPolicy.defaults(), List.of("nonexistent"));

        assertThatIllegalArgumentException()
            .isThrownBy(() -> factory.create(descriptor, CTX, NOOP_REASONER, null))
            .withMessageContaining("nonexistent");
    }

    @Test
    void create_wiresBudgetGate() {
        var registry = new MapPromptSectionRegistry(Map.of());
        var bus = new EventStreamBus<DecisionOutcome>();
        var factory = new DecisionLoopFactory(registry, bus);
        var descriptor = new DecisionLoopDescriptor(
            "test", ObservationStrategy.EVENT_DRIVEN,
            DecisionScope.FULL_AUTONOMY, CommunicationMode.STIGMERGY,
            new AgentBudgetPolicy(2, Long.MAX_VALUE), List.of());

        var loop = factory.create(descriptor, CTX, NOOP_REASONER, null);
        assertThat(loop.budget().callsUsed()).isEqualTo(0);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl agency-core -Dtest=DecisionLoopFactoryTest -q`
Expected: compilation failure

- [ ] **Step 3: Implement DecisionLoopFactory**

```java
package io.casehub.agency.orchestration.loop;

import io.casehub.agency.stigmergy.llm.InFlightGate;
import io.casehub.agency.summarisation.*;
import io.casehub.agency.orchestration.swarm.SwarmInvocationTracker;
import io.casehub.api.model.coordination.ObservationStrategy;
import org.jspecify.annotations.Nullable;
import java.time.Duration;
import java.util.concurrent.CompletableFuture;

public class DecisionLoopFactory {

    private final PromptSectionRegistry sectionRegistry;
    private final EventStreamBus<DecisionOutcome> outputBus;

    public DecisionLoopFactory(PromptSectionRegistry sectionRegistry,
                               EventStreamBus<DecisionOutcome> outputBus) {
        this.sectionRegistry = sectionRegistry;
        this.outputBus = outputBus;
    }

    public AgentDecisionLoop create(
            DecisionLoopDescriptor descriptor,
            DecisionContext context,
            ContentSummariser<ObservationEvent, DecisionOutcome> reasoner,
            @Nullable SwarmInvocationTracker sharedBudget) {

        var sections = sectionRegistry.resolveAll(descriptor.promptSectionNames());
        var budget = new AgentBudgetTracker(descriptor.budgetPolicy(), sharedBudget);

        ContentSummariser<ObservationEvent, DecisionOutcome> withSections =
            new SectionAwareContentSummariser(reasoner, sections, context);

        ContentSummariser<ObservationEvent, DecisionOutcome> budgetAware =
            (events, prev) -> {
                if (!budget.tryAcquire()) {
                    return CompletableFuture.completedFuture(prev);
                }
                return withSections.summarise(events, prev);
            };

        var gate = new InFlightGate<>(budgetAware);

        EmissionPolicy<ObservationEvent, ?> emissionPolicy = buildEmissionPolicy(
            descriptor.observationStrategy());

        var runner = SummarisationRunner.<ObservationEvent, DecisionOutcome>builder(
                gate.summariser().asSummariser(),
                outputBus,
                DecisionLoopLevels.DECISION)
            .emissionPolicy(gate.gating(emissionPolicy))
            .build();

        return new AgentDecisionLoop(descriptor, context, runner, budget);
    }

    private EmissionPolicy<ObservationEvent, ?> buildEmissionPolicy(
            ObservationStrategy strategy) {
        return switch (strategy) {
            case EVENT_DRIVEN -> EmissionPolicy.window(WindowPolicy.ofCount(1));
            case TICK_DRIVEN -> EmissionPolicy.window(
                WindowPolicy.ofDuration(Duration.ofSeconds(5)));
            case DEMAND_DRIVEN -> (buffered, state, now) -> false;
        };
    }
}
```

- [ ] **Step 4: Implement AgentDecisionLoop**

```java
package io.casehub.agency.orchestration.loop;

import io.casehub.agency.summarisation.LevelEvent;
import io.casehub.agency.summarisation.SummarisationRunner;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.CompletionStage;

public class AgentDecisionLoop {

    private static final System.Logger LOG =
        System.getLogger(AgentDecisionLoop.class.getName());

    private final DecisionLoopDescriptor descriptor;
    private final DecisionContext context;
    private final SummarisationRunner<ObservationEvent, DecisionOutcome> runner;
    private final AgentBudgetTracker budget;

    AgentDecisionLoop(DecisionLoopDescriptor descriptor,
                      DecisionContext context,
                      SummarisationRunner<ObservationEvent, DecisionOutcome> runner,
                      AgentBudgetTracker budget) {
        this.descriptor = descriptor;
        this.context = context;
        this.runner = runner;
        this.budget = budget;
    }

    public void collect(ObservationEvent event) {
        runner.collect(new LevelEvent<>(event, event.timestamp().toEpochMilli(),
            DecisionLoopLevels.OBSERVATION, null));
    }

    public CompletionStage<Void> tick(long now) {
        if (budget.isBudgetExhausted()) {
            LOG.log(System.Logger.Level.DEBUG,
                "Budget exhausted for {0} — skipping tick", context.agentId());
            return CompletableFuture.completedFuture(null);
        }
        return runner.tick(now);
    }

    public CompletionStage<Void> flush() {
        if (budget.isBudgetExhausted()) {
            return CompletableFuture.failedFuture(
                new BudgetExhaustedException(context.agentId()));
        }
        return runner.flush();
    }

    public void close() {
        runner.clear();
    }

    public DecisionLoopDescriptor descriptor() { return descriptor; }
    public DecisionContext context() { return context; }
    public AgentBudgetTracker budget() { return budget; }
}
```

- [ ] **Step 5: Run factory tests**

Run: `mvn test -pl agency-core -Dtest=DecisionLoopFactoryTest -q`
Expected: all tests pass

- [ ] **Step 6: Write AgentDecisionLoop integration test**

```java
package io.casehub.agency.orchestration.loop;

import io.casehub.agency.summarisation.ContentSummariser;
import io.casehub.agency.summarisation.EventStreamBus;
import io.casehub.api.model.coordination.*;
import org.junit.jupiter.api.Test;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.CompletableFuture;
import static org.assertj.core.api.Assertions.*;

class AgentDecisionLoopTest {

    static final DecisionContext CTX = new DecisionContext(
        UUID.randomUUID(), "agent-1", DecisionScope.FULL_AUTONOMY,
        CommunicationMode.STIGMERGY,
        new TopologyDescriptor(TopologyType.CENTRALISED, Map.of()));

    @Test
    void eventDriven_collectAndTick_producesOutcome() {
        var outcome = new DecisionOutcome.WriteBack(List.of(), List.of(), List.of());
        ContentSummariser<ObservationEvent, DecisionOutcome> reasoner =
            (items, prev) -> CompletableFuture.completedFuture(outcome);

        var bus = new EventStreamBus<DecisionOutcome>();
        var captured = new ArrayList<DecisionOutcome>();
        bus.subscribe(e -> true, e -> captured.add(e.payload()));

        var factory = new DecisionLoopFactory(
            new MapPromptSectionRegistry(Map.of()), bus);
        var descriptor = new DecisionLoopDescriptor(
            "test", ObservationStrategy.EVENT_DRIVEN,
            DecisionScope.FULL_AUTONOMY, CommunicationMode.STIGMERGY,
            AgentBudgetPolicy.defaults(), List.of());

        var loop = factory.create(descriptor, CTX, reasoner, null);
        loop.collect(ObservationEvent.of("signal", 0.9));
        loop.tick(System.currentTimeMillis()).toCompletableFuture().join();

        assertThat(captured).hasSize(1);
        assertThat(captured.get(0)).isSameAs(outcome);
    }

    @Test
    void demandDriven_tickDoesNotEmit_flushDoes() {
        var outcome = new DecisionOutcome.WriteBack(List.of(), List.of(), List.of());
        ContentSummariser<ObservationEvent, DecisionOutcome> reasoner =
            (items, prev) -> CompletableFuture.completedFuture(outcome);

        var bus = new EventStreamBus<DecisionOutcome>();
        var captured = new ArrayList<DecisionOutcome>();
        bus.subscribe(e -> true, e -> captured.add(e.payload()));

        var factory = new DecisionLoopFactory(
            new MapPromptSectionRegistry(Map.of()), bus);
        var descriptor = new DecisionLoopDescriptor(
            "test", ObservationStrategy.DEMAND_DRIVEN,
            DecisionScope.FULL_AUTONOMY, CommunicationMode.STIGMERGY,
            AgentBudgetPolicy.defaults(), List.of());

        var loop = factory.create(descriptor, CTX, reasoner, null);
        loop.collect(ObservationEvent.of("signal", 0.9));
        loop.tick(System.currentTimeMillis()).toCompletableFuture().join();
        assertThat(captured).isEmpty();

        loop.flush().toCompletableFuture().join();
        assertThat(captured).hasSize(1);
    }

    @Test
    void budgetExhausted_tickIsNoop() {
        var callCount = new java.util.concurrent.atomic.AtomicInteger(0);
        ContentSummariser<ObservationEvent, DecisionOutcome> reasoner =
            (items, prev) -> {
                callCount.incrementAndGet();
                return CompletableFuture.completedFuture(
                    new DecisionOutcome.WriteBack(List.of(), List.of(), List.of()));
            };

        var bus = new EventStreamBus<DecisionOutcome>();
        var factory = new DecisionLoopFactory(
            new MapPromptSectionRegistry(Map.of()), bus);
        var descriptor = new DecisionLoopDescriptor(
            "test", ObservationStrategy.EVENT_DRIVEN,
            DecisionScope.FULL_AUTONOMY, CommunicationMode.STIGMERGY,
            new AgentBudgetPolicy(1, Long.MAX_VALUE), List.of());

        var loop = factory.create(descriptor, CTX, reasoner, null);

        loop.collect(ObservationEvent.of("s1", 1));
        loop.tick(System.currentTimeMillis()).toCompletableFuture().join();
        assertThat(callCount.get()).isEqualTo(1);

        loop.collect(ObservationEvent.of("s2", 2));
        loop.tick(System.currentTimeMillis()).toCompletableFuture().join();
        assertThat(callCount.get()).isEqualTo(1);
    }

    @Test
    void budgetExhausted_flushReturnsFailed() {
        ContentSummariser<ObservationEvent, DecisionOutcome> reasoner =
            (items, prev) -> CompletableFuture.completedFuture(
                new DecisionOutcome.WriteBack(List.of(), List.of(), List.of()));

        var bus = new EventStreamBus<DecisionOutcome>();
        var factory = new DecisionLoopFactory(
            new MapPromptSectionRegistry(Map.of()), bus);
        var descriptor = new DecisionLoopDescriptor(
            "test", ObservationStrategy.DEMAND_DRIVEN,
            DecisionScope.FULL_AUTONOMY, CommunicationMode.STIGMERGY,
            new AgentBudgetPolicy(0, 0), List.of());

        var loop = factory.create(descriptor, CTX, reasoner, null);
        loop.collect(ObservationEvent.of("s", 1));

        assertThatExceptionOfType(java.util.concurrent.CompletionException.class)
            .isThrownBy(() -> loop.flush().toCompletableFuture().join())
            .withCauseInstanceOf(BudgetExhaustedException.class);
    }

    @Test
    void close_clearsAccumulatedEvents() {
        var callCount = new java.util.concurrent.atomic.AtomicInteger(0);
        ContentSummariser<ObservationEvent, DecisionOutcome> reasoner =
            (items, prev) -> {
                callCount.incrementAndGet();
                return CompletableFuture.completedFuture(
                    new DecisionOutcome.WriteBack(List.of(), List.of(), List.of()));
            };

        var bus = new EventStreamBus<DecisionOutcome>();
        var factory = new DecisionLoopFactory(
            new MapPromptSectionRegistry(Map.of()), bus);
        var descriptor = new DecisionLoopDescriptor(
            "test", ObservationStrategy.DEMAND_DRIVEN,
            DecisionScope.FULL_AUTONOMY, CommunicationMode.STIGMERGY,
            AgentBudgetPolicy.defaults(), List.of());

        var loop = factory.create(descriptor, CTX, reasoner, null);
        loop.collect(ObservationEvent.of("s", 1));
        loop.close();
        loop.flush().toCompletableFuture().join();
        assertThat(callCount.get()).isEqualTo(0);
    }

    @Test
    void multipleObservations_singleSummarisationCall() {
        var receivedCounts = new ArrayList<Integer>();
        ContentSummariser<ObservationEvent, DecisionOutcome> reasoner =
            (items, prev) -> {
                receivedCounts.add(items.size());
                return CompletableFuture.completedFuture(
                    new DecisionOutcome.WriteBack(List.of(), List.of(), List.of()));
            };

        var bus = new EventStreamBus<DecisionOutcome>();
        var factory = new DecisionLoopFactory(
            new MapPromptSectionRegistry(Map.of()), bus);
        var descriptor = new DecisionLoopDescriptor(
            "test", ObservationStrategy.DEMAND_DRIVEN,
            DecisionScope.FULL_AUTONOMY, CommunicationMode.STIGMERGY,
            AgentBudgetPolicy.defaults(), List.of());

        var loop = factory.create(descriptor, CTX, reasoner, null);
        loop.collect(ObservationEvent.of("s1", 1));
        loop.collect(ObservationEvent.of("s2", 2));
        loop.collect(ObservationEvent.of("s3", 3));
        loop.flush().toCompletableFuture().join();

        assertThat(receivedCounts).containsExactly(3);
    }
}
```

- [ ] **Step 7: Run all tests**

Run: `mvn test -pl agency-core -Dtest="DecisionLoopFactoryTest,AgentDecisionLoopTest" -q`
Expected: all tests pass

- [ ] **Step 8: Run full test suite to check for regressions**

Run: `mvn test -pl agency-core -q`
Expected: all existing tests still pass

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/220/agency add agency-core/src/main/java/io/casehub/agency/orchestration/loop/ agency-core/src/test/java/io/casehub/agency/orchestration/loop/
git -C /Users/mdproctor/claude/casehub/slots/220/agency commit -m "$(cat <<'EOF'
feat(#379): add DecisionLoopFactory + AgentDecisionLoop — full pipeline

Factory assembles the runtime from a descriptor: resolves prompt
sections, wires budget gate, builds emission policy from
ObservationStrategy, and composes SummarisationRunner.

Refs casehubio/agency#379

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## References

- `2026-11-02-agent-decision-loop-design.md` — design spec this plan implements
- `ContentSummariser.java` (summarisation-api) — LLM invocation primitive
- `SummarisationRunner.java` (summarisation-api) — tick-driven pipeline
- `EmissionPolicy.java` (summarisation-api) — emission control
- `InFlightGate.java` (agency-core/stigmergy/llm) — concurrency guard
- `SwarmInvocationTracker.java` (agency-core/swarm) — shared budget tracking
- `SwarmInvocationPolicy.java` (agency-core/swarm) — budget policy record
- `DecisionScope.java` (engine-api) — scope enforcement enum
- `ObservationStrategy.java` (engine-api) — trigger vocabulary enum
- `TopologyDescriptor.java` (engine-api) — topology snapshot record
- `RoutingDecision.java` (agency-core/routing) — return-value action type
- `AggregationResult.java` (agency-core/aggregation) — return-value action type
- `Observation.java` (engine-api/spi/observation) — engine observation record
- `ContentSummariserTest.java` (summarisation-api) — test patterns
- casehubio/agency#379 — focal issue
- casehubio/agency#378 — parent sub-epic
