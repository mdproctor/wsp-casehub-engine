# Handoff — 2026-07-12

## What's Done

**TenancyId threading (#680) — landed on main.** Root fix: `TenantAwareRepository.withTenantTransaction()` parameterized with explicit tenancyId — removes ambient `CurrentPrincipal` dependency. 10 event records gain tenancyId. PlanItemStore SPI evolved (updateStatus tenancyId overload, findDelegatedCrossTenant rename). Test workarounds removed. Design review (3 rounds, 12 issues, all resolved). Spec at `docs/specs/2026-07-12-thread-tenancyid-event-bus-design.md`.

**CBR generalization (#704, #672, #694) — landed on main.** All three repos pushed and delivered.

## Immediate Next Step

All pushed. No pending deliveries. Pick next work from What's Next.

## Cross-Module

*Unchanged — `git show HEAD~1:HANDOFF.md`*

New: engine#710 — work-adapter must populate tenancyId in ActionGate completion events (cross-repo follow-up from #680).

## What's Left

- #710 — work-adapter: populate tenancyId in ActionGate completion events · S · Low (cross-repo)
- #709 — SubCaseGroupLifecycleEvent tenancyId (CDI event consistency) · XS · Low
- #646 — per-case CONTEXT_CHANGED serialization · M · Med
- #702 — event/handler ExecutorRef migration · M · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #695 | DAG-aware parallel execution driver | L | High | Unblocked by #694 |
| #600 | HTN — hierarchical task decomposition | L | High | Under #595 epic |
| #689 | WorkItems boundary — typed payload | M | Med | |
| #635 | Rename io.casehub.api → io.casehub.engine.api | L | Low | Cross-repo |

## References

- Spec: `docs/specs/2026-07-12-thread-tenancyid-event-bus-design.md`
- Design review: `~/adr/casehub-engine/thread-tenancyid-event-bus-20260712-185729/`
- Garden: GE-20260712-1a82c4 (ide_edit_member truncation), GE-20260712-4a8a3c (record field reorder silent bug)
