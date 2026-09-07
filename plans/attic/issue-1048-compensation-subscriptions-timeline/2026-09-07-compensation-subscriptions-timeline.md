# Compensation Subscriptions + Enriched Timeline Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1048 — saga: compensation GraphQL subscriptions + enriched timeline for ops dashboard
**Issue group:** #1048

**Goal:** Add real-time step-level compensation subscription, restructure the timeline with grouped retry attempts, enrich faulted steps with error detail, and link sub-case compensation.

**Architecture:** A new `CompensationStepEvent` CDI event in engine-common bridges the existing EventLog writes in `CaseCompensationServiceImpl` to `CaseEventPublisher`'s Multi stream infrastructure. A new `compensationProgress` GraphQL subscription in engine-graphql consumes this stream. The timeline query is restructured from a flat step list to grouped `CompensationAttemptType` records, with error enrichment from `_diagnostics` and sub-case linkage from EventLog metadata.

**Tech Stack:** Quarkus CDI, SmallRye GraphQL (subscriptions via WebSocket), SmallRye Mutiny Multi, Jackson

## Global Constraints

- All changes in `engine-common` (CDI event) and `engine-graphql` (DTOs, subscription, query). No other modules modified except `engine-planning` (one CDI injection added to `CaseCompensationServiceImpl`).
- Pre-release: breaking changes to `CompensationTimelineType` are acceptable.
- GraphQL types use `@Type` annotations from `org.eclipse.microprofile.graphql`.
- Tests use JUnit 5 + AssertJ + Mockito (unit) following existing patterns in `CaseEventPublisherTest` and `CaseQueryResolverTest`.
- `BackPressureStrategy.DROP` for all subscription emitters (matches existing pattern).

---

## Batch 1: Subscription Infrastructure

### Task 1: CompensationStepEvent CDI event + CaseEventPublisher stream

**Files:**
- Create: `common/src/main/java/io/casehub/engine/common/internal/event/CompensationStepEvent.java`
- Modify: `graphql/src/main/java/io/casehub/engine/graphql/CaseEventPublisher.java`
- Modify: `graphql/src/test/java/io/casehub/engine/graphql/CaseEventPublisherTest.java`

**Interfaces:**
- Produces: `CompensationStepEvent(UUID caseId, String tenancyId, CaseHubEventType eventType, String originalBindingName, String compensatingBindingName, Instant timestamp)` — CDI event consumed by `CaseEventPublisher`
- Produces: `CaseEventPublisher.compensationStepStream() → Multi<CompensationStepEvent>` — consumed by `CaseSubscriptionResolver` (Task 2)

- [ ] **Step 1: Write failing test for CompensationStepEvent reaching subscriber**

Add to `CaseEventPublisherTest.java`:

```java
@Test
void compensation_step_event_reaches_subscriber() {
  AssertSubscriber<CompensationStepEvent> subscriber =
      publisher.compensationStepStream().subscribe().withSubscriber(AssertSubscriber.create(10));

  CompensationStepEvent event = new CompensationStepEvent(
      UUID.randomUUID(), "tenant-1", CaseHubEventType.COMPENSATION_STEP_STARTED,
      "irb-review", "irb-review-reversal", Instant.now());
  publisher.onCompensationStepEvent(event);

  subscriber.awaitItems(1, Duration.ofSeconds(1));
  assertThat(subscriber.getItems()).containsExactly(event);
}

@Test
void disconnected_subscriber_stops_receiving_compensation_events() {
  AssertSubscriber<CompensationStepEvent> subscriber =
      publisher.compensationStepStream().subscribe().withSubscriber(AssertSubscriber.create(10));

  CompensationStepEvent event1 = new CompensationStepEvent(
      UUID.randomUUID(), "tenant-1", CaseHubEventType.COMPENSATION_STEP_STARTED,
      "data-export", "data-export-cleanup", Instant.now());
  publisher.onCompensationStepEvent(event1);
  subscriber.awaitItems(1, Duration.ofSeconds(1));

  subscriber.cancel();

  CompensationStepEvent event2 = new CompensationStepEvent(
      UUID.randomUUID(), "tenant-1", CaseHubEventType.COMPENSATION_STEP_COMPLETED,
      "data-export", "data-export-cleanup", Instant.now());
  publisher.onCompensationStepEvent(event2);

  assertThat(subscriber.getItems()).hasSize(1);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl graphql -Dtest=CaseEventPublisherTest -Dsurefire.failIfNoSpecifiedTests=false -q`
