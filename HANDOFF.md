# Handoff — 2026-07-15

## What's Done

CI green. AML diagnostic complete. Unified execution model spec drafted.

- 10 commits landed on main (d7e79017..a6df47c4): SNAPSHOT drift fixes, null inputData NPE, binding re-dispatch loop, test profile contamination, Flyway migration, recovery test timing, YAML mapper fixes
- AML tenant mismatch: reproduced, diagnosed (package relocation CDI exclusion + null inputData + binding re-dispatch). Diagnostic sent. Consumer reproduction test added.
- Unified execution model spec at `docs/specs/2026-07-15-unified-execution-model-design.md` — has contradictions from iterative design discussion that need cleaning up before sharing with trebleel
- 4 stranded blog entries promoted to workspace main, 1 published to mdproctor.github.io

## Immediate Next Step

Revisit `docs/specs/2026-07-15-unified-execution-model-design.md` — resolve contradictions (choreography listed both as dispatch mode AND strategy name), incorporate LangChain4j GoalOrientedPlanner mapping, verify coverage against engine#101 sub-issues.

## Cross-Module

- AML — rebuild against latest engine SNAPSHOT (binding re-dispatch fix b1e9a4c3 + null inputData fix 55a602e6). Gate issue is CDI exclusion of relocated work-engine-adapter classes.
- casehub-work — engine SNAPSHOTs now published to GitHub Packages. CI should resolve.
- casehub-blocks — DAG unification (Phase 0: retire ExecutionPlan, adopt DagPlan) is first convergence step. Requires sequentialMerge() on DagPlan.

## What's Left

- Spec contradictions: choreography appears as both a dispatch archetype and a planning strategy name in the tables · S · Med
- engine#101 sub-issues: verify unified model covers all LangChain4j patterns (Sequential, Supervisor, GoalOriented, P2P, Loop, Conditional) · M · Med
- 202 local engine branches + 90 workspace branches stamped/closed — bulk deletion approved but not executed · XS · Low
- casehub-blocks: TrustRoutingPolicy 8th param propagation (3 test files) · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Rewrite spec: clean model (two dispatch modes, planning algorithms under orchestration, DagPlan as output format) | M | Med | Coordinate with trebleel before implementing |
| — | Phase 0: sequentialMerge() on DagPlan | S | Low | Prerequisite for blocks ExecutionPlan retirement |
| — | Phase 1: Retire ChoreographyLoopControl | M | Med | PlanningStrategyLoopControl becomes sole LoopControl |
| #732 | Wire CaseContextStoreFactory through recovery path | M | Med | Blocks durable factories |
| #600 | HTN — hierarchical task decomposition | L | High | Under #595 epic — spec informs design |
| #689 | WorkItems boundary — typed payload | M | Med | ContextBridge arc |
