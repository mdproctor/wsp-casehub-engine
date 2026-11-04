# Multi-Approver Oversight Gate Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #810 — Multi-approver OversightGateService — M-of-N approval for high-risk actions
**Issue group:** #810

**Goal:** Bridge engine's `RiskDecision.GateRequired` to work repo's `WorkItemSpawnGroup` for M-of-N multi-party approval of high-risk actions.

**Architecture:** Three-tier change. Engine-api owns `QuorumConfig` (governance semantics). Work-api extends `WorkItemCreator` with `createMultiInstance()` (mechanism). Work engine-adapter translates between the two — mapping `QuorumConfig` → `MultiInstanceConfig` on creation, and routing `WorkItemGroupLifecycleEvent` to gate events on completion.

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI, Vert.x EventBus

## Global Constraints

- Engine-api must not depend on work-api — `QuorumConfig` uses engine's `OnThresholdReached`, not work's
- Null quorum = single-approver (backward compatible, no existing tests may break)
- Children must NOT carry gate callerRef — only the parent does
- All `@Nullable` annotations use `org.jspecify.annotations.Nullable`
- Issue linkage: all commits reference `Refs #810`

---

### Task 1: `QuorumConfig` record + `RiskDecision.GateRequired` extension (engine-api)

**Files:**
- Create: `api/src/main/java/io/casehub/api/spi/QuorumConfig.java`
- Modify: `api/src/main/java/io/casehub/api/spi/RiskDecision.java:37-44` (add quorum field to GateRequired)
- Test: `api/src/test/java/io/casehub/api/spi/QuorumConfigTest.java`

**Interfaces:**
- Consumes: `io.casehub.api.model.OnThresholdReached` (existing enum: KEEP, CANCEL)
- Produces: `QuorumConfig(int instances, int required, @Nullable OnThresholdReached onThresholdReached, boolean allowSameAssignee)` — used by Task 2, Task 3, Task 4

- [ ] **Step 1: Write failing tests for QuorumConfig validation**

```java
package io.casehub.api.spi;

import io.casehub.api.model.OnThresholdReached;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class QuorumConfigTest {

    @Test
    void valid_config_created_successfully() {
        var config = new QuorumConfig(3, 2, null, false);
        assertEquals(3, config.instances());
        assertEquals(2, config.required());
        assertNull(config.onThresholdReached());
        assertFalse(config.allowSameAssignee());
    }

    @Test
    void instances_below_two_throws() {
        assertThrows(IllegalArgumentException.class, () -> new QuorumConfig(1, 1, null, false));
    }

    @Test
    void required_zero_throws() {
        assertThrows(IllegalArgumentException.class, () -> new QuorumConfig(3, 0, null, false));
    }

    @Test
    void required_exceeds_instances_throws() {
        assertThrows(IllegalArgumentException.class, () -> new QuorumConfig(3, 4, null, false));
    }

    @Test
    void required_equals_instances_is_unanimous() {
        var config = new QuorumConfig(3, 3, null, false);
        assertEquals(3, config.required());
    }

    @Test
    void onThresholdReached_cancel_accepted() {
        var config = new QuorumConfig(5, 3, OnThresholdReached.CANCEL, false);
        assertEquals(OnThresholdReached.CANCEL, config.onThresholdReached());
    }

    @Test
    void allowSameAssignee_true_accepted() {
        var config = new QuorumConfig(3, 2, null, true);
        assertTrue(config.allowSameAssignee());
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=QuorumConfigTest -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: FAIL — `QuorumConfig` does not exist

- [ ] **Step 3: Create `QuorumConfig` record**

Use `ide_create_file` to create `api/src/main/java/io/casehub/api/spi/QuorumConfig.java`:

```java
package io.casehub.api.spi;

import io.casehub.api.model.OnThresholdReached;
import org.jspecify.annotations.Nullable;

public record QuorumConfig(
    int instances,
    int required,
    @Nullable OnThresholdReached onThresholdReached,
    boolean allowSameAssignee) {

  public QuorumConfig {
    if (instances < 2) {
      throw new IllegalArgumentException("instances must be >= 2, got " + instances);
    }
    if (required < 1 || required > instances) {
      throw new IllegalArgumentException(
          "required must be 1.." + instances + ", got " + required);
    }
  }
}
```

- [ ] **Step 4: Add `quorum` field to `RiskDecision.GateRequired`**

Use `ide_edit_member` on `RiskDecision.java`, member `GateRequired`:

```java
  record GateRequired(
      String reason,
      boolean reversible,
      CandidateSetStrategy candidateGroups,
      Duration expiresIn,
      String scope,
      @Nullable Class<?> resolutionType,
      @Nullable QuorumConfig quorum)
      implements RiskDecision {}
