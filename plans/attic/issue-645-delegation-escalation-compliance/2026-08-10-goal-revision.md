# Goal Revision Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #806 — Goal revision — modify goal parameters based on outcomes
**Issue group:** #803, #806, #805, #808

**Goal:** Enable goal revision — agents accumulate outcome signals per goal,
and when thresholds are met, priority is adjusted via GoalEvolution (eidos)
and descriptions refined via LLM (engine GoalRevisionStrategy).

**Architecture:** GoalOutcomeRecorder records SUCCESS/FAILURE per goal via
GoalSignalStore (eidos). GoalRevisionEvaluator accumulates per-agent state
and triggers evaluation via GoalEvolution + GoalRevisionStrategy when
thresholds are met. Changes applied via AgentRegistry.register().

**Tech Stack:** Java 21, Quarkus 3.32, eidos-api (GoalSignalStore,
GoalEvolution, GoalOutcomeCounts), Mutiny Uni, virtual threads

## Global Constraints

- Pre-release platform — breaking changes are free
- IntelliJ MCP required for all code navigation and editing
- TDD: failing test first, then implementation
- Every commit references an issue: `Refs #806`
- GoalSignalStore / GoalEvolution / GoalOutcomeCounts are existing eidos-api
  types — use them directly, do not create engine-side equivalents
- `@DefaultBean @ApplicationScoped` for no-op defaults (eidos CDI pattern)
- `Instance<>` injection with `isResolvable()` guard for optional dependencies

---

### Task 1: eidos-api — GoalSignalStore + GoalEvolution implementations

Create in-memory default implementations of the existing eidos-api SPIs.
These exist as interfaces with no implementations. Also add
`AgentGoal.toBuilder()`.

**Files:**
- Create: `eidos/api/src/main/java/io/casehub/eidos/api/InMemoryGoalSignalStore.java`
- Create: `eidos/api/src/main/java/io/casehub/eidos/api/DefaultGoalEvolution.java`
- Modify: `eidos/api/src/main/java/io/casehub/eidos/api/AgentGoal.java` (add toBuilder + Builder)
- Test: `eidos/api/src/test/java/io/casehub/eidos/api/InMemoryGoalSignalStoreTest.java`
- Test: `eidos/api/src/test/java/io/casehub/eidos/api/DefaultGoalEvolutionTest.java`
- Test: `eidos/api/src/test/java/io/casehub/eidos/api/AgentGoalBuilderTest.java`

**Interfaces:**
- Consumes: `GoalSignalStore`, `GoalEvolution`, `GoalOutcome`, `GoalOutcomeCounts`, `GoalEvolutionResult`, `AgentGoal`, `AgentDescriptor`, `GoalPriority` (all existing eidos-api types)
- Produces: `InMemoryGoalSignalStore` (used by engine tests), `DefaultGoalEvolution` (used by engine at runtime), `AgentGoal.toBuilder()` (used by GoalRevisionEvaluator for description merging)

**Cross-repo:** This task runs in `/Users/mdproctor/claude/casehub/eidos`.
After completion, run `mvn install -DskipTests -q` to install the SNAPSHOT
so engine can resolve the dependency.

- [ ] **Step 1: Write InMemoryGoalSignalStore test**

```java
package io.casehub.eidos.api;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.junit.jupiter.api.Assertions.*;

class InMemoryGoalSignalStoreTest {

    private InMemoryGoalSignalStore store;

    @BeforeEach
    void setUp() {
        store = new InMemoryGoalSignalStore();
    }

    @Test
    void recordOutcome_incrementsSuccessCount() {
        store.recordOutcome("agent-1", "tenant-1", "goal-a", GoalOutcome.SUCCESS);
        store.recordOutcome("agent-1", "tenant-1", "goal-a", GoalOutcome.SUCCESS);
        Map<String, GoalOutcomeCounts> counts = store.outcomeCounts("agent-1", "tenant-1");
        assertEquals(2, counts.get("goal-a").successCount());
        assertEquals(0, counts.get("goal-a").failureCount());
    }

    @Test
    void recordOutcome_incrementsFailureCount() {
        store.recordOutcome("agent-1", "tenant-1", "goal-a", GoalOutcome.FAILURE);
        Map<String, GoalOutcomeCounts> counts = store.outcomeCounts("agent-1", "tenant-1");
        assertEquals(0, counts.get("goal-a").successCount());
        assertEquals(1, counts.get("goal-a").failureCount());
    }

    @Test
    void outcomeCounts_emptyWhenNoRecords() {
        Map<String, GoalOutcomeCounts> counts = store.outcomeCounts("agent-1", "tenant-1");
        assertTrue(counts.isEmpty());
    }

    @Test
    void outcomeCounts_isolatedByAgentAndTenancy() {
        store.recordOutcome("agent-1", "tenant-1", "goal-a", GoalOutcome.SUCCESS);
        store.recordOutcome("agent-2", "tenant-1", "goal-a", GoalOutcome.FAILURE);
        assertEquals(1, store.outcomeCounts("agent-1", "tenant-1").get("goal-a").successCount());
        assertEquals(1, store.outcomeCounts("agent-2", "tenant-1").get("goal-a").failureCount());
    }

    @Test
    void clear_resetsAllCounts() {
        store.recordOutcome("agent-1", "tenant-1", "goal-a", GoalOutcome.SUCCESS);
        store.recordOutcome("agent-1", "tenant-1", "goal-b", GoalOutcome.FAILURE);
        store.clear("agent-1", "tenant-1");
        assertTrue(store.outcomeCounts("agent-1", "tenant-1").isEmpty());
    }

    @Test
    void decay_reducesCounts() {
        for (int i = 0; i < 10; i++) {
            store.recordOutcome("agent-1", "tenant-1", "goal-a", GoalOutcome.SUCCESS);
        }
        store.decay("agent-1", "tenant-1", 0.5);
        Map<String, GoalOutcomeCounts> counts = store.outcomeCounts("agent-1", "tenant-1");
        assertEquals(5, counts.get("goal-a").successCount());
    }

    @Test
    void multipleGoals_trackedIndependently() {
        store.recordOutcome("agent-1", "tenant-1", "goal-a", GoalOutcome.SUCCESS);
        store.recordOutcome("agent-1", "tenant-1", "goal-b", GoalOutcome.FAILURE);
        Map<String, GoalOutcomeCounts> counts = store.outcomeCounts("agent-1", "tenant-1");
        assertEquals(1, counts.get("goal-a").successCount());
        assertEquals(1, counts.get("goal-b").failureCount());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl api -Dtest=InMemoryGoalSignalStoreTest -f /Users/mdproctor/claude/casehub/eidos/pom.xml`
