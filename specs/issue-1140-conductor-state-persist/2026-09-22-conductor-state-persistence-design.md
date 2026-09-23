# Conductor State Persistence — Design Spec

**Issue:** casehubio/engine#1140
**Epic:** casehubio/engine#1149 (Production Readiness)
**Parent spec:** `2026-09-21-command-centre-conductor-design.md` (#1132)
**Decisions:** D1–D5 in `decisions.md`
**Date:** 2026-09-22

## Problem

All conductor state stores are bare `ConcurrentHashMap` fields inside domain beans. On restart:
- Pending inbox entries → improvement cases stuck in GATED state, no recovery
- Watch patterns → operator escalation configuration silently lost
- Dynamic deny patterns → operator safety configuration silently lost
- Manual blocks → coordination state silently lost

The domain beans mix business logic with state management — no separation of concerns, no pluggable persistence. When durable persistence arrives (#1166 CaseContextRecoveryStrategy), there is no SPI boundary to swap in a durable implementation.

## Architecture

Extract four repository SPIs from three domain beans. Domain beans become stateless logic that injects SPIs. The SPI boundary enables future durable implementations without touching business logic.

```
                         common-core/spi
                    ┌──────────────────────────┐
                    │  ConductorInboxRepository │
                    │  WatchPatternStore        │
                    │  DenyPatternStore         │
                    │  ImprovementBlockStore    │
                    └──────────┬───────────────┘
                               │ implements
                    ┌──────────▼───────────────┐
                    │  runtime-core             │
                    │  InMemory* (@DefaultBean) │
                    │  Domain beans (inject SPI)│
                    └──────────────────────────┘
```

**Persistence direction (D1):** In-memory implementations now. All stores are case-scoped and low-mutation (human-timescale operator configuration). When durable implementations arrive, they will be JPA-backed entities with their own tables — these stores have typed query semantics (findPending, isBlocked) incompatible with CaseContext's flat key-value model. EventLog for audit only. The EventLog writes from the #1132 spec (`GATE_PENDING`, `DENY_PATTERN_ADDED`, etc.) remain correct as audit trail — not as the durability mechanism. EventLog-replay-based recovery has known data-loss bugs (#1151, #1152) and is being superseded by #1150. Restart loss impact differs by store lifecycle: inbox entries and blocks are transient coordination state (tolerable loss), while watch patterns and deny patterns are operator safety configuration (correctness gap — operators must re-apply after restart). This is a known limitation of the in-memory default.

**SPI placement (D2):** SPIs in `common-core/spi` alongside `CaseInstanceRepository` and `EventLogRepository`. Per contributor guide §SPI Architecture: persistence SPIs go in `common-core/spi`, operational SPIs go in `api/spi`. The conductor stores are persistence SPIs — they manage domain state storage, not pluggable operational behavior.

**Watch pattern split (D3):** Four SPIs instead of three. `WatchPatternStore` is separate from `ConductorInboxRepository` because inbox entries are transient case-lifecycle state (created at gate, resolved or timed out) while watch patterns are persistent operator configuration that survives across improvement cycles.

## SPI Interfaces

All methods are tenant-scoped (`tenancyId` parameter), matching `EventLogRepository`'s convention.

### ConductorInboxRepository

```java
// common-core, io.casehub.engine.common.spi
public interface ConductorInboxRepository {

  void save(ConductorInboxEntry entry, String tenancyId);

  @Nullable
  ConductorInboxEntry findById(UUID caseId, String entryId, String tenancyId);

  List<ConductorInboxEntry> findPending(UUID caseId, String tenancyId);

  int countPending(UUID caseId, String tenancyId);

  List<ConductorInboxEntry> findAll(UUID caseId, String tenancyId);
}
```

Covers the inbox entry lifecycle: enqueue (save), resolve (save with updated status), query pending/all. The `caseId` is read from the entry's own field for `save`; query methods take it as a parameter.

### WatchPatternStore

```java
// common-core, io.casehub.engine.common.spi
public interface WatchPatternStore {

  void save(UUID caseId, WatchPattern pattern, String tenancyId);

  void remove(UUID caseId, String patternId, String tenancyId);

  List<WatchPattern> findActive(UUID caseId, String tenancyId);
}
```

Operator-managed escalation watch patterns. Long-lived configuration, independent of inbox entry lifecycle.

### DenyPatternStore

```java
// common-core, io.casehub.engine.common.spi
public interface DenyPatternStore {

  void save(UUID caseId, String pattern, String tenancyId);

  void remove(UUID caseId, String pattern, String tenancyId);

  Set<String> findAll(UUID caseId, String tenancyId);
}
```

Dynamic deny patterns per case. Static deny patterns (`STRUCTURAL_DENIED_PATTERNS`) remain on `ImprovementBudgetEnforcer` — they are compile-time constants, not persistence.

### ImprovementBlockStore

```java
// common-core, io.casehub.engine.common.spi
public interface ImprovementBlockStore {

  void save(UUID caseId, UUID improvementCaseId, UUID blockerImprovementId,
      String tenancyId);

  void remove(UUID caseId, UUID improvementCaseId, String tenancyId);

  boolean isBlocked(UUID caseId, UUID improvementCaseId, String tenancyId);

  @Nullable
  UUID blockedBy(UUID caseId, UUID improvementCaseId, String tenancyId);
}
```

Manual improvement ordering semaphore. Complements `ConflictDetector` (automatic file-level conflicts).

## In-Memory Implementations

All in `runtime-core`, package `io.casehub.engine.internal.improvement`. All are `@DefaultBean @ApplicationScoped` (D4) — shipped engine defaults, overridable by any non-`@DefaultBean` implementation. All implement `Resettable` (D5).

`tenancyId` parameters are accepted but not used by in-memory impls (no tenant isolation in memory). They exist for the SPI contract so durable implementations can enforce tenant scoping. This matches `InMemoryCaseInstanceRepository` which also accepts `tenancyId` but uses a single-map store.

### InMemoryConductorInboxRepository

```java
@DefaultBean
@ApplicationScoped
public class InMemoryConductorInboxRepository
    implements ConductorInboxRepository, Resettable {

  private final ConcurrentHashMap<UUID, ConcurrentHashMap<String, ConductorInboxEntry>> entries =
      new ConcurrentHashMap<>();

  @Override
  public void save(ConductorInboxEntry entry, String tenancyId) {
    entries.computeIfAbsent(entry.caseId(), k -> new ConcurrentHashMap<>())
        .put(entry.id(), entry);
  }

  @Override
  public ConductorInboxEntry findById(UUID caseId, String entryId, String tenancyId) {
    var caseEntries = entries.get(caseId);
    return caseEntries != null ? caseEntries.get(entryId) : null;
  }

  @Override
  public List<ConductorInboxEntry> findPending(UUID caseId, String tenancyId) {
    var caseEntries = entries.get(caseId);
    if (caseEntries == null) return List.of();
    return caseEntries.values().stream()
        .filter(e -> e.status() == ConductorInboxEntry.Status.PENDING)
        .toList();
  }

  @Override
  public int countPending(UUID caseId, String tenancyId) {
    var caseEntries = entries.get(caseId);
    if (caseEntries == null) return 0;
    return (int) caseEntries.values().stream()
        .filter(e -> e.status() == ConductorInboxEntry.Status.PENDING)
        .count();
  }

  @Override
  public List<ConductorInboxEntry> findAll(UUID caseId, String tenancyId) {
    var caseEntries = entries.get(caseId);
    if (caseEntries == null) return List.of();
    return List.copyOf(caseEntries.values());
  }

  @Override
  public void reset() { entries.clear(); }
}
```

**Note:** `save()` reads `caseId` from `entry.caseId()`. The `ConductorInboxEntry` record must expose a `caseId` field. Currently it does not — see §Domain Bean Refactoring for the required change.

### InMemoryWatchPatternStore

```java
@DefaultBean
@ApplicationScoped
public class InMemoryWatchPatternStore
    implements WatchPatternStore, Resettable {

  private final ConcurrentHashMap<UUID, CopyOnWriteArrayList<WatchPattern>> patterns =
      new ConcurrentHashMap<>();

  @Override
  public void save(UUID caseId, WatchPattern pattern, String tenancyId) {
    patterns.computeIfAbsent(caseId, k -> new CopyOnWriteArrayList<>()).add(pattern);
  }

  @Override
  public void remove(UUID caseId, String patternId, String tenancyId) {
    var casePatterns = patterns.get(caseId);
    if (casePatterns != null) casePatterns.removeIf(p -> p.id().equals(patternId));
  }

  @Override
  public List<WatchPattern> findActive(UUID caseId, String tenancyId) {
    var casePatterns = patterns.get(caseId);
    return casePatterns != null ? List.copyOf(casePatterns) : List.of();
  }

  @Override
  public void reset() { patterns.clear(); }
}
```

### InMemoryDenyPatternStore

```java
@DefaultBean
@ApplicationScoped
public class InMemoryDenyPatternStore
    implements DenyPatternStore, Resettable {

  private final ConcurrentHashMap<UUID, Set<String>> patterns =
      new ConcurrentHashMap<>();

  @Override
  public void save(UUID caseId, String pattern, String tenancyId) {
    patterns.computeIfAbsent(caseId, k -> ConcurrentHashMap.newKeySet()).add(pattern);
  }

  @Override
  public void remove(UUID caseId, String pattern, String tenancyId) {
    var casePatterns = patterns.get(caseId);
    if (casePatterns != null) casePatterns.remove(pattern);
  }

  @Override
  public Set<String> findAll(UUID caseId, String tenancyId) {
    return Set.copyOf(patterns.getOrDefault(caseId, Set.of()));
  }

  @Override
  public void reset() { patterns.clear(); }
}
```

### InMemoryImprovementBlockStore

```java
@DefaultBean
@ApplicationScoped
public class InMemoryImprovementBlockStore
    implements ImprovementBlockStore, Resettable {

  private final ConcurrentHashMap<UUID, ConcurrentHashMap<UUID, UUID>> blocks =
      new ConcurrentHashMap<>();

  @Override
  public void save(UUID caseId, UUID improvementCaseId, UUID blockerImprovementId,
      String tenancyId) {
    blocks.computeIfAbsent(caseId, k -> new ConcurrentHashMap<>())
        .put(improvementCaseId, blockerImprovementId);
  }

  @Override
  public void remove(UUID caseId, UUID improvementCaseId, String tenancyId) {
    var caseBlocks = blocks.get(caseId);
    if (caseBlocks != null) caseBlocks.remove(improvementCaseId);
  }

  @Override
  public boolean isBlocked(UUID caseId, UUID improvementCaseId, String tenancyId) {
    var caseBlocks = blocks.get(caseId);
    return caseBlocks != null && caseBlocks.containsKey(improvementCaseId);
  }

  @Override
  public UUID blockedBy(UUID caseId, UUID improvementCaseId, String tenancyId) {
    var caseBlocks = blocks.get(caseId);
    return caseBlocks != null ? caseBlocks.get(improvementCaseId) : null;
  }

  @Override
  public void reset() { blocks.clear(); }
}
```

## Domain Bean Refactoring

### ConductorInboxManager

**Before:** owns `ConcurrentHashMap<UUID, ConcurrentHashMap<String, ConductorInboxEntry>>` entries and `ConcurrentHashMap<UUID, CopyOnWriteArrayList<WatchPattern>>` watchPatterns. Implements `Resettable`.

**After:** stateless logic bean. Injects `ConductorInboxRepository` and `WatchPatternStore`. No longer implements `Resettable`.

```java
@ApplicationScoped
public class ConductorInboxManager {

  private final ConductorInboxRepository inboxRepository;
  private final WatchPatternStore watchPatternStore;

  @Inject
  ConductorInboxManager(ConductorInboxRepository inboxRepository,
      WatchPatternStore watchPatternStore) {
    this.inboxRepository = inboxRepository;
    this.watchPatternStore = watchPatternStore;
  }

  public String enqueue(UUID caseId, ConductorInboxEntry entry, String tenancyId) {
    inboxRepository.save(entry, tenancyId);
    return entry.id();
  }

  public List<ConductorInboxEntry> pending(UUID caseId, String tenancyId) {
    return inboxRepository.findPending(caseId, tenancyId);
  }

  public int pendingCount(UUID caseId, String tenancyId) {
    return inboxRepository.countPending(caseId, tenancyId);
  }

  public List<ConductorInboxEntry> allEntries(UUID caseId, String tenancyId) {
    return inboxRepository.findAll(caseId, tenancyId);
  }

  public void resolve(UUID caseId, String entryId, ConductorDecision decision,
      String tenancyId) {
    var existing = inboxRepository.findById(caseId, entryId, tenancyId);
    if (existing == null) return;
    var resolved = new ConductorInboxEntry(
        existing.id(), existing.stage(), decision.outcome(),
        existing.category(), existing.areaId(), existing.improvementCaseId(),
        existing.summary(), existing.escalationTriggers(), existing.confidence(),
        existing.queuedAt(), Instant.now(), existing.timeoutMinutes(), decision);
    inboxRepository.save(resolved, tenancyId);
  }

  public List<WatchPattern> activeWatchPatterns(UUID caseId, String tenancyId) {
    return watchPatternStore.findActive(caseId, tenancyId);
  }

  public void addWatchPattern(UUID caseId, WatchPattern pattern, String tenancyId) {
    watchPatternStore.save(caseId, pattern, tenancyId);
  }

  public void removeWatchPattern(UUID caseId, String patternId, String tenancyId) {
    watchPatternStore.remove(caseId, patternId, tenancyId);
  }
}
```

**ConductorInboxEntry caseId field:** The `save()` method on `ConductorInboxRepository` reads `caseId` from the entry itself. The current `ConductorInboxEntry` record does not include a `caseId` field — the caseId was the outer map key. Add `UUID caseId` as the first field of the record. All call sites that construct `ConductorInboxEntry` must pass the caseId.

### ImprovementBudgetEnforcer

**Before:** owns `dynamicDenyPatterns` (ConcurrentHashMap), `activeImprovements`, `dailyCounts`, `lastCompletionTime`. Implements `Resettable`.

**After:** injects `DenyPatternStore`. Keeps `Resettable` for remaining state (`activeImprovements`, `dailyCounts`, `lastCompletionTime`). Only `dynamicDenyPatterns` is extracted.

```java
@ApplicationScoped
public class ImprovementBudgetEnforcer implements Resettable {

  private static final Set<String> STRUCTURAL_DENIED_PATTERNS = Set.of(/* unchanged */);

  private final DenyPatternStore denyPatternStore;
  private final ConcurrentHashMap<UUID, ImprovementRequest> activeImprovements =
      new ConcurrentHashMap<>();
  private final ConcurrentHashMap<LocalDate, AtomicInteger> dailyCounts =
      new ConcurrentHashMap<>();
  private volatile Instant lastCompletionTime = Instant.EPOCH;

  @Inject
  ImprovementBudgetEnforcer(DenyPatternStore denyPatternStore) {
    this.denyPatternStore = denyPatternStore;
  }

  public BudgetCheck check(UUID caseId, ImprovementBudget budget,
      ImprovementRequest request, String tenancyId) {
    // Structural deny check (unchanged — compile-time constants)
    for (String path : request.targetPaths()) {
      for (String pattern : STRUCTURAL_DENIED_PATTERNS) {
        if (path.contains(pattern)) {
          return new BudgetCheck.Denied("Structural self-modification denied: " + path);
        }
      }
    }

    // Dynamic deny check (delegated to store)
    var dynamic = denyPatternStore.findAll(caseId, tenancyId);
    for (String path : request.targetPaths()) {
      for (String pattern : dynamic) {
        if (path.contains(pattern)) {
          return new BudgetCheck.Denied("Path denied by dynamic deny pattern: " + path);
        }
      }
    }

    // Remaining budget checks (unchanged) ...
  }

  public void addDenyPattern(UUID caseId, String pattern, String tenancyId) {
    denyPatternStore.save(caseId, pattern, tenancyId);
  }

  public void removeDenyPattern(UUID caseId, String pattern, String tenancyId) {
    denyPatternStore.remove(caseId, pattern, tenancyId);
  }

  public Set<String> dynamicDenyPatterns(UUID caseId, String tenancyId) {
    return denyPatternStore.findAll(caseId, tenancyId);
  }

  public boolean isDenied(UUID caseId, ImprovementRequest request, String tenancyId) {
    for (String path : request.targetPaths()) {
      for (String pattern : STRUCTURAL_DENIED_PATTERNS) {
        if (path.contains(pattern)) return true;
      }
      var dynamic = denyPatternStore.findAll(caseId, tenancyId);
      for (String pattern : dynamic) {
        if (path.contains(pattern)) return true;
      }
    }
    return false;
  }

  // recordStart, recordCompletion, activeCount, activeImprovementRequests
  // unchanged — these don't touch persistent state

  @Override
  public void reset() {
    activeImprovements.clear();
    dailyCounts.clear();
    lastCompletionTime = Instant.EPOCH;
    // dynamicDenyPatterns no longer here — reset handled by InMemoryDenyPatternStore
  }
}
```

### ImprovementCoordinator

**Before:** owns `blocks` (ConcurrentHashMap). Implements `Resettable`.

**After:** stateless logic bean. Injects `ImprovementBlockStore`. No longer implements `Resettable`.

```java
@ApplicationScoped
public class ImprovementCoordinator {

  private final ImprovementBlockStore blockStore;

  @Inject
  ImprovementCoordinator(ImprovementBlockStore blockStore) {
    this.blockStore = blockStore;
  }

  public void block(UUID caseId, UUID improvementCaseId, UUID blockerImprovementId,
      String tenancyId) {
    blockStore.save(caseId, improvementCaseId, blockerImprovementId, tenancyId);
  }

  public void unblock(UUID caseId, UUID improvementCaseId, String tenancyId) {
    blockStore.remove(caseId, improvementCaseId, tenancyId);
  }

  public boolean isBlocked(UUID caseId, UUID improvementCaseId, String tenancyId) {
    return blockStore.isBlocked(caseId, improvementCaseId, tenancyId);
  }

  @Nullable
  public UUID blockedBy(UUID caseId, UUID improvementCaseId, String tenancyId) {
    return blockStore.blockedBy(caseId, improvementCaseId, tenancyId);
  }
}
```

## Caller Updates

Domain bean method signatures gain `tenancyId`. Callers that already have `tenancyId` in scope simply pass it through.

| Caller | Injects | Methods affected | tenancyId source |
|--------|---------|-----------------|-----------------|
| `DefaultEngineEvolutionApi` | `ConductorInboxManager`, `ImprovementBudgetEnforcer`, `ImprovementCoordinator` | All evolution API methods | Already a parameter on every API method |
| `ResearchPipelineOrchestrator` | `ConductorInboxManager` | `enqueue()` calls | Already has `tenancyId` in `execute()`/`resume()` |
| `ImprovementGoalFormationStrategy` | `ImprovementBudgetEnforcer` | `check()`, `isDenied()` | Needs `tenancyId` added to `proposeImprovements()` — sourced from `EvolutionTicker.tick()` which already has it |
| `ImprovementOutcomeEventCapture` | `ImprovementBudgetEnforcer` | `recordStart()`, `recordCompletion()` | These methods don't touch the deny pattern store — no tenancyId needed |

**No injection changes for callers.** `DefaultEngineEvolutionApi` still injects the domain beans (not the repository SPIs). The domain beans internally delegate to repositories. The only caller-visible change is the `tenancyId` parameter on domain bean methods.

## Test Strategy

### Contract tests (common-core/test)

Abstract contract test per SPI, following the `CaseInstanceRepositoryContractTest` pattern:

| Contract test | SPI |
|--------------|-----|
| `ConductorInboxRepositoryContractTest` | `ConductorInboxRepository` |
| `WatchPatternStoreContractTest` | `WatchPatternStore` |
| `DenyPatternStoreContractTest` | `DenyPatternStore` |
| `ImprovementBlockStoreContractTest` | `ImprovementBlockStore` |

Each abstract test class defines `abstract T createStore()`. Test methods exercise the full SPI contract: save/find/remove, boundary cases (not found, empty results), per-case isolation.

### In-memory impl tests (runtime-core/test)

Concrete tests extending contract tests:

| Test class | Extends |
|-----------|---------|
| `InMemoryConductorInboxRepositoryContractTest` | `ConductorInboxRepositoryContractTest` |
| `InMemoryWatchPatternStoreContractTest` | `WatchPatternStoreContractTest` |
| `InMemoryDenyPatternStoreContractTest` | `DenyPatternStoreContractTest` |
| `InMemoryImprovementBlockStoreContractTest` | `ImprovementBlockStoreContractTest` |

Each provides `createStore()` returning the in-memory implementation. Additionally tests `Resettable.reset()`.

### Updated domain bean tests

| Test class | Changes |
|-----------|---------|
| `ConductorInboxManagerTest` | Constructor injects in-memory repos. Tests business logic (enqueue/resolve semantics, pending filtering). Removes direct map assertions. Adds `tenancyId` to all calls. |
| `ImprovementBudgetEnforcerTest` | Constructor injects `InMemoryDenyPatternStore`. Adds `tenancyId` to `check()`, `addDenyPattern()`, `removeDenyPattern()`, `isDenied()`. Budget/cooldown/concurrency tests unchanged. |
| `ImprovementCoordinatorTest` | Constructor injects `InMemoryImprovementBlockStore`. Adds `tenancyId` to all calls. Tests unchanged except for the parameter addition. |

### Critical test scenarios

1. **Per-case isolation:** Store data for caseA and caseB → queries for caseA return only caseA data
2. **Save idempotency:** Saving the same inbox entry twice (same id) → overwrites, count stays 1
3. **Pending filter accuracy:** Mix of PENDING/APPROVED/REJECTED entries → `findPending` returns only PENDING
4. **Reset clears all:** Populate store → reset → all queries return empty
5. **Domain bean delegation:** Enqueue via `ConductorInboxManager` → entry retrievable via injected `ConductorInboxRepository`
6. **Budget check with deny store:** Add dynamic deny pattern via store → `ImprovementBudgetEnforcer.check()` returns Denied for matching paths

## Module Placement

| Component | Module | Package |
|-----------|--------|---------|
| `ConductorInboxRepository` | common-core | `io.casehub.engine.common.spi` |
| `WatchPatternStore` | common-core | `io.casehub.engine.common.spi` |
| `DenyPatternStore` | common-core | `io.casehub.engine.common.spi` |
| `ImprovementBlockStore` | common-core | `io.casehub.engine.common.spi` |
| `InMemoryConductorInboxRepository` | runtime-core | `io.casehub.engine.internal.improvement` |
| `InMemoryWatchPatternStore` | runtime-core | `io.casehub.engine.internal.improvement` |
| `InMemoryDenyPatternStore` | runtime-core | `io.casehub.engine.internal.improvement` |
| `InMemoryImprovementBlockStore` | runtime-core | `io.casehub.engine.internal.improvement` |
| `ConductorInboxManager` (refactored) | runtime-core | `io.casehub.engine.internal.improvement` |
| `ImprovementBudgetEnforcer` (refactored) | runtime-core | `io.casehub.engine.internal.improvement` |
| `ImprovementCoordinator` (refactored) | runtime-core | `io.casehub.engine.internal.improvement` |

## References

- `CaseInstanceRepository.java` (common-core/spi — SPI interface pattern, tenant-scoped methods)
- `EventLogRepository.java` (common-core/spi — SPI interface pattern, `tenancyId` convention)
- `InMemoryCaseInstanceRepository.java` (engine-support-core — in-memory implementation pattern)
- `CaseInstanceRepositoryContractTest.java` (common-core/test — contract test pattern)
- `DefaultSummarizationProvider.java` (runtime-core — `@DefaultBean` pattern)
- `DefaultEscalationProvider.java` (runtime-core — `@DefaultBean` pattern)
- `ConductorInboxManager.java` (runtime-core — current implementation, lines 31–118)
- `ImprovementBudgetEnforcer.java` (runtime-core — current implementation, lines 32–206)
- `ImprovementCoordinator.java` (runtime-core — current implementation, lines 25–58)
- `DefaultEngineEvolutionApi.java` (runtime-core — caller, injects all 3 domain beans)
- `ResearchPipelineOrchestrator.java` (runtime-core — caller, injects ConductorInboxManager)
- `ImprovementGoalFormationStrategy.java` (runtime-core — caller, injects ImprovementBudgetEnforcer)
- `ImprovementOutcomeEventCapture.java` (runtime-core — caller, injects ImprovementBudgetEnforcer)
- `2026-09-21-command-centre-conductor-design.md` (#1132 — conductor architecture)
- casehubio/engine#1150 — persistence layer coherence epic
- casehubio/engine#1166 — CaseContextRecoveryStrategy SPI
- casehubio/engine#1151, #1152 — EventLog-replay data-loss bugs
- PP-20260921-b7c277 — no-defaultbean-multi-instance-spi protocol
- D1–D5 in `decisions.md`
