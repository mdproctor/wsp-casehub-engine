# Handoff — 2026-06-25

## What's Done

**engine#567 — Remove serverlessworkflow SDK from engine-api (CLOSED)**

Replaced hardcoded switch dispatch in DefaultWorkerExecutor with pluggable handler model. Two new SPIs: WorkerFunctionHandler (execution) and WorkerFunctionProvider (YAML construction). FlowWorkerFunction moved from api to flow module. WorkerFunction became marker interface. SDK removed from api, schema, common, runtime — only flow keeps it.

3 commits on main:
- `docs(#567): design spec — remove serverlessworkflow SDK via pluggable handler model`
- `refactor(#567): remove serverlessworkflow SDK from engine-api, pluggable handler model`
- `docs(#567): update CLAUDE.md for pluggable handler model`

Filed: engine#570 (output schema evaluation should use ExpressionEngineRegistry).

## What's Left

- engine#570 — output schema evaluation should use ExpressionEngineRegistry · S · Low
- Consumer repo import migration (aml#69, clinical#92, devtown#96, life#44, claudony#157) · M · Low each

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #570 | Output schema evaluation uses ExpressionEngineRegistry | S | Low | Surfaced during #567 design |
| aml#69 | Propagate worker-api imports to aml | S | Low | Mechanical import swap |
| clinical#92 | Propagate worker-api imports to clinical | S | Low | Mechanical import swap |
| devtown#96 | Propagate worker-api imports to devtown | S | Low | Mechanical import swap |
| life#44 | Propagate worker-api imports to life | M | Low | 15 files |
| claudony#157 | Propagate worker-api imports to claudony | M | Low | 19 files |