Expected: FAIL — `InMemoryGoalSignalStore` does not exist

- [ ] **Step 3: Implement InMemoryGoalSignalStore**

```java
package io.casehub.eidos.api;

import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.stream.Collectors;

@DefaultBean
@ApplicationScoped
public class InMemoryGoalSignalStore implements GoalSignalStore {

    private final ConcurrentHashMap<String, ConcurrentHashMap<String, int[]>> store =
        new ConcurrentHashMap<>();

    @Override
    public void recordOutcome(String agentId, String tenancyId, String goalName,
                              GoalOutcome outcome) {
        String key = agentId + "|" + tenancyId;
        store.computeIfAbsent(key, k -> new ConcurrentHashMap<>())
             .compute(goalName, (g, counts) -> {
                 if (counts == null) counts = new int[]{0, 0};
                 if (outcome == GoalOutcome.SUCCESS) counts[0]++;
                 else counts[1]++;
                 return counts;
             });
    }

    @Override
    public Map<String, GoalOutcomeCounts> outcomeCounts(String agentId, String tenancyId) {
        String key = agentId + "|" + tenancyId;
        ConcurrentHashMap<String, int[]> goalCounts = store.get(key);
        if (goalCounts == null) return Map.of();
        return goalCounts.entrySet().stream()
            .collect(Collectors.toUnmodifiableMap(
                Map.Entry::getKey,
                e -> new GoalOutcomeCounts(e.getValue()[0], e.getValue()[1])));
    }

    @Override
    public void decay(String agentId, String tenancyId, double decayFactor) {
        String key = agentId + "|" + tenancyId;
        ConcurrentHashMap<String, int[]> goalCounts = store.get(key);
        if (goalCounts == null) return;
        goalCounts.replaceAll((goal, counts) ->
            new int[]{(int) (counts[0] * decayFactor), (int) (counts[1] * decayFactor)});
        goalCounts.entrySet().removeIf(e -> e.getValue()[0] == 0 && e.getValue()[1] == 0);
    }

    @Override
    public void clear(String agentId, String tenancyId) {
        store.remove(agentId + "|" + tenancyId);
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl api -Dtest=InMemoryGoalSignalStoreTest -f /Users/mdproctor/claude/casehub/eidos/pom.xml`
Expected: PASS

- [ ] **Step 5: Write DefaultGoalEvolution test**

