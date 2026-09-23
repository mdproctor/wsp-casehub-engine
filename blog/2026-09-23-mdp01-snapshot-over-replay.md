---
title: "Snapshot Over Replay — Why Database Recovery Beats Event Log Reconstruction"
date: 2026-09-23
author: mdp
tags: [persistence, recovery, spi, architecture, event-sourcing]
entry_type: note
subtype: diary
---

The engine's context recovery had a design flaw hiding in plain sight. On cache miss, `rebuildStateContext()` would replay the entire event log to reconstruct `CaseContext` — scanning for seven event types, applying diffs and patches in sequence order, rebuilding milestone state, restoring episodic layer entries. O(N) in event count, and fragile in a way that only surfaces when you add new context-mutating event types.

Two bugs proved the fragility. `SCOPED_WORKER_OUTPUT` — the interim context writes from persistent workers — wasn't in the replay filter at all. A cache eviction mid-worker silently lost those writes. `CONTEXT_SIGNAL_APPLIED` was worse: it was in the event log but only stored key *names*, not values. Even adding it to the filter wouldn't help — the data needed for reconstruction wasn't there.

The implicit contract is the problem. Every event type that mutates `CaseContext` must: store replay-compatible data in event metadata, AND be handled in the replay method. Nothing enforces this at compile time, test time, or runtime. The first you hear about it is a user reporting that context values vanished after a restart.

## The snapshot strategy

We introduced a `CaseContextRecoveryStrategy` SPI with two implementations. The default — `SnapshotRecoveryStrategy` — serialises the full context as a JSON snapshot on the `CaseInstance` entity, persisted within the existing `updateStateAndAppendEvent` transaction. Recovery is a deserialisation call. O(1), no implicit replay contract, no event-type filter to keep in sync.

The event-log replay path still exists as `EventLogReplayRecoveryStrategy`, marked experimental. It has its uses — write-performance-sensitive deployments, audit verification, testing that your event log is complete. But it's no longer the default recovery mechanism.

## onContextChanged and the blackboard

The SPI has two methods: `recover()` and `onContextChanged()`. I initially proposed dropping `onContextChanged` and always writing the snapshot in the repository — simpler, no extension point to maintain. That was too narrow.

`onContextChanged` is the recovery strategy's participation in the blackboard propagation cycle. The `CaseContextChangedEvent` is the async notification that drives binding evaluation and worker dispatch. `onContextChanged` is the synchronous hook at the persistence layer — same semantic event, different architectural concern. The snapshot strategy serialises context here. The event-log strategy is a no-op. A future strategy could stream changes, replicate state, or — as I've been thinking about from my Drools days — capture state changes as function + input parameters for delta-based replay. Each mutation recorded as input values to a known function; replay re-applies those values instead of re-interpreting full event payloads.

That optimisation isn't built yet, but `onContextChanged` is the hook that makes it possible without restructuring the SPI.

## The replay gaps, fixed

With the SPI in place, fixing the two replay bugs was mechanical. `SCOPED_WORKER_OUTPUT`: add the event type to the filter, apply `contextChanges` from metadata — same pattern as `WORKER_EXECUTION_COMPLETED`. `CONTEXT_SIGNAL_APPLIED`: fix the write path first (store actual key-value pairs in TopLevel format, not just key names), then add the replay handler. Both fixes are scoped to the experimental path. The default snapshot strategy was never affected.

Fourteen issues remain in the persistence coherence epic. Next up: `PlanItemStore.updateStatus` tenancy inversion — a different kind of persistence problem, but the same principle. Fix the design, don't protect callers.
