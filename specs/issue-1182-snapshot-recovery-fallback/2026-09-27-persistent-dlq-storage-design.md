# Persistent DLQ Storage — Design Spec

**Issue:** casehubio/engine#1185
**Epic:** casehubio/engine#210 — cancellation, timeout, and error recovery
**Date:** 2026-09-27

## Problem

`DeadLetterQueue` stores entries in a `ConcurrentHashMap`. All DLQ entries are lost on restart. A worker failure followed by a restart means the failure is unrecoverable — no record exists for the admin to replay.

## Solution

### 1. DeadLetterEntryStore SPI (D9)

Define a store SPI in `resilience-core`:

```java
public interface DeadLetterEntryStore {
    DeadLetterEntry save(DeadLetterEntry entry);
    DeadLetterEntry findById(String deadLetterId);
    List<DeadLetterEntry> query(DeadLetterQuery query);
    void updateStatus(String deadLetterId, DeadLetterStatus status);
    void incrementReplayAttempts(String deadLetterId);
    void deleteAll();
}
```

### 2. InMemoryDeadLetterEntryStore

In `resilience-core`. Wraps a `ConcurrentHashMap<String, DeadLetterEntry>` — extracts the current `DeadLetterQueue` storage behavior into a standalone implementation. Default for tests.

### 3. DeadLetterQueue refactoring (D10)

`DeadLetterQueue` takes a `DeadLetterEntryStore` constructor parameter. Methods delegate:

| Queue method | Store delegation |
|-------------|-----------------|
| `add(caseId, workerId, ...)` | Constructs `DeadLetterEntry`, calls `store.save(entry)` |
| `query(query)` | `store.query(query)` |
| `findById(id)` | `store.findById(id)` |
| `discard(id)` | `store.updateStatus(id, DISCARDED)` |
| `markReplayed(id)` | `store.updateStatus(id, REPLAYED)` |
| `size()` | `store.query(DeadLetterQuery.all()).size()` |
| `clear()` | `store.deleteAll()` |

`DeadLetterEntry.setStatus()` and `incrementReplayAttempts()` remain for in-memory mutation but the store is the persistence authority.

### 4. JPA implementation (D11)

In `persistence-hibernate`. `DeadLetterEntryEntity` with fields:

| Field | Type | Notes |
|-------|------|-------|
| `id` | `Long` | Auto-generated PK |
| `deadLetterId` | `String` | Unique, indexed |
| `caseId` | `UUID` | Indexed |
| `workerId` | `String` | |
| `idempotencyHash` | `String` | |
| `inputContext` | `String` (JSONB) | Serialized `Map<String, Object>` |
| `retryState` | `String` (JSONB) | Serialized `RetryState` |
| `status` | `DeadLetterStatus` (enum) | |
| `replayAttempts` | `int` | |
| `arrivedAt` | `Instant` | |
| `lastReplayAttemptAt` | `Instant` | Nullable |
| `tenancyId` | `String` | From case context |

Schema managed by Hibernate `drop-and-create`. No Flyway migration (CLAUDE.md constraint).

`JpaDeadLetterEntryStore` maps between `DeadLetterEntry` and `DeadLetterEntryEntity`. `persistence-hibernate` gains `resilience-core` as a dependency.

### 5. CDI Wiring

`DeadLetterQueue` is currently produced in `ResilienceBeans` (or equivalent). Update the producer to inject `DeadLetterEntryStore` and pass it to the constructor.

The `InMemoryDeadLetterEntryStore` is `@ApplicationScoped` with `@DefaultBean` or low `@Priority` — the JPA impl overrides when the persistence module is on the classpath.

## Files Changed

| File | Module | Change |
|------|--------|--------|
| `DeadLetterEntryStore.java` | resilience-core | **Create**: SPI interface |
| `InMemoryDeadLetterEntryStore.java` | resilience-core | **Create**: ConcurrentHashMap implementation |
| `DeadLetterQueue.java` | resilience-core | **Modify**: constructor takes store, delegate all storage |
| `DeadLetterEntry.java` | resilience-core | **Modify**: make constructor public (store needs to create entries) |
| `DeadLetterEntryEntity.java` | persistence-hibernate | **Create**: JPA entity |
| `JpaDeadLetterEntryStore.java` | persistence-hibernate | **Create**: JPA implementation |
| `pom.xml` | persistence-hibernate | **Modify**: add resilience-core dependency |
| `DeadLetterQueueTest.java` | resilience-core/test | **Modify**: use InMemoryDeadLetterEntryStore |
| Producer/beans class | resilience or runtime | **Modify**: wire store into queue |

## What This Does NOT Cover

- Query pagination — not needed at pre-release scale
- DLQ entry TTL/expiry — entries persist indefinitely
- Spring JPA implementation — can be added following the same pattern

## References

- `DeadLetterQueue.java:33-113` — current in-memory implementation
- `DeadLetterEntry.java:31-109` — entry record
- `CaseQueueEntryStore.java` — similar SPI pattern
- `JpaCaseQueueEntryStore.java` — similar JPA implementation
- CLAUDE.md §No Migration Tooling — Flyway constraint
- decisions.md D9-D11