```java
package io.casehub.eidos.api;

import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import static org.junit.jupiter.api.Assertions.*;

class DefaultGoalEvolutionTest {

    private final DefaultGoalEvolution evolution = new DefaultGoalEvolution();

    private AgentDescriptor descriptorWithGoals(AgentGoal... goals) {
        return AgentDescriptor.builder()
            .agentId("agent-1").name("test-agent").slot("default")
            .tenancyId("tenant-1").goals(List.of(goals)).build();
    }

    private AgentGoal goal(String name, GoalPriority priority) {
        return new AgentGoal(name, "description for " + name, priority,
                             Visibility.PUBLIC, List.of());
    }

    @Test
    void promotesSecondaryOnHighSuccessRate() {
        var descriptor = descriptorWithGoals(goal("g1", GoalPriority.SECONDARY));
        var counts = Map.of("g1", new GoalOutcomeCounts(9, 1));
        var result = evolution.evaluate(descriptor, counts);
        assertInstanceOf(GoalEvolutionResult.Evolved.class, result);
        var evolved = (GoalEvolutionResult.Evolved) result;
        assertTrue(evolved.promotedGoals().contains("g1"));
        assertEquals(GoalPriority.PRIMARY,
            evolved.newGoals().stream().filter(g -> g.name().equals("g1"))
                   .findFirst().orElseThrow().priority());
    }

    @Test
    void demotesPrimaryOnHighFailureRate() {
        var descriptor = descriptorWithGoals(goal("g1", GoalPriority.PRIMARY));
        var counts = Map.of("g1", new GoalOutcomeCounts(2, 8));
        var result = evolution.evaluate(descriptor, counts);
        assertInstanceOf(GoalEvolutionResult.Evolved.class, result);
        var evolved = (GoalEvolutionResult.Evolved) result;
        assertTrue(evolved.demotedGoals().contains("g1"));
    }

    @Test
    void unchangedWhenRatesBetweenThresholds() {
        var descriptor = descriptorWithGoals(goal("g1", GoalPriority.SECONDARY));
        var counts = Map.of("g1", new GoalOutcomeCounts(6, 4));
        var result = evolution.evaluate(descriptor, counts);
        assertInstanceOf(GoalEvolutionResult.Unchanged.class, result);
    }

    @Test
    void dampenedWhenBelowMinOutcomes() {
        var descriptor = descriptorWithGoals(goal("g1", GoalPriority.SECONDARY));
        var counts = Map.of("g1", new GoalOutcomeCounts(3, 0));
        var result = evolution.evaluate(descriptor, counts);
        assertInstanceOf(GoalEvolutionResult.Dampened.class, result);
    }

    @Test
    void unchangedWhenNoOutcomes() {
        var descriptor = descriptorWithGoals(goal("g1", GoalPriority.SECONDARY));
        var result = evolution.evaluate(descriptor, Map.of());
        assertInstanceOf(GoalEvolutionResult.Unchanged.class, result);
    }

    @Test
    void multipleGoals_mixedOutcomes() {
        var descriptor = descriptorWithGoals(
            goal("promote-me", GoalPriority.SECONDARY),
            goal("demote-me", GoalPriority.PRIMARY),
            goal("leave-me", GoalPriority.SECONDARY));
        var counts = Map.of(
            "promote-me", new GoalOutcomeCounts(9, 1),
            "demote-me", new GoalOutcomeCounts(2, 8),
            "leave-me", new GoalOutcomeCounts(5, 5));
        var result = evolution.evaluate(descriptor, counts);
        assertInstanceOf(GoalEvolutionResult.Evolved.class, result);
        var evolved = (GoalEvolutionResult.Evolved) result;
        assertTrue(evolved.promotedGoals().contains("promote-me"));
        assertTrue(evolved.demotedGoals().contains("demote-me"));
        assertFalse(evolved.promotedGoals().contains("leave-me"));
        assertFalse(evolved.demotedGoals().contains("leave-me"));
    }
}
```

- [ ] **Step 6: Run test to verify it fails**

Run: `mvn test -pl api -Dtest=DefaultGoalEvolutionTest -f /Users/mdproctor/claude/casehub/eidos/pom.xml`
Expected: FAIL — `DefaultGoalEvolution` does not exist

- [ ] **Step 7: Implement DefaultGoalEvolution**

```java
package io.casehub.eidos.api;

import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;

@DefaultBean
@ApplicationScoped
public class DefaultGoalEvolution implements GoalEvolution {

    private static final int MIN_OUTCOMES = 5;
    private static final double PROMOTION_THRESHOLD = 0.8;
    private static final double DEMOTION_THRESHOLD = 0.7;
    private static final double DAMPENED_DECAY = 0.5;

    @Override
    public GoalEvolutionResult evaluate(AgentDescriptor descriptor,
                                         Map<String, GoalOutcomeCounts> counts) {
        if (descriptor.goals().isEmpty()) return new GoalEvolutionResult.Unchanged();

        boolean anyBelowMin = false;
        boolean anyChanged = false;
        List<AgentGoal> newGoals = new ArrayList<>();
        List<String> promoted = new ArrayList<>();
        List<String> demoted = new ArrayList<>();

        for (AgentGoal goal : descriptor.goals()) {
            GoalOutcomeCounts gc = counts.get(goal.name());
            if (gc == null || gc.successCount() + gc.failureCount() == 0) {
                newGoals.add(goal);
                continue;
            }
            int total = gc.successCount() + gc.failureCount();
            if (total < MIN_OUTCOMES) {
                anyBelowMin = true;
                newGoals.add(goal);
                continue;
            }
            double successRate = gc.successRate();
            double failureRate = 1.0 - successRate;
            GoalPriority newPriority = goal.priority();

            if (goal.priority() == GoalPriority.SECONDARY && successRate > PROMOTION_THRESHOLD) {
                newPriority = GoalPriority.PRIMARY;
                promoted.add(goal.name());
                anyChanged = true;
            } else if (goal.priority() == GoalPriority.PRIMARY && failureRate > DEMOTION_THRESHOLD) {
                newPriority = GoalPriority.SECONDARY;
                demoted.add(goal.name());
                anyChanged = true;
            }

            if (newPriority != goal.priority()) {
                newGoals.add(new AgentGoal(goal.name(), goal.description(), newPriority,
                                           goal.visibility(), goal.capabilities()));
            } else {
                newGoals.add(goal);
            }
        }

        if (anyChanged) {
            return new GoalEvolutionResult.Evolved(newGoals, promoted, demoted);
        }
        if (anyBelowMin && !counts.isEmpty()) {
            return new GoalEvolutionResult.Dampened(DAMPENED_DECAY);
        }
        return new GoalEvolutionResult.Unchanged();
    }
}
```

