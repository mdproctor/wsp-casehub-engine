# Handoff — 2026-07-12

*Updated: cross-module corrected, CBR + DAG/planning issues added.*

## What's Done

**CBR routing pipeline (#706) — landed on main.** Three child issues closed: #703 (CbrCaseRetainObserver stores PlanCbrCase on case close), #505 (strategies consume context.experiences(), dead code removed), #705 (superseded — RoutingFeatureExtractor deleted instead of promoted). Design review (4 rounds, 18 issues, all resolved).

## Immediate Next Step

**Fix flaky runtime tests.** ActionGateIntegrationTest (6 errors), ActionGateResolutionTest (3 errors), CaseLifecycleCdiEventTest (1 error) — pre-existing Awaitility timeouts, not caused by #706.

## Cross-Module

**We're blocking blocks:** #694 (DAG plan structure) gates blocks#44 (agentic planning architecture epic) · L · High

## What's Left

- Flaky tests — ActionGate + CaseLifecycle timeout failures · S · Med
- #680 — thread tenancyId through event bus messages · M · Med
- #646 — per-case CONTEXT_CHANGED serialization · M · Med
- #702 — event/handler ExecutorRef migration · M · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| **CBR follow-on** | | | | |
| #704 | CbrRetrievalService — generalize beyond PlanCbrCase | S | Low | |
| #672 | Feature-level similarity breakdown in RetrievedExperience | S | Med | |
| **DAG/Planning cluster** | | | | **blocks#44 waiting on #694** |
| #694 | DAG plan structure — ExecutionPlan with dependency edges | L | High | Blocks #695, #697, #698 |
| #695 | DAG-aware parallel execution driver | L | High | Depends on #694 |
| #697 | Plan versioning — immutable plan snapshots | M | Med | Depends on #694 |
| #696 | Multi-level recovery protocol | M | High | Depends on #694 + #697 |
| #698 | Context isolation per task | M | Med | Depends on #694 |
| **HTN** | | | | |
| #600 | HTN — hierarchical task decomposition | L | High | Under #595 epic |
| **Boundaries** | | | | |
| #689 | WorkItems boundary — typed payload/resolution | M | Med | |
| #690 | SubCase boundary — typed context passing | S | Med | |
| #691 | Signals boundary — typed signal overload | S | Med | |
| #692 | Connectors boundary — typed inbound payloads | S | Med | |
| **Other** | | | | |
| #635 | Rename io.casehub.api → io.casehub.engine.api | L | Low | Cross-repo |

## References

- Spec: `docs/specs/2026-07-11-cbr-routing-pipeline-design.md`
- Design review: `~/adr/casehub-engine/cbr-routing-pipeline-20260711-190819/`
- Garden: GE-20260712-626e51 (@DefaultBean in external module not discovered via index-dependency)
