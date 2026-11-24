## D1: FlowWorkerFunction execution model

**Choice:** In-process synchronous execution — `FlowWorkerFunctionHandler` already exists and handles this
**Alternatives:**
- Async via scheduler — more infrastructure overhead, inconsistent with the `do:` block mental model of small imperative fragments
- Both (detect from block) — premature complexity; async can be added later if needed
**Rationale:** `FlowWorkerFunctionHandler` already exists in `engine-support-core`, supports `SWFWorkerFunction` (sealed parent of `FlowWorkerFunction`), and executes via the Serverless Workflow SDK runtime (`WorkflowApplication`). No new handler needed.
**Trade-offs:** None — the handler already works. The gap is in parse-time wiring, not execution.
**Sources:** `FlowWorkerFunctionHandler.java` (engine-support-core), `WorkflowApplication` / `WorkflowInstance` (serverlessworkflow-impl-core)
**Exploration:** quick
**Status:** captured

## D2: Fix parse→dispatch wiring for do: blocks

**Choice:** Diagnose and fix the gap where `FlowWorkerFunctionProvider.create()` output doesn't reach the `Worker` record's function field at parse time
**Alternatives:**
- Create a new handler — unnecessary, `FlowWorkerFunctionHandler` already exists
- Flow-specific `@WorkerBackend` — overengineered, the existing scheduler backends + handler chain already work
**Rationale:** The handler infrastructure works. The error "No backend supports capability" means `canExecute(worker.function())` returns false, which happens when `worker.function()` is `WorkerFunction.NONE`. The fix is in the parse path: ensure `FlowWorkerFunctionProvider` is a CDI bean discovered by the registry, and that `convertWorkers()` Phase 2 correctly invokes it.
**Trade-offs:** None — this is a wiring fix, not an architectural choice.
**Sources:** `YamlCaseDefinitionConverter.convertWorkers()` Phase 2, `DefaultWorkerFunctionProviderRegistry.createFunction()`, `FlowWorkerFunctionProvider` CDI wiring in `FlowBeans`
**Exploration:** quick
**Status:** captured

## D3: AgentWorkerFunctionProvider coverage — widen existing provider

**Choice:** Extend `AgentWorkerFunctionProvider.handles()` to match all `agent:` block variants (slot, capability, domain), not just `slot:`
**Alternatives:**
- Separate providers per variant (`AgentCapabilityFunctionProvider`, `AgentDomainFunctionProvider`) — more classes with shared logic, the variants are small
**Rationale:** The handler (`AgentWorkerFunctionHandler`) already supports all resolution modes (eager, slot-based, capability-based via `DeferredAgentConfig` sealed variants). The provider is simply too narrow — it only matches `has("agent") && has("slot")`. Widening it and creating `AgentWorkerFunction.ofCapability()` with a `DeferredAgentConfig.CapabilityConfig` closes the gap.
**Trade-offs:** Slightly more complex `handles()`/`create()` logic in one class. Acceptable — the variants share structure.
**Sources:** `AgentWorkerFunctionProvider.java` (engine-support-core), `AgentWorkerFunctionHandler.java` (runtime), `DeferredAgentConfig.SlotConfig` / `DeferredAgentConfig.CapabilityConfig`
**Exploration:** quick
**Status:** captured

## D4: Auto-create CaseHub beans via CaseHubRegistry

**Choice:** New `CaseHubRegistry` (`@ApplicationScoped`) that holds named `CaseHub` instances created during YAML registration
**Alternatives:**
- Add `getCaseHub()` to `CaseDefinitionRegistry` — mixes concerns (definitions vs runtime API), `CaseHub` needs CDI-injected `CaseHubRuntime`
- Quarkus `BeanRegistrar` synthetic beans — requires build-time classpath scanning, duplicates `CaseYamlRegistrar` logic, harder to test
**Rationale:** Clean, testable, no dynamic CDI. The registry is the CDI bean; individual CaseHubs are looked up by name. Explicit `YamlCaseHub` subclasses with `augment()` still work as CDI beans via `Instance<CaseHub>`. Since `CaseHub` requires `@Inject CaseHubRuntime` (field injection), the registry-managed instances need `CaseHubRuntime` passed explicitly rather than via CDI.
**Trade-offs:** Consumers of auto-discovered cases inject `CaseHubRegistry` instead of a typed `CaseHub` bean. Acceptable — the YAML-only path is for cases that don't need type-safe injection.
**Sources:** `CaseYamlRegistrar.java` (runtime), `CaseHub.java` / `YamlCaseHub.java` (api), `CaseHubRuntime.java` (api), `DefaultCaseDefinitionRegistry.registerKnownDefinitions()` startup path
**Exploration:** quick
**Status:** captured
