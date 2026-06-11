# Handoff — 2026-06-11

**Head commit (engine):** e4413172 — feat(blackboard): add tenancyId to PlanItemCompletedEvent and SubCaseExecutionCompleted
**Head commit (workspace):** fc96142 — feat: promote blog from issue-460-multitenancy-fix-batch
**Both repos on:** main
**PR #470:** OPEN — multi-tenancy fix batch → casehubio/engine

## What Changed This Session

**#460, #459, #450, #429 — all closed.** Multi-tenancy fix batch on branch `issue-460-multitenancy-fix-batch` (now closed).

- **#460** — already fixed in prior session (dc37cd50); closed without code change
- **#459** — `WorkItemGroupLifecycleEvent.of()` SNAPSHOT cascade: three test call sites in `WorkItemLifecycleAdapterTest` updated with new `tenancyId` 9th arg
- **#450** — `CaseLedgerEntryRepository.caseEm` missing `@LedgerPersistenceUnit` qualifier; silent wrong-PU in multi-datasource
- **#429** — `tenancyId` added to `PlanItemCompletedEvent` and `SubCaseExecutionCompleted`; eliminates `CrossTenantCaseInstanceRepository` workaround in devtown#43

**New protocols:** PP-20260611-d4e5cf (CDI async events must carry tenancyId) · PP-20260611-cc63b4 (LedgerRepository subclass EM qualifier)

## Immediate Next Step

Review PR #470 or start `engine#465` (validate panel event model serves Drools re-fire triggers). Run `/work` to begin.

## What's Left

- **PR #470** — open on casehubio/engine, awaiting merge · XS · Low
- **engine#466** — review `casehub-platform` compile scope in `runtime/pom.xml` · XS · Low · not urgent
- **devtown#43** — can now use `event.tenancyId()` directly; devtown team should remove `CrossTenantCaseInstanceRepository` workaround

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| engine#465 | Validate panel event model serves Drools re-fire triggers | XS | Low | Do before #446 |
| engine#446 | WorkingMemoryBridge — typed Drools facts from named panels | M | Med | Unblocked |
| engine#5 | DroolsExpressionEvaluator | M | Med | Depends on #289 (done) |
| engine#207 | RulesRouter + RULES_DECISION lineage | L | Med | Final Drools piece — depends on #446 |
| engine#383 | Oversight response loop | M | Med | Unblocked |
| engine#448 | Worker(Plan.of(...)) function type | M | Med | Plan-based execution Phase 1 |

## Key References

- Blog: `wksp/blog/2026-06-11-mdp01-the-qualifier-nobody-inherits.md`
- Protocols: PP-20260611-d4e5cf (CDI async event tenancyId) · PP-20260611-cc63b4 (LedgerRepository EM qualifier)
- Garden: GE-20260611-a42c0b (unqualified EntityManager silently wrong PU in multi-datasource)
