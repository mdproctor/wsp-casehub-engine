# Handoff — 2026-06-30

## What's Done

**#490 Epic: Hybrid Orchestration — CLOSED (with #483, #484, #485)**

Made the engine truly hybrid choreography+orchestration via a four-tier model:

- **Tier 1 — WorkerRuntime** (#485): Per-invocation handle letting workers call other functions and spawn sub-cases. `WorkerExecutionContext.currentRuntime()` access pattern. `WorkerRuntimeFactory` creates instances per invocation. `execute()` never throws (wraps in `WorkerResult.failed()`). Supports Sync and AgentWorkerFunction; FlowWorkerFunction excluded (Tier 3). `spawnCase()`/`awaitCase()` are TODO stubs.

- **Tier 2 — SequentialPlanningStrategy** (#484): `PlanningStrategy` implementation (id="sequential") that selects one binding at a time. `PlanningStrategyLoopControl` changed from single injection to `Instance<PlanningStrategy>` with ID-based lookup. `CaseDefinition.planningStrategy` field added. Halts on non-COMPLETED terminal states.

- **signalAndAwait** (#483): Bulk `signal(UUID, Map)` for atomic multi-key context updates. `signalAndAwait()` with `SignalSettlementTracker` using generation-tagged signalId threading through 5 event types. Settlement resolves when expectedCount == completedCount AND fullyDispatched.

- **YAML support**: `planningStrategy:` and `sequence:` keys in CaseDefinitionYamlMapper. Sequence uses two-pass resolution.

- **WorkerFunctions.sequence()**: Convenience combinator for linear step execution.

Design-reviewed (10 rounds, 29 issues, $31.06). Code-reviewed (final whole-branch, 2 Critical fixed, 4 Important triaged, 4 Minor noted).

## Follow-Up Issues Filed

| # | Title | Context |
|---|-------|---------|
| #620 | Thread signalId through QuartzRetryService retry/exhaust path | Exception→retry path doesn't thread signalId — signalAndAwait hangs on worker exceptions that exhaust retries |
| #621 | SequentialPlanningStrategy getAllPlanItems() copy per select() call | Perf concern for large plans — builds Map from defensive copy on every context change cycle |
| #622 | Bulk signal event log should record updated keys | BulkSignalReceivedEvent stores only `{"type": "bulk_signal"}` — no audit of what was updated |

## Cross-Module

**Consumer repos still need capabilityNames migration** (from prior session):
- casehub-life#47, casehub-aml#85, casehub-devtown#117, casehub-desiredstate#50, casehubio/parent#328

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #620 | signalId threading through retry/exhaust path | S | Med | Needed for signalAndAwait robustness with throwing workers |
| #621 | SequentialPlanningStrategy perf optimization | XS | Low | Add CasePlanModel.getPlanItemByBindingName() |
| #622 | Bulk signal event log audit keys | XS | Low | Record updated keys in event log payload |
| #582 | Generalize GoalBasedCompletion beyond success/failure | M | Med | Follow-on from #581 |
| #592 | External-backend recovery gap | M | Med | Pre-existing gap |
| — | WorkerRuntime.spawnCase()/awaitCase() full implementation | M | Med | Event bus listener for CASE_STATUS_CHANGED |
| — | WorkflowPlanningStrategy (Tier 3) | L | High | SW-backed durable orchestration — future |
| — | SequentialPlanningStrategy integration test fix | S | Med | Bindings fire concurrently in @QuarkusTest — unit tests pass |
