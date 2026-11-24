# Worker Backend Registration and CaseHub Auto-Creation

**Issues:** casehubio/engine#1360, #1361, #1362
**Branch:** `issue-1360-flow-worker-function-prov`
**Date:** 2026-11-24

## Problem

Three gaps prevent YAML-defined case definitions from being fully functional without boilerplate Java code:

1. **#1360 — Flow workers:** Workers with `do:` blocks get "No backend supports capability" at dispatch time. `FlowWorkerFunctionProvider` creates a `FlowWorkerFunction` from `do:` blocks, and `FlowWorkerFunctionHandler` can execute it — but the function doesn't reach the `Worker` record, so `canExecute(worker.function())` fails.

2. **#1361 — Agent workers:** Workers with `agent:` blocks (capability-based or domain-based) get the same error. `AgentWorkerFunctionProvider` only matches `agent: { slot: ... }` — capability-based and domain-based variants fall through to `WorkerFunction.NONE`.

3. **#1362 — CaseHub beans:** `CaseYamlRegistrar` registers definitions into `CaseDefinitionRegistry` but doesn't create `CaseHub` beans. The runtime API (`startCase()`, `signal()`, `query()`) is unavailable for auto-discovered cases without an explicit Java subclass.

All three block casehubio/examples#212.

## Architecture Context

### Dispatch Chain

```
Parse-time:
  CaseYamlRegistrar.register()
    → CaseDefinitionYamlMapper.load(stream, mapper, registry, providerRegistry, agentProvider)
      → YamlCaseDefinitionConverter.convertWorkers()
        Phase 1: Worker records built with WorkerFunction.NONE
        Phase 2: WorkerFunctionProviderRegistry.createFunction(rawWorkerNode)
          → first provider where handles() returns true → create() → WorkerFunction
          → stored on Worker record

Dispatch-time:
  Binding fires (CapabilityTarget)
    → AgentCandidateFactory.buildCandidates() — matches workers by capability name
    → AgentRoutingStrategy.select() — picks one worker
    → WorkerScheduleEventHandler.handle()
      → CompositeWorkerExecutionManager.submit()
        → FirstSupportedRoutingStrategy.select()
          → backend.supports(capName, tenancyId)   [both return true]
          → backend.canExecute(worker.function())   [delegates to WorkerFunctionHandler.supports()]
        → "No backend supports capability" if no backend matches
```

### Existing Infrastructure

| Component | Status | Location |
|-----------|--------|----------|
| `FlowWorkerFunctionProvider` | Exists (parse-time) | `engine-support-core` |
| `FlowWorkerFunctionHandler` | Exists (execution-time) | `engine-support-core` |
| `AgentWorkerFunctionProvider` | Exists but narrow (slot-only) | `engine-support-core` |
| `AgentWorkerFunctionHandler` | Exists (all modes) | `runtime` |
| `DeferredAgentConfig.CapabilityConfig` | Exists | `api` |
| `AgentWorkerFunction.ofCapability()` | Exists | `api` |
| `CaseHubRegistry` | Missing | — |

## Design

### #1360: Fix Flow Worker Parse→Dispatch Wiring

**Root cause:** `FlowWorkerFunctionProvider` creates a valid `FlowWorkerFunction`, but the function doesn't reach the `Worker` record. This happens when:
- The provider isn't discovered by CDI (it's a plain POJO, not `@ApplicationScoped` — must be produced in `FlowBeans`)
- Or `convertWorkers()` Phase 2 receives a null provider registry (the simple `load(InputStream)` overload uses `EMPTY_PROVIDERS`)

**Fix:**
1. Verify `FlowBeans` produces `FlowWorkerFunctionProvider` as a CDI bean (if not, add a `@Produces` method)
2. Verify `CaseYamlRegistrar.register()` calls the `load()` overload that passes the real `WorkerFunctionProviderRegistry`
3. Trace through `convertWorkers()` Phase 2 to confirm `FlowWorkerFunctionProvider.handles()` returns true for `do:` block nodes and `create()` returns a valid `FlowWorkerFunction`
4. Add an integration test: define a worker with `do: call: http` and `capabilities: [notify]`, trigger the binding, verify execution reaches `FlowWorkerFunctionHandler`

**No new classes needed.** The handler and provider exist. This is a wiring diagnosis and fix.

### #1361: Widen AgentWorkerFunctionProvider