- [ ] **Step 8: Run test to verify it passes**

Run: `mvn test -pl api -Dtest=DefaultGoalEvolutionTest -f /Users/mdproctor/claude/casehub/eidos/pom.xml`
Expected: PASS

- [ ] **Step 9: Add AgentGoal.toBuilder()**

Add a `Builder` inner class and `toBuilder()` method to `AgentGoal`, following
the `AgentDescriptor.toBuilder()` pattern. Use `ide_insert_member` to add
after the `AgentGoal` compact constructor.

```java
public Builder toBuilder() {
    return new Builder(name, description, priority, visibility, capabilities);
}

public static final class Builder {
    private String name, description;
    private GoalPriority priority;
    private Visibility visibility;
    private List<String> capabilities;

    Builder(String name, String description, GoalPriority priority,
            Visibility visibility, List<String> capabilities) {
        this.name = name;
        this.description = description;
        this.priority = priority;
        this.visibility = visibility;
        this.capabilities = capabilities;
    }

    public Builder name(String v) { this.name = v; return this; }
    public Builder description(String v) { this.description = v; return this; }
    public Builder priority(GoalPriority v) { this.priority = v; return this; }
    public Builder visibility(Visibility v) { this.visibility = v; return this; }
    public Builder capabilities(List<String> v) { this.capabilities = v; return this; }

    public AgentGoal build() {
        return new AgentGoal(name, description, priority, visibility, capabilities);
    }
}
```

- [ ] **Step 10: Write AgentGoal.toBuilder() test**

```java
package io.casehub.eidos.api;

import org.junit.jupiter.api.Test;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class AgentGoalBuilderTest {

    @Test
    void toBuilder_preservesAllFields() {
        var goal = new AgentGoal("g1", "desc", GoalPriority.PRIMARY,
                                  Visibility.PUBLIC, List.of("cap-1"));
        var copy = goal.toBuilder().build();
        assertEquals(goal, copy);
    }

    @Test
    void toBuilder_changesDescription() {
        var goal = new AgentGoal("g1", "old desc", GoalPriority.PRIMARY,
                                  Visibility.PUBLIC, List.of());
        var revised = goal.toBuilder().description("new desc").build();
        assertEquals("new desc", revised.description());
        assertEquals(goal.name(), revised.name());
        assertEquals(goal.priority(), revised.priority());
    }

    @Test
    void toBuilder_changesPriority() {
        var goal = new AgentGoal("g1", "desc", GoalPriority.SECONDARY,
                                  Visibility.PUBLIC, List.of());
        var revised = goal.toBuilder().priority(GoalPriority.PRIMARY).build();
        assertEquals(GoalPriority.PRIMARY, revised.priority());
    }
}
```

- [ ] **Step 11: Run all eidos tests**

Run: `mvn test -pl api -f /Users/mdproctor/claude/casehub/eidos/pom.xml`
Expected: ALL PASS

- [ ] **Step 12: Commit and install**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add api/src/main/java/io/casehub/eidos/api/InMemoryGoalSignalStore.java api/src/main/java/io/casehub/eidos/api/DefaultGoalEvolution.java api/src/main/java/io/casehub/eidos/api/AgentGoal.java api/src/test/java/io/casehub/eidos/api/InMemoryGoalSignalStoreTest.java api/src/test/java/io/casehub/eidos/api/DefaultGoalEvolutionTest.java api/src/test/java/io/casehub/eidos/api/AgentGoalBuilderTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat: add GoalSignalStore/GoalEvolution implementations + AgentGoal.toBuilder()

Refs engine#806"
mvn install -DskipTests -q -f /Users/mdproctor/claude/casehub/eidos/pom.xml
```

---

### Task 2: Engine-api SPI types + GOAL_REVISED event type

Create GoalRevisionStrategy SPI, GoalRevisionContext, GoalRevisionProposal,
and GOAL_REVISED event type constant.

**Files:**
- Create: `api/src/main/java/io/casehub/api/spi/routing/GoalRevisionStrategy.java`
- Create: `api/src/main/java/io/casehub/api/spi/routing/GoalRevisionContext.java`
- Create: `api/src/main/java/io/casehub/api/spi/routing/GoalRevisionProposal.java`
- Modify: `api/src/main/java/io/casehub/api/model/event/CaseHubEventType.java`
- Test: `api/src/test/java/io/casehub/api/spi/routing/GoalRevisionProposalTest.java`

**Interfaces:**
- Consumes: `NamedStrategy` (platform-api), `AgentGoal` (eidos-api), `GoalOutcomeCounts` (eidos-api), `CaseDefinition` (engine-api)
- Produces: `GoalRevisionStrategy`, `GoalRevisionContext`, `GoalRevisionProposal` (used by Tasks 4, 5)

- [ ] **Step 1: Create GoalRevisionProposal record**

```java
package io.casehub.api.spi.routing;

import java.util.List;
import java.util.Objects;

