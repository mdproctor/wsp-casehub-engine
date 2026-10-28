# PlanItemView Sub-Case Identity Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1260 — feat: expose sub-case identity in PlanItemView or dedicated query
**Issue group:** #1260

**Goal:** Extend PlanItemView with `capabilityName` and `subCaseIds` so scaffold BFF can show sub-case delegations, and fix `TargetType.SUB_CASE` to actually be used.

**Architecture:** Extend the PlanItemView record with two nullable fields. Store `capabilityName` on PlanItemRecord at write time (extracted from SubCaseTarget). Source `subCaseIds` from SubCaseGroupRepository at query time — unify grouped and ungrouped sub-cases by registering both. Fix TargetType resolution so SUB_CASE is no longer dead code.

**Tech Stack:** Java 21, Quarkus 3.32.2, Hibernate, JPA

## Global Constraints

- Hibernate manages schema via `drop-and-create` — no migration scripts needed
- `TargetType.SUB_CASE` exists but is currently dead code — this plan activates it
- SubCaseGroupRepository SPI has 4 implementations: InMemory, Jpa, SpringJpa, Test (extends InMemory)
- PlanItemStore SPI has 7 implementations — only InMemory, Jpa, SpringJpa are production
- Use `ide_edit_member` for record declarations, `ide_insert_member` for new methods

---

## Batch 1: Data Model + SPI Changes

### Task 1: Add bindingName to SubCaseGroup and findAllByParentCaseId to SubCaseGroupRepository

**Files:**
- Modify: `common-core/src/main/java/io/casehub/engine/common/internal/model/SubCaseGroup.java`
- Modify: `common-core/src/main/java/io/casehub/engine/common/spi/SubCaseGroupRepository.java`
- Modify: `persistence-jpa-common/src/main/java/io/casehub/persistence/jpa/SubCaseGroupEntity.java`
- Modify: `engine-support-core/src/main/java/io/casehub/engine/persistence/memory/InMemorySubCaseGroupRepository.java`
- Modify: `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaSubCaseGroupRepository.java`
- Modify: `persistence-spring-jpa/src/main/java/io/casehub/persistence/spring/SpringJpaSubCaseGroupRepository.java`
- Modify: `common-core/src/test/java/io/casehub/engine/common/spi/SubCaseGroupRepositoryContractTest.java`
- Test: `common-core/src/test/java/io/casehub/engine/common/spi/SubCaseGroupRepositoryContractTest.java`

**Interfaces:**
- Produces: `SubCaseGroup.getBindingName()`, `SubCaseGroup.setBindingName(String)`, `SubCaseGroupRepository.findAllByParentCaseId(UUID, String) → List<SubCaseGroup>`
- Produces: `SubCaseGroupEntity.bindingName` column

- [ ] **Step 1: Write failing contract test for findAllByParentCaseId**

Add test to `SubCaseGroupRepositoryContractTest`:

```java
@Test
void findAllByParentCaseId_returnsAllGroupsForParent() {
    UUID parentCaseId = UUID.randomUUID();
    UUID child1 = UUID.randomUUID();
    UUID child2 = UUID.randomUUID();
    repository().getOrCreate(parentCaseId, "g1", 2, 2, OnThresholdReached.KEEP, tenancyId());
    repository().registerChild(parentCaseId, "g1", child1, tenancyId());
    repository().getOrCreate(parentCaseId, "g2", 1, 1, OnThresholdReached.KEEP, tenancyId());
    repository().registerChild(parentCaseId, "g2", child2, tenancyId());

    List<SubCaseGroup> groups = repository().findAllByParentCaseId(parentCaseId, tenancyId());

    assertThat(groups).hasSize(2);
    assertThat(groups).extracting(SubCaseGroup::getGroupId).containsExactlyInAnyOrder("g1", "g2");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl common-core -Dtest=SubCaseGroupRepositoryContractTest#findAllByParentCaseId_returnsAllGroupsForParent -q`
Expected: compilation error — `findAllByParentCaseId` does not exist

