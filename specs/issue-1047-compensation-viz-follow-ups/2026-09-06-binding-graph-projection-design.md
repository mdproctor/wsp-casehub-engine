# Binding Graph Projection — Design Spec

**Issue:** casehubio/engine#1047
**Branch:** issue-1047-compensation-viz-follow-ups
**Date:** 2026-09-06

## Problem

`CompensationGraphProjection` projects only compensation edges (`compensateRef` links) from bindings. A design-time viewer needs the full binding DAG — compensation edges, data flow edges (`produces`/`consumes` channels), and trigger dependency edges (which binding's output activates which downstream binding).

The binding model has an asymmetry: `producedKeys` declares what a binding writes to the context, but nothing declares what a binding reads. This prevents reliable trigger dependency inference without fragile JQ expression parsing.

## Design

### 1. Add `requiredKeys` to Binding (engine-api)

New field on `Binding`: `requiredKeys: Set<String>` — the symmetric counterpart to `producedKeys`. Declares which context keys must be present for this binding to fire meaningfully.

```java
// Binding.java
private Set<String> requiredKeys;

public void setRequiredKeys(Set<String> requiredKeys) { ... }
public Set<String> getRequiredKeys() { ... }
```

Builder support:
```java
// Binding.Builder
public Builder requiredKeys(Set<String> requiredKeys) { ... }
```

YAML:
```yaml
bindings:
  - name: fraud-check
    requiredKeys: [entityResolution, transactionAmount]
    producedKeys: [fraudScore]
```

`CaseDefinitionYamlMapper` parses `requiredKeys:` array alongside `producedKeys:`. Same pattern — `Set<String>` from YAML string array.

### 2. Generalize graph types (engine-graphql)

**Delete:**
- `CompensationGraphProjection`
- `CompensationGraphType`
- `CompensationNodeType`
- `CompensationEdgeType`

**Create:**

```java
// EdgeKind.java
public enum EdgeKind {
    COMPENSATION,
    DATA_FLOW,
    TRIGGER_DEPENDENCY
}
```

```java
// BindingNodeType.java
@Type("BindingNode")
public record BindingNodeType(
    String bindingName,
    String targetType,
    boolean compensationOnly) {}
```

```java
// BindingEdgeType.java
@Type("BindingEdge")
public record BindingEdgeType(
    String sourceBinding,
    String targetBinding,
    EdgeKind edgeType,
    String label) {}
```

`label` is nullable metadata for the edge:
- COMPENSATION: null (the relationship is implicit in the type)
- DATA_FLOW: channel name (e.g. `"orders-channel"`)
- TRIGGER_DEPENDENCY: key names (e.g. `"entityResolution, amount"`)

```java
// BindingGraphType.java
@Type("BindingGraph")
public record BindingGraphType(
    List<BindingNodeType> nodes,
    List<BindingEdgeType> edges,
    List<String> compensationGaps) {}
```

`compensationGaps` retains the existing gap detection — non-compensation bindings without a compensating binding.

### 3. BindingGraphProjection (engine-graphql)

Static utility replacing `CompensationGraphProjection`:

```java
public final class BindingGraphProjection {

    public static BindingGraphType project(List<Binding> bindings) {
        List<BindingNodeType> nodes = new ArrayList<>();
        List<BindingEdgeType> edges = new ArrayList<>();
        List<String> compensationGaps = new ArrayList<>();

        // Build lookup maps
        Map<String, Binding> byName = ...;
        Map<String, String> channelProducers = ...; // channelName → bindingName
        Map<String, Set<String>> producedKeysIndex = ...; // key → bindingNames

        for (Binding b : bindings) {
            // Nodes
            nodes.add(new BindingNodeType(
                b.getName(), targetTypeName(b.target()), b.isCompensation()));

            // Compensation edges
            if (b.getCompensateRef() != null) {
                edges.add(new BindingEdgeType(
                    b.getName(), b.getCompensateRef(),
                    EdgeKind.COMPENSATION, null));
            } else if (!b.isCompensation()) {
                compensationGaps.add(b.getName());
            }

            // Data flow edges (consumer side)
            if (b.getConsumes() != null) {
                String producer = channelProducers.get(b.getConsumes());
                if (producer != null) {
                    edges.add(new BindingEdgeType(
                        producer, b.getName(),
                        EdgeKind.DATA_FLOW, b.getConsumes()));
                }
            }

            // Track producers for data flow
            if (b.getProduces() != null) {
                channelProducers.put(b.getProduces(), b.getName());
            }

            // Track producedKeys for trigger dependency
            if (b.getProducedKeys() != null) {
                for (String key : b.getProducedKeys()) {
                    producedKeysIndex
                        .computeIfAbsent(key, k -> new LinkedHashSet<>())
                        .add(b.getName());
                }
            }
        }

        // Trigger dependency edges (second pass — need full producedKeysIndex)
        for (Binding b : bindings) {
            if (b.getRequiredKeys() == null || b.getRequiredKeys().isEmpty()) continue;
            for (String requiredKey : b.getRequiredKeys()) {
                Set<String> producers = producedKeysIndex.get(requiredKey);
                if (producers != null) {
                    for (String producer : producers) {
                        if (!producer.equals(b.getName())) {
                            edges.add(new BindingEdgeType(
                                producer, b.getName(),
                                EdgeKind.TRIGGER_DEPENDENCY, requiredKey));
                        }
                    }
                }
            }
        }

        return new BindingGraphType(nodes, edges, compensationGaps);
    }
}
```

Two-pass design: first pass builds nodes and indices; second pass resolves trigger dependencies. Data flow and compensation edges are computed in the first pass since they don't need the full index.

Note: when multiple `requiredKeys` match multiple `producedKeys` from the same producer, this produces one edge per key. The viewer can deduplicate or merge labels as needed. This is intentional — per-key granularity lets the viewer show exactly which data dependency links two bindings.

### 4. CaseDefinitionType update

```java
@Type("CaseDefinitionResponse")
public record CaseDefinitionType(
    String namespace, String name, String version,
    String title, String summary,
    List<String> capabilities,
    BindingGraphType bindingGraph) {  // was: CompensationGraphType compensationGraph

    public static CaseDefinitionType from(CaseDefinition def) {
        // ... existing capability extraction ...
        BindingGraphType graph = def.getBindings() != null && !def.getBindings().isEmpty()
            ? BindingGraphProjection.project(def.getBindings())
            : null;
        return new CaseDefinitionType(..., graph);
    }
}
```

### 5. Unchanged

- `compensationTimeline` query — runtime PlanItem + EventLog query, not design-time graph
- `compensationChain` query — ledger causal chain, not design-time graph
- `CaseCompensationService` — runtime saga coordinator
- All compensation EventLog types

### 6. Schema changes

Update `schema/src/main/resources/schema/CaseDefinition.yaml` to include `requiredKeys` property on binding definitions, same pattern as `producedKeys`:

```yaml
requiredKeys:
  type: array
  items:
    type: string
  description: Context keys this binding requires to fire meaningfully
```

## Test plan

1. **BindingGraphProjection unit tests:**
   - Compensation edges — same as existing tests (full coverage, gaps, compensation-only bindings)
   - Data flow edges — bindings with matching produces/consumes channels
   - Data flow with no matching consumer — no edge, no error
   - Trigger dependency edges — bindings with matching producedKeys/requiredKeys
   - Trigger dependency self-reference — filtered (no self-edges)
   - Mixed edge types — all three in one graph
   - Empty bindings — empty graph
   - Target type mapping — all BindingTarget subtypes

2. **Binding.requiredKeys:**
   - Builder round-trip
   - Null/empty handling
   - YAML parsing via CaseDefinitionYamlMapper

## References

- `CompensationGraphProjection.java` — existing projection (graphql module)
- `Binding.java:64` — `producedKeys` field (api module)
- `GoapAction` — effects/preconditions model (engine-api, `io.casehub.engine.plan`)
- `CaseDefinitionYamlMapper` — YAML parsing for binding properties
- casehubio/engine#1047 — issue
- casehubio/work#238 — saga compensation epic (parent)
- casehubio/work#390 — compensation visualization (design-time + runtime)
