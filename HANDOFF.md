# Handoff — 2026-07-11

## What's Done

**Unified orchestration model (#700) — landed on main.** Introduced shared types: TaskStatus (replaces PlanItemStatus), TaskDescriptor (behavioral interface, PlanItem implements it), TaskSnapshot (read model), RoutingResult (replaces AgentAssignment), ExecutorRef on PlanItem (replaces workerName), Assignment, OutcomeKind. Adversarial design review (3 rounds, 15 issues). Four deferred issues filed for blocks adoption (blocks#50-52, engine#702).

## Immediate Next Step

**Fix flaky runtime tests.** `ActionGateIntegrationTest` (6 errors), `ActionGateResolutionTest` (3 errors), `CaseLifecycleCdiEventTest` (1 error) — Awaitility timeouts, not assertion failures. Pre-existing, not caused by #700.

## Cross-Module

**Worker repo** has branch `issue-203-context-bridge-protocol` that needs to land alongside engine changes.

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
