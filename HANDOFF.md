# Handoff — 2026-06-24

## What's Done

**engine#543 — Worker primitives migration (CLOSED)**

Migrated 9 Worker type files from engine-api to casehub-worker-api and casehub-platform-api foundation dependencies. 128 files changed across all 15 modules. All tests pass.

4 commits landed on main:
- `docs(#543): worker primitives migration design spec`
- `feat(#543): add AgentWorkerFunction, FlowWorkerFunction, ClassificationContext`
- `feat(#543): add AgentDescriptor map to CaseDefinition`
- `refactor(#543): migrate Worker primitives to casehub-worker-api`

Filed: engine#567 (remove serverlessworkflow SDK from engine-api). 3 garden entries submitted (lambda ambiguity, null schemas, RetryPolicy validation).

## What's Left

- engine#562 — propagate worker-api imports to consumer repos · M · Low
- engine#567 — remove serverlessworkflow SDK from engine-api · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #562 | Propagate worker-api imports to consumer repos | M | Low | Downstream of #543 |
| #567 | Remove serverlessworkflow SDK from engine-api | M | Med | Requires mapper restructuring |
