# Design Decisions — #1332 DataStore Platform SPI

## D1: SPI approach — Rich Collection SPI with Schema Descriptors

**Choice:** Comprehensive DataStore SPI with schema descriptors (fields, types, indexes, relations, storage hints). Single interface covers schema-less document storage through fully-typed relational collections.

**Alternatives:**
- Minimal Core + Extension SPIs — too thin for neocortex migration, extension boundaries become leaky abstractions, consumers must instanceof-check for capabilities
- Capability-Negotiated SPI — pushes complexity onto every consumer, runtime negotiation makes the SPI harder to reason about, testing is combinatorial

**Rationale:** The SPI must serve three audiences: (1) internal platform modules (engine case persistence, neocortex memory/mindmap/observability), (2) platform infrastructure (pages document storage), and (3) end-user applications built on the platform. A rich SPI with optional schema descriptors lets simple apps ignore schema entirely while complex apps declare fields, indexes, and relations. The implementation decides physical storage strategy — the consumer doesn't know or care.

**Trade-offs:** Larger API surface upfront. Risk of designing too much before learning from real usage. Mitigated by making schema descriptors optional — the core CRUD path works without them.

**Sources:**
- Salesforce MT_Data universal table architecture (metadata-driven types)
- JCR/Oak NodeStore SPI (storage knows nothing about types — schema is a layer above)
- Keycloak MapStorageProvider evolution (per-entity SPIs → generic area-based SPI)
- Neocortex storage modules (memory-sqlite, mindmap-sqlite, cognitive-observability-sqlite)
- CaseHub existing persistence pattern (CaseInstanceRepository, InMemory/JPA/Spring implementations)

**Exploration:** deep-analysis
**Status:** captured

## D2: Record format — ObjectNode

**Choice:** Jackson `ObjectNode` as the record type in the SPI contract.

**Alternatives:**
- `Map<String, Object>` — no Jackson dependency but weakly typed, nested structures need casting, no schema validation at the type level
- Custom `Record` interface — avoids Jackson coupling but adds a new type to learn and maintain with no clear benefit over ObjectNode

**Rationale:** ObjectNode is already the platform's de facto interchange format. It handles typed fields (numbers, strings, booleans, arrays, objects) naturally, works for both JSON blob storage and typed-column mapping, and every module already has Jackson on the classpath.

**Trade-offs:** Couples platform-api to Jackson. Acceptable because Jackson is already a universal dependency across the platform.

**Depends on:** D1 (SPI approach)
**Exploration:** quick
**Status:** captured

## D3: SPI home — casehub-platform-api

**Choice:** `io.casehub.platform.api.data` package in `casehub-platform-api`.

**Alternatives:**
- New `casehub-data-api` module — avoids growing platform-api but adds another dependency coordinate for every consumer. `platform-api-scope` protocol suggests this for domain SPIs without confirmed cross-repo adoption.
- Engine repo initially — limits adoption by other repos (pages, neocortex) and requires graduation later

**Rationale:** Cross-repo need is confirmed: engine (CaseInstanceRepository), qhorus (20+ per-entity stores), neocortex (4 SQLite stores), and pages (JsonCollectionStore) all implement the same pattern. This overrides the protocol's "start where the first consumer lives" guidance — the first consumer is the platform itself, not a single repo. Consistent with existing platform abstractions (Path, Preferences, EndpointRegistry).

**Trade-offs:** Introduces Jackson as platform-api's first external dependency (for ObjectNode). Accepted because Jackson is a universal dependency across the platform.

**Depends on:** D1 (SPI approach)
**Exploration:** quick → revised after review (R1-02, R1-03)
**Status:** captured

## D4: Scope — SPI + two implementations + engine adapter

**Choice:** This issue delivers: (1) DataStore SPI in platform-api, (2) document-backed SQLite implementation, (3) table-backed SQLite implementation, (4) CaseInstanceRepository adapter in engine, (5) contract tests. Neocortex migration, DataSource SPI abstraction, and pages refactor are follow-up issues.

**Alternatives:**
- Full platform migration (SPI + all migrations) — too large for one issue, needs an epic with 5+ children
- SPI + document impl only — doesn't prove the SPI works for table-backed storage, risks bias

**Rationale:** Building both implementations validates the SPI is genuinely implementation-agnostic. The engine adapter proves it works for real domain persistence. Follow-up issues handle migration of existing consumers.

**Trade-offs:** SQLite is hardwired in both implementations for now — DataSource SPI abstraction is deferred. Acceptable because the implementations are behind the DataStore SPI, so swapping the backing store later doesn't affect consumers.

**Depends on:** D1 (SPI approach), D3 (SPI home)
**Exploration:** quick
**Status:** captured

## D5: Naming — DataStore

**Choice:** `DataStore` as the core SPI interface. `DataCollection` for collection access. `DataContext` for tenant context.

**Alternatives:**
- `CollectionStore` — more specific, avoids qhorus collision, but risks confusion with `java.util.Collection`
- `RecordStore` — implies flat key-value without relational/query capabilities

**Rationale:** Generic name matches the platform's role as the canonical data persistence SPI. The temporary naming collision with `io.casehub.qhorus.api.store.DataStore` is accepted — qhorus' `DataStore` is a migration target that will be removed when qhorus adopts the platform SPI. Design for the end state, not the transitional conflict.

**Depends on:** D1 (SPI approach)
**Exploration:** quick → revised after review (R1-01, qhorus analysis)
**Status:** captured

## D6: Qhorus DataStore collision — design for end state

**Choice:** Keep `DataStore` as the platform name. Qhorus' `DataStore` interface is a migration target, not a peer. During qhorus migration, its 20+ per-entity store SPIs (including `DataStore`) will be replaced by registered collections on the platform DataStore SPI.

**Alternatives:**
- Rename platform SPI to avoid collision — compromises the best long-term name for a temporary conflict
- Rename qhorus' DataStore now — creates unnecessary churn before migration

**Rationale:** Qhorus has 20+ per-entity store SPIs (ChannelStore, MessageStore, InstanceStore, DataStore, etc.), each with in-memory, JPA, and Spring implementations. This is the exact per-entity SPI proliferation pattern that Keycloak consolidated. The platform DataStore SPI is designed to subsume all of these. Naming it around qhorus' current interface would embed a transitional compromise into the platform's canonical API.

**Trade-offs:** Temporary import ambiguity during the migration period — any class importing both `io.casehub.platform.api.data.DataStore` and `io.casehub.qhorus.api.store.DataStore` needs qualified names. Accepted as a short-term cost.

**Depends on:** D5 (Naming)
**Exploration:** deep-analysis (qhorus codebase investigation)
**Status:** captured
