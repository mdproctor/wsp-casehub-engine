---
layout: post
title: "One Store to Rule Them All"
date: 2026-11-22
entry_type: note
subtype: diary
projects: [casehubio/engine]
tags: [architecture, spi, persistence, data-store, platform]
series: issue-1332-collection-store-spi
---

# One Store to Rule Them All

Every platform I've studied that lets modules persist data eventually arrives at the same crossroads: do you give each module its own storage abstraction, or do you build one SPI that everyone shares?

CaseHub had been walking down the first path. The engine has `CaseInstanceRepository`. Qhorus has `ChannelStore`, `MessageStore`, `InstanceStore`, and seventeen more like them. Neocortex has four separate SQLite stores. Pages has `JsonCollectionStore`. Each does roughly the same thing — store records, look them up, filter them — but each reinvents the contract, the tenant isolation, the versioning, and the error handling. Each gets its own in-memory test double. Each has its own quirks that surprise you when you cross a module boundary.

Keycloak had exactly this problem. Their per-entity `MapStorageProvider` SPIs proliferated until maintaining them became the bottleneck. They consolidated into an area-based generic SPI. Salesforce's MT_Data goes further — a single universal table architecture where types are metadata, not schema. JCR/Oak separates the concerns cleanly: `NodeStore` knows about bytes and trees, never about content types. The type system lives above.

The pattern is clear enough: storage SPIs should be generic, with types and schema as a layer above. We built that for CaseHub.

## The SPI

`DataStore` is the entry point. You register a `CollectionDescriptor` if you want schema — fields with types, indexes, relations — or you skip it entirely and get schemaless document storage. Either way, the `DataCollection` interface gives you the same operations: insert, get, update, delete, query, count, search.

```java
public interface DataStore {
  void register(CollectionDescriptor descriptor);
  DataCollection collection(String name);
}
```

```java
public interface DataCollection {
  String insert(ObjectNode record, DataContext ctx);
  String insert(String id, ObjectNode record, DataContext ctx);
  Optional<ObjectNode> get(String id, DataContext ctx);
  void update(String id, ObjectNode changes, OptionalInt expectedVersion,
              MergeStrategy mergeStrategy, DataContext ctx);
  // ...
}
```

Records are Jackson `ObjectNode` — the platform's de facto interchange format. Every module already has Jackson on the classpath. System fields (`_id`, `_version`, `_created_at`, `_updated_at`) are managed by the implementation, stripped on write, injected on read. The caller never thinks about them.

`DataContext` carries tenant and app identity. Every operation is tenant-scoped. There's no ambient state, no CDI-injected tenant — the context is explicit on every call, same pattern as `CaseInstanceRepository`.

## Why ObjectNode and not Map

I considered `Map<String, Object>` and a custom `Record` interface. Maps are weakly typed — nested structures need casting, and there's no way to represent the difference between a missing field and a null one. A custom type would avoid the Jackson coupling but adds something to learn with no clear upside.

ObjectNode handles typed fields naturally, works for both JSON blob storage and typed-column mapping, and the Jackson dependency is already universal. The coupling is real but the cost was already paid.

## Two Storage Strategies

The SPI ships with two SQLite implementations that prove the abstraction works at both ends of the spectrum.

**Document-backed** (`SqliteDocumentDataStore`) stores every record as a JSON blob in a single `data_records` table. The composite key is `(tenant_id, app_id, collection, doc_id)`. Filtering and sorting happen in Java after loading. Simple, correct, good enough for small collections.

**Table-backed** (`SqliteTypedDataStore`) uses the same table structure but pushes filter and sort operations down to SQLite using `json_extract()` on declared fields. A registered collection with field descriptors gets SQL-level query pushdown. An unregistered collection falls back to the document behaviour. Same data, different query path.

Both pass the same `DataStoreContractTest` — an abstract test class that any implementation must satisfy. Seventeen tests covering CRUD, optimistic concurrency, merge strategies, pagination, sorting, tenant isolation, defensive copies, and schemaless access. Add a new implementation, extend the contract test, provide a `DataStore` and a `DataContext`, and you know immediately if it's correct.

## The Engine Adapter

`DataStoreCaseInstanceRepository` implements the existing `CaseInstanceRepository` SPI by delegating to `DataStore`. `CaseInstanceMapper` handles the bidirectional conversion between the domain model and `ObjectNode`, flattening `CaseMetaModel` fields — namespace, name, version — to top-level record fields so they're queryable through the standard filter mechanism.

The adapter passes all nine contract tests that any `CaseInstanceRepository` implementation must satisfy. No special cases, no workarounds. The domain model's `UUID` becomes the document ID via the explicit-id `insert` overload — the SPI doesn't generate a random ID when the domain already has one.

## What This Opens Up

The immediate follow-on is qhorus. Twenty-plus per-entity store SPIs, each with in-memory, JPA, and Spring implementations. That's the exact proliferation pattern Keycloak consolidated. Each qhorus store becomes a registered collection on a shared `DataStore` instance — `ChannelStore` becomes `dataStore.collection("channels")` with a `CollectionDescriptor` declaring the fields that need indexing.

Neocortex has four SQLite stores that could unify at the infrastructure layer. Memory and cognitive-observability are straightforward migrations. Mindmap is graph-structured and may stay domain-specific — but even there, the document storage mode could handle the node and edge records while graph traversal remains a domain concern.

The bigger play is `DataSource` SPI abstraction — right now SQLite is hardwired in both implementations. Abstracting the connection source would let the same `DataStore` SPI serve PostgreSQL in production and SQLite in tests, without the consumer knowing or caring. That's a separate issue, and it only makes sense once the SPI has proven itself in real use.
