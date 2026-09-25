# HANDOFF — casehub-engine

## Last Session

Closed epic #1150 (persistence layer coherence, 18 issues). Final two issues this session:

- **#1165** — DLQ replay integration test. Exposed a bug where `DeadLetterReplayService` didn't set `ExecutionMode.REINVOKED`, causing the idempotency guard in `WorkerScheduleEventHandler` to silently skip replayed work. Fixed by adding `REINVOKED` mode and `ExecutionOrigin.REPLAY`. Added `DeadLetterReplayIntegrationTest` (4 tests, all green).

- **#1179** — `CaseRecoveryService.unfault(UUID caseId)` in `runtime-core`. Transitions FAULTED → RUNNING: sets state synchronously, re-registers completion tracker, dispatches `CaseStatusChanged` for persistence and binding re-evaluation. CDI producer in `RuntimeBeans`.

Branch squashed (27 → 18 commits), rebased onto origin/main, merged, and pushed.

## What's Next

Recovery hardening — 5 issues queued in `.plan`:

| # | Issue | Scale | Complexity | Notes |
|---|-------|-------|------------|-------|
| 1 | #1182 — SnapshotRecoveryStrategy fallback on null snapshot | S | Low | Fall back to event-log replay instead of throwing. Migration path concern. |
| 2 | #732 — wire CaseContextStoreFactory through recovery path | M | Med | Pre-existing issue. Recovery hardcodes InMemory factory. Prerequisite for durable stores. |
| 3 | #1183 — harden CaseRecoveryService.unfault | M | Med | Re-register evicted engine registries, fix JPA persistence race, document CANCELLED policy. |
| 4 | #1184 — enforce replay handler registration | S | Med | Contract test that fails when a new event type lacks a replay handler. Prevents #1151/#1152 class of bugs. |
| 5 | #1185 — persistent DLQ storage | M | Med | DLQ entries lost on restart. Add DeadLetterEntryStore SPI + JPA implementation. |

Recommended order: #1182 first (smallest, unblocks migration path), then #732 (prerequisite for durable stores), then #1183 (depends on understanding from #732), then #1184 and #1185 in either order.

## References

| Artifact | Path |
|----------|------|
| Design spec | `wksp/specs/issue-1150-persistence-coherence/2026-09-23-case-context-recovery-strategy-design.md` |
| Decisions | `wksp/specs/issue-1150-persistence-coherence/decisions.md` |
| Diary (SPI design) | `wksp/blog/2026-09-23-mdp01-snapshot-over-replay.md` |
| Diary (DLQ replay) | `wksp/blog/2026-09-25-mdp01-the-replay-the-engine-ignored.md` |
| Parent epic for recovery | #210 — cancellation, timeout, and error recovery |
