# Decisions — Recovery Hardening (branch: issue-1182-snapshot-recovery-fallback)

## D1: Branch in recovery service, not in strategy SPI

**Choice:** `DefaultWorkerExecutionRecoveryService` resolves the factory and branches: durable → `CaseContextImpl.loadFromStore(factory, caseId)` directly; volatile → delegate to existing `CaseContextRecoveryStrategy`. The SPI is not changed.
**Alternatives:**
- New `DurableStoreRecoveryStrategy` implementing `CaseContextRecoveryStrategy` — clean SPI separation but strategies can't construct properly-wired contexts (they don't resolve factories), and per-case vs per-container selection is a mismatch
- Extend `SnapshotRecoveryStrategy` fallback chain — god-object risk, mixes three recovery paths in one class
**Rationale:** `CaseContextImpl.storeFactory` is `final`. Recovery must create the context with the correct factory from the start — there's no "recover then re-wire" path. Factory resolution requires `CaseDefinitionRegistry` and `StrategyResolver`, which are service-level dependencies, not strategy-level. The service already orchestrates recovery; factory resolution is a natural addition.
**Trade-offs:** The recovery service gains two new dependencies (`CaseDefinitionRegistry`, `StrategyResolver`) and factory-awareness. This is acceptable — the service is already the orchestrator.
**Sources:** `DefaultWorkerExecutionRecoveryService.java:63`, `CaseContextImpl.java:52` (final storeFactory field), `CaseContextImpl.java:129` (layer() uses storeFactory), `CaseHubRuntimeImpl.java:116` (resolveFactory pattern)
**Exploration:** deep-analysis
**Status:** captured

## D2: Static factory method `CaseContextImpl.loadFromStore`

**Choice:** Add `CaseContextImpl.loadFromStore(CaseContextStoreFactory factory, UUID caseId)` — static method that calls `factory.loadStore(layerName, caseId)` per built-in layer.
**Alternatives:**
- Constructor with mode enum (`CREATE` vs `LOAD`) — boolean-trap on a class with five constructors already
**Rationale:** Matches the existing `fromLayerDocument(JsonNode)` pattern — static methods for alternative construction paths. Clearer at the call site than a constructor flag.
**Trade-offs:** None significant — the static method delegates to a private constructor or init method.
**Sources:** `CaseContextImpl.java:630` (`fromLayerDocument` pattern), `CaseContextImpl.java:81` (factory constructor), `CaseContextStoreFactory.java:35` (`loadStore` default method)
**Exploration:** quick
**Status:** captured
**Depends on:** D1 (the service calls this method for durable factories)

## D3: Fall back to volatile recovery on factory resolution failure

**Choice:** If `CaseMetaModel` is null, definition isn't registered, or factory bean is missing — log WARN and delegate to existing `CaseContextRecoveryStrategy`. The case recovers with an in-memory context.
**Alternatives:**
- Throw on resolution failure — loses the case entirely, unacceptable for a recovery path
**Rationale:** Degraded but functional. Handles the migration path: existing cases started before factory wiring was added won't have the metadata for factory resolution and fall back gracefully.
**Trade-offs:** A durable factory case that hits this path loses its store connection. The WARN log alerts operators.
**Sources:** `CaseMetaModel.java:31` (name field), `CaseDefinitionRegistry.java:67` (getCaseDefinition), `DefaultWorkerExecutionRecoveryService.java:63` (recovery entry point)
**Exploration:** quick
**Status:** captured
**Depends on:** D1 (the fallback is the else-branch in the service)

---

## D4: Keep writing snapshots for durable factory cases

**Choice:** `SnapshotRecoveryStrategy.onContextChanged()` continues to write JSON snapshots for all cases, including those backed by durable factories. Belt-and-suspenders for pre-release.
**Alternatives:**
- Skip snapshot writes for durable factories — cleaner, avoids redundancy, but creates a consistency question (which is authoritative?) and removes a secondary recovery path
**Rationale:** The snapshot write is cheap (sets a field within the existing `updateStateAndAppendEvent` transaction). Having both the durable store AND the snapshot means: snapshot is a secondary recovery path if the durable store is corrupted; no behavioral change to `onContextChanged`; zero risk of silent data loss during the transition to durable factories.
**Trade-offs:** Slight storage overhead (JSONB column per case). Negligible for the safety benefit.
**Sources:** `SnapshotRecoveryStrategy.java:60` (onContextChanged), `CaseInstanceEntity.java` (contextSnapshot column)
**Exploration:** quick
**Status:** captured

---

## D5: Persist state synchronously in unfault()

