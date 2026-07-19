# HumanTask CBR Routing Enrichment — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #741 — Support humanTask routing enrichment via CBR plan traces
**Issue group:** #741

**Goal:** Extend the engine's routing pipeline so CBR plan trace data flows
through humanTask bindings — enabling both historical scoring of candidates
and experience context forwarding to the work repo.

**Architecture:** New `HumanTaskRoutingStrategy` SPI (sealed result, context/candidates
separation) in `api/spi/routing/`. Default no-op in `runtime/internal/routing/`.
Handler plumbing threads experiences through `publishHumanTaskSchedule` and calls
the strategy between candidate resolution and event publishing. Retention side
generalised to include humanTask PlanItems. Cross-repo nullability changes in
neocortex `PlanTrace` and `AdaptedStep`.

**Tech Stack:** Java 21, Quarkus (CDI, Mutiny), engine-api SPI conventions

## Global Constraints

- Routing strategy convention (engine#634): `select()` method, context/candidates
  separation, sealed result type, `NamedStrategy` + `StrategyResolver`
- `@DefaultBean @ApplicationScoped @Unremovable` for no-op defaults
  (protocol PP-20260514-engine-spi-noops-defaultbean)
- IntelliJ MCP for all code navigation and editing
- TDD: write failing test → verify fail → implement → verify pass → commit

---

### Task 1: Nullability changes — `PlanTrace`, `AdaptedStep`, `ExperiencePlanStep`

Cross-repo change in neocortex + engine-api. Makes `capabilityName` nullable
on all three records so humanTask plan traces can store `null` (no capability).

**Files:**
- Modify: `neocortex/memory-api/src/main/java/io/casehub/neocortex/memory/cbr/PlanTrace.java`
- Modify: `neocortex/memory-api/src/main/java/io/casehub/neocortex/memory/cbr/AdaptedStep.java`
- Modify: `neocortex/memory-api/src/test/java/io/casehub/neocortex/memory/cbr/AdaptedStepTest.java`
- Modify: `engine/api/src/main/java/io/casehub/api/spi/routing/ExperiencePlanStep.java`
- Modify: `engine/api/src/test/java/io/casehub/api/spi/routing/ExperiencePlanStepTest.java`

**Interfaces:**
- Produces: `PlanTrace(bindingName, null, ...)` — valid construction
- Produces: `AdaptedStep(bindingName, null, ...)` — valid construction
- Produces: `ExperiencePlanStep(bindingName, null, ...)` — valid construction

- [ ] **Step 1: Write failing tests — null capabilityName accepted**

In `ExperiencePlanStepTest.java`, change `null_capabilityName_throws` to verify
null is accepted:

```java
@Test
void null_capabilityName_accepted() {
    var step = new ExperiencePlanStep("b", null, "w", "ok", 0, Map.of());
    assertNull(step.capabilityName());
}
```

In `AdaptedStepTest.java`, change `nullCapabilityNameRejected` to verify null
is accepted:

```java
@Test void nullCapabilityNameAccepted() {
    var step = new AdaptedStep("b1", null, "w1", "SUCCESS", 0,
            Map.of("k", "v"), AdaptationAction.RETAINED, null);
    assertThat(step.capabilityName()).isNull();
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl api -Dtest=ExperiencePlanStepTest#null_capabilityName_accepted -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: FAIL (NullPointerException from `Objects.requireNonNull`)

Run: `mvn test -pl memory-api -Dtest=AdaptedStepTest#nullCapabilityNameAccepted -f /Users/mdproctor/claude/casehub/neocortex/pom.xml`
Expected: FAIL (NullPointerException from `Objects.requireNonNull`)

- [ ] **Step 3: Remove requireNonNull on capabilityName**

`PlanTrace.java` — remove `Objects.requireNonNull(capabilityName, "capabilityName required");`

`AdaptedStep.java` — remove `Objects.requireNonNull(capabilityName, "capabilityName");`

`ExperiencePlanStep.java` — remove `Objects.requireNonNull(capabilityName, "capabilityName must not be null");`

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -pl memory-api -Dtest=AdaptedStepTest -f /Users/mdproctor/claude/casehub/neocortex/pom.xml`
Expected: PASS

Run: `mvn test -pl api -Dtest=ExperiencePlanStepTest -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: PASS

- [ ] **Step 5: Install neocortex locally, commit both repos**

```bash
mvn install -DskipTests -q -f /Users/mdproctor/claude/casehub/neocortex/pom.xml
```

Commit neocortex:
```bash
git -C /Users/mdproctor/claude/casehub/neocortex add memory-api/src
git -C /Users/mdproctor/claude/casehub/neocortex commit -m "feat(#741): make PlanTrace and AdaptedStep capabilityName nullable for humanTask traces

Refs casehubio/engine#741"
```

Commit engine:
```bash
git -C /Users/mdproctor/claude/casehub/engine add api/src
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(#741): make ExperiencePlanStep.capabilityName nullable

Refs #741"
```

---

### Task 2: `ExperienceAnalyser` predicate overload

Adds the generalized overload that accepts a `Predicate<ExperiencePlanStep>`
instead of a `String capabilityName`. Existing overload delegates.

**Files:**
- Modify: `engine/api/src/main/java/io/casehub/api/spi/routing/ExperienceAnalyser.java`
- Modify: `engine/api/src/test/java/io/casehub/api/spi/routing/ExperienceAnalyserTest.java`

**Interfaces:**
- Consumes: `ExperiencePlanStep` (nullable `capabilityName` from Task 1)
- Produces: `ExperienceAnalyser.workerSuccessRates(experiences, eligible, Predicate, weights)`

- [ ] **Step 1: Write failing test — predicate overload**

```java
@Test
void predicateOverload_matchesByBindingName() {
    var step = new ExperiencePlanStep("review-binding", null, "reviewer-a", "SUCCESS", 0, Map.of());
    var exp = new RetrievedExperience(
        "problem", "solution", "COMPLETED", 1.0, 0.8, Map.of(), List.of(step), Map.of());
    Map<String, Double> result = ExperienceAnalyser.workerSuccessRates(
        List.of(exp),
        Set.of("reviewer-a"),
        s -> "review-binding".equals(s.bindingName()),
        ExperienceAnalyser.DEFAULT_OUTCOME_WEIGHTS);
    assertThat(result).containsEntry("reviewer-a", 1.0);
}

@Test
void predicateOverload_nullCapabilityName_noMatchOnCapability() {
    var step = new ExperiencePlanStep("review-binding", null, "reviewer-a", "SUCCESS", 0, Map.of());
    var exp = new RetrievedExperience(
        "problem", "solution", "COMPLETED", 1.0, 0.8, Map.of(), List.of(step), Map.of());
    Map<String, Double> result = ExperienceAnalyser.workerSuccessRates(
        List.of(exp),
        Set.of("reviewer-a"),
        "review-binding",
        ExperienceAnalyser.DEFAULT_OUTCOME_WEIGHTS);
    assertThat(result).isEmpty();
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl api -Dtest=ExperienceAnalyserTest#predicateOverload_matchesByBindingName -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: FAIL — method does not exist

- [ ] **Step 3: Implement predicate overload**

Add the new overload and make the existing one delegate:

```java
public static Map<String, Double> workerSuccessRates(
    final List<RetrievedExperience> experiences,
    final Set<String> eligibleWorkerIds,
    final Predicate<ExperiencePlanStep> stepFilter,
    final Map<RoutingOutcome, Double> outcomeWeights) {
  // Same body as current method, replacing:
  //   if (!capabilityName.equals(step.capabilityName()) ...)
  // with:
  //   if (!stepFilter.test(step) ...)
}
```

Existing overload becomes:

```java
public static Map<String, Double> workerSuccessRates(
    final List<RetrievedExperience> experiences,
    final Set<String> eligibleWorkerIds,
    final String capabilityName,
    final Map<RoutingOutcome, Double> outcomeWeights) {
  return workerSuccessRates(experiences, eligibleWorkerIds,
      step -> capabilityName.equals(step.capabilityName()), outcomeWeights);
}
```

Add import: `java.util.function.Predicate`

- [ ] **Step 4: Run full ExperienceAnalyserTest**

Run: `mvn test -pl api -Dtest=ExperienceAnalyserTest -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: ALL PASS (existing tests verify delegation, new tests verify predicate path)

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add api/src
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(#741): generalize ExperienceAnalyser with predicate-based step filter

Refs #741"
```

---

### Task 3: SPI types — `HumanTaskRoutingStrategy`, context, candidates, result

Creates the four new SPI types in `api/spi/routing/`.

**Files:**
- Create: `engine/api/src/main/java/io/casehub/api/spi/routing/HumanTaskRoutingStrategy.java`
- Create: `engine/api/src/main/java/io/casehub/api/spi/routing/HumanTaskRoutingContext.java`
- Create: `engine/api/src/main/java/io/casehub/api/spi/routing/HumanTaskCandidates.java`
- Create: `engine/api/src/main/java/io/casehub/api/spi/routing/HumanTaskRoutingResult.java`
- Create: `engine/api/src/test/java/io/casehub/api/spi/routing/HumanTaskCandidatesTest.java`
- Create: `engine/api/src/test/java/io/casehub/api/spi/routing/HumanTaskRoutingResultTest.java`

**Interfaces:**
- Consumes: `RetrievedExperience` (from Task 1/2), `NamedStrategy` (platform-api)
- Produces: `HumanTaskRoutingStrategy.select(context, candidates) → Uni<HumanTaskRoutingResult>`

- [ ] **Step 1: Write tests for `HumanTaskCandidates` and `HumanTaskRoutingResult`**

`HumanTaskCandidatesTest.java`:
```java
package io.casehub.api.spi.routing;

import org.junit.jupiter.api.Test;
import java.util.Set;
import static org.assertj.core.api.Assertions.assertThat;

class HumanTaskCandidatesTest {
    @Test void validConstruction() {
        var c = new HumanTaskCandidates(Set.of("group-a"), Set.of("user-1"));
        assertThat(c.groups()).containsExactly("group-a");
        assertThat(c.users()).containsExactly("user-1");
    }
    @Test void nullGroupsDefaultsToEmpty() {
        var c = new HumanTaskCandidates(null, Set.of("user-1"));
        assertThat(c.groups()).isEmpty();
    }
    @Test void nullUsersDefaultsToEmpty() {
        var c = new HumanTaskCandidates(Set.of("group-a"), null);
        assertThat(c.users()).isEmpty();
    }
    @Test void defensiveCopy() {
        var groups = new java.util.HashSet<>(Set.of("g"));
        var c = new HumanTaskCandidates(groups, Set.of());
        groups.add("g2");
        assertThat(c.groups()).doesNotContain("g2");
    }
}
```

`HumanTaskRoutingResultTest.java`:
```java
package io.casehub.api.spi.routing;

import org.junit.jupiter.api.Test;
import java.util.Map;
import java.util.Set;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatNullPointerException;

class HumanTaskRoutingResultTest {
    @Test void enriched_defensiveCopies() {
        var groups = new java.util.HashSet<>(Set.of("g1"));
        var scores = new java.util.HashMap<String, Double>();
        scores.put("u1", 0.8);
        var result = new HumanTaskRoutingResult.Enriched(groups, Set.of("u1"), scores);
        groups.add("g2");
        scores.put("u2", 0.5);
        assertThat(result.candidateGroups()).doesNotContain("g2");
        assertThat(result.candidateScores()).doesNotContainKey("u2");
    }
    @Test void unchanged_isInstanceOf() {
        HumanTaskRoutingResult result = new HumanTaskRoutingResult.Unchanged();
        assertThat(result).isInstanceOf(HumanTaskRoutingResult.Unchanged.class);
    }
    @Test void escalated_carriesReason() {
        var result = new HumanTaskRoutingResult.Escalated("constraint violation");
        assertThat(result.reason()).isEqualTo("constraint violation");
    }
    @Test void sealedExhaustive() {
        HumanTaskRoutingResult result = new HumanTaskRoutingResult.Unchanged();
        String label = switch (result) {
            case HumanTaskRoutingResult.Enriched e -> "enriched";
            case HumanTaskRoutingResult.Unchanged u -> "unchanged";
            case HumanTaskRoutingResult.Escalated e -> "escalated";
        };
        assertThat(label).isEqualTo("unchanged");
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl api -Dtest=HumanTaskCandidatesTest -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: FAIL — class not found

- [ ] **Step 3: Create the four SPI types**

`HumanTaskRoutingContext.java`:
```java
package io.casehub.api.spi.routing;

import com.fasterxml.jackson.databind.JsonNode;
import java.util.List;
import java.util.UUID;

public record HumanTaskRoutingContext(
    UUID caseId,
    String bindingName,
    String tenancyId,
    JsonNode caseContext,
    List<RetrievedExperience> experiences) {}
```

`HumanTaskCandidates.java`:
```java
package io.casehub.api.spi.routing;

import java.util.Set;

public record HumanTaskCandidates(Set<String> groups, Set<String> users) {
    public HumanTaskCandidates {
        groups = groups != null ? Set.copyOf(groups) : Set.of();
        users = users != null ? Set.copyOf(users) : Set.of();
    }
}
```

`HumanTaskRoutingResult.java`:
```java
package io.casehub.api.spi.routing;

import java.util.Map;
import java.util.Set;

public sealed interface HumanTaskRoutingResult
    permits HumanTaskRoutingResult.Enriched,
            HumanTaskRoutingResult.Unchanged,
            HumanTaskRoutingResult.Escalated {

    record Enriched(
        Set<String> candidateGroups,
        Set<String> candidateUsers,
        Map<String, Double> candidateScores) implements HumanTaskRoutingResult {
        public Enriched {
            candidateGroups = Set.copyOf(candidateGroups);
            candidateUsers = Set.copyOf(candidateUsers);
            candidateScores = Map.copyOf(candidateScores);
        }
    }

    record Unchanged() implements HumanTaskRoutingResult {}

    record Escalated(String reason) implements HumanTaskRoutingResult {}
}
```

`HumanTaskRoutingStrategy.java`:
```java
package io.casehub.api.spi.routing;

import io.casehub.platform.api.routing.NamedStrategy;
import io.smallrye.mutiny.Uni;

public interface HumanTaskRoutingStrategy extends NamedStrategy {
    Uni<HumanTaskRoutingResult> select(
        HumanTaskRoutingContext context, HumanTaskCandidates candidates);
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -pl api -Dtest="HumanTaskCandidatesTest,HumanTaskRoutingResultTest" -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add api/src
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(#741): add HumanTaskRoutingStrategy SPI — context, candidates, sealed result

Refs #741"
```

---

### Task 4: Default impl + CaseDefinition field + EngineStrategyResolver

Wires the new SPI into the engine runtime: default no-op implementation,
`humanTaskRouting` field on `CaseDefinition`, strategy resolver registration.

**Files:**
- Create: `engine/runtime/src/main/java/io/casehub/engine/internal/routing/NoOpHumanTaskRoutingStrategy.java`
- Modify: `engine/api/src/main/java/io/casehub/api/model/CaseDefinition.java` (add field, getter, setter, builder method)
- Modify: `engine/runtime/src/main/java/io/casehub/engine/internal/routing/EngineStrategyResolver.java` (add Instance injection)
- Modify: `engine/runtime/src/test/java/io/casehub/engine/internal/routing/EngineStrategyResolverTest.java`

**Interfaces:**
- Consumes: `HumanTaskRoutingStrategy`, `HumanTaskRoutingResult` (from Task 3)
- Produces: `CaseDefinition.getHumanTaskRouting()`, `NoOpHumanTaskRoutingStrategy` (id="default")
- Produces: `EngineStrategyResolver.resolve(HumanTaskRoutingStrategy.class, id)`

- [ ] **Step 1: Write failing tests**

`EngineStrategyResolverTest.java` — add test for HumanTaskRoutingStrategy resolution:
```java
@Test
void resolvesHumanTaskRoutingStrategy() {
    var noop = new NoOpHumanTaskRoutingStrategy();
    var resolver = EngineStrategyResolver.forTest(List.of(
        new EngineStrategyResolver.TestHandle<>(noop, true)));
    var result = resolver.resolve(HumanTaskRoutingStrategy.class, null);
    assertThat(result.id()).isEqualTo("default");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl runtime -Dtest=EngineStrategyResolverTest#resolvesHumanTaskRoutingStrategy -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: FAIL — class not found

- [ ] **Step 3: Create `NoOpHumanTaskRoutingStrategy`**

```java
package io.casehub.engine.internal.routing;

import io.casehub.api.spi.routing.HumanTaskCandidates;
import io.casehub.api.spi.routing.HumanTaskRoutingContext;
import io.casehub.api.spi.routing.HumanTaskRoutingResult;
import io.casehub.api.spi.routing.HumanTaskRoutingStrategy;
import io.quarkus.arc.DefaultBean;
import io.quarkus.arc.Unremovable;
import io.smallrye.mutiny.Uni;
import jakarta.enterprise.context.ApplicationScoped;

@DefaultBean
@ApplicationScoped
@Unremovable
public class NoOpHumanTaskRoutingStrategy implements HumanTaskRoutingStrategy {
    @Override
    public String id() {
        return "default";
    }

    @Override
    public Uni<HumanTaskRoutingResult> select(
        HumanTaskRoutingContext ctx, HumanTaskCandidates candidates) {
        return Uni.createFrom().item(new HumanTaskRoutingResult.Unchanged());
    }
}
```

- [ ] **Step 4: Add `humanTaskRouting` field to `CaseDefinition`**

Add field (after `implementationRouting`):
```java
private String humanTaskRouting;
```

Add getter/setter:
```java
public String getHumanTaskRouting() { return humanTaskRouting; }
public void setHumanTaskRouting(String humanTaskRouting) {
    this.humanTaskRouting = humanTaskRouting;
}
```

Add builder field and method (after `implementationRouting`):
```java
private String humanTaskRouting;

public Builder humanTaskRouting(String humanTaskRouting) {
    this.humanTaskRouting = humanTaskRouting;
    return this;
}
```

Add to `build()` method (after `setImplementationRouting`):
```java
caseHubDefinition.setHumanTaskRouting(humanTaskRouting);
```

- [ ] **Step 5: Add `HumanTaskRoutingStrategy` to `EngineStrategyResolver` constructor**

Add parameter to constructor:
```java
@Any Instance<HumanTaskRoutingStrategy> humanTaskStrategies,
```

Add registration call:
```java
registerStrategies(humanTaskStrategies);
```

- [ ] **Step 6: Run tests**

Run: `mvn test -pl runtime -Dtest=EngineStrategyResolverTest -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: ALL PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add api/src runtime/src
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(#741): NoOpHumanTaskRoutingStrategy, CaseDefinition.humanTaskRouting, resolver wiring

Refs #741"
```

---

### Task 5: `HumanTaskScheduleEvent` + handler plumbing

Adds `experiences` and `candidateScores` fields to `HumanTaskScheduleEvent`.
Threads experiences through `publishHumanTaskSchedule`, calls the strategy.

**Files:**
- Modify: `engine/common/src/main/java/io/casehub/engine/common/internal/event/HumanTaskScheduleEvent.java`
- Modify: `engine/runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java`
- Modify: `engine/runtime/src/test/java/io/casehub/engine/HumanTaskTargetDispatchTest.java` (update event recorder)
- Create: `engine/runtime/src/test/java/io/casehub/engine/internal/routing/HumanTaskRoutingStrategyWiringTest.java`

**Interfaces:**
- Consumes: `HumanTaskRoutingStrategy` (from Task 3), `EngineStrategyResolver` (from Task 4)
- Produces: `HumanTaskScheduleEvent` with `experiences` and `candidateScores` fields

- [ ] **Step 1: Write failing test — strategy called during humanTask dispatch**

`HumanTaskRoutingStrategyWiringTest.java`:
```java
package io.casehub.engine.internal.routing;

// Test that when a HumanTaskTarget binding fires,
// the HumanTaskRoutingStrategy.select() is called with the correct context
// and the result enriches the HumanTaskScheduleEvent.
// Uses a recording strategy (@Alternative @Priority(1)) that captures the context
// and returns Enriched with scores.
```

Full test body: define an inner `CaseHub` with a humanTask binding, a recording
`HumanTaskRoutingStrategy` that captures the context and returns `Enriched`,
and assert the `HumanTaskScheduleEvent` carries the enriched candidates and scores.

- [ ] **Step 2: Add fields to `HumanTaskScheduleEvent`**

Add two new fields to the record:
```java
List<RetrievedExperience> experiences,
Map<String, Double> candidateScores
```

These go at the end of the parameter list (after `resolvedExpiresIn`).

- [ ] **Step 3: Update `publishHumanTaskSchedule` signature**

Add `CaseDefinition caseDefinition` and `List<RetrievedExperience> experiences` parameters.

- [ ] **Step 4: Update `publishByTarget` to thread parameters**

Change the `HumanTaskTarget` branch:
```java
case HumanTaskTarget ht -> publishHumanTaskSchedule(
    caseInstance, caseDefinition, binding, ht, experiences);
```

- [ ] **Step 5: Add strategy invocation in `publishHumanTaskSchedule`**

After `groupsUni`/`usersUni` resolution, before event publishing:

```java
// Resolve humanTask routing strategy
final HumanTaskRoutingStrategy humanTaskStrategy =
    strategyResolver.resolve(
        HumanTaskRoutingStrategy.class,
        caseDefinition.getHumanTaskRouting());
final var routingCtx = new HumanTaskRoutingContext(
    caseInstance.getUuid(),
    binding.getName(),
    caseInstance.tenancyId,
    caseContext,
    experiences);
final var htCandidates = new HumanTaskCandidates(resolvedGroups, resolvedUsers);
```

Then call strategy and switch on result:
```java
return humanTaskStrategy.select(routingCtx, htCandidates)
    .chain(routingResult -> {
        final Set<String> finalGroups;
        final Set<String> finalUsers;
        final Map<String, Double> scores;
        switch (routingResult) {
            case HumanTaskRoutingResult.Enriched e -> {
                finalGroups = e.candidateGroups();
                finalUsers = e.candidateUsers();
                scores = e.candidateScores();
            }
            case HumanTaskRoutingResult.Unchanged u -> {
                finalGroups = resolvedGroups;
                finalUsers = resolvedUsers;
                scores = Map.of();
            }
            case HumanTaskRoutingResult.Escalated e -> {
                LOG.warnf("HumanTask routing escalated for caseId=%s binding=%s: %s",
                    caseInstance.getUuid(), binding.getName(), e.reason());
                finalGroups = resolvedGroups;
                finalUsers = resolvedUsers;
                scores = Map.of();
            }
        }
        // publish event with finalGroups, finalUsers, scores, experiences
        eventBus.publish(EventBusAddresses.HUMAN_TASK_SCHEDULE,
            new HumanTaskScheduleEvent(
                caseInstance.getUuid(), caseInstance.tenancyId,
                binding.getName(), target, inputData,
                payloadTypeName, resolutionTypeName,
                finalGroups, finalUsers,
                caseBudgetDeadline, expiresAtDeadline,
                resolvedTitle, resolvedScope, resolvedExpiresIn,
                experiences, scores));
        return Uni.createFrom().voidItem();
    });
```

- [ ] **Step 6: Fix existing test event recorders**

`HumanTaskTargetDispatchTest.HumanTaskEventRecorder` — add the two new fields
to the `HumanTaskScheduleEvent` construction (null/empty defaults).

`HumanTaskTypedContextTest.TypedEventRecorder` — same update.

- [ ] **Step 7: Run tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest="HumanTaskTargetDispatchTest,HumanTaskRoutingStrategyWiringTest" -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add common/src runtime/src
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(#741): thread experiences through humanTask dispatch, call HumanTaskRoutingStrategy

Refs #741"
```

---

### Task 6: Retention — `CbrCaseRetainObserver` generalisation

Generalises `buildCapabilityNameMap()` to include humanTask bindings so their
PlanItems are stored in CBR plan traces.

**Files:**
- Modify: `engine/runtime/src/main/java/io/casehub/engine/internal/memory/CbrCaseRetainObserver.java`
- Modify: `engine/runtime/src/test/java/io/casehub/engine/internal/memory/CbrCaseRetainObserverTest.java`

**Interfaces:**
- Consumes: `PlanTrace` with nullable `capabilityName` (from Task 1)
- Produces: plan traces that include humanTask PlanItems with null `capabilityName`

- [ ] **Step 1: Write failing test — humanTask PlanItems retained**

Add test in `CbrCaseRetainObserverTest.java` that creates a `CaseDefinition` with
a humanTask binding, PlanItemRecords for both CapabilityTarget and HumanTaskTarget
bindings, and asserts both appear in the stored `PlanCbrCase.planTrace()`.

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — humanTask PlanItems filtered out by `buildCapabilityNameMap`

- [ ] **Step 3: Rename and generalise `buildCapabilityNameMap`**

Rename to `buildRoutingKeyMap`. Add humanTask branch:

```java
private Map<String, String> buildRoutingKeyMap(CaseDefinition definition) {
    Map<String, String> map = new LinkedHashMap<>();
    for (Binding binding : definition.getBindings()) {
        switch (binding.target()) {
            case CapabilityTarget ct -> map.put(binding.getName(), ct.capability().name());
            case HumanTaskTarget ht -> map.put(binding.getName(), null);
            default -> { /* SubCase, Extension — not retained */ }
        }
    }
    return map;
}
```

- [ ] **Step 4: Run full CbrCaseRetainObserverTest**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=CbrCaseRetainObserverTest -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add runtime/src
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(#741): generalize CbrCaseRetainObserver to include humanTask PlanItems in plan traces

Refs #741"
```

---

### Task 7: Build verification + CLAUDE.md update

Full build, diagnostics, update CLAUDE.md with new SPI documentation.

**Files:**
- Modify: `engine/CLAUDE.md` (add HumanTaskRoutingStrategy section)

- [ ] **Step 1: Full build**

```bash
mvn install -DskipTests -q -f /Users/mdproctor/claude/casehub/engine/pom.xml
```

- [ ] **Step 2: Run all affected test suites**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api,runtime -f /Users/mdproctor/claude/casehub/engine/pom.xml
```

- [ ] **Step 3: IntelliJ diagnostics**

Run `ide_diagnostics` on each modified file to verify no compilation errors.

- [ ] **Step 4: Update CLAUDE.md**

Add section documenting the new SPI, its convention, default implementation,
and integration point in the handler pipeline.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add CLAUDE.md
git -C /Users/mdproctor/claude/casehub/engine commit -m "docs(#741): document HumanTaskRoutingStrategy SPI in CLAUDE.md

Refs #741"
```
