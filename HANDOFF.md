# Handoff — 2026-06-26

## What's Done

**engine#570 — Use ExpressionEngineRegistry for output schema evaluation (CLOSED)**

Added `transform()` to `ExpressionEngine` SPI and `ExpressionEngineRegistry`. Replaced hardcoded `JQEvaluator` in `DefaultWorkerExecutor` with the pluggable registry — same infrastructure as all other expression evaluation. 8 files, 181 insertions. All tests green.

## What's Left

- Consumer repo import migration (aml#69, clinical#92, devtown#96, life#44) · M · Low each
- work-adapter template-mode tests have 9 pre-existing failures (HumanTaskScheduleHandlerTest, HumanTaskPlannerIntegrationTest) — not introduced by this branch

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| aml#69 | Propagate worker-api imports to aml | S | Low | Mechanical import swap |
| clinical#92 | Propagate worker-api imports to clinical | S | Low | Mechanical import swap |
| devtown#96 | Propagate worker-api imports to devtown | S | Low | Mechanical import swap |
| life#44 | Propagate worker-api imports to life | M | Low | 15 files |
