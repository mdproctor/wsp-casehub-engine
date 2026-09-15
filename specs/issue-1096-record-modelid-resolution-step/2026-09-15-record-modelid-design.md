# Record modelId in ResolutionStep.parameters()

**Issue:** engine#1096
**Scale:** S | **Complexity:** Low

## Problem

Blocks' `ModelPreferenceSignalProvider` scores agents by `(workerId, modelId)` compound key from CBR experience data, reading `modelId` from `ExperiencePlanStep.parameters()`. Currently, `CbrCaseRetainObserver.toResolutionStep()` always passes `Map.of()` for the parameters field, so no model ID is ever recorded.

## Design

### 1. Add `modelId` to `Agent`

`Agent` gains a nullable `String modelId` field, set via `AgentBuilder.modelId(String)`. This stores the declared model name (e.g. `claude-sonnet-4-20250514`) for later retrieval.

### 2. Wire in `AgentConverter`

`AgentConverter.toApiAgent()` reads `modelName` from the YAML agent node and passes it to `builder.modelId(modelName)`. This is the same `modelName` value already used to construct the `ChatModelProvider`.

### 3. Populate parameters in `CbrCaseRetainObserver`

`toResolutionStep()` looks up the worker by `executorName` from the `CaseDefinition` (already available in `doRetain()`). If the worker's function is an `AgentWorkerFunction`, extracts `agent().modelId()` and passes it in the parameters map.

```java
Map<String, Object> params = resolveParameters(record, definition);
// ...
return new ResolutionStep(
    record.bindingName(), ..., params, record.variantId());
```

Where `resolveParameters` does:
```java
private Map<String, Object> resolveParameters(PlanItemRecord record, CaseDefinition definition) {
    return definition.getWorkers().stream()
        .filter(w -> w.name().equals(record.executorName()))
        .findFirst()
        .filter(w -> w.function() instanceof AgentWorkerFunction)
        .map(w -> ((AgentWorkerFunction) w.function()).agent().modelId())
        .filter(java.util.Objects::nonNull)
        .map(id -> Map.<String, Object>of("modelId", id))
        .orElse(Map.of());
}
```

### 4. Data flow (unchanged downstream)

`ResolutionStep.parameters()` → `CbrRetrievalService.mapResolutionStep()` (already passes through `t.parameters()`) → `ExperiencePlanStep.parameters()` → `ModelPreferenceSignalProvider` reads it.

## Scope

- `Agent.java` — add `modelId` field + constructor param
- `AgentBuilder.java` — add `modelId(String)` setter, thread through `build()`
- `AgentConverter.java` — read `modelName` from YAML, set on builder
- `CbrCaseRetainObserver.java` — resolve modelId from definition, pass in parameters

## Out of scope

- A2A, MCP, ReAct worker types (no `Agent` object — separate issues)
- Dynamic model selection at runtime (declared model is sufficient for CBR correlation)
- `protocolMetadata` / EventLog threading (unnecessary given definition-time availability)

## References

- `CbrCaseRetainObserver.java:335-344` — recording site
- `CbrRetrievalService.java:714-725` — passthrough mapping
- `ModelPreferenceSignalProvider.java:63-70` — consumer in blocks
- `AgentConverter.java:34-70` — YAML agent construction
