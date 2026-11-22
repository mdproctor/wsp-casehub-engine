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
