# MapCaseFile Design

**Date:** 2026-05-12  
**Issue:** treblereel/casehub-engine#239  
**Status:** Approved

## Context

casehub-poc's `CaseFile` was a god object: key-value store + case identity +
lifecycle + graph relationships + change listeners in one interface. Workers
wrote `caseFile.get("key", Type.class)` and `caseFile.put("key", value)`.

casehub-engine disaggregated those concerns correctly:

| Poc `CaseFile` capability | Engine equivalent |
|---|---|
| `get/put/contains/keys/snapshot/putIfAbsent` | `CaseContext` |
| `putIfVersion` (per-key OCC) | `CaseContext.compareAndSet()` |
| `getPropagationContext()` | `PropagationContext` via `WorkerContext` |
| `getId/getStatus/getCreatedAt` | `CaseInstance` |
| `complete()/fail()` | Engine lifecycle — workers signal via return/exception |
| `onChange/onAnyChange` | CDI events (`CaseContextChangedEvent`) — durable, not in-memory |
| `getParentCase/getChildCases` | Sub-case model in `casehub-blackboard` |

No functionality is lost — each capability moved to the right abstraction. The
engine is strictly more capable: `onChange` became a durable CDI event rather
than an in-memory-only listener; lifecycle is engine-driven rather than
worker-driven; graph relationships are a first-class blackboard concern.

## Problem

Worker code migrated from the poc has a naming mismatch at the call sites:
`put/get/keys` vs `set/getAs/getKeys`. `MapCaseFile` bridges this gap as a
stepping-stone shim, named to signal its migration intent.

Analysis of poc `execute()` methods shows 36 of 40 CaseFile calls (90%) are
`get` or `put` — the exact methods this shim targets.

## Decision

**Approach A — subclass `CaseContextImpl` in `runtime/`.** `MapCaseFile`
extends `CaseContextImpl` in the new public package
`io.casehub.engine.context`, adding three alias methods. Inheritance is
appropriate: `MapCaseFile` is a `CaseContextImpl` with a poc-compatible
surface; no new state is introduced.

Alternatives rejected:
- **Composition in `api/`** — 25+ delegation methods of pure boilerplate for a
  migration shim; no one depends on `api/` without `runtime/`
- **Add methods to `CaseContextImpl`** — pollutes the internal implementation
  with migration API; aliases would not appear on the `CaseContext` interface

## Design

### Class

```
runtime/src/main/java/io/casehub/engine/context/MapCaseFile.java
```

Package `io.casehub.engine.context` — new non-internal public package in
`runtime`. `CaseContextImpl` stays in `internal/context/`.

```java
public class MapCaseFile extends CaseContextImpl {

    public MapCaseFile() {}

    public MapCaseFile(Map<String, Object> initial) {
        super(initial);
    }

    public void put(String key, Object value)        { set(key, value); }
    public <T> T get(String key, Class<T> type)      { return getAs(key, type); }
    public Set<String> keys()                        { return getKeys(); }
}
```

Everything else — `contains`, `putIfAbsent`, `snapshot`, `getOrDefault`,
`compareAndSet`, locking, versioning, `asJsonNode`, `diff`, `applyDiff` —
is inherited unchanged from `CaseContextImpl`.

`get(key, Class<T>)` returns nullable `T` (not `Optional<T>`), consistent
with the engine's `CaseContext` convention and platform protocol
PP-20260512-5f055d (Optional only when absence is the primary return contract;
map accessors use null + `contains()`/`getOrDefault()`).

### Thread safety

Inherited from `CaseContextImpl` (ReadWriteLock on all operations). No
additional synchronisation needed.

### Scope and intent

`MapCaseFile` is a **migration stepping-stone**, not a long-term API. It eases
the rename burden during poc migration. Once migrated, call sites can switch to
`CaseContextImpl` directly or use the `CaseContext` interface.

## Testing

Plain JUnit 5 unit test — no Quarkus, no container.

```
runtime/src/test/java/io/casehub/engine/context/MapCaseFileTest.java
```

Coverage:
- **Happy path:** `put/get` roundtrip with type casting; `keys()` reflects
  all puts; `contains()` returns true after put
- **Correctness:** `get()` returns null for missing key; `putIfAbsent` does
  not overwrite existing value; `snapshot()` is an independent copy;
  `MapCaseFile instanceof CaseContext` is true
- **Robustness:** null value round-trip; construction from existing map;
  empty construction