- [ ] **Step 3: Add bindingName to SubCaseGroup**

Use `ide_insert_member` to add to `SubCaseGroup.java` (after the `childCaseIds` field):

```java
private String bindingName;
```

And add getter/setter:

```java
public String getBindingName() {
    return bindingName;
}

public void setBindingName(String v) {
    bindingName = v;
}
```

- [ ] **Step 4: Add bindingName column to SubCaseGroupEntity**

Use `ide_insert_member` to add to `SubCaseGroupEntity.java` (after `tenancyId` field):

```java
@Column(name = "binding_name", length = 255)
public String bindingName;
```

- [ ] **Step 5: Add findAllByParentCaseId to SubCaseGroupRepository interface**

Use `ide_insert_member` to add after `findByChildCaseId`:

```java
List<SubCaseGroup> findAllByParentCaseId(UUID parentCaseId, String tenancyId);
```

Add `java.util.List` import.

- [ ] **Step 6: Implement in InMemorySubCaseGroupRepository**

Use `ide_insert_member` to add:

```java
@Override
public List<SubCaseGroup> findAllByParentCaseId(UUID parentCaseId, String tenancyId) {
    return groups.values().stream()
        .filter(g -> g.getParentCaseId().equals(parentCaseId))
        .toList();
}
```

Add `java.util.List` import.

- [ ] **Step 7: Update InMemorySubCaseGroupRepository.getOrCreate to set bindingName from groupId**

The `getOrCreate` method creates a new `SubCaseGroup`. Currently `bindingName` is not set. For now, leave it null — Task 4 will pass it explicitly. The `toDomain` / creation code just needs to pass through whatever is stored.

No change needed here — `bindingName` defaults to null.

- [ ] **Step 8: Implement in JpaSubCaseGroupRepository**

Use `ide_insert_member` to add:

```java
@Override
@Transactional
public List<SubCaseGroup> findAllByParentCaseId(UUID parentCaseId, String tenancyId) {
    tcm.setTenantContext(tenancyId);
    return em.createQuery(
            "SELECT g FROM SubCaseGroupEntity g WHERE g.parentCaseId = :pid AND g.tenancyId = :tid",
            SubCaseGroupEntity.class)
        .setParameter("pid", parentCaseId)
        .setParameter("tid", tenancyId)
        .getResultList()
        .stream()
        .map(this::toDomain)
        .toList();
}
```

Also update `toDomain` to map `bindingName`:
```java
g.setBindingName(e.bindingName);
```

And update `getOrCreate` to persist `bindingName`:
After `e.onThresholdReached = ...;` add:
```java
// bindingName set by caller via SubCaseGroup after creation if needed
```
Actually, bindingName on the entity should be set during `getOrCreate`. But `getOrCreate`'s current signature doesn't have a `bindingName` parameter. We need to add it to the SPI.

**Revised approach:** Add `bindingName` parameter to `getOrCreate` on the SPI.

Go back to `SubCaseGroupRepository.java` and change `getOrCreate` signature:

```java
SubCaseGroup getOrCreate(
    UUID parentCaseId,
    String groupId,
    int totalInGroup,
    int requiredCount,
    OnThresholdReached onThresholdReached,
    String tenancyId,
    String bindingName);
```

Then update ALL implementations and ALL callers.

- [ ] **Step 9: Update SubCaseGroupRepository.getOrCreate signature to include bindingName**

Use `ide_edit_member` on `SubCaseGroupRepository.java` method `getOrCreate`:

```java
SubCaseGroup getOrCreate(
    UUID parentCaseId,
    String groupId,
    int totalInGroup,
    int requiredCount,
    OnThresholdReached onThresholdReached,
    String tenancyId,
    String bindingName);
```

- [ ] **Step 10: Update InMemorySubCaseGroupRepository.getOrCreate**

Use `ide_edit_member` to update the method signature and body. Add `bindingName` parameter, and set it on the new group:

In the `computeIfAbsent` lambda, add `group.setBindingName(bindingName);` after setting `onThresholdReached`.

