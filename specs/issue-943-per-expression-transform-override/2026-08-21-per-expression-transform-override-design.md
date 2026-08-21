# Per-Expression Override for Data Transform Projections — Design Spec

**Issue:** engine#943
**Date:** 2026-08-21

---

## Problem

Data transform projections (`inputProjection`, `outputProjection`, `inputProjectionOverride`) are
hardcoded to JQ throughout the engine. Boolean conditions were unified with per-expression language
override in engine#925 — the same `{lang: expr}` YAML map syntax and `ExpressionEngineRegistry`
dispatch. But projections bypass the registry entirely: runtime call sites create `JqTransformer`
or call `jqEvaluator.eval()` directly.

The SPI infrastructure is already in place:
- `ExpressionEngine.transform(evaluator, json) → List<JsonNode>` (default throws UnsupportedOp)
- `ExpressionEngineRegistry.transform(evaluator, json) → List<JsonNode>` (dispatch by type)
- `JQExpressionEngine.transform()` (JQ implementation)
- `MvelExpressionEngine` does not override `transform()` — correct for v1

The gap is threefold: (1) model types carry projections as raw Strings with no language type,
(2) runtime call sites bypass the registry, (3) the YAML mapper doesn't parse projection fields
through `resolveExpression()`.

---

## What Is Not Changing

- `Capability` (foundation tier, `worker-api`) — `inputSchema` and `outputSchema` remain Strings.
  Foundation tier has no dependency on `platform-api` where `ExpressionEvaluator` lives.
- `ExpressionEngine`, `ExpressionEngineRegistry`, `DefaultExpressionEngineRegistry` — already
  have `transform()` methods. No changes needed.
- `JQExpressionEngine.transform()` — already implemented. No changes needed.
- `resolveExpression()` on `CaseDefinitionYamlMapper` — already handles `{lang: expr}` map syntax.
  Reused for projections without modification.

---

## Design

### 1. CapabilityTarget — carry resolved projection evaluators

```java
public record CapabilityTarget(
    Capability capability,
    ExpressionEvaluator inputProjection,
    ExpressionEvaluator outputProjection
) implements BindingTarget {

    public CapabilityTarget(Capability capability) {
        this(capability,
             capability.inputSchema() != null
                 ? new JQExpressionEvaluator(capability.inputSchema()) : null,
             capability.outputSchema() != null
                 ? new JQExpressionEvaluator(capability.outputSchema()) : null);
    }
}
```

The 1-arg convenience constructor wraps Capability Strings in `JQExpressionEvaluator` — same
pattern as `Binding.when(String)`. Java DSL callers that only use JQ continue unchanged.
`CaseDefinitionYamlMapper` uses the 3-arg constructor with evaluators from `resolveExpression()`.

---

### 2. Binding — inputProjectionOverride becomes ExpressionEvaluator

```java
// Before:
private String inputProjectionOverride;
public String effectiveInputProjection(Capability capability) {
    return inputProjectionOverride != null ? inputProjectionOverride : capability.inputSchema();
}

// After:
private ExpressionEvaluator inputProjectionOverride;
public ExpressionEvaluator effectiveInputProjection(CapabilityTarget capTarget) {
    return inputProjectionOverride != null ? inputProjectionOverride : capTarget.inputProjection();
}
```

`effectiveInputProjection()` now takes `CapabilityTarget` (which carries the resolved evaluator)
instead of `Capability` (which carries the raw String). The return type is `ExpressionEvaluator`.

Builder convenience: `inputProjectionOverride(String)` wraps in `JQExpressionEvaluator` for
backward compat. New overload `inputProjectionOverride(ExpressionEvaluator)` for explicit
language control.

---

### 3. WorkerScheduleEvent — carry ExpressionEvaluator

`inputProjectionOverride` field changes from `String` to `ExpressionEvaluator`.
`effectiveInputProjection()` returns `ExpressionEvaluator` instead of `String`.

Construction sites that currently pass a String projection override are updated to pass the
resolved `ExpressionEvaluator` from the `Binding`. Construction sites that pass `null` continue
passing `null`.

---

### 4. SubCaseMapping.Expression — carry ExpressionEvaluator

```java
// Before:
public record Expression(String expression) implements SubCaseMapping { ... }

// After:
public record Expression(ExpressionEvaluator evaluator) implements SubCaseMapping { ... }
```

Pre-release platform — the sealed record change is the right design. The YAML mapper creates
the evaluator via `resolveExpression()`. Java DSL callers use `SubCaseMapping.of(String)` which
wraps in `JQExpressionEvaluator`.

