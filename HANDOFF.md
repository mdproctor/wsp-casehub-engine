# Handoff — 2026-06-27

## What's Done

**#574: M-of-N sub-case groups in YAML + per-child outputMapping fix — CLOSED**

Added `groupId`, `totalInGroup`, `requiredCount`, `onThresholdReached` to SubCase YAML schema and mapper. Fixed silent data loss in `SubCaseCompletionService.handleGroupedCompletion()` — outputMapping now applies for every completing child, not just the threshold-triggering one. Unblocks devtown#11 bisection.

**Known pre-existing:** work-adapter compilation fails against current casehub-work SNAPSHOT — `WorkItemStatus` and `WorkItemCreateRequest` moved packages upstream. Tracked as #565.

## What's Left

- Consumer repo import migration (aml#69, devtown#96) · S · Low each
- engine#565: re-publish SNAPSHOT · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| aml#69 | Propagate worker-api imports to aml | S | Low | Mechanical import swap |
| devtown#96 | Propagate worker-api imports to devtown | S | Low | Mechanical import swap |