public record GoalRevisionProposal(
    List<RevisedGoal> revisions,
    String rationale
) {
    public GoalRevisionProposal {
        Objects.requireNonNull(revisions, "revisions must not be null");
        revisions = List.copyOf(revisions);
    }

    public record RevisedGoal(
        String goalName,
        String revisedDescription,
        String revisionReason
    ) {
        public RevisedGoal {
            Objects.requireNonNull(goalName, "goalName must not be null");
            Objects.requireNonNull(revisionReason, "revisionReason must not be null");
        }
    }
}
```

- [ ] **Step 2: Create GoalRevisionContext record**

```java
package io.casehub.api.spi.routing;

import io.casehub.api.model.CaseDefinition;
import io.casehub.eidos.api.AgentGoal;
import io.casehub.eidos.api.GoalOutcomeCounts;
import java.util.List;
import java.util.Map;
import java.util.Objects;

public record GoalRevisionContext(
    String agentId,
    String tenancyId,
    List<AgentGoal> goals,
    Map<String, GoalOutcomeCounts> counts,
    CaseDefinition definition
) {
    public GoalRevisionContext {
        Objects.requireNonNull(agentId, "agentId must not be null");
        Objects.requireNonNull(tenancyId, "tenancyId must not be null");
        Objects.requireNonNull(goals, "goals must not be null");
        goals = List.copyOf(goals);
        Objects.requireNonNull(counts, "counts must not be null");
        counts = Map.copyOf(counts);
    }
}
```

- [ ] **Step 3: Create GoalRevisionStrategy interface**

```java
package io.casehub.api.spi.routing;

import io.casehub.platform.api.routing.NamedStrategy;
import io.smallrye.mutiny.Uni;

public interface GoalRevisionStrategy extends NamedStrategy {
    Uni<GoalRevisionProposal> revise(GoalRevisionContext context);

    @Override
    default String id() { return "llm"; }
}
```

- [ ] **Step 4: Add GOAL_REVISED to CaseHubEventType**

Use `ide_edit_member` to add after `PLAN_ADAPTED`:

```java
GOAL_REVISED, // agent goal revised based on accumulated outcome signals
```

- [ ] **Step 5: Write GoalRevisionProposal test**

```java
package io.casehub.api.spi.routing;

import org.junit.jupiter.api.Test;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class GoalRevisionProposalTest {

    @Test
    void constructsWithValidData() {
        var revision = new GoalRevisionProposal.RevisedGoal("g1", "new desc", "better fit");
        var proposal = new GoalRevisionProposal(List.of(revision), "test rationale");
        assertEquals(1, proposal.revisions().size());
        assertEquals("g1", proposal.revisions().get(0).goalName());
    }

    @Test
    void revisionsListIsImmutable() {
        var proposal = new GoalRevisionProposal(List.of(), "rationale");
        assertThrows(UnsupportedOperationException.class,
            () -> proposal.revisions().add(
                new GoalRevisionProposal.RevisedGoal("g1", null, "reason")));
    }

    @Test
    void nullRevisionsThrows() {
        assertThrows(NullPointerException.class,
            () -> new GoalRevisionProposal(null, "rationale"));
    }

    @Test
    void nullGoalNameThrows() {
        assertThrows(NullPointerException.class,
            () -> new GoalRevisionProposal.RevisedGoal(null, "desc", "reason"));
    }

    @Test
    void nullDescriptionAllowed() {
        var revision = new GoalRevisionProposal.RevisedGoal("g1", null, "no change needed");
        assertNull(revision.revisedDescription());
    }
}
```

- [ ] **Step 6: Run tests**

Run: `mvn test -pl api -Dtest=GoalRevisionProposalTest -f /Users/mdproctor/claude/casehub/slots/94/engine/pom.xml`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add api/src/main/java/io/casehub/api/spi/routing/GoalRevisionStrategy.java api/src/main/java/io/casehub/api/spi/routing/GoalRevisionContext.java api/src/main/java/io/casehub/api/spi/routing/GoalRevisionProposal.java api/src/main/java/io/casehub/api/model/event/CaseHubEventType.java api/src/test/java/io/casehub/api/spi/routing/GoalRevisionProposalTest.java
git commit -m "feat(#806): add GoalRevisionStrategy SPI types + GOAL_REVISED event type"
```

---

### Task 3: GoalOutcomeRecorder — replace GoalFailureRecorder

Rename GoalFailureRecorder to GoalOutcomeRecorder, switch from
BehavioralSignalStore to GoalSignalStore, add SUCCESS recording, null
capability guard. Update WorkflowExecutionCompletedHandler call sites.

**Files:**
- Rename: `GoalOutcomeRecorder` -> `GoalOutcomeRecorder` (use `ide_refactor_rename`)
- Modify: `runtime/src/main/java/io/casehub/engine/internal/routing/GoalOutcomeRecorder.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/WorkflowExecutionCompletedHandler.java`
- Rename: `GoalOutcomeRecorderTest` -> `GoalOutcomeRecorderTest` (use `ide_refactor_rename`)
- Modify: `runtime/src/test/java/io/casehub/engine/internal/routing/GoalOutcomeRecorderTest.java`

**Interfaces:**
- Consumes: `GoalSignalStore` (eidos-api), `GoalOutcome` (eidos-api), `CaseDefinitionRegistry`, `CaseDefinition`, `AgentDescriptor`
- Produces: `GoalOutcomeRecorder.record()` — called by WorkflowExecutionCompletedHandler on both paths

