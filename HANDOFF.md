# Handoff — 2026-06-23

## What's Done

**engine#543 — Worker primitives migration (IN PROGRESS)**

Spec approved (round 3). New types created with TDD: `FlowWorkerFunction`, `AgentWorkerFunction`, `ClassificationContext`, `CaseDefinition.agentDescriptorFor()`. SPI updated: `ActionRiskClassifier.classify(PlannedAction, ClassificationContext)`.

9 old types deleted. Production source compiled clean across all modules. Test compilation partially fixed (~100 errors resolved, ~100 remaining).

**Blocked by linter/subagent interaction.** Subagents recreated deleted files (GE-20260623-ec9c80). Spotless linter reverts POM and import changes between tool calls. Working tree is in a mixed state.

Filed: parent#304 (BOM entry), parent#305 (PLATFORM.MD types), engine#562 (consumer import propagation).

## Immediate Next Step

Reset working tree to clean state: `git -C $PROJECT checkout .` then replay the migration atomically. Six new untracked files survive the reset — commit them first:
`AgentWorkerFunction.java`, `FlowWorkerFunction.java`, `ClassificationContext.java` (production) + 3 test files.

Execution order: (1) commit new untracked files, (2) add `casehub-worker-api` dep to root + api POMs, (3) delete 9 old types and commit, (4) fix downstream imports/accessors — commit per module, (5) fix tests. See spec at `docs/specs/2026-06-22-worker-primitives-migration-design.md` and `design/JOURNAL.md` for session notes on what went wrong.

## What's Left

- Complete engine#543 migration — working tree needs reset and clean replay · L · Med (spec done, execution is mechanical)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #543 | Migrate Worker primitives to casehub-worker-api | L | Med | Spec done, replay needed |