---

### 5. CaseDefinitionYamlMapper — parse projection fields through resolveExpression

Projection fields in the YAML that currently read as Strings are parsed through
`resolveExpression()` to support the `{lang: expr}` map syntax:

| YAML field | Parse site | Current | After |
|-----------|-----------|---------|-------|
| `capabilities[].inputProjection` | `convertToApiModel` | `getString()` → `Capability(String)` | `resolveExpression(rawNode)` → `CapabilityTarget(cap, evaluator, evaluator)` |
| `capabilities[].outputProjection` | `convertToApiModel` | `getString()` → `Capability(String)` | same |
| `bindings[].inputProjectionOverride` | `convertBinding` | `getString()` → `builder.inputProjectionOverride(String)` | `resolveExpression(rawBindingNode)` → `builder.inputProjectionOverride(ExpressionEvaluator)` |
| `agent.inputProjection` | `AgentConverter.toApiAgent` | `getString()` → `builder.inputProjection(String)` | see section 6 |
| `agent.outputProjection` | `AgentConverter.toApiAgent` | `getString()` → `builder.outputProjection(String)` | see section 6 |
| `subCase.inputMapping` | `convertSubCase` | `getString()` → `Expression(String)` | `resolveExpression(rawNode)` → `Expression(ExpressionEvaluator)` |
| `subCase.outputMapping` | `convertSubCase` | `getString()` → `Expression(String)` | `resolveExpression(rawNode)` → `Expression(ExpressionEvaluator)` |

**YAML syntax** (same as #925 for conditions):
```yaml
# Default (JQ, same as today)
inputProjection: ".transaction"

# Per-expression override
inputProjection: { mvel: "transaction" }

# Explicit JQ
inputProjection: { jq: ".transaction" }
```

Raw `JsonNode` parsing is required (not the generated schema model getter) because the YAML
value may be a map. The existing pattern from `when:` parsing applies: read `rawBindingNode.get("inputProjectionOverride")` and pass to `resolveExpression()`.

---

### 6. AgentConverter — pass registry, resolve projections

`AgentConverter.toApiAgent()` gains an `ExpressionEngineRegistry` parameter and the raw agent
`JsonNode`:

```java
// Before:
public static Agent toApiAgent(Agent schemaAgent) { ... }

// After:
public static Agent toApiAgent(
    Agent schemaAgent, JsonNode rawAgentNode,
    ExpressionEngineRegistry registry, String expressionLang) { ... }
```

Inside, projection fields are resolved via `resolveExpression()` and wrapped into
`UnaryOperator<JsonNode>` lambdas:

```java
ExpressionEvaluator inputEval = resolveExpression(
    rawAgentNode != null ? rawAgentNode.get("inputProjection") : null,
    registry, expressionLang);
if (inputEval != null) {
    builder.inputTransformer(input -> {
        List<JsonNode> result = registry.transform(inputEval, input);
        return result.isEmpty() ? input : result.get(0);
    });
}
```

The Agent stays a pure execution unit — it receives a `UnaryOperator<JsonNode>`, never sees
the expression engine. `AgentBuilder.inputProjection(String)` remains for Java DSL callers
(JQ-only convenience).

The no-CDI `JQ_ONLY` fallback in `CaseDefinitionYamlMapper` handles the registry-less path
for unit tests that call `load(is)` directly. Its `transform()` delegates to `JQEvaluator`
(in `common`) directly — NOT `JQExpressionEngine` (in `runtime`), which would create an
upward api→runtime dependency.

---

### 7. Runtime call sites — route through registry.transform()

Every `evalJqAsJsonNode` / `evalJqAsMap` call site is replaced with `registry.transform()`.
Full enumeration:

| # | Call site | Current | After |
|---|----------|---------|-------|
| 1 | `WorkerScheduleEventHandler:114` | `evalJqAsJsonNode(ctx, event.effectiveInputProjection())` | `transformSingle(event.effectiveInputProjection(), ctx)` |
| 2 | `CaseContextChangedEventHandler:877` | `evalJqAsMap(ctx, inputMapping)` for CapabilityTarget | `transformAsMap(evaluator, ctx)` |
| 3 | `CaseContextChangedEventHandler:976` | `evalJqAsMap(ctx, expr)` for SubCase input | `transformAsMap(subCaseMapping.evaluator(), ctx)` |
| 4 | `CaseContextChangedEventHandler` | `instanceof JQExpressionEvaluator` for HumanTask input | `transformAsMap(target.inputMapping(), ctx)` (registry dispatch, no instanceof) |
| 5 | `DefaultWorkOrchestrator:219` | `evalJqAsMap(ctx, capability.inputSchema())` | Receives `CapabilityTarget`, calls `transformAsMap(capTarget.inputProjection(), ctx)` |
| 6 | `DefaultPersistentScope:89` | `jqEvaluator.eval(inputProjection, snapshot)` | Receives `ExpressionEvaluator`, calls `registry.transform(evaluator, snapshot)` |
| 7 | `PersistentWorkerFunctionHandler:101` | `binding.effectiveInputProjection(cap)` → String | `binding.effectiveInputProjection(capTarget)` → `ExpressionEvaluator` |
| 8 | `AgentBuilder:157-162` | `new JqTransformer(inputProjection)::apply` | Already handled by section 6 (converter wraps) |
| 9 | `CaseContextChangedEventHandler.tryProvision()` | inline `evalJqAsMap(ctx, effectiveProjection)` | `transformAsMap(evaluator, ctx)` using binding's `effectiveInputProjection(capTarget)` |
| 10 | `DefaultPersistentScope.emit()` | `jqEvaluator.eval(outputSchema, outputNode)` | Receives `ExpressionEvaluator` for output, calls `registry.transform(evaluator, outputNode)` |
| 11 | `PersistentWorkerFunctionHandler` output | `ct.capability().outputSchema()` → String | `capTarget.outputProjection()` → ExpressionEvaluator, threaded to `DefaultPersistentScope` |
| 12 | `QuartzWorkerExecutionJob:218` output | `capability.outputSchema()` → String | Resolved from `CapabilityTarget.outputProjection()` via EventLog metadata, passed to executor |

**DefaultWorkOrchestrator access to CapabilityTarget:** `doSubmitWork()` receives `CaseDefinition`
and looks up `Capability` by name — it has no Binding or CapabilityTarget. Solution: look up
the binding from `CaseDefinition.getBindings()` by capability name, get the CapabilityTarget
from the binding's target, and use `capTarget.inputProjection()`. If no binding matches (direct
capability submission), fall back to wrapping `capability.inputSchema()` in `JQExpressionEvaluator`.