- [ ] **Step 1: Rename GoalFailureRecorder -> GoalOutcomeRecorder**

Use `ide_refactor_rename` on the class. This updates all references including
the field in `WorkflowExecutionCompletedHandler`, the test class name, and
any other references.

- [ ] **Step 2: Modify GoalOutcomeRecorder — replace BehavioralSignalStore with GoalSignalStore**

Replace the `Instance<BehavioralSignalStore>` injection with
`Instance<GoalSignalStore>`. Replace the signal recording logic:

```java
@ApplicationScoped
public class GoalOutcomeRecorder {

  private static final Logger LOG = Logger.getLogger(GoalOutcomeRecorder.class);

  private final Instance<GoalSignalStore> signalStore;
  private final CaseDefinitionRegistry registry;

  @Inject
  public GoalOutcomeRecorder(
      Instance<GoalSignalStore> signalStore, CaseDefinitionRegistry registry) {
    this.signalStore = signalStore;
    this.registry = registry;
  }

  public void record(
      CaseInstance caseInstance,
      String workerName,
      String capabilityName,
      WorkerOutcome<?> outcome) {
    if (!signalStore.isResolvable()) return;
    if (capabilityName == null) return;

    CaseDefinition definition;
    try {
      definition = registry.getCaseDefinition(caseInstance.getCaseMetaModel());
    } catch (Exception e) {
      return;
    }

    Optional<AgentDescriptor> descriptorOpt = definition.agentDescriptorFor(workerName);
    if (descriptorOpt.isEmpty()) return;

    AgentDescriptor descriptor = descriptorOpt.get();
    if (descriptor.goals().isEmpty()) return;

    GoalOutcome goalOutcome = mapOutcome(outcome);
    String agentId = descriptor.agentId();
    String tenancyId = caseInstance.tenancyId;

    for (var goal : descriptor.goals()) {
      if (!goal.capabilities().isEmpty()
          && !goal.capabilities().contains(capabilityName)) {
        continue;
      }
      signalStore.get().recordOutcome(agentId, tenancyId, goal.name(), goalOutcome);
      LOG.debugf("Goal outcome recorded: agent=%s goal=%s outcome=%s capability=%s",
          agentId, goal.name(), goalOutcome, capabilityName);
    }
  }

  private static GoalOutcome mapOutcome(WorkerOutcome<?> outcome) {
    return switch (outcome) {
      case WorkerOutcome.Success<?> s -> GoalOutcome.SUCCESS;
      case WorkerOutcome.Completed<?> c -> GoalOutcome.SUCCESS;
      case WorkerOutcome.Declined<?> d -> GoalOutcome.FAILURE;
      case WorkerOutcome.Failed<?> f -> GoalOutcome.FAILURE;
      case WorkerOutcome.Expired<?> e -> GoalOutcome.FAILURE;
    };
  }
}
```

- [ ] **Step 3: Add success-path call in WorkflowExecutionCompletedHandler**

The handler currently calls `goalOutcomeRecorder.record()` only on the failure
path (in `handleSemanticFailure`). Add a call on the success path, after
`agentGoalCompletionMarker.markGoalsCompleted()`:

```java
goalOutcomeRecorder.record(
    caseInstance,
    worker.name(),
    extractCapabilityTag(caseInstance, worker, bindingName),
    event.outcome());
```

- [ ] **Step 4: Update GoalOutcomeRecorderTest**

Rewrite tests to use `GoalSignalStore` instead of `BehavioralSignalStore`.
Add tests for SUCCESS recording and null capability guard. Key test cases:

- SUCCESS records GoalOutcome.SUCCESS
- COMPLETED records GoalOutcome.SUCCESS
- DECLINED/FAILED/EXPIRED record GoalOutcome.FAILURE
- Null capabilityName returns early (no recording)
- Capability filtering unchanged
- No GoalSignalStore -> no-op

- [ ] **Step 5: Run tests**

Run: `mvn test -pl runtime -Dtest=GoalOutcomeRecorderTest -f /Users/mdproctor/claude/casehub/slots/94/engine/pom.xml`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/casehub/engine/internal/routing/GoalOutcomeRecorder.java runtime/src/main/java/io/casehub/engine/internal/engine/handler/WorkflowExecutionCompletedHandler.java runtime/src/test/java/io/casehub/engine/internal/routing/GoalOutcomeRecorderTest.java
git commit -m "feat(#806): replace GoalFailureRecorder with GoalOutcomeRecorder using GoalSignalStore"
```

---

### Task 4: GoalAbandonmentEvaluator — evolve to use GoalSignalStore

Migrate from BehavioralSignalStore with `__goal__` sentinel to
GoalSignalStore.outcomeCounts().

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/routing/GoalAbandonmentEvaluator.java`
- Modify: `runtime/src/test/java/io/casehub/engine/internal/routing/GoalAbandonmentEvaluatorTest.java`

**Interfaces:**
- Consumes: `GoalSignalStore` (eidos-api), `GoalOutcomeCounts` (eidos-api)
- Produces: `GoalAbandonmentEvaluator.isAbandoned()`, `activeGoals()` — existing signatures unchanged

- [ ] **Step 1: Modify GoalAbandonmentEvaluator**

