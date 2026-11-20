# Design: Agent Worker Function Provider for Eidos Dispatch

**Issue:** casehubio/engine#1333
**Date:** 2026-11-20
**Branch:** `issue-1333-agent-worker-function-provider`

## Summary

Promote agent workers from an inline fallback in `YamlCaseDefinitionConverter` to a first-class `WorkerFunctionProvider` SPI participant, and add a new `AgentTaskTarget` binding type for capability-based agent dispatch.

## Motivation

The existing agent worker path is a special case — `YamlCaseDefinitionConverter.convertWorkers()` has a fallback that constructs `AgentWorkerFunction` inline when no provider handles the node. Every other worker type (flow, MCP, A2A, ReAct, pattern, playbook) goes through the `WorkerFunctionProvider` SPI. This inconsistency means:

- Agent workers don't participate in `discoverWorkers()` — no auto-discovery
- The converter has hardcoded knowledge of agent wiring that belongs in a provider
- No path for YAML authors to reference eidos-registered agents by slot — only inline `systemPrompt`/`model` config

Additionally, there's no `agent-task:` binding type — the only way to route work to agents is through direct `CapabilityTarget` bindings with agent-backed workers. A dedicated `AgentTaskTarget` enables capability-based agent matching via eidos.

## Design

### 1. Slot-Based Deferred Agent Resolution (Java API)

The core change: `AgentWorkerFunction` gains support for slot-based deferred resolution. Instead of requiring an eager `AgentProvider` at construction time, a new construction path accepts a slot reference. At execution time, the handler resolves the full invocation chain:

```
slot → AgentRegistry.findById(slot, tenancyId)
     → AgentDescriptor
     → SystemPromptRenderer.render(descriptor, context)
     → AgentProvider (from descriptor's provider config)
     → WorkerAgentInvoker
     → invocation
```

The `WorkerAgentInvoker` constructor already accepts all the pieces (`AgentProvider`, `systemPrompt`, `model`, `userMessageTemplate`, transformers). The new code constructs it at execution time from the resolved descriptor rather than at YAML parse time.

**`AgentWorkerFunction` structure change:** The existing record `AgentWorkerFunction(WorkerAgentInvoker invoker)` gains a new construction path. Two modes, distinguished by which fields are populated:

- **Eager mode** (existing): `invoker` is non-null, `slot` is null. Used by Java API when the caller supplies a pre-built invoker. Existing tests and code paths unchanged.
- **Deferred mode** (new): `slot` is non-null, `invoker` is null. The handler resolves the invoker at execution time from the slot. Used by `AgentWorkerFunctionProvider` and by Java API callers who want slot-based resolution.

A new `AgentSlotConfig` record holds the slot reference plus optional overrides (model, userMessageTemplate, inputProjection, outputProjection). `AgentWorkerFunction` becomes `AgentWorkerFunction(WorkerAgentInvoker invoker, AgentSlotConfig slotConfig)` with factory methods `AgentWorkerFunction.eager(invoker)` and `AgentWorkerFunction.ofSlot(slot)`.

**Optional overrides** (model, userMessage, inputProjection, outputProjection) are stored in `AgentSlotConfig` and applied on top of the descriptor-derived defaults at execution time.

### 2. AgentWorkerFunctionProvider

New class in `engine-support-core`, package `io.casehub.engine.agent`.

```java
public class AgentWorkerFunctionProvider implements WorkerFunctionProvider {

  @Override
  public boolean handles(JsonNode rawWorkerNode) {
    return rawWorkerNode.has("agent");
  }

  @Override
  public WorkerFunction<?, ?> create(JsonNode rawWorkerNode) {
    JsonNode agentNode = rawWorkerNode.get("agent");
    String slot = agentNode.get("slot").asText();
    // Parse optional overrides: model, userMessage, inputProjection, outputProjection
    return new AgentWorkerFunction(slot, overrides);
  }
}
```

**YAML surface:**

```yaml
workers:
  - name: sentiment-analyst
    agent:
      slot: "sentiment-analyst"
      model: "claude-sonnet-5-5"        # optional
      userMessage: |                     # optional
        Analyse the sentiment of: {{text}}
      inputProjection: ".ticket"         # optional JQ
      outputProjection: ".sentiment"     # optional JQ
    capabilities:
      - name: analyse-sentiment
        inputSchema: ".ticket.description"
        outputSchema: ".sentiment"
```

**Java equivalent:**

```java
Worker.builder()
    .name("sentiment-analyst")
    .function(AgentWorkerFunction.ofSlot("sentiment-analyst")
        .model("claude-sonnet-5-5")
        .userMessage("Analyse the sentiment of: {{text}}")
        .inputProjection(".ticket")
        .outputProjection(".sentiment")
        .build())
    .capability(Capability.builder()
        .name("analyse-sentiment")
        .inputSchema(".ticket.description")
        .outputSchema(".sentiment")
        .build())
    .build();
```

### 3. Execution-Time Resolution in AgentWorkerFunctionHandler

The existing `AgentWorkerFunctionHandler` already handles `AgentWorkerFunction`. It needs to be extended to support the slot-based path:

1. Check if the function has a slot reference (new path) vs. an eager invoker (existing path)
2. For slot-based: resolve via `AgentRegistry`, render system prompt, construct `WorkerAgentInvoker`, execute
3. For eager invoker: existing behaviour unchanged (backward compatibility during migration)

The handler needs `AgentRegistry`, `SystemPromptRenderer`, and `AgentProvider` injected. These are already available as CDI beans when eidos is on the classpath.

### 4. AgentTaskTarget — New BindingTarget Permit

New sealed permit on `BindingTarget` for capability-based agent task dispatch.

```java
public final class AgentTaskTarget implements BindingTarget {
    private final String capability;        // eidos capability name (required)
    private final String domain;            // optional domain filter
    private final ExpressionEvaluator inputMapping;
    private final ExpressionEvaluator outputMapping;
}
```

**YAML surface:**

```yaml
bindings:
  - name: analyse-ticket
    agent-task:
      capability: "sentiment-analysis"
      domain: "customer-support"
      inputMapping: ".ticket"
      outputMapping: ".analysis"
    on:
      contextChange: ".ticket"
```

**Java equivalent:**

```java
Binding.builder()
    .name("analyse-ticket")
    .target(AgentTaskTarget.builder()
        .capability("sentiment-analysis")
        .domain("customer-support")
        .inputMapping(".ticket")
        .outputMapping(".analysis")
        .build())
    .on(".ticket")
    .build();
```

**Dispatch resolution:** At dispatch time, the resolver queries:

```java
AgentQuery query = AgentQuery.builder()
    .capabilityName(target.capability())
    .domain(target.domain())
    .tenancyId(tenancyId)
    .build();
List<AgentMatch> candidates = agentRegistry.find(query);
// Default strategy: pick best match (first in list — ordered by match quality)
AgentMatch selected = candidates.get(0);
```

The pool infrastructure supports multi-candidate resolution; the default strategy picks the single best match. Future iterations can add other selection strategies.

### 5. Migration: Remove Inline Agent Fallback

Remove the fallback path in `YamlCaseDefinitionConverter.convertWorkers()` that handles `yw.agent() != null` when no provider matches. The `AgentWorkerFunctionProvider` now owns the `agent:` key through the SPI.

The `AgentConverter.toWorkerInvoker()` utility is retained — the provider delegates to it for constructing `WorkerAgentInvoker` internals.

Update `agent-worker-example.yaml` to use the new slot-based shape.

### 6. BindingExecutorResolver Update

Add `AgentTaskTarget` case to the exhaustive switch in `BindingExecutorResolver.resolve()`. Because `BindingTarget` is sealed, the compiler identifies every other switch site that needs updating.

## Module Placement

| Artifact | Module |
|----------|--------|
| `AgentWorkerFunctionProvider` | `engine-support-core` |
| `AgentWorkerFunction` changes | `api` |
| `AgentTaskTarget` | `api` |
| `AgentWorkerFunctionHandler` changes | `runtime` |
| `BindingExecutorResolver` changes | `common-core` |
| `YamlCaseDefinitionConverter` migration | `api` (or codegen-generated converter) |

## Error Handling

- **Missing slot at execution time:** `AgentRegistry.findById(slot, tenancyId)` returns `Optional.empty()` → `WorkerResult.failed("Agent not found for slot: " + slot)`. Not a runtime exception — the case continues with a failed worker result.
- **No matching agents for agent-task:** `AgentRegistry.find(query)` returns empty list → binding dispatch fails with a clear error identifying the unresolvable capability.
- **AgentProvider unavailable:** Existing `WorkerAgentInvoker` exception handling applies — `AgentException` wraps LLM invocation failures.

## Testing Strategy

- **Unit tests:** `AgentWorkerFunctionProviderTest` — YAML parsing, slot extraction, override handling
- **Unit tests:** `AgentTaskTargetTest` — builder validation, field access
- **Unit tests:** `AgentWorkerFunctionHandlerTest` — extend existing test for slot-based resolution path
- **Integration test:** End-to-end YAML case with `agent:` worker and `agent-task:` binding, using `InMemoryAgentRegistry` with test descriptors

## References

- `A2AWorkerFunctionProvider.java` — SPI pattern reference
- `McpWorkerFunctionProvider.java` — SPI pattern reference
- `FlowWorkerFunctionProvider.java` — SPI pattern reference
- `AgentWorkerFunction.java` — existing function record
- `WorkerAgentInvoker.java` — existing invoker
- `AgentWorkerFunctionHandler.java` — existing handler
- `AgentRegistry.java` (eidos-api) — slot/capability lookup
- `SystemPromptRenderer.java` (eidos-api) — prompt rendering
- `AgentProvider.java` (platform-agent-api) — LLM invocation
- `BindingTarget.java` — sealed interface for binding targets
- `HumanTaskTarget.java` — existing task target pattern
- `BindingExecutorResolver.java` — dispatch switch
- `YamlCaseDefinitionConverter` — inline fallback to remove
- casehubio/engine#1333 — issue
