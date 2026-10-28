# Design: Expose Sub-Case Identity in PlanItemView

**Issue:** casehubio/engine#1260
**Consumer:** casehubio/scaffold#74 (getCaseDetail BFF)
**Branch:** issue-1260-planitemview-subcase-identity

## Problem

`PlanItemView(UUID id, String name, TaskStatus status, String type, UUID parentId)` carries no sub-case identity. Consumers cannot determine which case definition a plan item delegates to or which child cases were spawned.

**Additional finding:** `TargetType.SUB_CASE` is dead code — never set on any `PlanItemSaveRequest`. Sub-case plan items are created in-memory in `PlanningStrategyLoopControl:121` with the actual `SubCaseTarget` binding, but plan item persistence paths (goal decomposition, adaptation) hardcode `TargetType.CAPABILITY`. This means the `type` field on `PlanItemView` currently does NOT distinguish sub-case items — it shows `"capability"` for all items regardless of binding target.

This issue fixes both: (a) exposing sub-case identity fields, and (b) correctly setting `TargetType.SUB_CASE` so the `type` field works.

## Solution

Extend `PlanItemView` with two fields:

```java
public record PlanItemView(
    UUID id,
    String name,
    TaskStatus status,
    String type,
    UUID parentId,
    String capabilityName,     // nullable — null for non-sub-case items
    List<UUID> subCaseIds      // nullable — null for non-sub-case items; empty list when not yet spawned
) {}
```

- `capabilityName`: the case definition name of the sub-case target (e.g. `"auto-assessment"`), stored at write time on PlanItemRecord
- `subCaseIds`: child case UUIDs, sourced from SubCaseGroupRepository at query time. Empty list when the plan item is sub-case type but child cases haven't been spawned yet. Handles M-of-N (multiple IDs) and ungrouped (single ID) uniformly.

## Write Path — capabilityName and TargetType fix

### Fix TargetType for sub-case plan items

Sub-case plan items are created in `PlanningStrategyLoopControl:121`:
```java
PlanItem item = PlanItem.create(binding.getName(), executor, 0, binding.target());
plan.addPlanItemIfAbsent(item);
```

The `PlanItem` carries the `SubCaseTarget` in memory, but plan items are currently only saved to `PlanItemStore` through goal decomposition and adaptation paths, which hardcode `TargetType.CAPABILITY`. The fix: when saving a plan item whose binding has a `SubCaseTarget`, use `TargetType.SUB_CASE` instead of `TargetType.CAPABILITY`.

The plan item save happens in `DefaultGoalDecomposer.decomposeGoal()`, `DefaultPlanAdaptationEvaluator.applyRevision()`, and `DeeperDecompositionHandler.tryDecompose()`. These need to resolve the target type from the binding target rather than hardcoding `TargetType.CAPABILITY`.

### Store capabilityName

Store `capabilityName` at plan-item creation time. The value comes from `SubCase.name()` which is available when the binding target is resolved.

**Changes:**

1. **PlanItemSaveRequest** — add `String capabilityName` field (nullable)
2. **PlanItemRecord** — add `String capabilityName` field (nullable)
3. **PlanItemEntity** — add `@Column(name = "capability_name", length = 255) public String capabilityName`
4. **PlanItemStore implementations** — pass capabilityName through save and findByCaseId
5. **Plan item save sites** (DefaultGoalDecomposer, DefaultPlanAdaptationEvaluator, DeeperDecompositionHandler) — resolve `TargetType` from `BindingTarget` (`SubCaseTarget → SUB_CASE`, all others → existing type), extract `subCase.name()` as `capabilityName` when target is `SubCaseTarget`
6. **Add utility** — static `TargetType resolveFrom(BindingTarget target)` and `String capabilityNameFrom(BindingTarget target)` on `PlanItemSaveRequest` or a helper

This follows the existing pattern for `executorName` and `description` — known at creation time, stored on the entity, returned in the record.

## Read Path — subCaseIds

Source child case IDs from SubCaseGroupRepository. Two changes unify grouped and ungrouped sub-cases under this data source:

### 1. Register ungrouped sub-cases in SubCaseGroupRepository

Currently `handleUngrouped()` does NOT create a SubCaseGroup entry. Extend it to register a synthetic group:

```java
// In SubCaseExecutionHandler.handleUngrouped():
subCaseGroupRepository.getOrCreate(
    parent.getUuid(),
    bindingName,          // use bindingName as groupId for ungrouped
    1,                    // totalInGroup = 1
    1,                    // requiredCount = 1
    OnThresholdReached.KEEP,
    parent.tenancyId);
subCaseGroupRepository.registerChild(
    parent.getUuid(), bindingName, childCaseId, parent.tenancyId);
```

This means ALL sub-case child relationships are tracked in SubCaseGroupRepository.

### 2. Add bindingName to SubCaseGroup

Add a `bindingName` field to SubCaseGroup and SubCaseGroupEntity. This bridges group → plan item. Set it when creating the group in both `handleGrouped()` and `handleUngrouped()`.

**Changes:**

1. **SubCaseGroup** — add `String bindingName` field with getter/setter
2. **SubCaseGroupEntity** — add `@Column(name = "binding_name", length = 255) public String bindingName`
3. **SubCaseGroupRepository** — add method: `List<SubCaseGroup> findAllByParentCaseId(UUID parentCaseId, String tenancyId)`
4. **All 4 implementations** (InMemory, Jpa, SpringJpa, Test) — implement `findAllByParentCaseId` and pass `bindingName` through `getOrCreate`
5. **SubCaseExecutionHandler.handleGrouped()** — pass `bindingName` to `getOrCreate`
6. **SubCaseExecutionHandler.handleUngrouped()** — create SubCaseGroup entry (see above)