Replace `Instance<BehavioralSignalStore>` with `Instance<GoalSignalStore>`.
Replace `signalStore.get().count(agentId, tenancyId, GOAL_CAPABILITY_SENTINEL, goalName, BehavioralSignal.DECLINE)` with:

```java
Map<String, GoalOutcomeCounts> counts = signalStore.get().outcomeCounts(agentId, tenancyId);
GoalOutcomeCounts gc = counts.get(goalName);
int failureCount = gc != null ? gc.failureCount() : 0;
return failureCount >= threshold;
```

Remove the `GOAL_CAPABILITY_SENTINEL` constant (no longer needed).

- [ ] **Step 2: Update GoalAbandonmentEvaluatorTest**

Replace mock `BehavioralSignalStore` with `InMemoryGoalSignalStore`.
Test logic is unchanged — just the signal store type changes.

- [ ] **Step 3: Run tests**

Run: `mvn test -pl runtime -Dtest=GoalAbandonmentEvaluatorTest -f /Users/mdproctor/claude/casehub/slots/94/engine/pom.xml`
Expected: PASS

- [ ] **Step 4: Commit**

```bash
git add runtime/src/main/java/io/casehub/engine/internal/routing/GoalAbandonmentEvaluator.java runtime/src/test/java/io/casehub/engine/internal/routing/GoalAbandonmentEvaluatorTest.java
git commit -m "feat(#806): evolve GoalAbandonmentEvaluator to use GoalSignalStore"
```

---

### Task 5: GoalRevisionEvaluator + LlmGoalRevisionStrategy

Core evaluator with threshold trigger, GoalEvolution integration,
GoalRevisionStrategy delegation, and LLM implementation.

**Files:**
- Create: `runtime/src/main/java/io/casehub/engine/internal/routing/GoalRevisionEvaluator.java`
- Create: `runtime/src/main/java/io/casehub/engine/internal/routing/LlmGoalRevisionStrategy.java`
- Test: `runtime/src/test/java/io/casehub/engine/internal/routing/GoalRevisionEvaluatorTest.java`
- Test: `runtime/src/test/java/io/casehub/engine/internal/routing/LlmGoalRevisionStrategyTest.java`

**Interfaces:**
- Consumes: `GoalSignalStore`, `GoalEvolution`, `AgentRegistry` (eidos-api), `GoalRevisionStrategy`, `GoalRevisionContext`, `GoalRevisionProposal` (engine-api), `CaseDefinitionRegistry`, `EventLogRepository`, `EngineStrategyResolver` (runtime), `ChatModelProvider` (api/model/ai)
- Produces: `GoalRevisionEvaluator.record()` — called by Task 6 (handler wiring)

- [ ] **Step 1: Write GoalRevisionEvaluator test — skip when disabled**

```java
@Test
void record_skipsWhenNotEnabled() {
    // enabled defaults to false
    evaluator.record(caseInstance, "worker-1", "cap-x", WorkerOutcome.success());
    verifyNoInteractions(goalSignalStore);
}
```

- [ ] **Step 2: Write test — skip when GoalSignalStore not resolvable**

- [ ] **Step 3: Write test — accumulates below threshold without triggering**

- [ ] **Step 4: Write test — triggers when outcomeCount >= minOutcomes**

Test that after `minOutcomes` calls to `record()`, GoalEvolution.evaluate()
is called.

- [ ] **Step 5: Write test — on Evolved, calls register with updated descriptor**

- [ ] **Step 6: Write test — on Dampened, calls GoalSignalStore.decay()**

- [ ] **Step 7: Write test — on Unchanged, no action**

- [ ] **Step 8: Write test — strategy failure, priority-only still applies**

- [ ] **Step 9: Write test — per-goal error isolation on description validation**

- [ ] **Step 10: Write test — per-agent lock prevents concurrent revision**

- [ ] **Step 11: Write test — writes GOAL_REVISED EventLog**

- [ ] **Step 12: Implement GoalRevisionEvaluator**

Full implementation following the spec's evaluation logic. Key structure:

```java
@ApplicationScoped
public class GoalRevisionEvaluator {
    private final Instance<GoalSignalStore> goalSignalStore;
    private final Instance<GoalEvolution> goalEvolution;
    private final Instance<AgentRegistry> agentRegistry;
    private final CaseDefinitionRegistry caseDefinitionRegistry;
    private final EngineStrategyResolver strategyResolver;
    private final EventLogRepository eventLogRepository;
    private final ConcurrentHashMap<String, RevisionState> states = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, ReentrantLock> locks = new ConcurrentHashMap<>();

    @ConfigProperty(name = "casehub.engine.goal.revision.enabled", defaultValue = "false")
    boolean enabled;
    @ConfigProperty(name = "casehub.engine.goal.revision.strategy", defaultValue = "llm")
    String strategyId;
    @ConfigProperty(name = "casehub.engine.goal.revision.min-outcomes", defaultValue = "10")
    int minOutcomes;
    @ConfigProperty(name = "casehub.engine.goal.revision.importance-threshold", defaultValue = "3.0")
    double importanceThreshold;

    // record() method with trigger evaluation
    // evaluateRevision() on virtual thread
    // mergeDescriptions() for LLM description merging
    // writeAuditLog() for GOAL_REVISED EventLog
}
```

