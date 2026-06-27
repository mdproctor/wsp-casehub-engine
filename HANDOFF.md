*Updated: clinical#92, life#44 closed — removed from backlog.*

# Handoff — 2026-06-27

## What's Done

**Batch XS/S fixes (#575, #569, #576, #568, #563) — ALL CLOSED**

Single-commit batch: PlanItemStatus Javadoc (#575), AgentBuilder.model public (#569), GateDecision→GateOutcome rename (#563), template-mode test fix (#576), CI rename (#568). CLAUDE.md updated for template store pattern and visibility change.

**Known pre-existing:** work-adapter compilation fails against current casehub-work SNAPSHOT — `WorkItemStatus` and `WorkItemCreateRequest` moved packages upstream. Tracked as #565 (re-publish SNAPSHOT). Core modules build and test clean.

## What's Left

- Consumer repo import migration (aml#69, devtown#96) · S · Low each
- engine#565: re-publish SNAPSHOT — resolves itself when CI runs on main (just pushed)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #574 | Expose M-of-N sub-case groups in YAML + fix per-child outputMapping | S | Med | Blocks devtown#11 |
| aml#69 | Propagate worker-api imports to aml | S | Low | Mechanical import swap |
| devtown#96 | Propagate worker-api imports to devtown | S | Low | Mechanical import swap |
