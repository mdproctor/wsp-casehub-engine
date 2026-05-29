# Design Journal — issue-392-sxs-batch

### 2026-05-29 · §Dependencies and SPI

`WorkerProvisioner.provision()` and `ReactiveWorkerProvisioner.provision()` now return `ProvisionResult` instead of `Worker`. `Worker` is a case-definition artifact — capabilities, function, execution policy declared by the case author. Provisioning outcomes (causal ledger entry linkage via `causedByEntryId`) are not part of that definition and should not be grafted onto it. `ProvisionResult(UUID causedByEntryId)` is the typed outcome envelope. No-op defaults still throw `ProvisioningException`; the return type change is in the declaration only. See protocol PP-20260529-bcbbb5.

### 2026-05-29 · §Worker Execution Lifecycle

`CaseContextChangedEventHandler.tryProvision()` now fires `CaseLifecycleEvent("WorkerStarted", commandType="ProvisionWorker")` after a successful external provisioner call — previously the provisioner return value was discarded with `.replaceWithVoid()`. The audit trail for external provisioning is now complete. `traceId` is captured synchronously before the reactive chain (OTel ThreadLocal is severed at async boundaries — matches the pattern used by all other handlers). Claudony wiring tracked in claudony#140.

`CaseStatusChangedHandler.onCaseStatusChangedHandler()` restructured to await CDI lifecycle event delivery. `lifecycleEvents.fireAsync()` moved from `.invoke()` (discards CompletionStage) to `.chain(() -> Uni.createFrom().completionStage(...))` so the handler's Uni does not complete until all `@ObservesAsync` observers have run. Observer failures are logged and recovered — ledger errors must not fail case transitions. Five remaining handlers with the same `.invoke()` discard pattern tracked in engine#397. See protocol PP-20260529-3237bd.
