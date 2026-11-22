# Agent-Task Runtime Dispatch Handler

**Issue:** casehubio/engine#1339
**Follow-up from:** casehubio/engine#1333 (agent-task binding model)
**Prerequisite context:** casehubio/engine#1338 (YAML parser — not yet implemented but design is independent)

## Problem

`AgentTaskTarget` is wired into the sealed `BindingTarget` hierarchy and `BindingExecutorResolver` returns `ExecutorRef.of(capability)` for it — a name stub. But the execution pipeline (`WorkerExecutionOrchestrator`) looks up workers by name from `CaseDefinition.getWorkers()`, and no Worker exists for an agent-task binding. The agent lives in the external `AgentRegistry`, discovered at runtime via capability matching.

## Approach: Synthetic Worker with Platform Selection

When a binding has an `AgentTaskTarget`, create a synthetic `Worker` with an `AgentWorkerFunction` at CaseDefinition load time. The synthetic Worker uses a sealed `DeferredAgentConfig` hierarchy — specifically `CapabilityConfig` — carrying capability, domain, evaluators, and model override. At dispatch time, `AgentWorkerFunctionHandler` queries `AgentRegistry.find(byCapabilityAndDomain(...))`, delegates to the platform's `AgentSelector` for health-aware, trust-informed selection, renders the system prompt, and executes.

## Component Changes

### 1. AgentTaskTarget — add model field

Add an optional `model` field for binding-level model override (parity with slot-based mode).

```java
public final class AgentTaskTarget implements BindingTarget {
  private final String capability;          // existing
  private final String domain;              // existing
  private final ExpressionEvaluator inputMapping;   // existing
  private final ExpressionEvaluator outputMapping;  // existing
  private final String model;               // NEW — nullable
}
```

No `matchStrategy` field — selection delegates to `AgentSelector` SPI.

### 2. Sealed DeferredAgentConfig hierarchy

Replace `AgentSlotConfig` as the deferred config type with a sealed interface:

```java
public sealed interface DeferredAgentConfig
    permits DeferredAgentConfig.SlotConfig, DeferredAgentConfig.CapabilityConfig {

  record SlotConfig(
      String slot,
      String model,
      String userMessageTemplate,
      String inputProjection,
      String outputProjection
  ) implements DeferredAgentConfig {
    public SlotConfig {
      Objects.requireNonNull(slot, "slot is required");
    }
  }

  record CapabilityConfig(
      String capability,
      String domain,
      String model,
      ExpressionEvaluator inputEvaluator,
      ExpressionEvaluator outputEvaluator
  ) implements DeferredAgentConfig {
    public CapabilityConfig {
      Objects.requireNonNull(capability, "capability is required");
    }
  }
}
```

Each subtype carries only its relevant fields. `SlotConfig` preserves the existing `AgentSlotConfig` contract (all fields, `requireNonNull(slot)`). `CapabilityConfig` carries capability, domain, model override, and evaluators.

### 3. AgentWorkerFunction — deferredConfig()

Replace `slotConfig()` with `deferredConfig()`:

```java
public record AgentWorkerFunction(WorkerAgentInvoker invoker, DeferredAgentConfig deferredConfig)
    implements WorkerFunction<Map<String, Object>, Map<String, Object>> {

  public static AgentWorkerFunction eager(WorkerAgentInvoker invoker) { ... }
  public static AgentWorkerFunction ofSlot(DeferredAgentConfig.SlotConfig config) { ... }
  public static AgentWorkerFunction ofCapability(DeferredAgentConfig.CapabilityConfig config) { ... }

  public boolean isDeferred() { return deferredConfig != null; }
}
```

### 4. AgentWorkerFunctionHandler — capability-based dispatch

The handler gains two new injected dependencies: `AgentSelector` and `ExpressionEngineRegistry`. The execute method pattern-matches on the config type:

