# Handoff — 2026-05-29

**Head commit (engine):** b13f5da — docs(claude-md): add ProvisionResult SPI return type + WorkerStarted wiring row
**Head commit (workspace):** (branch issue-392-sxs-batch, workspace main via stash/commit)
**Project on:** issue-392-sxs-batch
**Workspace on:** issue-392-sxs-batch

## What Changed This Session

**`issue-392-sxs-batch` in progress — all three S/XS issues committed, branch open for PR.**

All three issues resolved (5 commits on engine):
- #392 (XS) — disabled-path guard tests for `CaseLedgerEventCapture` and `WorkerDecisionEventCapture`
- #389 (S) — `ProvisionResult` SPI return type; `WorkerProvisioner.provision()` now returns `ProvisionResult` not `Worker`; `tryProvision()` fires `WorkerStarted` lifecycle event; claudony#140 filed
- #393 (S) — `CaseStatusChangedHandler` restructured to await CDI delivery via `.chain(completionStage())`; engine#397 filed for 5 remaining handlers

Garden: GE-20260513-b15933 REVISED (inner-class @ObservesAsync build-time non-registration); GE-20260529-e43076 (`.invoke()` → `.chain(completionStage())` technique).
Protocols: PP-20260529-3237bd (CDI event await chain rule), PP-20260529-bcbbb5 (ProvisionResult SPI rule).
Blog: `blog/2026-05-29-mdp03-worker-carries-a-definition.md`.

## Immediate Next Step

Push `issue-392-sxs-batch` to engine remote and open PR. Then start **engine#274** — BlackboardRegistry hydration from PlanItemStore on restart (M·Med).

## What's Left

- engine#274 — BlackboardRegistry hydration from PlanItemStore on restart · M · Med
- engine#383 — Oversight response loop: COMMAND → re-trigger routing · M · Med
- engine#384 — PlanItem state during escalation (ESCALATING state?) · M · Med
- engine#397 — await fireAsync in remaining 5 lifecycle event handlers · S · Low
- parent#87 — PLATFORM.md capability table stale · S · Low
- parent#88 — PLATFORM.md casehub-engine-ai and AgentEmbeddingProvider · S · Low
- ledger#100 — sequence race under READ COMMITTED · M · Med
- claudony#140 — wire causedByEntryId through provisioner (depends on engine#231)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| engine#274 | BlackboardRegistry hydration from PlanItemStore on restart | M | Med | After PR#394 merge |
| engine#397 | await fireAsync in 5 remaining lifecycle handlers | S | Low | Pattern established — apply to GoalReached, Milestone, Signal, CaseStarted, WorkflowCompleted |
| engine#383 | Oversight response loop: COMMAND re-triggers routing | M | Med | — |
| parent#87 | PLATFORM.md capability table stale | S | Low | — |

## Key References

- Branch: `issue-392-sxs-batch` (engine + workspace, not yet pushed to remote)
- PR: open PR#394 still awaiting merge (from previous session)
- Blog: `blog/2026-05-29-mdp03-worker-carries-a-definition.md`
- Protocols: PP-20260529-3237bd, PP-20260529-bcbbb5
- Filed: claudony#140, engine#397
