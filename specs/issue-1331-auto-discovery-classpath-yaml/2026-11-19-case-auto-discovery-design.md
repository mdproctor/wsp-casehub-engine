# Case Engine Auto-Discovery — Full-Pipeline YAML Registration

**Issue:** casehubio/engine#1331
**Date:** 2026-11-19

## Summary

Enhance `CaseYamlRegistrar` to use the full YAML loading pipeline — expression engines, worker function providers (`FlowWorkerFunctionProvider` for `do:` blocks), agent providers, and overlay composition. This enables zero-Java case applications where `casehub/*.yaml` on the classpath is sufficient to run a fully-wired case.

## Current State

`CaseYamlRegistrar` (priority 200, `ClasspathYamlRegistrar` SPI) scans `casehub/` and `cases/` directories on the classpath and calls `CaseDefinitionYamlMapper.load(InputStream)` — the simple overload that uses JQ-only expressions, no worker providers, and no agent providers. Workers with `do:` blocks are not wired.

The full pipeline is available via `CaseDefinitionYamlMapper.load(JsonNode, ObjectMapper, ExpressionEngineRegistry, WorkerFunctionProviderRegistry, AgentProvider)`, used by `YamlCaseHub.getDefinition()`.

## Design

### Changes to `CaseYamlRegistrar`

**New CDI-injected fields:**

```java
@Inject ExpressionEngineRegistry expressionEngineRegistry;
@Inject @YamlMapper ObjectMapper objectMapper;
@Inject WorkerFunctionProviderRegistry workerFunctionProviderRegistry;
@Inject Instance<AgentProvider> agentProviderInstance;
```

**Enhanced `register(String resourcePath, InputStream content)`:**

1. Read the `InputStream` as a `JsonNode` via the injected `objectMapper`
2. Derive the convention overlay path using `YamlCaseHub.deriveConventionPath(resourcePath)`
3. Look up the overlay on the classpath; if found, merge via `YamlMerger.merge(base, overlay)`
4. Call `CaseDefinitionYamlMapper.load(mergedNode, objectMapper, expressionEngineRegistry, workerFunctionProviderRegistry, agentProvider)`
5. Register via `caseDefinitionRegistry.registerCaseDefinition(definition)`

### Precedence

`DefaultCaseDefinitionRegistry.onStart()` runs at priority 10, registering all explicit CDI `CaseHub` beans. `CaseYamlRegistrar` runs at priority 200. The registry's existing dedup check (`RegistryEntry existing = registry.get(key); if (existing != null) return`) means explicit CaseHub beans always win. No additional precedence logic is needed.

A debug log line is added when a YAML file is skipped because a definition with the same key was already registered.

### Overlay Support

Mirrors `YamlCaseHub`'s overlay composition convention. For a base YAML at `casehub/helpdesk-ticket.yaml`, the registrar checks for `casehub/helpdesk-ticket-overrides.yaml` on the classpath. If found, it deep-merges the overlay onto the base via `YamlMerger.merge()` before loading.

The `deriveConventionPath()` method is already `static` on `YamlCaseHub` (package-private). Make it `public static` — it's a pure utility function with no side effects, and sharing it avoids duplicating the convention logic.

### Module Boundaries

All changes are in the `runtime` module where `CaseYamlRegistrar` already lives. The injected types (`ExpressionEngineRegistry`, `WorkerFunctionProviderRegistry`, `ObjectMapper`) are from the `api` module, already a dependency. `YamlMerger` is from `casehub-platform-api`, also already available.

### Error Handling

Errors during full-pipeline loading (expression validation failures, worker parse errors) are logged at ERROR level and the YAML file is skipped — same as the current behavior. The registrar does not fail-fast to avoid blocking startup for one malformed YAML file.

## Testing

1. **Existing `CaseYamlRegistrarTest`** — update to verify the registrar uses the full load pipeline (mock the injected dependencies)
2. **New test: overlay composition** — verify that a base YAML + convention overlay produces a merged definition
3. **New test: precedence** — verify that when a CDI CaseHub bean and a classpath YAML both define the same case key, the CDI bean's definition is used
4. **New test: worker wiring** — verify that a YAML with `do:` blocks has its worker functions resolved via `FlowWorkerFunctionProvider`

## References

- `CaseYamlRegistrar.java:30` — existing registrar
- `YamlCaseHub.java:68` — full pipeline load in getDefinition()
- `YamlCaseHub.java:143` — deriveConventionPath() for overlay convention
- `CaseDefinitionYamlMapper.java:213` — full load(JsonNode, ...) overload
- `DefaultCaseDefinitionRegistry.java:103` — priority 10 startup, dedup at line 157
- `FlowWorkerFunctionProvider.java:34` — handles `do:` blocks
- `PP-20260921-b7c277` — no-defaultbean-multi-instance-spi protocol (not applicable — registrar uses plain @ApplicationScoped, not @DefaultBean)
