## D1: FlowWorkerFunction execution model

**Choice:** In-process synchronous execution via a new `FlowWorkerFunctionHandler`
**Alternatives:**
- Async via scheduler — more infrastructure overhead, inconsistent with the `do:` block mental model of small imperative fragments
- Both (detect from block) — premature complexity; async can be added later if needed
**Rationale:** `do:` blocks represent small imperative workflow fragments (e.g. `call: http`). Synchronous in-process execution matches the mental model, avoids scheduler overhead, and slots into the existing `WorkerFunctionHandler` SPI without new infrastructure.
**Trade-offs:** Long-running `do:` blocks won't benefit from scheduler persistence/retry. Acceptable — users can use explicit worker bindings for long-running work.
**Sources:** `FlowWorkerFunctionProvider.java` (engine-support-core), `WorkerFunctionHandler.java` (common-core), `QuartzWorkerExecutionManager.canExecute()` dispatch path
**Exploration:** quick
**Status:** captured

## D2: FlowWorkerFunctionHandler — new handler class

**Choice:** Create a separate `FlowWorkerFunctionHandler` implementing `WorkerFunctionHandler`
**Alternatives:**
- Extend `FlowWorkerFunctionProvider` to also implement `WorkerFunctionHandler` — conflates parse-time and execution-time responsibilities, different dependency sets
- Flow-specific `@WorkerBackend` — overengineered for sync execution, adds routing complexity to `CompositeWorkerExecutionManager`
**Rationale:** The handler SPI exists precisely for this. Adding a handler is the minimal correct extension point. The scheduler backends' `canExecute()` will discover it automatically via `Instance<WorkerFunctionHandler>`.
**Trade-offs:** One more class, but clean separation of parse vs execute.
**Sources:** `AgentWorkerFunctionHandler.java` (existing handler pattern), `WorkerFunctionHandler.java` SPI, `FirstSupportedRoutingStrategy.select()` dispatch chain
**Exploration:** quick
**Status:** captured

## D3: AgentWorkerFunctionProvider coverage — widen existing provider

**Choice:** Extend `AgentWorkerFunctionProvider.handles()` to match all `agent:` block variants (slot, capability, domain), not just `slot:`
**Alternatives:**
- Separate providers per variant (`AgentCapabilityFunctionProvider`, `AgentDomainFunctionProvider`) — more classes with shared logic, the variants are small
**Rationale:** The handler (`AgentWorkerFunctionHandler`) already supports all three resolution modes (eager, slot-based, capability-based). The provider is simply too narrow — it only matches `has("agent") && has("slot")`. Widening it closes the gap with minimal change.
**Trade-offs:** Slightly more complex `handles()`/`create()` logic in one class. Acceptable — the variants share structure.
**Sources:** `AgentWorkerFunctionProvider.java` (engine-support-core), `AgentWorkerFunctionHandler.java` (runtime), `DeferredAgentConfig` variants
**Exploration:** quick
**Status:** captured

## D4: Auto-create CaseHub beans via CaseHubRegistry

**Choice:** New `CaseHubRegistry` (`@ApplicationScoped`) that holds named `CaseHub` instances created by `CaseYamlRegistrar`
**Alternatives:**
- Add `getCaseHub()` to `CaseDefinitionRegistry` — mixes concerns (definitions vs runtime API), `CaseHub` needs CDI-injected `CaseHubRuntime`
- Quarkus `BeanRegistrar` synthetic beans — requires build-time classpath scanning, duplicates `CaseYamlRegistrar` logic, harder to test
**Rationale:** Clean, testable, no dynamic CDI. The registry is the CDI bean; individual CaseHubs are looked up by name. Explicit `YamlCaseHub` subclasses with `augment()` still work as CDI beans via `Instance<CaseHub>`.
**Trade-offs:** Consumers of auto-discovered cases inject `CaseHubRegistry` instead of a typed `CaseHub` bean. Acceptable — the YAML-only path is for cases that don't need type-safe injection.
**Sources:** `CaseYamlRegistrar.java` (runtime), `CaseDefinitionRegistry.java` (common-core), `YamlCaseHub.java` (api), `DefaultCaseDefinitionRegistry.registerKnownDefinitions()` startup path
**Exploration:** quick
**Status:** captured