- [ ] **Step 13: Run GoalRevisionEvaluator tests**

Run: `mvn test -pl runtime -Dtest=GoalRevisionEvaluatorTest -f /Users/mdproctor/claude/casehub/slots/94/engine/pom.xml`
Expected: PASS

- [ ] **Step 14: Write LlmGoalRevisionStrategy test**

Test cases: produces proposal from canned LLM response, no-op when
ChatModelProvider absent, invalid JSON fails, empty revisions valid.

- [ ] **Step 15: Implement LlmGoalRevisionStrategy**

- [ ] **Step 16: Run LlmGoalRevisionStrategy tests**

Run: `mvn test -pl runtime -Dtest=LlmGoalRevisionStrategyTest -f /Users/mdproctor/claude/casehub/slots/94/engine/pom.xml`
Expected: PASS

- [ ] **Step 17: Commit**

```bash
git add runtime/src/main/java/io/casehub/engine/internal/routing/GoalRevisionEvaluator.java runtime/src/main/java/io/casehub/engine/internal/routing/LlmGoalRevisionStrategy.java runtime/src/test/java/io/casehub/engine/internal/routing/GoalRevisionEvaluatorTest.java runtime/src/test/java/io/casehub/engine/internal/routing/LlmGoalRevisionStrategyTest.java
git commit -m "feat(#806): add GoalRevisionEvaluator + LlmGoalRevisionStrategy"
```

---

### Task 6: Wire GoalRevisionEvaluator into WorkflowExecutionCompletedHandler

Add GoalRevisionEvaluator call on both success and failure paths.

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/WorkflowExecutionCompletedHandler.java`

**Interfaces:**
- Consumes: `GoalRevisionEvaluator.record()` (from Task 5)
- Produces: Complete wiring — goal revision fires on every worker completion

- [ ] **Step 1: Inject GoalRevisionEvaluator into handler**

Add field: `private final GoalRevisionEvaluator goalRevisionEvaluator;`
Add constructor parameter.

- [ ] **Step 2: Add call on success path**

After `goalOutcomeRecorder.record()` (added in Task 3), add:

```java
goalRevisionEvaluator.record(
    caseInstance,
    worker.name(),
    extractCapabilityTag(caseInstance, worker, bindingName),
    event.outcome());
```

- [ ] **Step 3: Add call on failure path**

In `handleSemanticFailure`, after `goalOutcomeRecorder.record()`:

```java
goalRevisionEvaluator.record(
    caseInstance,
    worker.name(),
    capabilityTag,
    event.outcome());
```

- [ ] **Step 4: Run handler compilation check**

Run: `mvn compile -pl runtime -f /Users/mdproctor/claude/casehub/slots/94/engine/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/casehub/engine/internal/engine/handler/WorkflowExecutionCompletedHandler.java
git commit -m "feat(#806): wire GoalRevisionEvaluator into WorkflowExecutionCompletedHandler"
```

---

### Task 7: Integration test

Full-flow `@QuarkusTest` verifying the complete goal revision pipeline.

**Files:**
- Create: `runtime/src/test/java/io/casehub/engine/internal/routing/GoalRevisionIntegrationTest.java`

**Interfaces:**
- Consumes: All components from Tasks 1-6

- [ ] **Step 1: Write integration test**

Test structure:
1. Define a CaseHub subclass with an agent having SECONDARY and PRIMARY goals
2. Enable goal revision via test `application.properties`
3. Run multiple worker completions (SUCCESS for SECONDARY goal, FAILURE for PRIMARY)
4. Verify GoalEvolution fires and AgentRegistry is updated
5. Verify EventLog contains GOAL_REVISED with correct metadata
6. Verify promoted/demoted goals

- [ ] **Step 2: Add test config properties**

In `runtime/src/test/resources/application.properties`:

```properties
casehub.engine.goal.revision.enabled=true
casehub.engine.goal.revision.min-outcomes=3
```

- [ ] **Step 3: Run integration test**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=GoalRevisionIntegrationTest -f /Users/mdproctor/claude/casehub/slots/94/engine/pom.xml`
Expected: PASS

- [ ] **Step 4: Run full test suite**

Run: `mvn test -pl runtime -f /Users/mdproctor/claude/casehub/slots/94/engine/pom.xml`
Expected: ALL PASS (no regressions)

- [ ] **Step 5: Commit**

```bash
git add runtime/src/test/java/io/casehub/engine/internal/routing/GoalRevisionIntegrationTest.java runtime/src/test/resources/application.properties
git commit -m "feat(#806): add GoalRevisionIntegrationTest — full pipeline verification"
```

---

### Task 8: CLAUDE.md + spec documentation

Update CLAUDE.md with goal revision documentation and update the spec
to reflect any implementation deviations.

**Files:**
- Modify: `CLAUDE.md`

- [ ] **Step 1: Add Goal Revision section to CLAUDE.md**

Add section documenting GoalRevisionEvaluator, GoalOutcomeRecorder,
GoalRevisionStrategy SPI, configuration properties, and the eidos
GoalSignalStore/GoalEvolution integration.

- [ ] **Step 2: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: add Goal Revision section to CLAUDE.md

Refs #806"
```