**Output projection threading:** `PersistentWorkerFunctionHandler` and `QuartzWorkerExecutionJob`
currently read `capability.outputSchema()` as a String. After this change, they read
`capTarget.outputProjection()` as an `ExpressionEvaluator`. For `QuartzWorkerExecutionJob`, the
evaluator type is stored in EventLog metadata alongside the expression (same pattern as
`contextBridgeType` for ContextBridge). `PersistentWorkerFunctionHandler` receives the evaluator
from the binding's CapabilityTarget directly.

**Deferred:** `exchangeProjectionExpression` on Binding — this is part of the Exchange/DataChannel
subsystem (engine#633) with its own `ExchangeProjectionStrategy` dispatch mechanism. Converting
it to ExpressionEvaluator is a separate concern. Filed as a follow-up issue.

**Helper methods** replace `evalJqAsJsonNode` / `evalJqAsMap` on each handler:

```java
private JsonNode transformSingle(ExpressionEvaluator evaluator, JsonNode input) {
    if (evaluator == null) return input;
    try {
        List<JsonNode> result = registry.transform(evaluator, input);
        return result.isEmpty() ? input : result.get(0);
    } catch (Exception e) {
        LOG.warnf(e, "transform failed for expression '%s'", evaluator);
        return input;
    }
}

private Map<String, Object> transformAsMap(ExpressionEvaluator evaluator, JsonNode input) {
    if (evaluator == null) return Map.of();
    try {
        List<JsonNode> result = registry.transform(evaluator, input);
        if (result.isEmpty()) return Map.of();
        return OBJECT_MAPPER.convertValue(result.get(0), MAP_TYPE);
    } catch (Exception e) {
        LOG.warnf(e, "transform failed for expression '%s'", evaluator);
        return Map.of();
    }
}
```

**Error handling preserved:** The `try/catch` with `LOG.warnf` + fallback matches the existing
`evalJqAsJsonNode` / `evalJqAsMap` behavior. Projection failures produce a warning and graceful
degradation — never crash the case. This is intentional: a failing projection should not prevent
case progression.

**`List<JsonNode>` → single result:** The `result.get(0)` reduction is explicit in the helpers.
JQ can produce multiple outputs; other languages typically produce one. The single-result contract
is documented in the helper methods.

---

### 8. JQ_ONLY fallback — extend for transforms

The existing `JQ_ONLY` anonymous `ExpressionEngineRegistry` in `CaseDefinitionYamlMapper` needs
a `transform()` implementation for the no-CDI path:

```java
@Override
public List<JsonNode> transform(ExpressionEvaluator evaluator, JsonNode input) {
    if (!JQExpressionEvaluator.TYPE.equals(evaluator.type())) {
        throw new IllegalArgumentException(
            "Only 'jq' supported without CDI. Got: " + evaluator.type());
    }
    // Use JQEvaluator (common module) directly — NOT JQExpressionEngine (runtime)
    JQExpressionEvaluator jqEval = (JQExpressionEvaluator) evaluator;
    ValidationResult vr = new JQEvaluator().eval(jqEval.expression(), input);
    return vr.ok() && vr.output() != null ? vr.output() : List.of(input);
}
```

---

## Blast Radius

| Layer | Files affected | Nature of change |
|-------|---------------|-----------------|
| `api` (model types) | `CapabilityTarget`, `Binding`, `SubCaseMapping` | String → ExpressionEvaluator |
| `api` (converter) | `AgentConverter`, `CaseDefinitionYamlMapper` | Pass registry, use resolveExpression |
| `common` (events) | `WorkerScheduleEvent` | String → ExpressionEvaluator field |
| `runtime` (handlers) | `WorkerScheduleEventHandler`, `CaseContextChangedEventHandler`, `DefaultWorkOrchestrator`, `PersistentWorkerFunctionHandler`, `DefaultPersistentScope` | evalJq → registry.transform |
| `runtime` (tests) | Existing projection tests | Update to use evaluator types |

Foundation tier (`worker-api`) is **not affected**. `scheduler-quartz` (`QuartzWorkerExecutionJob`)
is affected for output projection threading. Persistence: `SubCaseCompletionService` stores
expression Strings — restore path wraps in `JQExpressionEvaluator` (default convention, same
as `SubCaseMapping.of(String)`).

---

## Testing

**`CaseDefinitionYamlMapperTest`** — add:
- inputProjection plain string → creates JQExpressionEvaluator via resolveExpression
- inputProjection `{mvel: "expr"}` → creates TypedMvelExpressionEvaluator
- outputProjection same two variants
- inputProjectionOverride same two variants
- SubCase inputMapping/outputMapping same two variants
- Agent inputProjection/outputProjection same two variants

**`CapabilityTargetTest`** — add:
- 1-arg constructor wraps Capability strings in JQExpressionEvaluator
- 3-arg constructor preserves provided evaluators
- null inputSchema → null inputProjection evaluator

**`BindingTest`** — add:
- effectiveInputProjection returns override evaluator when present
- effectiveInputProjection returns CapabilityTarget evaluator when no override
- Builder inputProjectionOverride(String) wraps in JQExpressionEvaluator

**`WorkerScheduleEventHandler` integration tests** — update existing projection tests to verify
registry dispatch (not direct JQ evaluation). Add test with non-JQ evaluator type.

**`DefaultPersistentScope` tests** — add:
- Input projection via registry.transform (not direct jqEvaluator.eval)
- Output projection via registry.transform in emit()

**`DefaultWorkOrchestrator` tests** — add:
- Input projection resolves through CapabilityTarget, not capability.inputSchema()
- Fallback when no binding matches (wraps in JQExpressionEvaluator)

**`SubCaseMapping` persistence round-trip** — verify String restore wraps in JQExpressionEvaluator

**`DefaultExpressionEngineRegistryTest`** — existing `transform()` tests should already pass.
Add test with null evaluator returning identity.

---

## References

- [ExpressionEngine.java:154](api/src/main/java/io/casehub/api/engine/ExpressionEngine.java) — transform() SPI method
- [ExpressionEngineRegistry.java:110](api/src/main/java/io/casehub/api/engine/ExpressionEngineRegistry.java) — transform() registry dispatch
- [CaseDefinitionYamlMapper.java:1039](api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java) — resolveExpression() utility
- [2026-06-09-expression-evaluator-factory-design.md](docs/specs/2026-06-09-expression-evaluator-factory-design.md) — original expression factory design
- [GE-20260609-3c86d1] — ADR-0009 superseded by #925; per-expression override scope
- [GE-20260704-d6aacc] — ExpressionEngineRegistry polymorphic dispatch pattern
- engine#925 — per-expression override for boolean conditions (closed, foundation for this work)
- engine#289 — expression evaluator factory (closed, created the registry infra)