**Root cause:** `AgentWorkerFunctionProvider.handles()` checks `rawWorkerNode.has("agent") && rawWorkerNode.get("agent").has("slot")`. Workers with capability-based or domain-based agent references don't match.

**Fix:** Widen `handles()` and `create()` to support all `agent:` block variants:

```
handles(rawWorkerNode):
  return rawWorkerNode.has("agent")
  // No longer requires "slot" — any agent: block matches

create(rawWorkerNode):
  agentNode = rawWorkerNode.get("agent")
  if agentNode.has("slot"):
    return AgentWorkerFunction.ofSlot(new SlotConfig(slot, model, userMessage, inputProj, outputProj))
  if agentNode.has("capability"):
    return AgentWorkerFunction.ofCapability(new CapabilityConfig(capability, domain, model, inputEval, outputEval))
  throw IllegalArgumentException("agent: block requires slot: or capability:")
```

**New YAML structures supported:**

```yaml
workers:
  - name: classifier
    capabilities: [classify-ticket]
    agent:
      capability: text-classification
      domain: support
      model: gpt-4o

  - name: summariser
    capabilities: [summarise-ticket]
    agent:
      slot: summariser-agent
      model: claude-sonnet
```

The `AgentWorkerFunctionHandler` already handles both modes via pattern matching on `DeferredAgentConfig`:
- `SlotConfig` → `executeBySlot()` (lookup by slot name)
- `CapabilityConfig` → `executeByCapability()` (lookup by capability + domain, then `AgentSelector.select()`)

**Expression evaluator wiring:** `CapabilityConfig` takes `ExpressionEvaluator` for input/output transforms. When parsing from YAML, resolve evaluators via the `ExpressionEngineRegistry` injected into the provider (or accept raw expression strings and resolve at execution time).

### #1362: CaseHubRegistry for Auto-Discovered Cases

**New class:** `CaseHubRegistry` in `runtime` module.

```
@ApplicationScoped
public class CaseHubRegistry {
    private final Map<String, CaseHub> hubs = new ConcurrentHashMap<>();

    void register(String name, CaseHub hub)
    Optional<CaseHub> get(String name)
    Collection<CaseHub> all()
}
```

**Registration flow:**
1. `DefaultCaseDefinitionRegistry.registerCaseDefinition(definition)` runs as today
2. After successful registration, it checks `CaseHubRegistry` — if no CaseHub exists for this definition's name, it auto-creates one and registers it
3. `registerKnownDefinitions()` (CDI `CaseHub` beans at startup) also registers each explicit CaseHub into the registry
4. `CaseHubRegistry` becomes the universal lookup for all CaseHub instances, regardless of origin (YAML, annotations, programmatic)

**CaseHub wrapper:** Since `CaseHub` is an abstract class with `@Inject CaseHubRuntime runtime` (field injection), a non-CDI-managed subclass can't use field injection. Options:

**Option A — Constructor-wired inner class:**
```java
class RegistryCaseHub extends CaseHub {
    private final CaseDefinition definition;
    private final CaseHubRuntime runtime;

    RegistryCaseHub(CaseDefinition definition, CaseHubRuntime runtime) {
        this.definition = definition;
        this.runtime = runtime;
    }

    @Override CaseDefinition getDefinition() { return definition; }
}
```

This requires `CaseHub` to either (a) support setting the runtime via constructor/setter, or (b) the subclass overrides the delegating methods directly. Currently `CaseHub` delegates `startCase()` etc. to the `@Inject`-ed `runtime` field. A registry-managed hub would need that field set.

**Option B — Reflective field set:**
Set the `runtime` field via reflection after construction. Works but fragile.

**Option A is cleaner.** Add a `protected` constructor on `CaseHub` that accepts `CaseHubRuntime` as a parameter, storing it in the existing `runtime` field. The no-arg constructor (used by CDI) remains for subclasses with field injection. This is a small, backward-compatible API change that enables non-CDI-managed instances cleanly.

**Consumer usage:**
```java
@Inject CaseHubRegistry caseHubRegistry;

CaseHub helpdesk = caseHubRegistry.get("helpdesk-ticket").orElseThrow();
UUID caseId = helpdesk.startCase(inputData);
```

**Coexistence with explicit CaseHub beans:** Explicit `YamlCaseHub` subclasses (with `augment()` overrides) continue to work as CDI beans, discovered via `Instance<CaseHub>` at startup. `CaseHubRegistry` is for auto-discovered cases only. `DefaultCaseDefinitionRegistry.registerKnownDefinitions()` handles CDI beans; `CaseYamlRegistrar` handles auto-discovered YAML.

