# Agent-Task Runtime Dispatch Handler

**Issue:** casehubio/engine#1339
**Follow-up from:** casehubio/engine#1333 (agent-task binding model)
**Prerequisite context:** casehubio/engine#1338 (YAML parser — not yet implemented but design is independent)

## Problem

`AgentTaskTarget` is wired into the sealed `BindingTarget` hierarchy and `BindingExecutorResolver` returns `ExecutorRef.of(capability)` for it — a name stub. But the execution pipeline (`WorkerExecutionOrchestrator`) looks up workers by name from `CaseDefinition.getWorkers()`, and no Worker exists for an agent-task binding. The agent lives in the external `AgentRegistry`, discovered at runtime via capability matching.

## Approach: Synthetic Worker

When a binding has an `AgentTaskTarget`, create a synthetic `Worker` with an `AgentWorkerFunction` at CaseDefinition load time. The synthetic Worker uses an extended `AgentSlotConfig` carrying capability + domain (instead of slot). At dispatch time, `AgentWorkerFunctionHandler` queries `AgentRegistry.find(byCapabilityAndDomain(...))`, selects the best match, renders the system prompt, and executes.

This follows the existing slot-based deferred resolution pattern — same handler, different query path.

## Component Changes

### 1. AgentTaskTarget — add matchStrategy

Add a `matchStrategy` field (String, nullable). Default: `"first"`. Known values: `"first"`, `"best-domain"`, `"round-robin"`.

```java
public final class AgentTaskTarget implements BindingTarget {
  private final String capability;          // existing
  private final String domain;              // existing
  private final ExpressionEvaluator inputMapping;   // existing
  private final ExpressionEvaluator outputMapping;  // existing
  private final String matchStrategy;       // NEW — nullable, default "first"
}
```

The handler interprets this string at dispatch time. Invalid values produce a descriptive error at execution.

### 2. AgentSlotConfig — extend with capability/domain and evaluators

Add optional fields for the capability-based path. Exactly one of `slot` or `capability` must be non-null (validation invariant).

```java
public record AgentSlotConfig(
    String slot,                    // existing — null for capability path
    String capability,              // NEW — null for slot path
    String domain,                  // NEW — optional
    String matchStrategy,           // NEW — nullable, default "first"
    String model,                   // existing
    String userMessageTemplate,     // existing
    String inputProjection,         // existing (JQ string, slot path)
    String outputProjection,        // existing (JQ string, slot path)
    ExpressionEvaluator inputEvaluator,   // NEW — capability path
    ExpressionEvaluator outputEvaluator   // NEW — capability path
) {
  public AgentSlotConfig {
    if (slot == null && capability == null) {
      throw new IllegalArgumentException("either slot or capability is required");
    }
    if (slot != null && capability != null) {
      throw new IllegalArgumentException("slot and capability are mutually exclusive");
    }
  }

  public boolean isCapabilityBased() {
    return capability != null;
  }
}
```

Existing factory method `AgentSlotConfig(slot, model, userMessage, inputProj, outputProj)` stays — callers pass nulls for new fields via a new factory or builder.

### 3. AgentWorkerFunctionHandler — capability-based deferred resolution

Add `executeByCapability()` alongside existing `executeDeferred()`. The `execute()` method branches:

```java
@Override
public HandlerResult execute(Worker worker, Capability capability,
                             Object inputData, WorkerContext context,
                             ExecutionMetadata metadata) {
  AgentWorkerFunction agent = (AgentWorkerFunction) worker.function();
  if (agent.isDeferred()) {
    AgentSlotConfig config = agent.slotConfig();
    if (config.isCapabilityBased()) {
      return executeByCapability(config, worker, inputData, metadata);
    }
    return executeDeferred(config, worker, inputData, metadata);
  }
  return executeWithInvoker(agent.invoker(), worker, inputData);
}
```

`executeByCapability()` flow:

1. Build query: `AgentQuery.byCapabilityAndDomain(config.capability(), config.domain(), metadata.tenancyId())`
2. Query: `agentRegistry.find(query)`
3. If empty → return `WorkerResult.failed("No agent found for capability: " + config.capability())`
4. Select match using `config.matchStrategy()`:
   - `"first"` (default/null) → `matches.get(0)`
   - `"best-domain"` → first match where `descriptor.domain()` equals `config.domain()`; fall back to `matches.get(0)` if none
   - `"round-robin"` → stateless round-robin via atomic counter mod matches.size()
5. Render: `systemPromptRenderer.render(descriptor, AgentPromptContext.forFormat(MARKDOWN))`
6. Build transformers from evaluators (not JQ strings):
   - `inputEvaluator != null` → `node -> evaluator.evaluate(node)` (adapt to `UnaryOperator<JsonNode>`)
   - Otherwise → `UnaryOperator.identity()`
7. Build `WorkerAgentInvoker` with rendered prompt, model, transformers
8. Delegate to `executeWithInvoker()`

