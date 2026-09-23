# Session Handover — 2026-09-23

## What happened

Implemented `CaseContextRecoveryStrategy` SPI (#1166) — the keystone issue for the persistence coherence epic. Two implementations: `SnapshotRecoveryStrategy` (default, O(1) DB snapshot) and `EventLogReplayRecoveryStrategy` (experimental, extracted from `rebuildStateContext()`). Strategy is wired into `DefaultWorkerExecutionRecoveryService` for recovery and into all `updateStateAndAppendEvent` repository implementations for `onContextChanged`. Config-driven selection via `@DefaultBean` + `@IfBuildProperty`.

Fixed two event-log replay gaps: #1151 (SCOPED_WORKER_OUTPUT not in replay filter) and #1152 (CONTEXT_SIGNAL_APPLIED stored key names only, not values — fixed both write and replay paths).

## Decisions

- `onContextChanged` kept as a two-method SPI — architecturally part of the blackboard propagation cycle, not just snapshot persistence. Future delta-based replay (Drools command pattern) hooks into this.
- `recover(CaseInstance)` signature instead of `recover(UUID, String)` — avoids redundant DB lookup since the recovery service already loads the instance.
- Snapshot stored as JSONB column on `CaseInstanceEntity` — no separate table or SPI needed.

## Queue

Position 4/17. Active: #1153 — PlanItemStore.updateStatus tenancy inversion.

## References

| Artifact | Path |
|----------|------|
| Design spec | `wksp/specs/issue-1150-persistence-coherence/2026-09-23-case-context-recovery-strategy-design.md` |
| Implementation plan | `wksp/plans/2026-09-23-case-context-recovery-strategy.md` |
| Decisions | `wksp/specs/issue-1150-persistence-coherence/decisions.md` |
| Journal | `wksp/JOURNAL.md` |
| Blog | `wksp/blog/2026-09-23-mdp01-snapshot-over-replay.md` |
