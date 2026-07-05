# Semantic Types and Labels on CaseDefinition

**Issue:** casehubio/engine#652
**Date:** 2026-07-05
**Status:** Approved

## Summary

Add `types: Set<Path>` and `labels: Set<Path>` to `CaseDefinition`, establishing a platform convention for classifying definable entities. Remove vestigial `tags` (object) and `metadata` (object) from the YAML schema — both are dead code from the original Serverless Workflow scaffold, never mapped to the Java API, never consumed by any runtime code. (The bundled `document-processing.yaml` example uses `metadata` in a vestigial Kubernetes-style pattern — it is updated as part of this cleanup.)

## Motivation

`CaseDefinition` has identity (`namespace/name/version`) but no classification. casehub-desiredstate#59 needs to label response cases semantically — "this case is a situation-response of type replan" — so the platform can filter cases by purpose without requiring a new `CaseHub` subclass per type.

casehub-work already uses `Path`-based classification at the definition layer — `LabelDefinition.path` is typed `io.casehub.platform.api.path.Path` with a JPA `PathAttributeConverter`. The runtime instance layer (`WorkItemLabel.path`) stores a plain `String` for JPA embeddability. CaseDefinition is a definition-layer entity, so using typed `Path` is consistent with this layering: `Path` at definition time, string serialization at persistence/instance time.

## Platform Convention

