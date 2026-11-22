# Handoff — #1332 DataStore Platform SPI

**Branch:** `issue-1332-collection-store-spi`
**Slot:** 227
**Date:** 2026-11-22

## Session Summary

This session implemented the DataStore platform SPI in full — 3 new Maven modules, 75 tests, code review with fixes, blog entry written.

### What was done

1. **Implemented Batch 1** — `data-api` module: DataStore/DataCollection/DataContext SPI interfaces, 18 supporting types (descriptors, query, filter, exceptions), InMemoryDataStore + InMemoryDataCollection, DataStoreContractTest (abstract base, 20 tests)
2. **Implemented Batch 2** — `data-sqlite` module: SqliteDocumentDataStore (JSON blob storage), SqliteTypedDataStore (json_extract pushdown), SqliteDataStoreFactory, 41 tests total
3. **Implemented Batch 3** — `persistence-datastore` module: CaseInstanceMapper (bidirectional CaseInstance ↔ ObjectNode), DataStoreCaseInstanceRepository (implements CaseInstanceRepository via DataStore SPI), CDI producer, 14 tests
4. **Code review** — 3 parallel review forks, found and fixed: SQL injection via field name interpolation in SqliteTypedDataCollection, NPE on missing migration resource, HikariCP pool leak in tests, added 3 missing contract tests (delete_nonExistent, search, RFC 7396 merge)
5. **Blog entry** — "One Store to Rule Them All" covering design rationale and what it opens up
6. **Filed issues** — #1348 (count() omits free-text filter), #1349 (document collection silently ignores GT/GTE/LT/LTE filter ops)

### Key decisions (from previous session, implemented here)

| # | Decision | Choice |
|---|----------|--------|
| D1 | SPI approach | Rich Collection SPI with schema descriptors |
| D2 | Record format | Jackson ObjectNode |
| D3 | SPI home | `io.casehub.platform.api.data` in casehub-platform-api |
| D4 | Scope | SPI + two impls (document + table-backed SQLite) + engine adapter |
| D5 | Naming | DataStore / DataCollection / DataContext |
| D6 | Qhorus collision | Design for end state — qhorus DataStore is a migration target |

### Active item state

**#1332 is FINISHED.** All 6 tasks across 3 batches are implemented and tested. Code review complete. Blog written.

## What's Next

Queue has remaining items. Next unfinished: #1333 (call agent worker function provider for eidos dispatch), #1336 (integration test auto-discovered YAML wires do block workers).

## Follow-up Issues Filed

- casehubio/engine#1347 — Engine companion SPI adapters (EventLog, CaseMetaModel, PlanItem)
- casehubio/engine#1344 — Qhorus store consolidation (20+ per-entity SPIs → DataStore)
- casehubio/engine#1345 — Neocortex storage migration
- casehubio/engine#1346 — DataSource SPI abstraction (abstract SQLite away)
- casehubio/engine#1348 — CaseInstanceQuery.count() omits free-text filter
- casehubio/engine#1349 — DataStore document collection silently ignores filter ops

## Things the Next Session Must Know

- **updateStateAndAppendEvent drops EventLog** — the DataStore adapter only updates the case instance. EventLog persistence is tracked in #1347 (companion SPI adapters).
- **IntelliJ is open** for this slot at `/Users/mdproctor/claude/casehub/slots/227/engine`.
