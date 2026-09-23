# Session Handover — 2026-09-23 (session 2)

## What happened

Completed 1 issue from the persistence coherence epic (#1150), advancing the queue from position 12/17. Active issue is now #1161 (just completed, ready to advance to #1162). All changes are on `issue-1150-persistence-coherence` branch, committed but not pushed.

### Issues completed

| # | Title | Scale | Files |
|---|-------|-------|-------|
| #1161 | feat: add JPA and Spring JPA implementations for CaseQueueEntryStore | M | 11 |

### Key changes

- **CaseQueueEntryEntity** in `persistence-jpa-common` — JPA entity with UUID `@Id`, status stored as `String` to avoid adding heavy `engine-support-core` dependency to the shared entity module. Unique constraint on `(case_id, view_id)`, indexes on `case_id`, `view_id+tenancy_id`, `tenancy_id`.
- **JpaCaseQueueEntryStore** in `persistence-hibernate` — `@Alternative @Priority(2) @ApplicationScoped`, overrides the InMemory producer in `QueueBeans`. Uses pessimistic locking for `claimIfPending`. `toEntity`/`toModel` handle `QueueEntryStatus` ↔ String conversion.
- **SpringJpaCaseQueueEntryStore** in `persistence-spring-jpa` — package-private, `@Transactional` class-level, registered as bean in `PersistenceAutoConfiguration`.
- **Bean disambiguation:** Quarkus uses `@Alternative @Priority(2)` on the JPA impl. Spring uses `@ConditionalOnMissingBean(CaseQueueEntryStore.class)` on the InMemory bean in `EngineSupportAutoConfiguration`.
- **CaseQueueEntryStoreContractTest** — 16 abstract tests covering all SPI operations plus full-field round-trip. `InMemoryCaseQueueEntryStoreTest` refactored to extend it.
- **POM changes:** Added `engine-support-core` as dependency to `persistence-hibernate` and `persistence-spring-jpa` (needed for `CaseQueueEntryStore` SPI, `CaseQueueEntry` model, `QueueEntryStatus` enum).
- **Indentation normalization** — `CaseQueueEntryStore`, `ExecutionSnapshotStore`, `PlanVersionStore` interfaces normalized from 4-space to 2-space by IntelliJ reformatting.

## Decisions

- **Status as String in entity:** `CaseQueueEntryEntity.status` is `String`, not `@Enumerated(QueueEntryStatus.class)`. Avoids adding `engine-support-core` (which transitively brings `runtime-core`, `planning-core`, `ledger-api`) to `persistence-jpa-common`. Store implementations handle the String ↔ enum conversion.
- **UUID as @Id:** CaseQueueEntry uses UUID as its domain identity. Entity uses UUID directly as `@Id` (following `ExecutionSnapshotEntity` pattern) rather than a synthetic `Long` with a separate UUID field.

## Known issues

- **Pre-existing Spring drift detection failures** in `runtime-spring` — `verify-drift` goal fails due to 23+ Quarkus types with no Spring equivalent. Not caused by this branch.
- **Pre-existing schema validation failure** in `JpaExecutionSnapshotStoreTest` — `context_snapshot` column missing from Flyway migration (added by #1166 CaseContextRecoveryStrategy work). The test uses `validate` schema mode against the Flyway-managed schema.

## Queue

Position 12/17. Active: #1161 (completed, needs `work next` to advance).
Next: #1162 — fix: resolve duplicate JpaPlanItemStore in persistence-hibernate and work-adapter.

## References

| Artifact | Path |
|----------|------|
| Design spec | `wksp/specs/issue-1150-persistence-coherence/2026-09-23-case-context-recovery-strategy-design.md` |
| Decisions | `wksp/specs/issue-1150-persistence-coherence/decisions.md` |
| Journal | `wksp/JOURNAL.md` |
