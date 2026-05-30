# Handoff — 2026-05-30

**Head commit (engine):** 6e987d0 — fix: await WorkerStarted fireAsync; trim candidateGroups; comment last-wins
**Head commit (workspace):** b7047ee — feat: promote blog + settings from issue-392-sxs-batch
**Both repos on:** main

## What Changed This Session

**`issue-392-sxs-batch` closed — PR#401 merged to casehubio/engine.**

S/XS batch (9 commits, 8 issues closed):
- #392 (XS) — disabled-path guard tests for CaseLedger + WorkerDecision captures
- #389 (S) — ProvisionResult return type + WorkerStarted lifecycle event
- #393 (S) — await CaseLifecycleEvent CDI delivery (fireAsync in .invoke() fix, pattern established)
- #395 (XS) — Flyway migrations path: `db/migration/` → `db/engine-ledger/migration/` (**AML unblocked**)
- #399 (XS) — callerRef passed as assigneeIdOverride bug in HumanTaskScheduleHandler
- #397 (S) — await fireAsync in 5 remaining lifecycle event handlers + workerDecisionEvents
- #396 (S) — CaseLedgerEntryRepository @DefaultBean yields to selected alternatives (**AML unblocked**)
- #400 (S) — WorkItem escalation → `workItemEscalated` context signal

Code review caught one critical miss: WorkerStarted fireAsync in tryProvision() also used .invoke() — fixed before merge.

Garden: GE-20260530-9a5474 (gh auth token lacks read:packages). Protocol: PP-20260530-8725fa (engine library @Alternative subclass → @DefaultBean); PP-20260529-3237bd revised (broadened to all CDI fireAsync).

## Immediate Next Step

Start **engine#274** — BlackboardRegistry hydration from PlanItemStore on restart (M·Med, unblocked since engine#273 closed). Run `/work` to begin.

## Cross-Module

**We're blocking:** none — #395 and #396 merged; AML can now add casehub-engine-ledger.

**AML tracker:** casehubio/aml#14, casehubio/aml#9 — Layer 6 unblocked; AML can wire casehub-engine-ledger now that CDI ambiguity and Flyway path are fixed.

## What's Left

- engine#274 — BlackboardRegistry hydration from PlanItemStore on restart · M · Med ← NEXT
- engine#398 — HumanTask completion silently dropped after JVM restart · M · Med
- engine#383 — oversight response loop: COMMAND re-triggers routing · M · Med ← UNBLOCKED
- engine#384 — PlanItem state during escalation (ESCALATING?) · M · Med ← UNBLOCKED
- engine#387 — humanTask: dynamic candidateGroups from case context · M · Med
- engine#299 — multi-tenancy foundation · L · High ← UNBLOCKED (platform#17 closed)
- parent#87 — PLATFORM.md capability table stale · S · Low
- parent#88 — PLATFORM.md casehub-engine-ai and AgentEmbeddingProvider · S · Low
- ledger#100 — sequence race under READ COMMITTED (pre-existing) · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| engine#274 | BlackboardRegistry hydration on restart | M | Med | Unblocked, do next |
| engine#383 | Oversight response loop | M | Med | Unblocked |
| engine#398 | HumanTask completion lost after JVM restart | M | Med | Related to #274 |
| engine#384 | PlanItem escalation state | M | Med | Unblocked |
| parent#87 | PLATFORM.md capability table stale | S | Low | — |
| parent#88 | PLATFORM.md casehub-engine-ai | S | Low | — |

## Key References

- Blog: `blog/2026-05-30-mdp01-six-handlers-one-miss.md`
- Protocol: PP-20260530-8725fa — engine library @Alternative subclass → @DefaultBean
- Garden: GE-20260530-9a5474 — gh auth token lacks read:packages (tools)
