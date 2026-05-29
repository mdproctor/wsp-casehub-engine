<<<<<<< HEAD
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
=======
# Design Journal — issue-392-sxs-batch

### 2026-05-29 · §Dependencies and SPI

`WorkerProvisioner.provision()` and `ReactiveWorkerProvisioner.provision()` now return `ProvisionResult` instead of `Worker`. `Worker` is a case-definition artifact — capabilities, function, execution policy declared by the case author. Provisioning outcomes (causal ledger entry linkage via `causedByEntryId`) are not part of that definition and should not be grafted onto it. `ProvisionResult(UUID causedByEntryId)` is the typed outcome envelope. No-op defaults still throw `ProvisioningException`; the return type change is in the declaration only. See protocol PP-20260529-bcbbb5.

### 2026-05-29 · §Worker Execution Lifecycle

`CaseContextChangedEventHandler.tryProvision()` now fires `CaseLifecycleEvent("WorkerStarted", commandType="ProvisionWorker")` after a successful external provisioner call — previously the provisioner return value was discarded with `.replaceWithVoid()`. The audit trail for external provisioning is now complete. `traceId` is captured synchronously before the reactive chain (OTel ThreadLocal is severed at async boundaries — matches the pattern used by all other handlers). Claudony wiring tracked in claudony#140.

`CaseStatusChangedHandler.onCaseStatusChangedHandler()` restructured to await CDI lifecycle event delivery. `lifecycleEvents.fireAsync()` moved from `.invoke()` (discards CompletionStage) to `.chain(() -> Uni.createFrom().completionStage(...))` so the handler's Uni does not complete until all `@ObservesAsync` observers have run. Observer failures are logged and recovered — ledger errors must not fail case transitions. Five remaining handlers with the same `.invoke()` discard pattern tracked in engine#397. See protocol PP-20260529-3237bd.
>>>>>>> 06a0b14 (tmp: journal + index for main cherry-pick)
