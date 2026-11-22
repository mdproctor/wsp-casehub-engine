# Handoff — #1332 DataStore Platform SPI

**Branch:** `issue-1332-collection-store-spi`
**Slot:** 227
**Date:** 2026-11-22

## Session Summary

This session designed the DataStore platform SPI from scratch. No code was written — all output is design artifacts in the workspace.

### What was done

1. **Explored the problem space** — investigated CaseInstanceRepository/CaseInstanceCache SPIs, JsonCollectionStore API, module structure, platform boundaries (PLATFORM.md, INDEX.md)
2. **Reframed the scope** — user drove expansion from a simple CaseInstanceCache adapter to a platform-level universal data persistence SPI. Key insight: "there is no such thing as specialized json storage" — the SPI should be generic, not JSON-specific
3. **Researched best practices** — Salesforce MT_Data, JCR/Oak NodeStore, Keycloak MapStorageProvider. All converge: storage SPI is generic, type/schema is a layer above, collections are the type boundary
4. **Discovered qhorus consolidation opportunity** — qhorus has 20+ per-entity store SPIs (ChannelStore, MessageStore, etc.) — same proliferation pattern Keycloak consolidated. Strongest case for the platform SPI
5. **Investigated neocortex** — 4 SQLite stores, unification at infrastructure layer. Memory and cognitive-observability are migration candidates; mindmap (graph-structured) may stay domain-specific
6. **Captured 6 decisions** (D1–D6) in `specs/issue-1332-collection-store-spi/decisions.md`
7. **Wrote design spec** at `specs/issue-1332-collection-store-spi/2026-11-22-datastore-platform-spi-design.md`
8. **Ran standard spec review** — incorporated findings: naming collision (DataStore vs qhorus), Jackson in platform-api, extract from JsonCollectionStore
9. **Wrote implementation plan** at `plans/2026-11-22-datastore-platform-spi.md` — 3 batches, 6 tasks

### Key decisions

| # | Decision | Choice |
|---|----------|--------|
| D1 | SPI approach | Rich Collection SPI with schema descriptors |
| D2 | Record format | Jackson ObjectNode |
| D3 | SPI home | `io.casehub.platform.api.data` in casehub-platform-api |
| D4 | Scope | SPI + two impls (document + table-backed SQLite) + engine adapter |
| D5 | Naming | DataStore / DataCollection / DataContext |
| D6 | Qhorus collision | Design for end state — qhorus DataStore is a migration target |

### Active item state

**#1332 is NOT finished.** Design and planning are complete. Implementation (6 tasks across 3 batches) is the next session's work.

## What's Next

Execute the implementation plan at `plans/2026-11-22-datastore-platform-spi.md`:

1. **Batch 1:** DataStore SPI interfaces in `data-api` module + InMemory implementation + contract tests
2. **Batch 2:** SQLite implementations (document-backed + table-backed) in `data-sqlite` module
3. **Batch 3:** CaseInstanceMapper + DataStoreCaseInstanceRepository adapter in `persistence-datastore` module

Use `executing-plans` skill — tasks are sequential, not parallelizable.

## Follow-up Issues Filed

- casehubio/engine#1347 — Engine companion SPI adapters (EventLog, CaseMetaModel, PlanItem)
- casehubio/engine#1344 — Qhorus store consolidation (20+ per-entity SPIs → DataStore)
- casehubio/engine#1345 — Neocortex storage migration
- casehubio/engine#1346 — DataSource SPI abstraction (abstract SQLite away)

## Things the Next Session Must Know

- **DataCollection.insert() needs an overload** accepting an explicit ID (for cases where the domain assigns UUIDs like CaseInstance). Task 6 Step 4 in the plan flags this — fold it into Task 1.
- **SqliteDocumentDataCollection needs protected access** for the table-backed subclass. Task 4 in the plan notes fields/methods that must change from private to protected.
- **The spec was rewritten on disk by the standard review** to a narrow CaseInstanceCache-only scope. I restored it to the platform SPI scope that the user confirmed. If the file looks like a narrow adapter spec, the review overwrote it — restore from git.
- **IntelliJ is open** for this slot at `/Users/mdproctor/claude/casehub/slots/227/engine` — opened during plan writing.
