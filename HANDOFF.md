# Handoff — 2026-06-17

**Branch:** `issue-463-function-worker-design` — CLOSED, merged to main, pushed to upstream

## What's Done

- engine#463 closed — sealed WorkerFunction, WorkerExecutor SPI, DefaultWorkerExecutor, QuartzRetryService, fire-and-forget Quartz adapter, WORKFLOW_EXECUTION_FAILED deleted
- PR#499 merged (signal API #493, implementation routing #476, repeatable stage #482, auto-registration #497)
- engine#498 closed (CDI protocol updated in garden)
- parent#261 filed (casehub-engine.md deep-dive stale WorkflowExecutionFailed reference)
- GE-20260616-0175da submitted (ReactiveUtils.runOnSafeVertxContext mock Vertx gotcha)
- AgentDescriptor briefing parameter fix (casehub-eidos-api dependency update)
- 1,837 tests green across all modules

## Immediate Next Step

DevTown needs engine#502, #503, #504, #506. Start with #502 (agent exclusion — S/Low, no blockers).

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #502 | Agent exclusion — read excludedAgents from case context in routing | S | Low | For DevTown |
| #503 | Semantic DECLINE/FAIL — write structured failure state to blackboard | L | High | For DevTown |
| #504 | OutcomePolicy — retry/re-route for speech-act outcomes | L | High | Blocked by #503 |
| #506 | Failure goals → COMPLETED not FAULTED | S | Med | For DevTown |