**Choice:** `unfault()` calls `CaseInstanceRepository.update(instance, instance.tenancyId)` to persist RUNNING to DB before dispatching the async `CaseStatusChanged` event. Also adds `caseInstanceCache.put(instance)` for cache consistency.
**Alternatives:**
- Synchronous event dispatch — change CaseStatusChanged from async to sync for FAULTED→RUNNING. Couples unfault() to handler execution time and all cascading events. Scaling concern for batch unfault.
- CompletableFuture return — caller decides whether to wait. Over-engineering for an admin operation.
- Change DLQ replay to check cache first — fragile, ties correctness to cache behavior rather than persistence. Cache can be evicted at any time.
**Rationale:** The race is real and JPA-specific. `VertxEventDispatcher.dispatch()` calls `eventBus.publish()` which returns immediately. The handler runs on a separate Vert.x worker thread. `DeadLetterReplayService.doReplay()` uses `CrossTenantCaseInstanceRepository.findByUuid()` which always creates a NEW CaseInstance from the DB entity — it never sees in-memory state. In-memory tests pass because `InMemoryCaseInstanceRepository` implements both interfaces on the same backing store. The handler's subsequent `updateStateAndAppendEvent` is idempotent (writes RUNNING again + appends EventLog).
**Trade-offs:** One extra DB write per unfault (the handler writes state again). Negligible for an admin operation. Adds `CaseInstanceRepository` as a dependency to `CaseRecoveryService`.
**Sources:** `VertxEventDispatcher.java:101` (eventBus.publish — async), `CaseStatusChangedEventBusAdapter.java:29` (@ConsumeEvent blocking=true — worker thread), `JpaCrossTenantCaseInstanceRepository.java:60` (fromEntity creates NEW object), `InMemoryCaseInstanceRepository.java:36` (implements both interfaces), `DeadLetterReplayService.java:121` (findByUuid → DB query), `JpaCaseInstanceRepository.java:86` (update method)
**Exploration:** deep-analysis
**Status:** captured

## D6: CANCELLED cases are non-recoverable

**Choice:** CANCELLED remains permanently terminal. `unfault()` rejects CANCELLED cases with a specific WARN log: "Cannot recover a CANCELLED case — cancellation is an administrative decision. Start a new instance with the same context to retry." Javadoc documents the exclusion.
**Alternatives:**
- Support CANCELLED→RUNNING via unfault() — blurs the distinction between system failure recovery and operator decision reversal. Different authorization, audit, and semantic implications.
- Separate `uncancel()` method — YAGNI. The mechanical work is identical to unfault(), so adding it later costs the same. No door closes.
**Rationale:** FAULTED is system-initiated (the case didn't want to end). CANCELLED is human-initiated (an operator decided to end it). Recovery from FAULTED aligns with original intent. Recovery from CANCELLED reverses a deliberate decision — a fundamentally different operation. Accidental cancellation is a UI problem (confirmation dialogs), not a recovery problem. Starting a new instance preserves audit trail clarity.
**Trade-offs:** If a genuine need for uncancel() emerges, it must be implemented separately. This is acceptable — the separate implementation allows proper authorization and audit semantics.
**Sources:** `CaseStatusChangedHandler.java:195` (terminal cleanup applies to all terminal states), `CaseRecoveryService.java:71` (current generic rejection log)
**Exploration:** deep-analysis
**Status:** captured

## D7: Infrastructure restoration in unfault(), not in handler

**Choice:** `unfault()` calls `caseChannelProvider.openChannel(caseId, "coordination")` and `schedulerService.registerScheduledTriggers(instance)` directly, mirroring `CaseStartedEventHandler`. Recovery is fully synchronous and self-contained.
**Alternatives:**
- In CaseStatusChangedHandler — handler detects FAULTED→RUNNING and restores. Handler already has dependencies, but restoration is async (gap between unfault return and handler execution). Adds special-case logic to an already-large handler.
- Introduce CaseResumedEvent — clean event-driven architecture but adds new event type, new handlers, more complexity. Over-engineering.
**Rationale:** The caller of `unfault()` expects a recovered case, not a "recovery started" state. Mirroring `CaseStartedEventHandler.onCaseStarted()` (lines 101-104) keeps the infrastructure setup pattern consistent. The dependency growth (CaseChannelProvider, SchedulerService) is justified — a recovery service should own the infrastructure it restores.
**Trade-offs:** CaseRecoveryService grows from 4 to 7 constructor parameters. Acceptable for a service that owns recovery.
**Sources:** `CaseStartedEventHandler.java:101` (openChannel), `CaseStartedEventHandler.java:104` (registerScheduledTriggers), `CaseStatusChangedHandler.java:203-241` (terminal cleanup that must be reversed)
**Exploration:** deep-analysis
**Status:** captured
**Depends on:** D5 (state must be persisted before infrastructure restoration)
