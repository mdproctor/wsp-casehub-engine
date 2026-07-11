# Handoff — 2026-07-12

## What's Done

**CBR routing pipeline (#706) — landed on main.** Three child issues closed: #703 (CbrCaseRetainObserver stores PlanCbrCase on case close), #505 (strategies consume context.experiences(), dead code removed), #705 (superseded — RoutingFeatureExtractor deleted instead of promoted). Design review (4 rounds, 18 issues, all resolved). Blocks repo updated with 3 commits (strategy refactors + dead code removal).

## Immediate Next Step

**Fix flaky runtime tests.** ActionGateIntegrationTest (6 errors), ActionGateResolutionTest (3 errors), CaseLifecycleCdiEventTest (1 error) — pre-existing Awaitility timeouts, not caused by #706.

## Cross-Module

**Blocks repo** has 3 new commits on main (AgentAssignment→RoutingResult migration fix, strategy refactors, dead code removal). These depend on the latest engine-api SNAPSHOT.

**Blocks adoption** — four issues filed:
- blocks#50 — `AgentRef extends ExecutorRef`
- blocks#51 — `PlannedTask implements TaskDescriptor`
- blocks#52 — `SubTaskStatus` → `TaskStatus`

## What's Left

- Flaky tests — ActionGate + CaseLifecycle timeout failures · S · Med
- #680 — thread tenancyId through event bus messages · M · Med
- #646 — per-case CONTEXT_CHANGED serialization · M · Med
- #702 — event/handler ExecutorRef migration · M · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #694 | DAG plan structure | L | High | Natural vehicle for shared Plan type |
| #689 | WorkItems boundary — typed payload/resolution | M | Med | |
| #690 | SubCase boundary — typed context passing | S | Med | |
| #691 | Signals boundary — typed signal overload | S | Med | |
| #692 | Connectors boundary — typed inbound payloads | S | Med | |
| #635 | Rename io.casehub.api → io.casehub.engine.api | L | Low | Cross-repo |

## References

- Spec: `docs/specs/2026-07-11-cbr-routing-pipeline-design.md`
- Design review: `~/adr/casehub-engine/cbr-routing-pipeline-20260711-190819/`
- Garden: GE-20260712-626e51 (@DefaultBean in external module not discovered via index-dependency)