- [ ] **Step 11: Update JpaSubCaseGroupRepository.getOrCreate**

Add `String bindingName` parameter. In the entity creation block, add `e.bindingName = bindingName;` before `em.persist(e);`.

Also update `toDomain` to map `bindingName`: add `g.setBindingName(e.bindingName);`.

- [ ] **Step 12: Update SpringJpaSubCaseGroupRepository.getOrCreate**

Same changes as JPA: add parameter, set `e.bindingName = bindingName;`, update `toDomain`.

- [ ] **Step 13: Fix callers of getOrCreate — SubCaseExecutionHandler.handleGrouped**

Add `bindingName` as the last argument to the `getOrCreate` call at line 216:

```java
subCaseGroupRepository.getOrCreate(
    parent.getUuid(),
    groupId,
    subCase.totalInGroup(),
    subCase.requiredCount(),
    subCase.onThresholdReached(),
    parent.tenancyId,
    bindingName);
```

- [ ] **Step 14: Fix test callers of getOrCreate**

Search for all test usages of `getOrCreate` and add `null` as the bindingName argument (since tests don't care about it yet):
- `SubCaseGroupRepositoryContractTest`
- `PersistenceIntegrationTest`
- `SubCaseRecursionDepthTest`
- `SubCaseExecutionHandlerTest`

Use `ide_find_references` on `getOrCreate` to find all callers.

- [ ] **Step 15: Run tests to verify contract test passes**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl common-core -Dtest=SubCaseGroupRepositoryContractTest -q`
Expected: PASS

- [ ] **Step 16: Commit**

```bash
git add -A
git commit -m "feat(#1260): add bindingName to SubCaseGroup and findAllByParentCaseId to SubCaseGroupRepository

Refs #1260"
```

---

### Task 2: Add capabilityName to PlanItemSaveRequest/PlanItemRecord/PlanItemEntity and extend PlanItemView

**Files:**
- Modify: `common-core/src/main/java/io/casehub/engine/common/internal/model/PlanItemSaveRequest.java`
- Modify: `common-core/src/main/java/io/casehub/engine/common/internal/model/PlanItemRecord.java`
- Modify: `persistence-jpa-common/src/main/java/io/casehub/persistence/jpa/PlanItemEntity.java`
- Modify: `api/src/main/java/io/casehub/api/view/PlanItemView.java`
- Modify: `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaPlanItemStore.java`
- Modify: `persistence-spring-jpa/src/main/java/io/casehub/persistence/spring/SpringJpaPlanItemStore.java`
- Modify: `engine-support-core/src/main/java/io/casehub/engine/persistence/memory/InMemoryPlanItemStore.java`
- Modify: `rest-core/src/main/java/io/casehub/engine/rest/service/CaseService.java`

**Interfaces:**
- Consumes: nothing from Task 1
- Produces: `PlanItemView.capabilityName()`, `PlanItemView.subCaseIds()`, `PlanItemRecord.capabilityName()`, `PlanItemSaveRequest.capabilityName()`

- [ ] **Step 1: Add capabilityName to PlanItemRecord**

Use `ide_edit_member` on `PlanItemRecord` declaration to add `String capabilityName` after `variantId`:

```java
public record PlanItemRecord(
    UUID caseId,
    String planItemId,
    String bindingName,
    TaskStatus status,
    Instant createdAt,
    Instant completedAt,
    TargetType targetType,
    String outputMappingExpression,
    String tenancyId,
    String description,
    String executorName,
    String executorDescription,
    PlanItemType planItemType,
    String planningStrategy,
    String completionSemantics,
    String dispatchMode,
    boolean repeatable,
    String parentCompoundId,
    String lifecycleScope,
    com.fasterxml.jackson.databind.JsonNode activationContext,
    String variantId,
    String capabilityName) {
```

Also update the `primitive()` factory to pass `null` for `capabilityName`.

- [ ] **Step 2: Add capabilityName to PlanItemSaveRequest**

Use `ide_edit_member` on `PlanItemSaveRequest` declaration to add `String capabilityName` after `activationContext`:

```java
public record PlanItemSaveRequest(
    UUID caseId,
    String planItemId,
    String bindingName,
    TaskStatus status,
    Instant createdAt,
    TargetType targetType,
    String outputMappingExpression,
    String tenancyId,
    String description,
    String executorName,
    String executorDescription,
    PlanItemType planItemType,
    String planningStrategy,
    String completionSemantics,
    String dispatchMode,
    boolean repeatable,
    String parentCompoundId,
    String lifecycleScope,
    com.fasterxml.jackson.databind.JsonNode activationContext,
    String capabilityName) {
```

Also update the `primitive()` factory to pass `null` for `capabilityName`.

- [ ] **Step 3: Add capabilityName column to PlanItemEntity**

Use `ide_insert_member` to add after `activationContext`:

```java
@Column(name = "capability_name", length = 255)
public String capabilityName;
```

- [ ] **Step 4: Update JpaPlanItemStore.save to persist capabilityName**

In the `save` method, add after the `activationContext` line:
```java
e.capabilityName = request.capabilityName();
```

Update `toRecord` to pass `capabilityName`:
Add `e.capabilityName` as the last argument to `new PlanItemRecord(...)`.

- [ ] **Step 5: Update SpringJpaPlanItemStore save and toRecord**

Same changes as JPA implementation.

- [ ] **Step 6: Update InMemoryPlanItemStore save and toRecord**

The InMemory implementation stores PlanItemRecords directly. Update the `save` method to include `capabilityName` in the record construction. Update the `toRecord`/record creation to pass `null` or `request.capabilityName()`.

- [ ] **Step 7: Extend PlanItemView with capabilityName and subCaseIds**

Use `ide_edit_member` on `PlanItemView` declaration:

```java
public record PlanItemView(UUID id, String name, TaskStatus status, String type, UUID parentId,
    String capabilityName, List<UUID> subCaseIds) {}
```

Add `java.util.List` import.

- [ ] **Step 8: Update CaseService.mapPlanItem to pass nulls for new fields**

The existing single-arg `mapPlanItem(PlanItemRecord)` now needs to pass `null, null` for the two new fields:

```java
public PlanItemView mapPlanItem(PlanItemRecord record) {
    return new PlanItemView(
        UUID.fromString(record.planItemId()),
        record.bindingName(),
        record.status(),
        record.targetType().name().toLowerCase().replace('_', '-'),
        record.parentCompoundId() != null ? UUID.fromString(record.parentCompoundId()) : null,
        null,
        null);
}
```

- [ ] **Step 9: Fix compilation across all modules**

Run: `mvn install -DskipTests -q`

Fix any remaining compilation errors (test stubs, NoOp implementations that construct PlanItemRecord directly).

Expected breakages:
- `NoOpPlanItemStore` (planning-core) — no-op, no change needed
- `NoOpPlanItemStore` (runtime-core) — no-op, no change needed
- `StubPlanItemStore` (CbrCaseRetainObserverTest) — update record construction
- `RecordingPlanItemStore` (JudgmentWorkItemSchedulerRegistryTest) — just records, no change
- `PlanItemStoreContractTest` — update `request()` helper
- Various test callers that assert on PlanItemRecord fields

- [ ] **Step 10: Run full test suite to verify green**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -q`
Expected: PASS

- [ ] **Step 11: Commit**

```bash
git add -A
git commit -m "feat(#1260): add capabilityName to plan item data model and extend PlanItemView

Refs #1260"
```

---

## Batch 2: Write Path + Read Path + Integration Test

### Task 3: Fix TargetType resolution at save sites and wire ungrouped sub-case registration

**Files:**
- Modify: `planning-core/src/main/java/io/casehub/engine/planning/decomposition/DefaultGoalDecomposer.java`
- Modify: `planning-core/src/main/java/io/casehub/engine/planning/adaptation/DefaultPlanAdaptationEvaluator.java`
- Modify: `planning-core/src/main/java/io/casehub/engine/planning/adaptation/DeeperDecompositionHandler.java`
- Modify: `planning-core/src/main/java/io/casehub/engine/planning/subcase/SubCaseExecutionHandler.java`
- Modify: `rest-core/src/main/java/io/casehub/engine/rest/service/CaseService.java`
- Test: `planning/src/test/java/io/casehub/engine/planning/subcase/SubCaseExecutionHandlerTest.java`

**Interfaces:**
- Consumes: `PlanItemSaveRequest.capabilityName()` (Task 2), `SubCaseGroupRepository.findAllByParentCaseId` (Task 1), `SubCaseGroupRepository.getOrCreate` with `bindingName` (Task 1)
- Produces: Correctly typed plan items, ungrouped sub-case groups, enriched PlanItemView

- [ ] **Step 1: Write failing test for TargetType resolution**

Add test to `DefaultGoalDecomposerTest` (or a new test class) that verifies when a binding has a `SubCaseTarget`, the saved PlanItemSaveRequest has `targetType == TargetType.SUB_CASE` and `capabilityName` set.

Use the existing test patterns — mock `planItemStore`, capture the `PlanItemSaveRequest`, assert on `targetType()` and `capabilityName()`.

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — `targetType()` is `CAPABILITY`, not `SUB_CASE`

- [ ] **Step 3: Fix DefaultGoalDecomposer to resolve TargetType from BindingTarget**

In `decomposeGoal()`, where `PlanItemSaveRequest.primitive()` is called (around line 269), replace hardcoded `TargetType.CAPABILITY` with resolution from the binding:

```java
BindingTarget target = resolved.binding().target();
TargetType resolvedTargetType = target instanceof SubCaseTarget ? TargetType.SUB_CASE : TargetType.CAPABILITY;
String capName = target instanceof SubCaseTarget st ? st.subCase().name() : null;
```

Then pass `resolvedTargetType` and create a new save request that includes `capabilityName`. Since `primitive()` doesn't take `capabilityName`, either:
- Add a new factory method, or
- Use the canonical constructor directly

Use the canonical constructor with the extra field:

```java
planItemStore.save(
    new PlanItemSaveRequest(
        instance.getUuid(),
        resolved.step().id(),
        resolved.binding().getName(),
        TaskStatus.PENDING,
        Instant.now(),
        resolvedTargetType,
        null,
        instance.tenancyId,
        resolved.step().description(),
        null,
        null,
        PlanItemType.PRIMITIVE,
        null,
        null,
        null,
        false,
        null,
        null,
        null,
        capName),
    instance.tenancyId);
```

Add imports for `SubCaseTarget`, `BindingTarget`.

- [ ] **Step 4: Apply same fix to DefaultPlanAdaptationEvaluator.applyRevision**

Same pattern — resolve `TargetType` and `capabilityName` from the binding target instead of hardcoding `TargetType.CAPABILITY`.

- [ ] **Step 5: Apply same fix to DeeperDecompositionHandler.tryDecompose**

Same pattern.

- [ ] **Step 6: Run test to verify it passes**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl planning -q`
Expected: PASS

- [ ] **Step 7: Write failing test for ungrouped sub-case group registration**

In `SubCaseExecutionHandlerTest`, add a test that verifies `handleUngrouped` calls `subCaseGroupRepository.getOrCreate()` and `registerChild()`:

```java
@Test
void onSubCaseSchedule_ungrouped_registersSubCaseGroup() {
    // setup: subCase with groupId = null, waitForCompletion = true
    // verify: subCaseGroupRepository.getOrCreate was called with bindingName as groupId
    // verify: subCaseGroupRepository.registerChild was called with the childCaseId
}
```

- [ ] **Step 8: Run test to verify it fails**

Expected: FAIL — `getOrCreate` is not called for ungrouped sub-cases

- [ ] **Step 9: Modify SubCaseExecutionHandler.handleUngrouped to register group**

Add to `handleUngrouped()`, before the EventLog creation:

```java
subCaseGroupRepository.getOrCreate(
    parent.getUuid(),
    bindingName,
    1,
    1,
    OnThresholdReached.KEEP,
    parent.tenancyId,
    bindingName);
subCaseGroupRepository.registerChild(
    parent.getUuid(), bindingName, childCaseId, parent.tenancyId);
```

- [ ] **Step 10: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl planning -Dtest=SubCaseExecutionHandlerTest -q`
Expected: PASS

- [ ] **Step 11: Enrich CaseService.getPlanItems with sub-case data**

Inject `SubCaseGroupRepository` into `CaseService`. Use `ide_insert_member` to add after the existing `@Inject CaseStreamBroadcaster` field in `CaseService.java`:

```java
@Inject SubCaseGroupRepository subCaseGroupRepository;
```

Then update `getPlanItems` and add enriched `mapPlanItem` overload:

```java
public List<PlanItemView> getPlanItems(UUID caseId, String tenancyId) {
    requireCaseAccess(caseId, AclAction.READ);
    String resolved = tenancyId != null ? tenancyId : currentPrincipal.tenancyId();
    List<PlanItemRecord> records = planItemStore.findByCaseId(caseId, resolved);

    Map<String, List<UUID>> childMap = subCaseGroupRepository
        .findAllByParentCaseId(caseId, resolved).stream()
        .collect(Collectors.toMap(
            SubCaseGroup::getBindingName,
            g -> List.copyOf(g.getChildCaseIds()),
            (a, b) -> a));

    return records.stream()
        .map(r -> mapPlanItem(r, childMap))
        .toList();
}

private PlanItemView mapPlanItem(PlanItemRecord record, Map<String, List<UUID>> childMap) {
    boolean isSubCase = record.capabilityName() != null;
    return new PlanItemView(
        UUID.fromString(record.planItemId()),
        record.bindingName(),
        record.status(),
        record.targetType().name().toLowerCase().replace('_', '-'),
        record.parentCompoundId() != null ? UUID.fromString(record.parentCompoundId()) : null,
        record.capabilityName(),
        isSubCase ? childMap.getOrDefault(record.bindingName(), List.of()) : null);
}
```

Add imports for `SubCaseGroupRepository`, `SubCaseGroup`, `Collectors`, `Map`.

- [ ] **Step 12: Run full test suite**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -q`
Expected: PASS

- [ ] **Step 13: Commit**

```bash
git add -A
git commit -m "feat(#1260): wire TargetType resolution, ungrouped sub-case registration, and CaseService enrichment

Closes #1260"
```

## References

- [2026-10-28-planitemview-subcase-identity-design.md] — design spec this plan implements
- `api/src/main/java/io/casehub/api/view/PlanItemView.java` — current record (5 fields)
- `common-core/src/main/java/io/casehub/engine/common/internal/model/PlanItemSaveRequest.java` — plan item write model
- `common-core/src/main/java/io/casehub/engine/common/internal/model/PlanItemRecord.java` — plan item read model
- `persistence-jpa-common/src/main/java/io/casehub/persistence/jpa/PlanItemEntity.java` — JPA entity
- `common-core/src/main/java/io/casehub/engine/common/spi/SubCaseGroupRepository.java` — SPI for child case tracking
- `planning-core/src/main/java/io/casehub/engine/planning/subcase/SubCaseExecutionHandler.java` — sub-case dispatch
- `planning-core/src/main/java/io/casehub/engine/planning/decomposition/DefaultGoalDecomposer.java:275` — hardcoded TargetType.CAPABILITY
- `planning-core/src/main/java/io/casehub/engine/planning/control/PlanningStrategyLoopControl.java:121` — PlanItem creation with actual BindingTarget
- `rest-core/src/main/java/io/casehub/engine/rest/service/CaseService.java:213` — mapPlanItem conversion
- `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaPlanItemStore.java` — JPA save/load
- GitHub #1260 — focal issue
- GitHub scaffold#74 — consumer blocked on this