```java
@Override
public HandlerResult execute(Worker worker, Capability capability,
                             Object inputData, WorkerContext context,
                             ExecutionMetadata metadata) {
  AgentWorkerFunction agent = (AgentWorkerFunction) worker.function();
  if (agent.isDeferred()) {
    return switch (agent.deferredConfig()) {
      case DeferredAgentConfig.SlotConfig sc -> executeBySlot(sc, worker, inputData, metadata);
      case DeferredAgentConfig.CapabilityConfig cc -> executeByCapability(cc, worker, inputData, metadata);
    };
  }
  return executeWithInvoker(agent.invoker(), worker, inputData);
}
```

`executeByCapability()` flow:

1. **Query**: `AgentRegistry.find(AgentQuery.byCapabilityAndDomain(cc.capability(), cc.domain(), metadata.tenancyId()))`
2. **Select**: `AgentSelector.select(matches, SelectionContext.of(metadata.tenancyId(), cc.capability(), cc.domain()))`
3. **Handle selection result**:
   - `AgentSelection.Selected` → proceed with selected agent descriptor
   - `AgentSelection.NoneQualified` → return `WorkerResult.failed(reason)`
   - `AgentSelection.Escalated` → return `WorkerResult.failed("Escalated: " + reason)` (escalation handling is a future concern for the case engine layer)
4. **Render**: `systemPromptRenderer.render(descriptor, AgentPromptContext.forFormat(MARKDOWN))`
5. **Build transformers** using `ExpressionEngineRegistry`:
   - `cc.inputEvaluator() != null` → `node -> registry.transform(cc.inputEvaluator(), node).get(0)`
   - Otherwise → `UnaryOperator.identity()`
   - Same pattern for output evaluator
6. **Build invoker**: `new WorkerAgentInvoker(agentProvider, rendered.content(), null, inputTransformer, outputTransformer, null, effectiveModel, null)` where `effectiveModel` is `cc.model()` if non-null, else descriptor's model
7. **Delegate**: `executeWithInvoker(invoker, worker, inputData)`

### 5. BindingExecutorResolver — resolve to synthetic worker name

```java
case AgentTaskTarget at -> ExecutorRef.of("__agent-task__" + binding.getName());
```

The resolver already takes `Binding` (which has `getName()`), so no signature change needed.

### 6. Synthetic Worker creation

Public static utility:

```java
public static Worker createSyntheticAgentTaskWorker(String bindingName, AgentTaskTarget target) {
  var config = new DeferredAgentConfig.CapabilityConfig(
      target.capability(),
      target.domain(),
      target.model(),
      target.inputMapping(),
      target.outputMapping()
  );

  return Worker.builder()
      .name("__agent-task__" + bindingName)
      .description("Synthetic agent-task worker for capability: " + target.capability())
      .function(AgentWorkerFunction.ofCapability(config))
      .build();
}
```

Called from the CaseDefinition converter when a binding's target is `AgentTaskTarget`. The converter validates no user-authored Worker uses the `__agent-task__` prefix, failing fast on collision.

### 7. Naming collision protection

The converter validates after all workers and synthetic workers are created:

```java
for (Worker w : definition.getWorkers()) {
  if (w.name().startsWith("__agent-task__") && !syntheticWorkerNames.contains(w.name())) {
    throw new CaseDefinitionException(
        "Worker name '" + w.name() + "' uses reserved prefix '__agent-task__'");
  }
}
```

## Data Flow

```
YAML: binding with agent-task target
  → CaseDefinition converter: resolveTarget creates AgentTaskTarget
  → Converter: createSyntheticAgentTaskWorker(bindingName, target) → adds Worker to definition
  → Converter: validates no user Worker name collision
  → BindingExecutorResolver: resolves to "__agent-task__<bindingName>"
  → Planning loop dispatches → WorkerTaskData with workerId
  → WorkerExecutionOrchestrator: finds synthetic Worker by name
  → DefaultWorkerExecutor → AgentWorkerFunctionHandler.supports(AgentWorkerFunction) ✓
  → executeByCapability():
      AgentRegistry.find(byCapabilityAndDomain(cap, domain, tenancyId))
      → AgentSelector.select(matches, SelectionContext) — health, trust, routing
      → AgentSelection.Selected → descriptor
      → SystemPromptRenderer.render(descriptor)
      → ExpressionEngineRegistry.transform() for input/output evaluators
      → build WorkerAgentInvoker with model override
      → executeWithInvoker() → HandlerResult
```