### 3. Enrich PlanItemView in CaseService

```java
public List<PlanItemView> getPlanItems(UUID caseId, String tenancyId) {
    requireCaseAccess(caseId, AclAction.READ);
    String resolved = tenancyId != null ? tenancyId : currentPrincipal.tenancyId();
    List<PlanItemRecord> records = planItemStore.findByCaseId(caseId, resolved);

    // Build bindingName → childCaseIds map from SubCaseGroups
    Map<String, List<UUID>> childMap = subCaseGroupRepository
        .findAllByParentCaseId(caseId, resolved).stream()
        .collect(Collectors.toMap(
            SubCaseGroup::getBindingName,
            g -> List.copyOf(g.getChildCaseIds()),
            (a, b) -> a));  // first wins (shouldn't conflict)

    return records.stream()
        .map(r -> mapPlanItem(r, childMap))
        .toList();
}

public PlanItemView mapPlanItem(PlanItemRecord record, Map<String, List<UUID>> childMap) {
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

CaseService needs `SubCaseGroupRepository` injected. The existing `mapPlanItem(PlanItemRecord)` single-arg method is used by other callers — keep it with null/null for the new fields, add the enriched overload.

## Impact on Existing Callers

The existing `mapPlanItem(PlanItemRecord)` method must still compile. Options:
- Add a second overload that takes the childMap
- Update the original to pass `null` for both new fields (backward compatible — callers that don't need sub-case identity get nulls)

The REST API response JSON gains two new nullable fields. Existing consumers that don't read them are unaffected (additive change).

## Test Strategy

1. **Unit test:** `CaseService.mapPlanItem` correctly populates `capabilityName` and `subCaseIds` for sub-case items, and nulls for non-sub-case items
2. **Unit test:** `SubCaseExecutionHandler.handleUngrouped` registers a SubCaseGroup entry
3. **Integration test:** End-to-end — start a case with a sub-case binding, verify `getPlanItems` returns `capabilityName` and `subCaseIds` on the sub-case plan item
4. **Contract test:** `SubCaseGroupRepository.findAllByParentCaseId` works across all implementations

## Files Changed

| Module | File | Change |
|--------|------|--------|
| api | `PlanItemView.java` | Add `capabilityName`, `subCaseIds` fields |
| common-core | `PlanItemRecord.java` | Add `capabilityName` field |
| common-core | `PlanItemSaveRequest.java` | Add `capabilityName` field |
| common-core | `SubCaseGroup.java` | Add `bindingName` field |
| common-core | `SubCaseGroupRepository.java` | Add `findAllByParentCaseId` method |
| persistence-jpa-common | `PlanItemEntity.java` | Add `capability_name` column |
| persistence-jpa-common | `SubCaseGroupEntity.java` | Add `binding_name` column |
| persistence-hibernate | `JpaSubCaseGroupRepository.java` | Implement `findAllByParentCaseId`, pass `bindingName` |
| persistence-spring-jpa | `SpringJpaSubCaseGroupRepository.java` | Implement `findAllByParentCaseId`, pass `bindingName` |
| engine-support-core | `InMemorySubCaseGroupRepository.java` | Implement `findAllByParentCaseId`, pass `bindingName` |
| testing | `TestSubCaseGroupRepository.java` | Implement `findAllByParentCaseId`, pass `bindingName` |
| planning-core | `SubCaseExecutionHandler.java` | Register ungrouped sub-cases in SubCaseGroupRepository, pass `bindingName` to `getOrCreate` |
| rest-core | `CaseService.java` | Inject SubCaseGroupRepository, enrich `getPlanItems` with childMap lookup |
| planning-core | `DefaultGoalDecomposer.java` | Resolve `TargetType` from `BindingTarget`, extract `capabilityName` from `SubCaseTarget` |
| planning-core | `DefaultPlanAdaptationEvaluator.java` | Same TargetType + capabilityName resolution |
| planning-core | `DeeperDecompositionHandler.java` | Same TargetType + capabilityName resolution |

## References

- `api/src/main/java/io/casehub/api/view/PlanItemView.java` — current record
- `api/src/main/java/io/casehub/api/model/SubCaseTarget.java` — SubCase target binding
- `api/src/main/java/io/casehub/api/model/SubCase.java` — namespace, name, version
- `rest-core/src/main/java/io/casehub/engine/rest/service/CaseService.java:213` — mapPlanItem conversion
- `planning-core/src/main/java/io/casehub/engine/planning/subcase/SubCaseExecutionHandler.java` — grouped/ungrouped sub-case dispatch
- `common-core/src/main/java/io/casehub/engine/common/spi/SubCaseGroupRepository.java` — data source for child case IDs
- `persistence-jpa-common/src/main/java/io/casehub/persistence/jpa/SubCaseGroupEntity.java` — JPA entity
- `planning-core/src/main/java/io/casehub/engine/planning/control/PlanningStrategyLoopControl.java:121` — PlanItem creation with actual BindingTarget
- `planning-core/src/main/java/io/casehub/engine/planning/decomposition/DefaultGoalDecomposer.java:275` — hardcoded TargetType.CAPABILITY
- `common-core/src/main/java/io/casehub/engine/common/internal/model/TargetType.java` — SUB_CASE enum constant (currently dead code)
- casehubio/engine#1260 — issue
- casehubio/scaffold#74 — consumer blocked on this
