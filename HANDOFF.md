# Session Handover — 2026-09-23

## What happened

Completed 8 issues from the persistence coherence epic (#1150), advancing the queue from position 4/17 to 12/17. All changes are on `issue-1150-persistence-coherence` branch, committed but not pushed.

### Issues completed

| # | Title | Scale | Files |
|---|-------|-------|-------|
| #1153 | PlanItemStore.updateStatus abstract/default swap for tenancy safety | XS | 17 |
| #1154 | Episodic layer replay — restore baseline and goal tracking | S | 3 |
| #1155 | CrossTenant* Javadoc — system services, not recovery only | XS | 2 |
| #1156 | Extract cross-tenant methods from PlanItemStore to CrossTenantPlanItemStore | S | 15 |
| #1157 | Document Repository vs Store SPI naming convention | XS | 1 |
| #1158 | Standardize null vs Optional across persistence SPIs | M | 31 |
| #1159 | Move ExecutionSnapshotStore to common.spi.recovery package | XS | 44 |
| #1160 | Contract Javadoc for PlanVersionStore, ExecutionSnapshotStore, CaseQueueEntryStore | XS | 3 |

### Key changes

- **PlanItemStore SPI split:** Tenant-scoped methods on `PlanItemStore`, cross-tenant on new `CrossTenantPlanItemStore`. `findDelegated(UUID, String)` is now abstract (was default delegating to cross-tenant). `updateStatus` requires tenancyId (2-arg removed).
- **Optional standardization:** `CaseInstanceRepository.findByUuid`, `CrossTenantCaseInstanceRepository.findByUuid`, `CaseMetaModelRepository.findByKey`, `CrossTenantEventLogRepository.findById` all return `Optional` now. ~30 callers updated.
- **Package moves:** `ExecutionSnapshotStore` → `common.spi.recovery`. InMemory impls → `common.internal.store`.
- **Episodic replay:** `initBaseline()` before replay, `GOAL_REACHED` in replay filter, `recordGoalReached` wired into `GoalReachedEventHandler`.

## Decisions

- InMemory impls for ExecutionSnapshotStore/PlanVersionStore stayed in `common-core` (in `common.internal.store`) rather than moving to `engine-support-core` — Maven cycle prevents `common-core` test-scope dep on `engine-support-core`.
- `BlackboardRegistry` resolves `CrossTenantPlanItemStore` via `instanceof` check on the `PlanItemStore` constructor param — avoids breaking 20+ test constructors.

## Known issues

- **Pre-existing Spring drift detection failures** in `runtime-spring`, `planning-spring`, `engine-support-spring`, `ledger-spring` — `verify-drift` goal fails due to 23+ Quarkus types with no Spring equivalent. Not caused by this branch.
- **Pre-existing `PersistenceAutoConfiguration` fix** from #1166 was included in the #1153 commit (constructor needed `CaseContextRecoveryStrategy` ObjectProvider).

## Queue

Position 12/17. Active: #1161 — feat: add JPA and Spring JPA implementations for CaseQueueEntryStore.

## References

| Artifact | Path |
|----------|------|
| Design spec | `wksp/specs/issue-1150-persistence-coherence/2026-09-23-case-context-recovery-strategy-design.md` |
| Decisions | `wksp/specs/issue-1150-persistence-coherence/decisions.md` |
| Journal | `wksp/JOURNAL.md` |
