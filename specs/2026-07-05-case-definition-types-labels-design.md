# Semantic Types and Labels on CaseDefinition

**Issue:** casehubio/engine#652
**Date:** 2026-07-05
**Status:** Approved

## Summary

Add `types: Set<Path>` and `labels: Set<Path>` to `CaseDefinition`, establishing a platform convention for classifying definable entities. Remove vestigial `tags` (object) and `metadata` (object) from the YAML schema — both are dead code from the original Serverless Workflow scaffold, never mapped to the Java API, never consumed by any code or consumer YAML definition.

## Motivation

`CaseDefinition` has identity (`namespace/name/version`) but no classification. casehub-desiredstate#59 needs to label response cases semantically — "this case is a situation-response of type replan" — so the platform can filter cases by purpose without requiring a new `CaseHub` subclass per type.

casehub-work already uses `Path`-based labels on WorkItems (`WorkItemLabel.path`, `LabelDefinition.path`). CaseDefinition should use the same platform primitive for classification, establishing a consistent pattern.

## Platform Convention

New platform boundary rule:

> Every definable entity carries `types: Set<Path>` (what it IS — behavioral contracts that affect engine routing, dispatch, and evaluation) and `labels: Set<Path>` (how it's organized — operational classification that affects queues, dashboards, and analytics). Both use `Path` from `casehub-platform-api`.

| | `types` | `labels` |
|---|---|---|
| Answers | "What contracts does this implement?" | "How is this organized?" |
| Affects | Engine behavior — routing, dispatch, completion | Operations — queues, dashboards, analytics |
| Set by | Definition author at design time | Definition author or engine at runtime |
| Example | `situation-response/replan`, `compliance/auditable` | `priority/high`, `team/infrastructure` |

Types use `implements` semantics (multi-valued) — a case definition can implement multiple behavioral contracts across orthogonal dimensions. Each `Path` provides vertical hierarchy (sub-typing via `isAncestorOf`); the set provides horizontal breadth.

First adopter: `CaseDefinition` (this issue). Second adopter: `WorkItem`/`WorkItemTemplate` (deferred — casehub-work issue).

## Design

### YAML Schema Changes

Three changes to `schema/src/main/resources/schema/CaseDefinition.yaml`:

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

**Rename `tags` → `labels`, change from object to array:**

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

**Remove `metadata`:** Dead weight — never consumed by any code or consumer.

### Java API — `CaseDefinition`

New fields (immutable, builder-constructed):

```java
private Set<Path> types;    // unmodifiable, never null, empty if none
private Set<Path> labels;   // unmodifiable, never null, empty if none
```

Builder methods:

```java
.type(Path.of("situation-response", "replan"))
.type(Path.of("compliance", "auditable"))
.types(Set.of(Path.of("situation-response", "replan")))
.label(Path.of("priority", "high"))
.labels(Set.of(Path.of("priority", "high")))
```

Accessors:

```java
public Set<Path> getTypes()   // unmodifiable, never null
public Set<Path> getLabels()  // unmodifiable, never null
```

No setters — builder-only, consistent with `capabilities`, `workers`, `bindings`.

### CaseDefinitionYamlMapper

Parses both fields from the generated schema model via `Path.parse()`:

```java
if (schemaModel.getTypes() != null) {
    schemaModel.getTypes().forEach(t -> builder.type(Path.parse(t)));
}
if (schemaModel.getLabels() != null) {
    schemaModel.getLabels().forEach(l -> builder.label(Path.parse(l)));
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

`DefaultCaseDefinitionRegistry` implementation — ancestor matching via `Path.isAncestorOf()`:

```java
@Override
public List<CaseDefinition> findByType(Path type) {
    return registeredDefinitions().stream()
        .filter(def -> def.getTypes().stream()
            .anyMatch(t -> t.equals(type) || type.isAncestorOf(t)))
        .toList();
}
```

Same pattern for `findByLabel`. `findByType(Path.of("situation-response"))` returns all situation-response cases regardless of sub-type.

## Cleanup

- Remove `tags` property from YAML schema (replaced by `labels`)
- Remove `metadata` property from YAML schema (dead code)
- Generated schema model regenerates automatically on next build

## Deferred

- **casehub-work:** Add `types: Set<Path>` to `WorkItemTemplate` and `WorkItem` — second adopter of the platform convention. Separate issue.
- **CaseMetaModel persistence:** Types/labels are persisted implicitly in the `definition` JsonNode column. Explicit columns for direct DB-level querying deferred until needed.
- **Vocabulary validation:** Types and labels are free-form `Path` values. Vocabulary enforcement against `LabelVocabulary` is a separate concern if needed later.
- **Runtime mutation:** Types and labels are design-time on `CaseDefinition`, not instance-level on `CaseInstance`.

## Files Changed

| File | Change |
|------|--------|
| `schema/src/main/resources/schema/CaseDefinition.yaml` | Add `types`, rename `tags` → `labels` (object → array), remove `metadata` |
| `api/src/main/java/io/casehub/api/model/CaseDefinition.java` | Add `types`, `labels` fields + builder methods + accessors |
| `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java` | Parse `types` and `labels` via `Path.parse()` |
| `common/src/main/java/io/casehub/engine/common/spi/CaseDefinitionRegistry.java` | Add `findByType(Path)`, `findByLabel(Path)` default methods |
| `runtime/src/main/java/io/casehub/engine/internal/engine/DefaultCaseDefinitionRegistry.java` | Implement `findByType`, `findByLabel` with ancestor matching |
| `api/src/test/java/...` | Contract tests for types/labels on CaseDefinition |
| `runtime/src/test/java/...` | Registry query tests for ancestor matching |
| `api/src/test/java/.../CaseDefinitionYamlMapperTest.java` | YAML parsing tests for types and labels |
| PLATFORM.MD (parent repo) | Add types/labels platform convention |

## Platform Coherence

- Uses `Path` from `casehub-platform-api` — the existing platform classification primitive
- Aligns with casehub-work's `WorkItemLabel` model (also `Path`-based)
- Removes parallel scope/classification types per platform boundary rule
- Removes dead schema fields that were never consumed
