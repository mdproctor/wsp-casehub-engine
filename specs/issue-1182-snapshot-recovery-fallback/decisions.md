# Decisions — Wire CaseContextStoreFactory Through Recovery (#732)

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

## D4: Keep writing snapshots for durable factory cases

**Choice:** `SnapshotRecoveryStrategy.onContextChanged()` continues to write JSON snapshots for all cases, including those backed by durable factories. Belt-and-suspenders for pre-release.
**Alternatives:**
- Skip snapshot writes for durable factories — cleaner, avoids redundancy, but creates a consistency question (which is authoritative?) and removes a secondary recovery path
**Rationale:** The snapshot write is cheap (sets a field within the existing `updateStateAndAppendEvent` transaction). Having both the durable store AND the snapshot means: snapshot is a secondary recovery path if the durable store is corrupted; no behavioral change to `onContextChanged`; zero risk of silent data loss during the transition to durable factories.
**Trade-offs:** Slight storage overhead (JSONB column per case). Negligible for the safety benefit.
**Sources:** `SnapshotRecoveryStrategy.java:60` (onContextChanged), `CaseInstanceEntity.java` (contextSnapshot column)
**Exploration:** quick
**Status:** captured