Expected: compilation errors — `CompensationStepEvent` and `compensationStepStream()` don't exist yet.

- [ ] **Step 3: Create CompensationStepEvent record**

Create `common/src/main/java/io/casehub/engine/common/internal/event/CompensationStepEvent.java`:

```java
package io.casehub.engine.common.internal.event;

import io.casehub.api.model.event.CaseHubEventType;
import java.time.Instant;
import java.util.UUID;

public record CompensationStepEvent(
    UUID caseId,
    String tenancyId,
    CaseHubEventType eventType,
    String originalBindingName,
    String compensatingBindingName,
    Instant timestamp) {}
```

- [ ] **Step 4: Add third emitter stream to CaseEventPublisher**

Add to `CaseEventPublisher.java`:
- New field: `private final List<MultiEmitter<? super CompensationStepEvent>> compensationStepEmitters = new CopyOnWriteArrayList<>();`
- New observer: `void onCompensationStepEvent(@ObservesAsync CompensationStepEvent event)` — same pattern as `onLifecycleEvent`
- New stream: `public Multi<CompensationStepEvent> compensationStepStream()` — same pattern as `lifecycleStream()`

Import: `io.casehub.engine.common.internal.event.CompensationStepEvent`

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn test -pl graphql -Dtest=CaseEventPublisherTest -q`
Expected: all tests PASS including the two new ones.

- [ ] **Step 6: Commit**

```bash
git add common/src/main/java/io/casehub/engine/common/internal/event/CompensationStepEvent.java graphql/src/main/java/io/casehub/engine/graphql/CaseEventPublisher.java graphql/src/test/java/io/casehub/engine/graphql/CaseEventPublisherTest.java
git commit -m "feat(#1048): add CompensationStepEvent and publisher stream Refs #1048"
```

### Task 2: CompensationProgressEventType DTO + GraphQL subscription

**Files:**
- Create: `graphql/src/main/java/io/casehub/engine/graphql/dto/CompensationProgressEventType.java`
- Modify: `graphql/src/main/java/io/casehub/engine/graphql/CaseSubscriptionResolver.java`
- Modify: `graphql/src/test/java/io/casehub/engine/graphql/CaseSubscriptionResolverTest.java`

**Interfaces:**
- Consumes: `CaseEventPublisher.compensationStepStream() → Multi<CompensationStepEvent>` (from Task 1)
- Produces: `CaseSubscriptionResolver.compensationProgress(UUID caseId) → Multi<CompensationProgressEventType>` — GraphQL subscription endpoint

- [ ] **Step 1: Write failing test for subscription filtering**

Add to `CaseSubscriptionResolverTest.java`:

```java
@Test
void compensationProgress_filters_by_caseId() {
  UUID targetCaseId = UUID.randomUUID();
  UUID otherCaseId = UUID.randomUUID();

  AssertSubscriber<CompensationProgressEventType> subscriber =
      resolver.compensationProgress(targetCaseId)
          .subscribe().withSubscriber(AssertSubscriber.create(10));

  publisher.onCompensationStepEvent(new CompensationStepEvent(
      targetCaseId, "tenant-1", CaseHubEventType.COMPENSATION_STEP_STARTED,
      "irb-review", "irb-review-reversal", Instant.now()));
  publisher.onCompensationStepEvent(new CompensationStepEvent(
      otherCaseId, "tenant-1", CaseHubEventType.COMPENSATION_STEP_COMPLETED,
      "data-export", "data-export-cleanup", Instant.now()));

  subscriber.awaitItems(1, Duration.ofSeconds(1));
  assertThat(subscriber.getItems()).hasSize(1);
  assertThat(subscriber.getItems().get(0).caseId()).isEqualTo(targetCaseId);
  assertThat(subscriber.getItems().get(0).eventType()).isEqualTo("COMPENSATION_STEP_STARTED");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl graphql -Dtest=CaseSubscriptionResolverTest#compensationProgress_filters_by_caseId -q`
Expected: compilation error — `CompensationProgressEventType` and `compensationProgress()` don't exist.

- [ ] **Step 3: Create CompensationProgressEventType DTO**

Create `graphql/src/main/java/io/casehub/engine/graphql/dto/CompensationProgressEventType.java`:

```java
package io.casehub.engine.graphql.dto;

import io.casehub.engine.common.internal.event.CompensationStepEvent;
import java.time.Instant;
import java.util.UUID;
import org.eclipse.microprofile.graphql.Type;

@Type("CompensationProgressEvent")
public record CompensationProgressEventType(
    UUID caseId,
    String eventType,
    String originalBindingName,
    String compensatingBindingName,
    Instant timestamp) {

  public static CompensationProgressEventType from(CompensationStepEvent event) {
    return new CompensationProgressEventType(
        event.caseId(),
        event.eventType().name(),
        event.originalBindingName(),
        event.compensatingBindingName(),
        event.timestamp());
  }
}
```

- [ ] **Step 4: Add compensationProgress subscription to CaseSubscriptionResolver**

Add to `CaseSubscriptionResolver.java`:

```java
@Subscription
@Description("Live compensation step progress — step starts and completions")
public Multi<CompensationProgressEventType> compensationProgress(@Name("caseId") UUID caseId) {
  return publisher
      .compensationStepStream()
      .filter(event -> event.caseId().equals(caseId))
      .map(CompensationProgressEventType::from);
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn test -pl graphql -Dtest=CaseSubscriptionResolverTest -q`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add graphql/src/main/java/io/casehub/engine/graphql/dto/CompensationProgressEventType.java graphql/src/main/java/io/casehub/engine/graphql/CaseSubscriptionResolver.java graphql/src/test/java/io/casehub/engine/graphql/CaseSubscriptionResolverTest.java
git commit -m "feat(#1048): add compensationProgress GraphQL subscription Refs #1048"
```

### Task 3: Fire CompensationStepEvent from CaseCompensationServiceImpl

**Files:**
- Modify: `planning/src/main/java/io/casehub/engine/planning/compensation/CaseCompensationServiceImpl.java`
- Modify: `planning/src/test/java/io/casehub/engine/planning/compensation/CaseCompensationServiceImplTest.java`

**Interfaces:**
- Consumes: `CompensationStepEvent` record (from Task 1)
- Produces: CDI `fireAsync(CompensationStepEvent)` calls that reach `CaseEventPublisher` (Task 1)

- [ ] **Step 1: Write failing test for CDI event firing**

Add to `CaseCompensationServiceImplTest.java` a test verifying `appendStepEvent` fires a CDI event. The test needs a recording CDI `Event<CompensationStepEvent>` injected into the service. Use a `List<CompensationStepEvent>` capture field:

```java
@Test
void appendStepEvent_fires_cdi_event() {
  // Trigger a compensation step that completes — this calls appendStepEvent internally
  // Setup: case in COMPENSATING state, one completed compensation PlanItem
  // Verify: firedEvents list contains a CompensationStepEvent with correct fields
  assertThat(firedCompensationStepEvents).hasSize(1);
  var fired = firedCompensationStepEvents.get(0);
  assertThat(fired.eventType()).isEqualTo(CaseHubEventType.COMPENSATION_STEP_COMPLETED);
  assertThat(fired.originalBindingName()).isEqualTo("irb-review");
  assertThat(fired.compensatingBindingName()).isEqualTo("irb-review-reversal");
}
```

The exact test setup depends on the existing test patterns in `CaseCompensationServiceImplTest` — follow the established fixture pattern for creating a COMPENSATING case with a compensation PlanItem.

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl planning -Dtest=CaseCompensationServiceImplTest#appendStepEvent_fires_cdi_event -q`
Expected: FAIL — no CDI event fired yet.

- [ ] **Step 3: Add CDI Event injection and fire in appendStepEvent**

Modify `CaseCompensationServiceImpl`:
1. Add field: `private final Event<CompensationStepEvent> compensationStepEvent;`
2. Add to constructor: `@Inject Event<CompensationStepEvent> compensationStepEvent` parameter
3. In `appendStepEvent()`, after the EventLog write, add:

```java
compensationStepEvent.fireAsync(new CompensationStepEvent(
    instance.getUuid(),
    instance.tenancyId,
    type,
    originalBindingName,
    compensatingBindingName,
    Instant.now()));
```

Import: `jakarta.enterprise.event.Event`, `io.casehub.engine.common.internal.event.CompensationStepEvent`

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl planning -Dtest=CaseCompensationServiceImplTest -q`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add planning/src/main/java/io/casehub/engine/planning/compensation/CaseCompensationServiceImpl.java planning/src/test/java/io/casehub/engine/planning/compensation/CaseCompensationServiceImplTest.java
git commit -m "feat(#1048): fire CompensationStepEvent CDI event from compensation coordinator Refs #1048"
```

---

## Batch 2: Enriched Timeline

### Task 4: CompensationStepType error enrichment + CompensationAttemptType

**Files:**
- Modify: `graphql/src/main/java/io/casehub/engine/graphql/dto/CompensationStepType.java`
- Create: `graphql/src/main/java/io/casehub/engine/graphql/dto/CompensationAttemptType.java`
- Modify: `graphql/src/main/java/io/casehub/engine/graphql/dto/CompensationTimelineType.java`

**Interfaces:**
- Produces: `CompensationStepType(planItemId, bindingName, targetType, status, createdAt, completedAt, compensatesBinding, compensatesItemId, errorReason, failureCategory)` — 10-field record
- Produces: `CompensationAttemptType(attemptNumber, startedAt, completedAt, outcome, triggeredBy, reason, List<CompensationStepType> steps)` — new DTO
- Produces: `CompensationTimelineType(caseId, status, List<TimelineStepType> forwardSteps, List<CompensationAttemptType> attempts, List<UUID> childCompensationCaseIds)` — restructured

- [ ] **Step 1: Write tests for new record construction**

Create `graphql/src/test/java/io/casehub/engine/graphql/dto/CompensationAttemptTypeTest.java`:

```java
package io.casehub.engine.graphql.dto;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.util.List;
import org.junit.jupiter.api.Test;

class CompensationAttemptTypeTest {

  @Test
  void constructs_with_all_fields() {
    var step = new CompensationStepType(
        "pi-1", "reversal", "capability", "COMPLETED",
        Instant.now(), Instant.now(), "original", "pi-0", null, null);
    var attempt = new CompensationAttemptType(
        1, Instant.now(), Instant.now(), "COMPLETED", "operator-1", "trial withdrawn", List.of(step));

    assertThat(attempt.attemptNumber()).isEqualTo(1);
    assertThat(attempt.outcome()).isEqualTo("COMPLETED");
    assertThat(attempt.steps()).hasSize(1);
  }

  @Test
  void in_progress_attempt_has_null_completedAt() {
    var attempt = new CompensationAttemptType(
        1, Instant.now(), null, "IN_PROGRESS", "operator-1", "correction", List.of());

    assertThat(attempt.completedAt()).isNull();
    assertThat(attempt.outcome()).isEqualTo("IN_PROGRESS");
  }
}
```

And add a test for the enriched `CompensationStepType` in the same directory:

```java
// In a new file or added to existing test
@Test
void faulted_step_carries_error_fields() {
  var step = new CompensationStepType(
      "pi-1", "reversal", "capability", "FAULTED",
      Instant.now(), Instant.now(), "original", "pi-0",
      "Agent declined: missing access credentials", "Knowledge");

  assertThat(step.errorReason()).isEqualTo("Agent declined: missing access credentials");
  assertThat(step.failureCategory()).isEqualTo("Knowledge");
}

@Test
void successful_step_has_null_error_fields() {
  var step = new CompensationStepType(
      "pi-1", "reversal", "capability", "COMPLETED",
      Instant.now(), Instant.now(), "original", "pi-0", null, null);

  assertThat(step.errorReason()).isNull();
  assertThat(step.failureCategory()).isNull();
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl graphql -Dtest=CompensationAttemptTypeTest -q`
Expected: compilation errors — `CompensationAttemptType` doesn't exist, `CompensationStepType` has wrong arity.

- [ ] **Step 3: Add errorReason and failureCategory to CompensationStepType**

Replace `CompensationStepType.java` record with 10 fields:

```java
@Type("CompensationStep")
public record CompensationStepType(
    String planItemId,
    String bindingName,
    String targetType,
    String status,
    Instant createdAt,
    Instant completedAt,
    String compensatesBinding,
    String compensatesItemId,
    String errorReason,
    String failureCategory) {}
```

- [ ] **Step 4: Create CompensationAttemptType DTO**

Create `graphql/src/main/java/io/casehub/engine/graphql/dto/CompensationAttemptType.java`:

```java
package io.casehub.engine.graphql.dto;

import java.time.Instant;
import java.util.List;
import org.eclipse.microprofile.graphql.Type;

@Type("CompensationAttempt")
public record CompensationAttemptType(
    int attemptNumber,
    Instant startedAt,
    Instant completedAt,
    String outcome,
    String triggeredBy,
    String reason,
    List<CompensationStepType> steps) {}
```

- [ ] **Step 5: Restructure CompensationTimelineType**

Replace `CompensationTimelineType.java` record:

```java
@Type("CompensationTimeline")
public record CompensationTimelineType(
    UUID caseId,
    String status,
    List<TimelineStepType> forwardSteps,
    List<CompensationAttemptType> attempts,
    List<UUID> childCompensationCaseIds) {}
```

- [ ] **Step 6: Fix compilation in CaseQueryResolver**

The existing `compensationTimeline()` method constructs the old `CompensationTimelineType`. It will not compile after the restructure. For now, update the constructor call to pass `List.of()` for `attempts` and `childCompensationCaseIds`, and `List.of()` for the removed fields. This is a temporary compilation fix — Task 5 rewrites the assembly logic.

```java
return new CompensationTimelineType(
    caseId,
    instance.getState().name(),
    forwardSteps,
    List.of(),
    List.of());
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `mvn test -pl graphql -Dtest="CompensationAttemptTypeTest,CaseQueryResolverTest,CaseEventPublisherTest,CaseSubscriptionResolverTest" -q`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add graphql/src/main/java/io/casehub/engine/graphql/dto/CompensationStepType.java graphql/src/main/java/io/casehub/engine/graphql/dto/CompensationAttemptType.java graphql/src/main/java/io/casehub/engine/graphql/dto/CompensationTimelineType.java graphql/src/main/java/io/casehub/engine/graphql/CaseQueryResolver.java graphql/src/test/java/io/casehub/engine/graphql/dto/CompensationAttemptTypeTest.java
git commit -m "feat(#1048): enrich CompensationStepType with error fields and add CompensationAttemptType Refs #1048"
```

### Task 5: Rewrite timeline assembly with attempt grouping, error enrichment, and sub-case linkage

**Files:**
- Modify: `graphql/src/main/java/io/casehub/engine/graphql/CaseQueryResolver.java` (method: `compensationTimeline`)
- Create: `graphql/src/test/java/io/casehub/engine/graphql/CompensationTimelineAssemblyTest.java`

**Interfaces:**
- Consumes: `CompensationAttemptType`, `CompensationStepType` (from Task 4)
- Consumes: `CompensationTimelineType` (restructured, from Task 4)
- Consumes: `CaseHubRuntime.eventLog(UUID, Set<CaseHubEventType>)` → `List<EventLog>`
- Consumes: `PlanItemStore.findByCaseId(UUID, String)` → `List<PlanItemRecord>`
- Consumes: `CaseHubRuntime.caseContext(UUID)` → `CaseContext` (for `_diagnostics`)

- [ ] **Step 1: Write tests for attempt grouping**

Create `graphql/src/test/java/io/casehub/engine/graphql/CompensationTimelineAssemblyTest.java`. This tests the assembly logic by mocking `CaseHubRuntime`, `CaseInstanceRepository`, `CaseDefinitionRegistry`, `PlanItemStore`, and `CurrentPrincipal`.

```java
package io.casehub.engine.graphql;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.*;

import io.casehub.api.model.Binding;
import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.event.CaseHubEventType;
import io.casehub.engine.common.internal.history.EventLog;
import io.casehub.engine.common.internal.history.EventStreamType;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.common.internal.model.CaseMetaModel;
import io.casehub.engine.common.internal.model.PlanItemRecord;
import io.casehub.api.model.CaseStatus;
import io.casehub.api.model.TaskStatus;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import java.time.Instant;
import java.util.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class CompensationTimelineAssemblyTest {
  // Test fixture setup with mocked dependencies
  // Tests cover:

  @Test
  void single_attempt_groups_all_steps() {
    // Setup: 1 COMPENSATION_STARTED, 2 STEP_STARTED, 2 STEP_COMPLETED, 1 COMPENSATION_COMPLETED
    // Verify: 1 attempt with 2 steps, outcome COMPLETED
  }

  @Test
  void two_attempts_groups_steps_by_time_window() {
    // Setup: attempt 1 starts at T1, step A starts at T2, step B faults at T3
    //        attempt 2 starts at T4, step B starts at T5, step A starts at T6, both complete
    // Verify: attempt 1 has 2 steps (A completed, B faulted), attempt 2 has 2 steps
  }

  @Test
  void in_progress_attempt_has_null_completedAt() {
    // Setup: 1 COMPENSATION_STARTED, 1 STEP_STARTED, no COMPLETED/FAULTED
    // Verify: 1 attempt with outcome IN_PROGRESS, completedAt null
  }

  @Test
  void faulted_step_enriched_with_error_from_diagnostics() {
    // Setup: FAULTED step with _diagnostics context containing latestDiagnosis
    // Verify: step has errorReason and failureCategory populated
  }

  @Test
  void child_compensation_case_ids_populated() {
    // Setup: COMPENSATION_STEP_STARTED EventLog with childCaseId in metadata
    // Verify: childCompensationCaseIds contains the child UUID
  }

  @Test
  void no_compensation_events_returns_null() {
    // Same as before — no COMPENSATION_STARTED in EventLog
    // Verify: returns null
  }
}
```

Each test needs full fixture construction following the patterns in the existing `CaseQueryResolverTest`. The test bodies are detailed because they exercise the grouping logic — build EventLog lists with specific timestamps, PlanItemRecords with matching bindingNames, and mock `_diagnostics` context for error enrichment.

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl graphql -Dtest=CompensationTimelineAssemblyTest -q`
Expected: FAIL — the assembly logic still returns empty attempts.

- [ ] **Step 3: Rewrite compensationTimeline() in CaseQueryResolver**

Replace the `compensationTimeline()` method body with the new assembly logic:

1. Query EventLog for all 5 compensation event types
2. If no `COMPENSATION_STARTED` found, return null
3. Group `COMPENSATION_STARTED` entries chronologically — each starts a new attempt
4. For each attempt, find the matching `COMPENSATION_COMPLETED` or `COMPENSATION_FAULTED` entry (timestamp after this attempt's start, before next attempt's start)
5. Query PlanItemStore — partition items into forward (non-compensation) and compensation groups
6. Assign compensation PlanItems to attempts by `createdAt` timestamp window
7. For FAULTED steps, read `_diagnostics.<bindingName>.latestDiagnosis` from case context via `runtime.caseContext(caseId)` and extract `category` and error reason
8. Collect `childCaseId` values from `COMPENSATION_STEP_STARTED` EventLog metadata entries that contain a `childCaseId` field
9. Construct and return the restructured `CompensationTimelineType`

The method needs a new injection: `@Inject CaseHubRuntime runtime` (already present as field on CaseQueryResolver).

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -pl graphql -Dtest=CompensationTimelineAssemblyTest -q`
Expected: PASS

- [ ] **Step 5: Run full graphql module tests**

Run: `mvn test -pl graphql -q`
Expected: all existing tests PASS (BindingGraphProjectionTest, CaseEventPublisherTest, CaseSubscriptionResolverTest, CompensationAttemptTypeTest, plus new tests)

- [ ] **Step 6: Commit**

```bash
git add graphql/src/main/java/io/casehub/engine/graphql/CaseQueryResolver.java graphql/src/test/java/io/casehub/engine/graphql/CompensationTimelineAssemblyTest.java
git commit -m "feat(#1048): rewrite timeline assembly with attempt grouping, error enrichment, sub-case linkage Refs #1048"
```

---

## Batch 3: Cross-Module Verification

### Task 6: Full build verification and cleanup

**Files:**
- No new files — verification only

**Interfaces:**
- Consumes: all previous tasks

- [ ] **Step 1: Install engine-common to local repo (for CompensationStepEvent)**

Run: `mvn install -pl common -DskipTests -q`

- [ ] **Step 2: Run full build across affected modules**

Run: `mvn clean test -pl common,planning,graphql -q`
Expected: all tests PASS

- [ ] **Step 3: Verify GraphQL schema generation**

Run: `mvn compile -pl graphql -q` and check that `target/classes/META-INF/microprofile-graphql.json` or schema output includes:
- `compensationProgress` subscription
- `CompensationProgressEvent` type
- `CompensationAttempt` type with `steps` field
- `CompensationStep` type with `errorReason` and `failureCategory` fields
- `CompensationTimeline` type with `attempts` and `childCompensationCaseIds` fields

- [ ] **Step 4: Commit any final adjustments**

If schema or compilation adjustments were needed, commit them:
```bash
git add -A
git commit -m "chore(#1048): cross-module verification fixes Refs #1048"
```

---

## References

- `2026-09-07-compensation-subscriptions-enriched-timeline-design.md` — design spec
- `CaseSubscriptionResolver.java:31-57` — existing subscription pattern
- `CaseEventPublisher.java:27-65` — CDI observer → Multi emitter pattern
- `CaseEventPublisherTest.java:31-121` — test patterns (AssertSubscriber, Duration.ofSeconds)
- `CaseCompensationServiceImpl.java:378-393` — appendStepEvent (EventLog write site)
- `CaseCompensationServiceImpl.java:63-89` — constructor injection pattern
- `CompensationStepType.java:22-30` — current 8-field record
- `CompensationTimelineType.java:24-32` — current flat structure
- `CaseQueryResolver.java:157-268` — current timeline assembly
- `PlanItemRecord.java:22-80` — record fields available for timeline
- `CaseHubEventType.java:101-105` — 5 compensation event type constants
- D1-D4 in decisions.md — design decisions
- casehubio/engine#1048 — issue