```

- [ ] **Step 5: Fix all compilation errors from the new record field**

Run `ide_build_project` to find all call sites that construct `GateRequired` (they now need an 8th arg).
Use `ide_find_references` on `GateRequired` to locate all construction sites.
Add `null` as the 8th argument at each site (backward compat — null quorum = single-approver).

- [ ] **Step 6: Run tests to verify they pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=QuorumConfigTest -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: PASS

- [ ] **Step 7: Verify full api module compiles**

Run: `ide_build_project` with `project_path=/Users/mdproctor/claude/casehub/engine/api`
Expected: 0 errors

- [ ] **Step 8: Commit**

```
feat(#810): add QuorumConfig record and quorum field on RiskDecision.GateRequired

Refs #810
```

---

### Task 2: `ChainedActionRiskClassifier` quorum comparison (engine-api)

**Files:**
- Modify: `api/src/main/java/io/casehub/api/spi/ChainedActionRiskClassifier.java:86-96` (narrower method)
- Test: `api/src/test/java/io/casehub/api/spi/ChainedActionRiskClassifierTest.java` (add quorum tests)

**Interfaces:**
- Consumes: `QuorumConfig` from Task 1
- Produces: Updated `narrower()` with quorum-aware comparison

- [ ] **Step 1: Write failing tests for quorum comparison**

Add these test methods to `ChainedActionRiskClassifierTest.java` using `ide_insert_member`:

```java
@Test
void twoClassifiers_quorumBeatsNoQuorum() {
    var withQuorum = new ActionRiskClassifier() {
        @Override
        public RiskDecision classify(PlannedAction action, ClassificationContext context) {
            return new RiskDecision.GateRequired(
                "needs multi-approval", false, null, null, null, null,
                new QuorumConfig(3, 2, null, false));
        }
    };
    var noQuorum = new ActionRiskClassifier() {
        @Override
        public RiskDecision classify(PlannedAction action, ClassificationContext context) {
            return new RiskDecision.GateRequired(
                "single approval", false, null, null, null, null, null);
        }
    };
    chain = new ChainedActionRiskClassifier(instanceOf(noQuorum, withQuorum));
    var result = (RiskDecision.GateRequired) chain.classify(anyAction(), anyContext());
    assertNotNull(result.quorum());
    assertEquals(2, result.quorum().required());
}

@Test
void twoClassifiers_higherRequiredWins() {
    var twoOfThree = new ActionRiskClassifier() {
        @Override
        public RiskDecision classify(PlannedAction action, ClassificationContext context) {
            return new RiskDecision.GateRequired(
                "2-of-3", false, null, null, null, null,
                new QuorumConfig(3, 2, null, false));
        }
    };
    var threeOfFive = new ActionRiskClassifier() {
        @Override
        public RiskDecision classify(PlannedAction action, ClassificationContext context) {
            return new RiskDecision.GateRequired(
                "3-of-5", false, null, null, null, null,
                new QuorumConfig(5, 3, null, false));
        }
    };
    chain = new ChainedActionRiskClassifier(instanceOf(twoOfThree, threeOfFive));
    var result = (RiskDecision.GateRequired) chain.classify(anyAction(), anyContext());
    assertEquals(3, result.quorum().required());
}

@Test
void twoClassifiers_equalRequired_lowerInstancesWins() {
    var twoOfTwo = new ActionRiskClassifier() {
        @Override
        public RiskDecision classify(PlannedAction action, ClassificationContext context) {
            return new RiskDecision.GateRequired(
                "2-of-2 (unanimous)", false, null, null, null, null,
                new QuorumConfig(2, 2, null, false));
        }
    };
    var twoOfFive = new ActionRiskClassifier() {
        @Override
        public RiskDecision classify(PlannedAction action, ClassificationContext context) {
            return new RiskDecision.GateRequired(
                "2-of-5", false, null, null, null, null,
                new QuorumConfig(5, 2, null, false));
        }
    };
    chain = new ChainedActionRiskClassifier(instanceOf(twoOfTwo, twoOfFive));
    var result = (RiskDecision.GateRequired) chain.classify(anyAction(), anyContext());
    assertEquals(2, result.quorum().instances());
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=ChainedActionRiskClassifierTest -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: FAIL — narrower() doesn't consider quorum

- [ ] **Step 3: Update `narrower()` with quorum comparison**

Use `ide_edit_member` on `ChainedActionRiskClassifier.java`, member `narrower`:

```java
  private GateRequired narrower(final GateRequired a, final GateRequired b) {
    final boolean aHasQuorum = a.quorum() != null;
    final boolean bHasQuorum = b.quorum() != null;
    if (aHasQuorum != bHasQuorum) return aHasQuorum ? a : b;
    if (aHasQuorum) {
      if (a.quorum().required() != b.quorum().required()) {
        return a.quorum().required() > b.quorum().required() ? a : b;
      }
      if (a.quorum().instances() != b.quorum().instances()) {
        return a.quorum().instances() < b.quorum().instances() ? a : b;
      }
    }
    final int sizeA = candidateSetSize(a.candidateGroups());
    final int sizeB = candidateSetSize(b.candidateGroups());
    if (sizeA != sizeB) return sizeA < sizeB ? a : b;
    if (a.expiresIn() != null && b.expiresIn() != null) {
      return a.expiresIn().compareTo(b.expiresIn()) <= 0 ? a : b;
    }
    if (a.expiresIn() != null) return a;
    if (b.expiresIn() != null) return b;
    return a;
  }
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=ChainedActionRiskClassifierTest -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: PASS (all existing + new tests)

- [ ] **Step 5: Commit**

```
feat(#810): quorum-aware narrower() in ChainedActionRiskClassifier

Refs #810
```

---

### Task 3: Nullability annotations on gate event records (engine-common)

**Files:**
- Modify: `common/src/main/java/io/casehub/engine/common/internal/event/ActionGateApprovedEvent.java:21` (add @Nullable to approvedBy, workItemResolution)
- Modify: `common/src/main/java/io/casehub/engine/common/internal/event/ActionGateRejectedEvent.java:35` (add @Nullable to rejectedBy, workItemResolution)

**Interfaces:**
- Consumes: nothing new
- Produces: Nullable-correct event records — used by Task 5

- [ ] **Step 1: Add @Nullable to ActionGateApprovedEvent fields**

Use `ide_edit_member` on `ActionGateApprovedEvent.java`, member `ActionGateApprovedEvent`:

```java
public record ActionGateApprovedEvent(
    UUID caseId,
    String tenancyId,
    long gateId,
    @Nullable String workItemResolution,
    @Nullable String approvedBy,
    @Nullable String resolutionTypeName) {}
```

- [ ] **Step 2: Add @Nullable to ActionGateRejectedEvent fields**

Use `ide_edit_member` on `ActionGateRejectedEvent.java`, member `ActionGateRejectedEvent`:

```java
public record ActionGateRejectedEvent(
    UUID caseId, String tenancyId, long gateId,
    @Nullable String workItemResolution, @Nullable String rejectedBy) {}
```

- [ ] **Step 3: Verify compilation**

Run: `ide_build_project` with `project_path=/Users/mdproctor/claude/casehub/engine/common`
Expected: 0 errors

- [ ] **Step 4: Commit**

```
fix(#810): add @Nullable to gate event actor and resolution fields

Refs #810
```

---

### Task 4: `WorkItemCreator.createMultiInstance()` SPI + implementation (work repo)

**Files:**
- Modify: `api/src/main/java/io/casehub/work/api/spi/WorkItemCreator.java:8` (add default method)
- Modify: `runtime/src/main/java/io/casehub/work/runtime/spi/WorkItemSpiAdapter.java:26` (implement createMultiInstance)
- Test: `runtime/src/test/java/io/casehub/work/runtime/spi/WorkItemSpiAdapterMultiInstanceTest.java`

**Interfaces:**
- Consumes: `WorkItemCreateRequest`, `MultiInstanceConfig`, `WorkItemRef`, `InstanceAssignmentStrategy`, `WorkItemSpawnGroup`, `WorkItemService`
- Produces: `WorkItemCreator.createMultiInstance(WorkItemCreateRequest, MultiInstanceConfig) → WorkItemRef` — used by Task 5

- [ ] **Step 1: Add default method to `WorkItemCreator` interface**

Use `ide_insert_member` on `WorkItemCreator.java`, position `after`, anchor `obsoleteByCallerRef`:

```java
default WorkItemRef createMultiInstance(WorkItemCreateRequest parentRequest,
                                       MultiInstanceConfig config) {
    throw new UnsupportedOperationException("Multi-instance creation not supported");
}
```

- [ ] **Step 2: Write failing test for createMultiInstance**

Create test file `runtime/src/test/java/io/casehub/work/runtime/spi/WorkItemSpiAdapterMultiInstanceTest.java`:

```java
package io.casehub.work.runtime.spi;

import io.casehub.work.api.MultiInstanceConfig;
import io.casehub.work.api.WorkItemCreateRequest;
import io.casehub.work.api.WorkItemRef;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
class WorkItemSpiAdapterMultiInstanceTest {

    @Inject WorkItemSpiAdapter adapter;

    @Test
    void createMultiInstance_creates_parent_and_children() {
        var request = WorkItemCreateRequest.builder()
                .title("Approve SAR filing")
                .candidateGroups("compliance-officers")
                .createdBy("casehub-engine")
                .callerRef("case:test-case-id/gate:1")
                .tenancyId("test-tenant")
                .build();

        var config = new MultiInstanceConfig(
                3, 2, null, "pool", null, false, null);

        WorkItemRef parentRef = adapter.createMultiInstance(request, config);

        assertNotNull(parentRef);
        assertNotNull(parentRef.id());
    }

    @Test
    void createMultiInstance_idempotent_on_same_callerRef() {
        var request = WorkItemCreateRequest.builder()
                .title("Approve containment")
                .candidateGroups("ir-leads")
                .createdBy("casehub-engine")
                .callerRef("case:idempotent-test/gate:99")
                .tenancyId("test-tenant")
                .build();

        var config = new MultiInstanceConfig(
                3, 2, null, "pool", null, false, null);

        WorkItemRef first = adapter.createMultiInstance(request, config);
        WorkItemRef second = adapter.createMultiInstance(request, config);

        assertEquals(first.id(), second.id());
    }

    @Test
    void createMultiInstance_children_have_no_callerRef() {
        var request = WorkItemCreateRequest.builder()
                .title("Approve treatment")
                .candidateGroups("attending-physicians")
                .createdBy("casehub-engine")
                .callerRef("case:no-ref-test/gate:42")
                .tenancyId("test-tenant")
                .build();

        var config = new MultiInstanceConfig(
                3, 2, null, "pool", null, false, null);

        adapter.createMultiInstance(request, config);

        // Children should NOT have gate callerRef — verify via the parent's children query
        // The exact verification depends on the work-engine's child query API
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=WorkItemSpiAdapterMultiInstanceTest -f /Users/mdproctor/claude/casehub/work/pom.xml`
Expected: FAIL — `createMultiInstance` throws `UnsupportedOperationException`

- [ ] **Step 4: Implement `createMultiInstance` in `WorkItemSpiAdapter`**

Use `ide_insert_member` on `WorkItemSpiAdapter.java`, position `after`, anchor `obsoleteByCallerRef`:

```java
@Override
@Transactional
public WorkItemRef createMultiInstance(WorkItemCreateRequest parentRequest,
                                      MultiInstanceConfig config) {
    if (parentRequest.callerRef != null) {
        Optional<WorkItemRef> existing = findActiveByCallerRef(parentRequest.callerRef);
        if (existing.isPresent()) {
            return existing.get();
        }
    }

    WorkItemRef parentRef = create(parentRequest);

    WorkItemSpawnGroup group = new WorkItemSpawnGroup();
    group.id = UUID.randomUUID();
    group.parentId = parentRef.id();
    group.idempotencyKey = parentRequest.callerRef != null
            ? parentRequest.callerRef : UUID.randomUUID().toString();
    group.createdAt = Instant.now();
    group.tenancyId = parentRequest.tenancyId;
    group.instanceCount = config.instanceCount();
    group.requiredCount = config.requiredCount();
    group.onThresholdReached = config.effectiveOnThresholdReached().name();
    group.allowSameAssignee = config.allowSameAssignee();
    group.parentRole = config.effectiveParentRole().name();
    group.persist();

    for (int i = 0; i < config.instanceCount(); i++) {
        WorkItemCreateRequest childRequest = parentRequest.toBuilder()
                .callerRef(null)
                .auditDetail("Multi-instance child " + (i + 1) + " of " + config.instanceCount())
                .build();
        WorkItemRef childRef = create(childRequest);
        workItemService.addRelation(childRef.id(), parentRef.id(), "PART_OF");
    }

    return parentRef;
}
```

Note: The exact implementation depends on `WorkItemService.addRelation()` availability. Check via `ide_find_symbol` for the relation creation API. If `addRelation` doesn't exist, use whatever mechanism the spawn infrastructure uses to establish PART_OF relations — follow the `SpawnPort` implementation for the pattern.

- [ ] **Step 5: Run tests to verify they pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=WorkItemSpiAdapterMultiInstanceTest -f /Users/mdproctor/claude/casehub/work/pom.xml`
Expected: PASS

- [ ] **Step 6: Commit**

```
feat(#810): WorkItemCreator.createMultiInstance() — programmatic M-of-N group creation

Refs #810
```

---

### Task 5: Adapter bridge — creation + completion (work engine-adapter)

**Files:**
- Modify: `engine-adapter/src/main/java/io/casehub/work/engine/ActionGateWorkItemHandler.java:54` (multi-instance branch)
- Modify: `engine-adapter/src/main/java/io/casehub/work/engine/ActionGateCompletionApplier.java:52` (add applyGroupCompletion)
- Modify: `engine-adapter/src/main/java/io/casehub/work/engine/WorkItemLifecycleAdapter.java:60` (gate group routing)
- Test: `engine-adapter/src/test/java/io/casehub/work/engine/WorkItemLifecycleAdapterTest.java` (add gate group tests)

**Interfaces:**
- Consumes: `QuorumConfig` (Task 1), `MultiInstanceConfig` (work-api), `WorkItemCreator.createMultiInstance()` (Task 4)
- Produces: Gate group events routed to engine — consumed by existing `ActionGateApprovedHandler`/`ActionGateRejectedHandler`

- [ ] **Step 1: Write failing tests for gate group completion routing**

Add to `WorkItemLifecycleAdapterTest.java` using `ide_insert_member`:

```java
@Test
void workItemGroupCompleted_gateCallerRef_publishesApproved() {
    var gateCallerRef = GateCallerRef.encode(caseId, 42L);
    var event = WorkItemGroupLifecycleEvent.of(
            UUID.randomUUID(), UUID.randomUUID(),
            3, 2, 2, 1,
            GroupStatus.COMPLETED, gateCallerRef, "test-tenant");

    adapter.onWorkItemGroupLifecycle(event);

    // Verify ActionGateApprovedEvent published on event bus
    verify(eventBus).publish(eq(EventBusAddresses.ACTION_GATE_APPROVED), any(ActionGateApprovedEvent.class));
}

@Test
void workItemGroupRejected_gateCallerRef_publishesRejected() {
    var gateCallerRef = GateCallerRef.encode(caseId, 42L);
    var event = WorkItemGroupLifecycleEvent.of(
            UUID.randomUUID(), UUID.randomUUID(),
            3, 2, 0, 2,
            GroupStatus.REJECTED, gateCallerRef, "test-tenant");

    adapter.onWorkItemGroupLifecycle(event);

    verify(eventBus).publish(eq(EventBusAddresses.ACTION_GATE_REJECTED), any(ActionGateRejectedEvent.class));
}
```

Note: The test infrastructure uses either Mockito or static recording fields — follow the existing test pattern in `WorkItemLifecycleAdapterTest`. Adjust verify/assertion style to match.

- [ ] **Step 2: Run tests to verify they fail**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl engine-adapter -Dtest=WorkItemLifecycleAdapterTest -f /Users/mdproctor/claude/casehub/work/pom.xml`
Expected: FAIL — gate group routing not implemented

- [ ] **Step 3: Add `applyGroupCompletion` to `ActionGateCompletionApplier`**

Use `ide_insert_member` on `ActionGateCompletionApplier.java`, position `after`, anchor `apply`:

```java
public void applyGroupCompletion(final GateCallerRef gateRef, final GroupStatus status,
                                 final String tenancyId) {
    switch (status) {
        case COMPLETED -> {
            eventBus.publish(
                    EventBusAddresses.ACTION_GATE_APPROVED,
                    new ActionGateApprovedEvent(
                            gateRef.caseId(), tenancyId, gateRef.gateId(),
                            null, null, null));
            LOG.infof("Gate approved (group quorum): caseId=%s gateId=%d",
                    gateRef.caseId(), gateRef.gateId());
        }
        case REJECTED -> {
            eventBus.publish(
                    EventBusAddresses.ACTION_GATE_REJECTED,
                    new ActionGateRejectedEvent(
                            gateRef.caseId(), tenancyId, gateRef.gateId(),
                            null, null));
            LOG.infof("Gate rejected (group quorum failed): caseId=%s gateId=%d",
                    gateRef.caseId(), gateRef.gateId());
        }
        default -> LOG.debugf("Gate group status %s for caseId=%s gateId=%d — no event",
                status, gateRef.caseId(), gateRef.gateId());
    }
}
```

- [ ] **Step 4: Add gate group routing to `WorkItemLifecycleAdapter.onWorkItemGroupLifecycle()`**

Use `ide_edit_member` on `WorkItemLifecycleAdapter.java`, member `onWorkItemGroupLifecycle`. Insert gate routing before the existing PlanItem routing:

```java
public void onWorkItemGroupLifecycle(@ObservesAsync WorkItemGroupLifecycleEvent event) {
    GroupStatus status = event.groupStatus();
    if (!status.isTerminal()) return;

    CallerRef ref = CallerRef.parse(event.callerRef());

    if (ref instanceof GateCallerRef gateRef) {
        gateApplier.applyGroupCompletion(gateRef, status, event.tenancyId());
        return;
    }

    if (!(ref instanceof PlanItemCallerRef piRef)) return;

    // ... existing PlanItem handling unchanged ...
```

- [ ] **Step 5: Add multi-instance branch to `ActionGateWorkItemHandler.onActionGateSchedule()`**

Use `ide_edit_member` on `ActionGateWorkItemHandler.java`, member `onActionGateSchedule`. After building the `WorkItemCreateRequest`, add the quorum branch:

```java
if (event.gateRequired().quorum() != null) {
    var q = event.gateRequired().quorum();
    var config = new MultiInstanceConfig(
            q.instances(),
            q.required(),
            io.casehub.work.api.ParentRole.COORDINATOR,
            "pool",
            q.onThresholdReached() != null
                    ? io.casehub.work.api.OnThresholdReached.valueOf(q.onThresholdReached().name())
                    : null,
            q.allowSameAssignee(),
            null);
    workItemCreator.createMultiInstance(request, config);
    LOG.infof("Gate multi-instance group created: caseId=%s gateId=%d instances=%d required=%d",
            event.caseId(), event.gateId(), q.instances(), q.required());
} else {
    workItemCreator.create(request);
    LOG.infof("Gate WorkItem created: caseId=%s gateId=%d callerRef=%s expiresAt=%s",
            event.caseId(), event.gateId(), callerRef, expiresAt);
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl engine-adapter -Dtest=WorkItemLifecycleAdapterTest -f /Users/mdproctor/claude/casehub/work/pom.xml`
Expected: PASS

- [ ] **Step 7: Build full engine-adapter module**

Run: `ide_build_project` with `project_path=/Users/mdproctor/claude/casehub/work/engine-adapter`
Expected: 0 errors

- [ ] **Step 8: Commit**

```
feat(#810): adapter bridge — multi-instance gate creation + group completion routing

Refs #810
```

---

### Task 6: Full module builds + cross-repo verification

**Files:**
- No new files — verification only

- [ ] **Step 1: Install engine to local .m2**

Run: `mvn install -DskipTests -q -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: SUCCESS (engine-api with QuorumConfig available to work repo)

- [ ] **Step 2: Build work repo**

Run: `mvn install -DskipTests -q -f /Users/mdproctor/claude/casehub/work/pom.xml`
Expected: SUCCESS

- [ ] **Step 3: Run engine-api tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: PASS (all existing + QuorumConfig + ChainedActionRiskClassifier quorum tests)

- [ ] **Step 4: Run work engine-adapter tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl engine-adapter -f /Users/mdproctor/claude/casehub/work/pom.xml`
Expected: PASS (all existing + gate group tests)

- [ ] **Step 5: Run engine runtime tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: PASS (existing tests unbroken — null quorum backward compat)

- [ ] **Step 6: File deferred-scope issues**

Create GitHub issues for each "Not In Scope" item from the spec:
1. Dynamic instance count from candidate group membership
2. `approvedBy` list aggregation on engine events
3. Typed resolution for multi-approver gates
4. YAML schema for quorum in case definitions
5. Escalation policy changes for multi-approver

- [ ] **Step 7: Commit any remaining fixes**

If any tests required fixes, commit them.
