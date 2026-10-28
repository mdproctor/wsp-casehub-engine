---
layout: post
title: "Why Your Plan Items Couldn't Tell You About Sub-Cases"
date: 2026-10-28
entry_type: article
subtype: diary
projects: [casehubio/engine]
tags: [planning, sub-cases, api, data-model]
---

# Why Your Plan Items Couldn't Tell You About Sub-Cases

A case engine that delegates work to child cases has a problem: the plan view doesn't know it's happening. The scaffold BFF calls `getPlanItems`, gets back a list of plan items with `id`, `name`, `status`, `type`, and `parentId` — and has no way to tell which ones spawned child cases, what kind of case they spawned, or where those children are.

The `type` field should help. It maps from `TargetType`, an enum that includes `SUB_CASE`. But here's what I found when I traced the data flow: `TargetType.SUB_CASE` was dead code. Every plan item save path hardcoded `TargetType.CAPABILITY`, regardless of the actual binding target. A sub-case delegation and a worker dispatch looked identical from the API.

## The gap ran deeper than a missing field

The obvious fix — add `subCaseId` and `capabilityName` to `PlanItemView` — turned out to be the easy part. The harder question was where the data comes from at query time.

`PlanItem` objects in memory carry a `BindingTarget` which can be a `SubCaseTarget`. That target knows the case definition name. But `PlanItemEntity` — the persisted form — only stored the `TargetType` enum. The case definition name was available at creation time and then thrown away.

Child case IDs had a different problem. For grouped sub-cases (M-of-N coordination), `SubCaseGroupRepository` tracked them. For ungrouped sub-cases — the common case, where one plan item spawns one child — nothing tracked them at all. The child case ID appeared in the `EventLog` as a `SUBCASE_STARTED` audit event and in the in-memory `BlackboardRegistry` completion index. Neither is a query source.

## Two decisions shaped the design

**Store `capabilityName` at write time.** The sub-case definition name (`SubCase.name()`) is available when the plan item is created. Storing it on `PlanItemSaveRequest`, `PlanItemRecord`, and `PlanItemEntity` follows the existing pattern for `executorName` and `description` — known at creation, persisted on the entity, returned in the record. Zero cost at query time.

**Unify child case tracking in `SubCaseGroupRepository`.** Rather than introducing a second data source or querying the event log, I extended `handleUngrouped` to register a synthetic group — `totalInGroup=1`, `requiredCount=1`, `groupId=bindingName`. Now every sub-case delegation, grouped or not, has its child case IDs in one place. A new `findAllByParentCaseId` query returns all groups for a case, and `CaseService.getPlanItems` builds a `bindingName → childCaseIds` map to enrich the view.

The result is two new nullable fields on `PlanItemView`:

```java
public record PlanItemView(
    UUID id, String name, TaskStatus status, String type, UUID parentId,
    String capabilityName,    // null for non-sub-case items
    List<UUID> subCaseIds     // null for non-sub-case; empty when not yet spawned
) {}
```

Consumers that don't read these fields are unaffected — additive JSON change.

## TargetType.SUB_CASE is no longer dead

The fix touched three plan item save sites: `DefaultGoalDecomposer`, `DefaultPlanAdaptationEvaluator`, and `DeeperDecompositionHandler`. All three had been hardcoding `TargetType.CAPABILITY`. Now they resolve the target type from the binding: `SubCaseTarget → SUB_CASE`, everything else → the existing type. The `type` field on `PlanItemView` finally returns `"sub-case"` when it should.

This is the kind of thing that's invisible until someone tries to use the data. The enum existed, the column existed, the mapping existed — but the write path never set it. The API dutifully returned `"capability"` for every plan item, and no consumer had complained because no consumer had tried to distinguish sub-case items until scaffold's `getCaseDetail` BFF needed to render them.

A `bindingName` field on `SubCaseGroup` bridges the last gap — connecting the child case IDs back to the plan item that spawned them. The SPI change propagated across four repository implementations (InMemory, JPA, Spring JPA, and the Test subclass), with `ide_change_signature` handling the mechanical update of all callers.

The scaffold team can now badge plan items as sub-case delegations, show what they delegate to, and link to the running child cases — all from a single `getPlanItems` call.
