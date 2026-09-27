# HANDOFF — casehub-engine

## Last Session

Completed 4 of 5 recovery hardening issues on branch `issue-1182-snapshot-recovery-fallback`. This session completed #1183 and #1184, and fully designed #1185 with a ready-to-execute plan.

### Previously completed (prior sessions)

- **#1182** (S/Low) — `SnapshotRecoveryStrategy` fallback to `EventLogReplayRecoveryStrategy` on null snapshot.
- **#732** (M/Med) — Wired `CaseContextStoreFactory` through recovery path.

### Completed this session

- **#1183** (M/Med) — Hardened `CaseRecoveryService.unfault()`. Fixed JPA-specific persistence race: `unfault()` now persists RUNNING to DB via `CaseInstanceRepository.update()` before dispatching the async `CaseStatusChanged` event. Added `caseInstanceCache.put()` for cache consistency. Re-opens coordination channel and re-registers scheduled triggers (mirrors `CaseStartedEventHandler` setup). CANCELLED cases documented as non-recoverable with specific WARN log. 8 unit tests + 4 integration tests pass. `CaseRecoveryService` constructor grew from 4 to 7 parameters (added `CaseInstanceRepository`, `CaseChannelProvider`, `SchedulerService`); `RuntimeBeans.caseRecoveryService()` producer updated.

- **#1184** (S/Med) — Contract test enforcing replay handler coverage. Extracted `REPLAYED_TYPES` constant from `EventLogReplayRecoveryStrategy.rebuildStateContext()`. Test asserts every `CaseHubEventType` is either in `REPLAYED_TYPES` (has a replay handler) or `NON_MUTATING_TYPES` (explicitly reviewed). Adding a new enum value without updating either set fails the build.

## Immediate Next Step

**#1185** — persistent DLQ storage. Design and plan are complete. Execute the plan at `wksp/plans/2026-09-27-persistent-dlq-storage.md`.

The plan has 2 batches, 4 tasks:
1. SPI + InMemory: Define `DeadLetterEntryStore` SPI in `resilience-core`, create `InMemoryDeadLetterEntryStore`
2. Facade: Refactor `DeadLetterQueue` to delegate to the store (constructor takes store)
3. JPA: `DeadLetterEntryEntity` + `JpaDeadLetterEntryStore` in `persistence-hibernate` (new `resilience-core` dep)
4. Integration: Verify full build, resolve CDI wiring (JPA vs InMemory bean precedence)

Key constraint: no Flyway — Hibernate `drop-and-create` manages schema.

## Pre-existing Build Issue

`api` module has a compilation error in `JsonNodeForEachAdapter.java` — `ForEachAdapter` interface method renamed (`getWhen` → `getCondition`). Not caused by this branch's work. Test modules compile and pass independently.

## References

| Artifact | Path |
|----------|------|
| Design spec (#1183) | `wksp/specs/issue-1182-snapshot-recovery-fallback/2026-09-27-unfault-hardening-design.md` |
| Design spec (#1185) | `wksp/specs/issue-1182-snapshot-recovery-fallback/2026-09-27-persistent-dlq-storage-design.md` |
| Replay handler contract spec (#1184) | `wksp/specs/issue-1182-snapshot-recovery-fallback/2026-09-27-replay-handler-contract-design.md` |
| Decisions (all issues) | `wksp/specs/issue-1182-snapshot-recovery-fallback/decisions.md` (D1-D11) |
| Implementation plan (#1183) | `wksp/plans/2026-09-27-unfault-hardening.md` |
| Implementation plan (#1185) | `wksp/plans/2026-09-27-persistent-dlq-storage.md` |
| Parent epic | #210 — cancellation, timeout, and error recovery |
