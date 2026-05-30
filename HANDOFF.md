# Handoff — 2026-05-31

**Head commit (engine):** c347873 — chore: branch closed
**Head commit (workspace):** 9f85c59 — docs: mark closed, deletion due 2026-06-14
**Both repos on:** main

## What Changed This Session

**`issue-274-registry-hydration-recovery` closed.**

- `BlackboardRegistry.get()` now lazily hydrates DELEGATED PlanItems from `PlanItemStore` on first miss after JVM restart — closes engine#274
- `HumanTaskRecoveryService` at `@Priority(25)` scans `findAllDelegated()` at startup and catches up WorkItems that terminated during downtime — closes engine#398
- Four `@ConsumeEvent` handlers in blackboard needed `blocking = true` as a consequence (JPA call now inside `registry.get()`)
- `PlanItemCompletionApplier` extracted — shared completion logic for both `WorkItemLifecycleAdapter` and `HumanTaskRecoveryService`
- `PlanItemStore` extended: `PlanItemSaveRequest` value object, `TargetType` enum, `findDelegated(UUID)`, `findAllDelegated()`
- Protocol PP-20260530-40a73c captured: `@ConsumeEvent` callers of `registry.get()` must declare `blocking = true`

**casehub-work:** `findByCallerRef()` added on branch `issue-235-sxs-sweep` (not on work main yet — needs PR/merge).

**Filed:** engine#404 — registry lifecycle analysis: eviction strategies, stateless-on-rest pattern, full Quartz restart recovery (RUNNING items + completionIndex).

## Immediate Next Step

Pick up engine#383 (oversight response loop) or engine#404 (registry lifecycle design). Run `/work` to begin.

## Cross-Module

**We're blocking:** none.

**AML tracker:** aml#14, aml#9 — unblocked since last session (engine#395, engine#396 already merged).

**casehub-work note:** `findByCallerRef()` is on branch `issue-235-sxs-sweep`, not main. Before consuming `WorkItemService.findByCallerRef()` from another module, that branch needs to be merged to work main.

## What's Left

- engine#404 — registry lifecycle analysis: eviction, stateless-on-rest, Quartz restart recovery · L · High ← NEW
- engine#383 — oversight response loop: COMMAND re-triggers routing · M · Med
- engine#384 — PlanItem state during escalation (ESCALATING?) · M · Med
- engine#387 — humanTask: dynamic candidateGroups from case context · M · Med
- engine#299 — multi-tenancy foundation · L · High
- parent#87 — PLATFORM.md capability table stale · S · Low
- parent#88 — PLATFORM.md casehub-engine-ai · S · Low
- ledger#100 — sequence race under READ COMMITTED (pre-existing) · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| engine#404 | Registry lifecycle analysis — eviction + stateless-on-rest | L | High | Design-only; groundwork laid by #274 |
| engine#383 | Oversight response loop | M | Med | Unblocked |
| engine#384 | PlanItem escalation state | M | Med | Unblocked |
| engine#387 | humanTask dynamic candidateGroups | M | Med | — |
| engine#299 | Multi-tenancy foundation | L | High | Unblocked (platform#17 closed) |
| parent#87 | PLATFORM.md capability table stale | S | Low | — |

## Key References

- Blog: `blog/2026-05-30-mdp02-registry-hydration-recovery.md`
- Protocol: PP-20260530-40a73c — `@ConsumeEvent` + `blocking = true` for `registry.get()` callers
- Spec: `docs/specs/2026-05-30-registry-hydration-recovery-design.md` (promoted to engine repo)
