# Design Decisions — #1332 CollectionStore Platform SPI

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

**Choice:** `io.casehub.platform.data` package in `casehub-platform-api`.

**Alternatives:**
- New `casehub-data-api` module — avoids growing platform-api but adds another dependency coordinate for every consumer
- Engine repo initially — limits adoption by other repos (pages, neocortex) and requires graduation later

**Rationale:** platform-api is zero-dep pure Java, any repo can depend on it. Consistent with existing platform abstractions (Path, Preferences, EndpointRegistry). The data SPI is a platform concern, not an engine concern.

**Trade-offs:** platform-api grows larger. Acceptable — the data SPI is a core platform capability, not a niche addition.

**Depends on:** D1 (SPI approach)
**Exploration:** quick
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
- `CollectionStore` — more specific but risks confusion with `java.util.Collection`
- `RecordStore` — implies flat key-value without relational/query capabilities

**Rationale:** Generic name matches the platform's role. Clean, no implementation leakage. Consistent with the aspiration that this serves end-user applications as a first-class data persistence API.

**Depends on:** D1 (SPI approach)
**Exploration:** quick
**Status:** captured