**Priority ordering:** If the same YAML is both auto-discovered AND has an explicit `YamlCaseHub` subclass, the explicit subclass wins — it's a CDI bean with `augment()`. `CaseYamlRegistrar` should check for duplicates (same namespace/name/version already registered by a CaseHub bean) and skip registry creation.

## Testing Strategy

### #1360 — Flow worker integration test
- Define a case with a worker: `capabilities: [notify]`, `do: call: http` (or `call: log` for simplicity)
- Trigger the binding
- Assert the worker executes via `FlowWorkerFunctionHandler`
- Assert `WorkerResult` contains the expected output

### #1361 — Agent worker provider test
- Unit test: `AgentWorkerFunctionProvider.handles()` returns true for `agent: { capability: ... }` and `agent: { slot: ... }`
- Unit test: `create()` returns `AgentWorkerFunction.ofCapability()` for capability-based, `ofSlot()` for slot-based
- Integration test: define a worker with `agent: { capability: text-classification }`, verify dispatch reaches `AgentWorkerFunctionHandler.executeByCapability()`

### #1362 — CaseHubRegistry test
- Unit test: `CaseHubRegistry` register/get/all
- Integration test: place a YAML file in `casehub/`, verify `CaseHubRegistry` contains a hub with the right name
- Integration test: `caseHubRegistry.get("test-case").get().startCase()` starts a case
- Duplicate test: explicit `YamlCaseHub` subclass + auto-discovered YAML → explicit wins, no duplicate

## Module Placement

| Change | Module | Rationale |
|--------|--------|-----------|
| `FlowWorkerFunctionProvider` CDI wiring fix | `engine-support-core` / `flow` | Where the provider lives |
| `AgentWorkerFunctionProvider` widening | `engine-support-core` | Where the provider lives |
| `CaseHubRegistry` | `runtime` | Needs `CaseHubRuntime`, close to `DefaultCaseDefinitionRegistry` |
| `CaseHub` runtime setter | `api` | Small API change to enable non-CDI usage |

## Protocols

- **PP-20260921-b7c277 (no @DefaultBean on multi-instance SPI):** `FlowWorkerFunctionHandler` and `AgentWorkerFunctionHandler` are correctly `@ApplicationScoped` (not `@DefaultBean`). Multiple handlers coexist. No violation.
- **PP-20260915-aa504e (API surface parity):** `CaseHubRegistry` is a new API surface. It's runtime-only (no YAML or annotation counterpart needed — it serves auto-discovered cases). No parity gap.

## References

- `FlowWorkerFunctionProvider.java` — `engine-support-core/src/main/java/io/casehub/engine/flow/`
- `FlowWorkerFunctionHandler.java` — `engine-support-core/src/main/java/io/casehub/engine/flow/`
- `FlowWorkerFunction.java` — `engine-support-core/src/main/java/io/casehub/engine/flow/`
- `AgentWorkerFunctionProvider.java` — `engine-support-core/src/main/java/io/casehub/engine/agent/`
- `AgentWorkerFunctionHandler.java` — `runtime/src/main/java/io/casehub/engine/runtime/executor/`
- `AgentWorkerFunction.java` — `api/src/main/java/io/casehub/api/model/`
- `DeferredAgentConfig.java` — `api/src/main/java/io/casehub/api/model/ai/`
- `CaseYamlRegistrar.java` — `runtime/src/main/java/io/casehub/engine/runtime/bootstrap/`
- `CaseHub.java` / `YamlCaseHub.java` — `api/src/main/java/io/casehub/api/engine/`
- `CaseHubRuntime.java` — `api/src/main/java/io/casehub/api/engine/`
- `CompositeWorkerExecutionManager.java` — `runtime-core/src/main/java/io/casehub/engine/runtime/worker/`
- `FirstSupportedRoutingStrategy.java` — `runtime-core/src/main/java/io/casehub/engine/runtime/routing/`
- `YamlCaseDefinitionConverter.convertWorkers()` — `api/src/main/java/io/casehub/api/model/converter/`
- `DefaultWorkerFunctionProviderRegistry.java` — `runtime-core/src/main/java/io/casehub/engine/runtime/worker/`
- `RuntimeBeans.java` producer methods — `runtime/src/main/java/io/casehub/engine/runtime/quarkus/`
