# Design Journal — issue-349-case-signal-sink

### 2026-05-26 · §Architecture Layers · §Execution Models · §Case Lifecycle · §Dependencies and SPI

The root architectural problem in engine#349 was a conflation of two concerns
in `CaseContextChangedEventHandler`: the RUNNING guard that blocked all evaluation
for non-RUNNING cases was doing two unrelated jobs — preventing duplicate dispatches
and blocking signal processing. Separating them required moving dedup responsibility
into `LoopControl`, which already owns dispatch decisions.

`PlanExecutionContext` now carries `CaseStatus`, and each `LoopControl` implementation
declares which states it safely handles. `ChoreographyLoopControl` (pure choreography,
no PlanItem tracking) restricts to RUNNING only. `PlanningStrategyLoopControl`
(blackboard active) allows RUNNING and WAITING, using `filterToDispatchable` to
prevent re-dispatch of in-flight bindings (RUNNING/DELEGATED PlanItems are skipped;
PENDING and terminal PlanItems pass through — terminal bindings can re-dispatch if
their trigger conditions fire again). The `CaseContextChangedEventHandler` guard
changed from `state != RUNNING` to `state != RUNNING && state != WAITING`, blocking
only SUSPENDED and terminal cases.

Two new signal sources were wired in: `QhorusMessageSignalBridge` observes Qhorus
`MessageReceivedEvent`s for commitment-resolving types (RESPONSE, DONE, DECLINE,
FAILURE) on `case-{caseId}/{purpose}` channels and calls `CaseHubRuntime.signal()`,
making human channel messages visible to WAITING cases for the first time. The channel
naming constant was extracted to `CaseChannel.CASE_CHANNEL_PREFIX` and a
`channelName()` factory so the bridge and channel providers share the format
definition rather than independent string literals (claudony#139 tracks adoption).

`WorkItemLifecycleAdapter` was corrected on two fronts: ESCALATED removed from the
terminal status filter (the WorkItem returns to PENDING — it is not terminal), and an
`@ObservesAsync WorkItemGroupLifecycleEvent` observer added for M-of-N SpawnGroup
outcomes (engine#339), routing COMPLETED/REJECTED group results to PlanItem
transitions and firing CONTEXT_CHANGED.
