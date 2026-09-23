# Design Journal — issue-1150-persistence-coherence

## 2026-09-23 — CaseContextRecoveryStrategy SPI + replay gap fixes

**Issues completed:** #1166, #1151, #1152

### Design evolution

The initial proposal simplified the SPI to a single `recover()` method, always writing snapshots in the repository and dropping `onContextChanged`. The user identified that `onContextChanged` is architecturally part of the blackboard propagation cycle — the recovery strategy's hook into context-change notification, not just a snapshot persistence mechanism. Future strategies (delta-based replay, replication, streaming) would use this extension point. Restored the two-method SPI.

The user also noted a connection to the Drools command-based persistence pattern: each state change captured as a function + input parameters, with replay re-applying those values. This shaped `onContextChanged` as the forward-looking hook for delta-based optimisation.

### Key decisions

- **`onContextChanged` called inside `updateStateAndAppendEvent`** — within the persistence transaction, before entity write. Simpler than hooking into the 15 `CaseContextChangedEvent` dispatch sites, and ensures the snapshot is persisted atomically with the state change.
- **`@DefaultBean` + `@IfBuildProperty`** — conditional CDI activation instead of a runtime producer. SnapshotRecoveryStrategy yields via `@DefaultBean`; EventLogReplayRecoveryStrategy activates only when `casehub.context.recovery-strategy=event-log` is set.
- **Signature refinement** — `recover(CaseInstance instance)` instead of `recover(UUID, String)` to avoid redundant DB lookups since the recovery service already loads the instance.

### Replay gap fixes

Both #1151 and #1152 followed the same pattern: add the event type to the replay filter, handle `contextChanges` from metadata. For #1152, the write path also needed fixing — `ContextSignalEventHandler` only stored key names, not values. Added TopLevel-format `contextChanges` to the event metadata.