### 4. BindingExecutorResolver — resolve to synthetic worker name

The resolve method needs the binding name for the synthetic worker name. Update signature to accept the binding:

```java
public static ExecutorRef resolve(Binding binding, CaseDefinition definition) {
  return switch (binding.target()) {
    // ...existing cases...
    case AgentTaskTarget at -> ExecutorRef.of("__agent-task__" + binding.getName());
    // ...
  };
}
```

Using the binding name (unique within a CaseDefinition) avoids collisions when multiple bindings target the same capability with different domains or mappings.

### 5. Synthetic Worker creation

A utility method (can be static on a helper class or in the converter):

```java
static Worker createSyntheticAgentTaskWorker(String bindingName, AgentTaskTarget target) {
  AgentSlotConfig config = new AgentSlotConfig(
      null,                           // slot — null
      target.capability(),            // capability
      target.domain(),                // domain
      target.matchStrategy(),         // matchStrategy
      null,                           // model — resolved from descriptor
      null,                           // userMessageTemplate — from descriptor
      null,                           // inputProjection — not used
      null,                           // outputProjection — not used
      target.inputMapping(),          // inputEvaluator
      target.outputMapping()          // outputEvaluator
  );

  return Worker.builder()
      .name(syntheticWorkerName(bindingName))
      .description("Synthetic agent-task worker for capability: " + target.capability())
      .function(AgentWorkerFunction.ofSlot(config))
      .build();
}
```

Called from the CaseDefinition converter when a binding's target is `AgentTaskTarget`. The synthetic Worker is added to `CaseDefinition.getWorkers()`.

## Data Flow

```
YAML: binding with agent-task target
  → CaseDefinition converter: resolveTarget creates AgentTaskTarget
  → Converter: createSyntheticAgentTaskWorker(target) → adds Worker to definition
  → BindingExecutorResolver: resolves to "__agent-task__<bindingName>"
  → Planning loop dispatches → WorkerTaskData with workerId
  → WorkerExecutionOrchestrator: finds synthetic Worker by name
  → DefaultWorkerExecutor → AgentWorkerFunctionHandler.supports(AgentWorkerFunction) ✓
  → executeByCapability():
      AgentRegistry.find(byCapabilityAndDomain(cap, domain, tenancyId))
      → select match (matchStrategy)
      → SystemPromptRenderer.render(descriptor)
      → build WorkerAgentInvoker with evaluator-based transformers
      → executeWithInvoker() → HandlerResult
```

## Error Handling

| Condition | Behaviour |
|-----------|-----------|
| No agents match capability | `WorkerResult.failed("No agent found for capability: X")` |
| AgentRegistry unavailable | Handler throws, retried via `RetryHandler` in orchestrator |
| System prompt render fails | Handler throws, retried |
| Match strategy unknown | `WorkerResult.failed("Unknown match strategy: X")` |
| Evaluator transform fails | Handler wraps in `AgentException`, retried |

## Testing Strategy

- **Unit: AgentSlotConfig validation** — slot/capability mutual exclusion, isCapabilityBased()
- **Unit: AgentTaskTarget.matchStrategy** — builder defaults, explicit values
- **Unit: BindingExecutorResolver** — AgentTaskTarget resolves to synthetic worker name
- **Unit: AgentWorkerFunctionHandler.executeByCapability()** — mock AgentRegistry, verify query params, match selection per strategy, evaluator-based transformers
- **Unit: Synthetic worker creation** — config carries correct values, name is deterministic
- **Integration: End-to-end dispatch** — CaseDefinition with agent-task binding dispatches through the full pipeline

## Scope Boundary

**In scope:**
- Runtime dispatch handler (query, select, render, invoke)
- AgentTaskTarget.matchStrategy field
- AgentSlotConfig extension
- Synthetic Worker creation utility
- BindingExecutorResolver update
- Unit and integration tests

**Out of scope:**
- YAML parser for agent-task binding target (#1338 — separate issue)
- Match strategy SPI (future — if string enum proves insufficient)
- Multi-agent fan-out from a single binding (future)

## References

- `AgentTaskTarget.java:22-97` — existing binding target class
- `BindingExecutorResolver.java:50` — current stub for AgentTaskTarget
- `AgentWorkerFunctionHandler.java:102-146` — existing deferred slot resolution path
- `AgentSlotConfig.java:20-29` — existing config record
- `WorkerExecutionOrchestrator.java:215-219` — worker lookup by name
- `AgentRegistry.java:6-47` — registry interface with find(AgentQuery)
- `AgentQuery.java:32-34` — byCapabilityAndDomain factory
- `AgentMatch.java:14` — match result with descriptor and resolvedCapability
- `SystemPromptRenderer.java:3` — renderer interface
- `DefaultWorkerExecutor.java:48-105` — composite executor dispatching to handlers
