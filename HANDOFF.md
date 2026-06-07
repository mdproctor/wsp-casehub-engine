# Handoff — 2026-06-07

**Head commit (engine):** a1e87bf — fix: remaining WorkerResult migration issues — CI green (engine#402)
**Head commit (workspace):** d2137fc — docs(issue-402-action-risk-classifier-spi): mark closed
**Both repos on:** main
**PR open:** casehubio/engine#435 — ActionRiskClassifier SPI (engine#402)

## What Changed This Session

**engine#402 — ActionRiskClassifier SPI: fully implemented, PR #435 open.**

Platform-level oversight gate for consequential worker actions. Workers declare `PlannedAction` in `WorkerResult`; engine classifies with `ReactiveActionRiskClassifier`; `GateRequired` creates a WorkItem pending human approval. Approved gates re-fire `WorkflowExecutionCompleted(plannedAction=null)` — reuses entire completion machinery. `pendingActionGate` is in-memory only (not JPA-persisted) — resolution handlers use `CaseInstanceCache`, not `CrossTenantRepo`.

Key components: `ChainedReactiveActionRiskClassifier` (@RiskClassifier qualifier, most-restrictive-wins), `WorkflowExecutionCompletedHandler` gate fork, `ActionGateWorkItemHandler`, `ActionGateCompletionApplier`, `CallerRef` sealed hierarchy (PlanItemCallerRef|GateCallerRef), gate resolution handlers (approved/rejected/expired), blackboard `ActionGateWorkerFaultedPlanItemHandler`.

Breaking change: all worker function lambdas and `Agent.execute()` return `WorkerResult` (51+ test files updated). SW task `function(lambda, Map.class)` lambdas must NOT be wrapped — see gotcha GE-20260607-115619.

CI note: "Build and test" passes. "Publish to GitHub Packages" fails with 403 — pre-existing fork configuration issue, not a code bug.

## Immediate Next Step

Review and merge PR #435. Then devtown#62 (set `bootstrapEscalationRequired=true`) is already unblocked (engine#427 merged last session).

## Cross-Module

**Blocking** (other modules waiting on PR #435 merge):
- `aml` — aml#42: SAR filing, account freeze, law enforcement referral · L · Med
- `clinical` — clinical#47: SUSAR filing, dose modification, patient withdrawal · L · Med
- `devtown` — devtown#56: production deploy, contributor access, security escalation · M · Med
- `life` — life#20: spend threshold, non-refundable bookings, contractor instruction · M · Low
- `openclaw` — openclaw#6: oversight channel gate (Epic 6 end-to-end wiring) · L · High

## What's Left

- PR #435 pending review/merge
- engine#433: persist `pendingActionGate` in `CaseInstanceEntity` (v2, restart resilience) · M · Med
- engine#434: integration test for classifier-throws fail-safe in full engine pipeline · S · Low
- parent#183: sync PLATFORM.md + casehub-engine.md deep-dive for engine#402 · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | AI Fusion typed fact space | XL | High | New module — own session |
| engine#404 | Registry lifecycle design | L | High | Design-only |
| engine#383 | Oversight response loop | M | Med | Unblocked |
| engine#384 | PlanItem escalation state | M | Med | Unblocked |
| engine#387 | humanTask dynamic candidateGroups | M | Med | — |

## Key References

- PR: https://github.com/casehubio/engine/pull/435
- Spec: `docs/specs/2026-06-05-action-risk-classifier-design.md`
- Blog: `blog/2026-06-07-mdp01-action-risk-classifier-gate.md`
- Garden: GE-20260607-b6478d (pendingActionGate in-memory), GE-20260607-cedf69 (@Startup/@PostConstruct), GE-20260607-115619 (SW lambda migration), GE-20260607-66daf2 (re-fire technique), GE-20260607-4bb9a7 (cross-test await)