Platform convention (validated by CaseDefinition as first adopter; #653 tracks second adopter validation):

> Every definable entity carries `types: Set<Path>` (what it IS — classification that may affect engine behavior such as routing, dispatch, and evaluation) and `labels: Set<Path>` (how it's organized — operational classification for queues, dashboards, and analytics). Both use `io.casehub.platform.api.path.Path` from `casehub-platform-api`.

| | `types` | `labels` |
|---|---|---|
| Answers | "What contracts does this implement?" | "How is this organized?" |
| Affects | Engine behavior — routing, dispatch, completion | Operations — queues, dashboards, analytics |
| Set by | Definition author at design time | Definition author at design time (instance-level runtime labels deferred — #656) |
| Example | `situation-response/replan`, `compliance/auditable` | `priority/high`, `team/infrastructure` |

Types use `implements` semantics (multi-valued) — a case definition can implement multiple behavioral contracts across orthogonal dimensions. Each `Path` provides vertical hierarchy (sub-typing via `isAncestorOf`); the set provides horizontal breadth.

First adopter: `CaseDefinition` (this issue). Second adopter: `WorkItem`/`WorkItemTemplate` (deferred — casehub-work issue).

## Design

### YAML Schema Changes

Changes to `schema/src/main/resources/schema/CaseDefinition.yaml`:

**Add `types`:**

```yaml
types:
  type: array
  description: >
    Behavioral type classifications for this case definition.
    Each entry is a hierarchical path string parsed via Path.parse().
    Types affect engine behavior — routing, dispatch, completion strategy.
  items:
    type: string
    minLength: 1
```

**Add `labels`** (new field — not a rename of `tags`, which was `type: object` with different semantics):

```yaml
labels:
  type: array
  description: >
    Operational classification labels for this case definition.
    Each entry is a hierarchical path string parsed via Path.parse().
    Labels affect organization — queues, dashboards, analytics.
  items:
    type: string
    minLength: 1
```

**Remove `tags`:** Dead code — `type: object` (key-value map), never mapped to the Java API, never consumed. Not a predecessor of `labels` (which is `type: array` of hierarchical path strings — different structure, different semantics).

**Remove `metadata`:** Dead code — never consumed by any runtime code. Also remove from the top-level `required: [ apiVersion, metadata, version, spec ]` array. (`apiVersion` is also absent from the properties list but is pre-existing — separate cleanup.)

### Java API — `CaseDefinition`

`CaseDefinition` currently uses a mutable pattern: constructor + setters + mutable `ArrayList` fields, with a builder that delegates to the same setters. The YAML mapper (`CaseDefinitionYamlMapper`) constructs via `new CaseDefinition(...)` and setters, not the builder. New fields follow this existing pattern for mapper compatibility.

New fields:

```java
private Set<Path> types = Set.of();    // never null, empty if none
private Set<Path> labels = Set.of();   // never null, empty if none
```

Setters (store unmodifiable copies, matching `setAgentDescriptors` pattern):

```java
public void setTypes(Set<Path> types) {
    this.types = types != null ? Set.copyOf(types) : Set.of();
}
public void setLabels(Set<Path> labels) {
    this.labels = labels != null ? Set.copyOf(labels) : Set.of();
}
```

Getters:

```java
public Set<Path> getTypes()   // returns unmodifiable set, never null
public Set<Path> getLabels()  // returns unmodifiable set, never null
```

Builder methods:

```java
.type(Path.of("situation-response", "replan"))
.type(Path.of("compliance", "auditable"))
.types(Set.of(Path.of("situation-response", "replan")))
.label(Path.of("priority", "high"))
.labels(Set.of(Path.of("priority", "high")))
```

Builder `build()` calls `setTypes()` / `setLabels()`, consistent with other fields.

### CaseDefinitionYamlMapper

Parses both fields from the generated schema model via `Path.parse()`, using the setter pattern consistent with the mapper's existing construction style:

```java
if (schemaModel.getTypes() != null) {
    def.setTypes(schemaModel.getTypes().stream()
        .map(Path::parse)
        .collect(Collectors.toCollection(LinkedHashSet::new)));
}
if (schemaModel.getLabels() != null) {
    def.setLabels(schemaModel.getLabels().stream()
        .map(Path::parse)
        .collect(Collectors.toCollection(LinkedHashSet::new)));
}
```

Invalid path strings fail at YAML load time — fail-fast via `Path.parse()`.

### CaseDefinitionRegistry Queries

Two new `default` methods (SPI evolution protocol — no breaking change):

```java
default List<CaseDefinition> findByType(Path type) {
    return List.of();
}

default List<CaseDefinition> findByLabel(Path label) {
    return List.of();
}
```

`DefaultCaseDefinitionRegistry` implementation — ancestor matching via `Path.isAncestorOf()`. Iterates the internal `Map<CaseKey, RegistryEntry> registry` (private `ConcurrentHashMap`):

```java
@Override
public List<CaseDefinition> findByType(Path type) {
    return registry.values().stream()
        .map(RegistryEntry::definition)
        .filter(def -> def.getTypes().stream()
            .anyMatch(t -> t.equals(type) || type.isAncestorOf(t)))
        .toList();
}
```

Same pattern for `findByLabel`. `findByType(Path.of("situation-response"))` returns all situation-response cases regardless of sub-type.

## Cleanup

- Remove `tags` property from YAML schema (dead code — not a predecessor of `labels`)
- Remove `metadata` property from YAML schema (dead code)
- Remove `metadata` from top-level `required` array
- Update `schema/src/main/resources/examples/document-processing.yaml` — remove vestigial Kubernetes-style fields (`apiVersion`, `kind`, `metadata`) and add required fields (`namespace`, `dsl`). This example uses dead schema fields and is likely non-functional today (mapper reads `schema.getName()` from top-level `name`, not `metadata.name`)
- Generated schema model regenerates automatically on next build

## Issue Reconciliation

Issue #652 lists `labels`, `tags`, and `categories`. This spec delivers `types` and `labels`:

- **`tags`** (YAML schema: `type: object`) — dead code. Never mapped to the Java API, never consumed by any runtime code. Removed as cleanup. `labels` is a new field (`type: array` of hierarchical path strings) — not a rename or successor of `tags`.
- **`categories`** — after analysis, the concept is captured by `types`: behavioral classification of what a case definition IS. The flat `tags` and vague `categories` from the issue are superseded by the typed `Path`-based `types` and `labels` distinction.
- **`types`** — new concept not in the original issue. Represents behavioral contracts that affect engine routing, dispatch, and evaluation. This is the semantic gap the issue was trying to fill: "so the platform can categorize cases by purpose."

Issue #652 should be updated to reflect this design.

## Deferred

- **casehub-work types/labels adoption (#653):** Add `types: Set<Path>` to `WorkItemTemplate` and `WorkItem` — second adopter of the platform convention.
- **CaseMetaModel persistence (#654):** The `CaseMetaModel` entity has a `JsonNode definition` column, but `DefaultCaseDefinitionRegistry.registerCaseDefinition()` never populates it — it is always null. Types and labels exist only in the in-memory `CaseDefinition` held in the registry's `ConcurrentHashMap`, rebuilt from YAML on every application restart. Populating the `definition` column during registration (for DB-level querying) is a separate concern.
- **Vocabulary validation (#655):** Types and labels are free-form `Path` values. Vocabulary enforcement against `LabelVocabulary` or an engine-level `TypeVocabulary` is a separate concern.
- **Instance-level types and labels (#656):** Types and labels on `CaseDefinition` are design-time only. Instance-level labels on `CaseInstance` (e.g. engine-assigned at runtime) are a separate concern, analogous to `WorkItemLabel` on `WorkItem`.

## Files Changed

| File | Change |
|------|--------|
| `schema/src/main/resources/schema/CaseDefinition.yaml` | Add `types` and `labels`, remove dead `tags` and `metadata`, remove `metadata` from `required` |
| `schema/src/main/resources/examples/document-processing.yaml` | Remove vestigial `apiVersion`/`kind`/`metadata`, add required `namespace`/`dsl` |
| `api/src/main/java/io/casehub/api/model/CaseDefinition.java` | Add `types`, `labels` fields + builder methods + accessors |
| `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java` | Parse `types` and `labels` via `Path.parse()` |
| `common/src/main/java/io/casehub/engine/common/spi/CaseDefinitionRegistry.java` | Add `findByType(Path)`, `findByLabel(Path)` default methods |
| `runtime/src/main/java/io/casehub/engine/internal/engine/DefaultCaseDefinitionRegistry.java` | Implement `findByType`, `findByLabel` with ancestor matching |
| `api/src/test/java/...` | Contract tests for types/labels on CaseDefinition |
| `runtime/src/test/java/...` | Registry query tests for ancestor matching |
| `api/src/test/java/.../CaseDefinitionYamlMapperTest.java` | YAML parsing tests for types and labels |
| PLATFORM.MD (parent repo) | Add types/labels platform convention |

## Platform Coherence

- Uses `io.casehub.platform.api.path.Path` from `casehub-platform-api` — the existing platform classification primitive
- Aligns with casehub-work's definition-layer `LabelDefinition` model (typed `Path` with `PathAttributeConverter`)
- Removes parallel scope/classification types per platform boundary rule
- Removes dead schema fields that were never consumed