## Error Handling

| Condition | Behaviour |
|-----------|-----------|
| No agents match capability | `AgentSelector` returns `NoneQualified` → `WorkerResult.failed(reason)` |
| All agents unhealthy | `AgentSelector` returns `NoneQualified` → `WorkerResult.failed(reason)` |
| Escalation triggered | `AgentSelector` returns `Escalated` → `WorkerResult.failed("Escalated: " + reason)` |
| AgentRegistry unavailable | Handler throws, retried via `RetryHandler` in orchestrator |
| System prompt render fails | Handler throws, retried |
| Evaluator transform fails | Handler wraps in `AgentException`, retried |
| Synthetic worker name collision | Converter throws `CaseDefinitionException` at load time |

## Testing Strategy

- **Unit: DeferredAgentConfig hierarchy** — SlotConfig validation, CapabilityConfig validation, sealed exhaustiveness
- **Unit: AgentWorkerFunction** — `ofSlot()`, `ofCapability()`, `isDeferred()`, `deferredConfig()` pattern match
- **Unit: BindingExecutorResolver** — AgentTaskTarget resolves to `__agent-task__<bindingName>`
- **Unit: AgentWorkerFunctionHandler.executeByCapability()** — mock AgentRegistry + AgentSelector, verify:
  - Correct query params passed to registry
  - SelectionContext built correctly
  - Selected → renders prompt, builds invoker with evaluator transforms and model override
  - NoneQualified → returns failed WorkerResult
  - Escalated → returns failed WorkerResult
- **Unit: Evaluator-based transforms** — ExpressionEngineRegistry.transform() called with correct evaluator, identity when evaluator is null
- **Unit: Synthetic worker creation** — config carries correct values, name is deterministic, collision validation
- **Unit: AgentTaskTarget.model** — builder with/without model
- **Integration: End-to-end dispatch** — CaseDefinition with agent-task binding dispatches through the full pipeline with InMemoryAgentRegistry

## Scope Boundary

**In scope:**
- Runtime dispatch handler (query, select via AgentSelector, render, invoke)
- DeferredAgentConfig sealed hierarchy (SlotConfig, CapabilityConfig)
- AgentWorkerFunction refactor (slotConfig → deferredConfig)
- AgentTaskTarget.model field
- Synthetic Worker creation utility (public static)
- BindingExecutorResolver update
- Naming collision protection
- ExpressionEngineRegistry injection for evaluator-based transforms
- Unit and integration tests

**Out of scope:**
- YAML parser for agent-task binding target (#1338 — separate issue)
- Escalation handling in the case engine layer (future — currently returns failed result)
- Multi-agent fan-out from a single binding (future)

## References

- `AgentTaskTarget.java:22-97` — existing binding target class
- `BindingExecutorResolver.java:50` — current stub for AgentTaskTarget
- `AgentWorkerFunctionHandler.java:62-199` — existing handler with slot-based deferred resolution
- `AgentSlotConfig.java:20-29` — existing config record (becomes SlotConfig)
- `WorkerExecutionOrchestrator.java:215-219` — worker lookup by name
- `AgentRegistry.java:6-47` — registry interface with find(AgentQuery)
- `AgentQuery.java:32-34` — byCapabilityAndDomain factory
- `AgentMatch.java:14` — match result with descriptor and resolvedCapability
- `AgentSelector.java:5-7` — selection SPI
- `EngineAwareAgentSelector.java:42-113` — production implementation with health + trust + routing
- `AgentSelection.java:3-22` — sealed selection result (Selected, NoneQualified, Escalated)
- `SelectionContext.java:5-21` — selection context record
- `AgentRoutingStrategy.java:46-61` — CDI-extensible routing strategy SPI
- `SystemPromptRenderer.java:3` — renderer interface
- `ExpressionEngineRegistry` — platform expression evaluation
- `DefaultWorkerExecutor.java:48-105` — composite executor dispatching to handlers
- `BindingTarget.java:27-34` — sealed interface precedent for DeferredAgentConfig pattern
