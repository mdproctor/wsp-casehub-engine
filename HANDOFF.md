# Handoff — 2026-05-31

**Head commit (engine):** 9f40f5c — docs(engine#299): add multi-tenancy foundation design spec
**Head commit (workspace):** ba8a068 — archive plan to attic
**Both repos on:** main

## What Changed This Session

**`issue-299-multi-tenancy-foundation` closed and merged to upstream.**

- Explicit `String tenancyId` on every SPI method (no CDI injection in repositories)
- `public String tenancyId` field on 4 domain objects + 4 records; `tenancy_id` column on 8 JPA entities
- All JPA repositories filter/write by tenancyId; update() includes tenancyId in WHERE
- Memory stores filter by tenancyId; `DefaultTestPrincipal` ships in persistence-memory for test classpath
- `BlackboardRegistry`: stored tenancyId in `CaseEntry`, O(1) evict, defense-in-depth in `get(UUID, String)`
- `CaseLifecycleEvent` gains `tenancyId` as 2nd component — **breaking for all observers**
- `CrossTenantEventLogRepository` + `CrossTenantCaseInstanceRepository` in `common/spi/` for recovery/Quartz/DLQ
- Subcase tenancyId inherits from parent (protocol PP-20260531-42fd93)
- Squashed to 9 clean commits, pushed to casehubio/engine main

**Filed during #299:** engine#405 (@CrossTenant CDI producer), engine#406 (DB RLS), engine#407 (WorkerDecisionEvent tenancyId), ADR (CaseMetaModel per-tenant decision pending)

## Immediate Next Step

Pick up engine#404 (registry lifecycle design — eviction, stateless-on-rest, Quartz restart recovery). Run `/work` to begin.

## Cross-Module

**We're breaking** (consumer repos need to update `@ObservesAsync CaseLifecycleEvent` observers — compile error when they pull engine main):
- `claudony` — claudony#143
- `devtown` — devtown#61
- `aml` — aml#47
- `clinical` — clinical#51

## What's Left

- engine#405 — @CrossTenant CDI producer pattern (system-actor principal pending) · S · Low
- engine#406 — DB-level RLS after application-level filtering stable · M · High
- engine#407 — WorkerDecisionEvent tenancyId audit · S · Low
- ADR — CaseMetaModel per-tenant vs global (sentinel tenancyId consideration) · S · Low
- engine#404 — registry lifecycle analysis: eviction + stateless-on-rest + Quartz restart recovery · L · High
- engine#383 — oversight response loop: COMMAND re-triggers routing · M · Med
- engine#384 — PlanItem escalation state · M · Med
- engine#387 — humanTask: dynamic candidateGroups from case context · M · Med
- parent#87 — PLATFORM.md capability table stale · S · Low
- parent#88 — PLATFORM.md casehub-engine-ai · S · Low
- ledger#100 — sequence race under READ COMMITTED (pre-existing) · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| engine#404 | Registry lifecycle analysis | L | High | Design-only; groundwork from #274+#299 done |
| engine#383 | Oversight response loop | M | Med | Unblocked |
| engine#384 | PlanItem escalation state | M | Med | Unblocked |
| engine#387 | humanTask dynamic candidateGroups | M | Med | — |
| parent#87 | PLATFORM.md capability table | S | Low | Quick |

## Key References

- Blog: `blog/2026-05-31-mdp01-tenancy-threading-explicit.md`
- Spec: `proj/docs/specs/issue-299-multi-tenancy-foundation/2026-05-31-multi-tenancy-foundation-design.md`
- Protocol PP-20260531-42fd93: subcase tenancyId inherits from parent
- Garden entries: GE-20260531-935576 (regex parens), GE-20260531-446fea (Quartz job data), GE-20260531-8b1f4e (Maven module direction), GE-20260531-22e747 (Java record cascade)
