# Wire CaseContextStoreFactory Through Recovery Path — Design Spec

**Issue:** casehubio/engine#732
**Epic:** casehubio/engine#210 — cancellation, timeout, and error recovery
**Date:** 2026-09-27

## Problem

When a case is started, `CaseHubRuntimeImpl` resolves the `CaseContextStoreFactory` from the `CaseDefinition` and creates a `CaseContextImpl(factory, caseId)`. Each layer is backed by a `CaseContextStore` from the factory. New layers accessed on demand also go through the factory (`layer()` and `writableLayer()` call `storeFactory.createStore()`).

When a case is recovered on cache miss, `DefaultWorkerExecutionRecoveryService` delegates to `CaseContextRecoveryStrategy.recover(instance)`. Both the snapshot and EventLog replay strategies create a `CaseContextImpl` with the no-arg constructor, which hardcodes `InMemoryCaseContextStoreFactory`. The recovered context is disconnected from the original factory.

This means:
1. Durable factories (`isDurable()==true`) cannot be deployed — a guard in `CaseHubRuntimeImpl.resolveFactory()` throws `UnsupportedOperationException` because recovery would silently lose state
2. Custom volatile factories lose their behaviour (logging, metrics) after recovery
3. New layers accessed after recovery use `InMemoryCaseContextStoreFactory` regardless of the original factory

## Constraint

`CaseContextImpl.storeFactory` is `final` (line 52). Both `layer()` (line 129) and `writableLayer()` (line 135) call `storeFactory.createStore()` for new layers. The factory cannot be swapped after construction — recovery must create the context with the correct factory from the start.

## Solution

Add factory resolution to the recovery service. For durable factories, bypass the recovery strategy entirely and load from the factory's stores. For volatile factories, use the existing recovery strategy (snapshot or EventLog replay).

### Recovery Service Changes

`DefaultWorkerExecutionRecoveryService.loadOrRestoreCaseInstance()` gains factory-aware branching:

```java
CaseInstance instance = caseInstanceRepository.findByUuid(caseId).orElseThrow(...);

CaseContextStoreFactory factory = resolveFactory(instance);
CaseContext context;
if (factory != null && factory.isDurable()) {
    context = CaseContextImpl.loadFromStore(factory, instance.getUuid());
} else {
    context = recoveryStrategy.recover(instance);
}

instance.setCaseContext(context);
caseInstanceCache.put(instance);
```

The service adds two new injected dependencies:
- `CaseDefinitionRegistry` — to look up the `CaseDefinition` from the `CaseMetaModel`
- `StrategyResolver` — to resolve the `CaseContextStoreFactory` from the definition

### Factory Resolution

```java
private CaseContextStoreFactory resolveFactory(CaseInstance instance) {
    CaseMetaModel metaModel = instance.getCaseMetaModel();
    if (metaModel == null) {
        LOG.errorf("Cannot resolve factory for caseId=%s — CaseMetaModel is null. "
            + "Falling back to volatile recovery. If this case uses a durable factory, "
            + "writes since last snapshot will be lost on next restart.", instance.getUuid());
        return null;
    }
    CaseDefinition definition = caseDefinitionRegistry.getCaseDefinition(metaModel);
    if (definition == null) {
        LOG.errorf("Cannot resolve factory for caseId=%s — CaseDefinition not registered "
            + "for metaModel '%s'. Falling back to volatile recovery.", 
            instance.getUuid(), metaModel.getName());
        return null;
    }
    String factoryName = definition.getContextStoreFactory();
    if (factoryName == null || factoryName.isBlank()) {
        return null; // default factory — volatile recovery is correct
    }
    try {
        return strategyResolver.resolve(CaseContextStoreFactory.class, factoryName);
    } catch (Exception e) {
        LOG.errorf(e, "Cannot resolve CaseContextStoreFactory '%s' for caseId=%s. "
            + "Falling back to volatile recovery.", factoryName, instance.getUuid());
        return null;
    }
}
```

**Failure semantics (from decision review):** Factory resolution failure for a durable-factory case is a corruption-shaped failure — the case appears to recover but writes are lost on next restart. ERROR-level logging is mandatory so operators can alert on it, not just discover it via log grep. A metric counter (`casehub.recovery.factory_resolution_failures`) should be added when the metrics SPI is available.

### `CaseContextImpl.loadFromStore`

New static factory method:

```java
public static CaseContextImpl loadFromStore(CaseContextStoreFactory factory, UUID caseId) {
    CaseContextImpl ctx = new CaseContextImpl();
    ctx.storeFactory = factory; // ← requires removing 'final' or using a private constructor
    ctx.caseId = caseId;
    ctx.layers.put(ContextLayer.WORKING,
        new WritableLayerImpl(ContextLayer.WORKING, factory.loadStore(ContextLayer.WORKING, caseId)));
    ctx.layers.put(ContextLayer.SEMANTIC,
        new WritableLayerImpl(ContextLayer.SEMANTIC, factory.loadStore(ContextLayer.SEMANTIC, caseId)));
    ctx.layers.put(ContextLayer.EPISODIC,
        new WritableLayerImpl(ContextLayer.EPISODIC, factory.loadStore(ContextLayer.EPISODIC, caseId)));
    return ctx;
}
```

**Implementation note:** `storeFactory` and `caseId` are currently `final`. The static factory method needs write access. Two options:
1. Add a private constructor `CaseContextImpl(CaseContextStoreFactory, UUID, boolean load)` that calls `loadStore()` instead of `createStore()` — keeps fields final
2. Remove `final` from the fields and use the static method as written above

Option 1 (private constructor) is preferred — keeps `final` semantics:

```java
private CaseContextImpl(CaseContextStoreFactory storeFactory, UUID caseId, boolean load) {
    this.storeFactory = storeFactory;
    this.caseId = caseId;
    for (String layer : List.of(ContextLayer.WORKING, ContextLayer.SEMANTIC, ContextLayer.EPISODIC)) {
        CaseContextStore store = load ? storeFactory.loadStore(layer, caseId) 
                                      : storeFactory.createStore(layer, caseId);
        layers.put(layer, new WritableLayerImpl(layer, store));
    }
}

public static CaseContextImpl loadFromStore(CaseContextStoreFactory factory, UUID caseId) {
    return new CaseContextImpl(factory, caseId, true);
}
```

The existing `public CaseContextImpl(CaseContextStoreFactory, UUID)` constructor delegates to the private one with `load=false`.

### Remove Durable Guard

Delete the `isDurable()` check in `CaseHubRuntimeImpl.resolveFactory()`:

```java
private CaseContextStoreFactory resolveFactory(CaseDefinition definition) {
    return strategyResolver.resolve(
        CaseContextStoreFactory.class,
        definition.getContextStoreFactory());
}
```

### Snapshot / Durable Store Precedence

When a case uses a durable factory, both the durable stores and the JSON snapshot on `CaseInstanceEntity` contain context data. Precedence:

- **Recovery:** Durable store wins. `loadFromStore()` loads from the factory's stores.
- **Fallback:** If factory resolution fails (D3), snapshot recovery kicks in via the existing `CaseContextRecoveryStrategy`.
- **`onContextChanged`:** Continues to write snapshots for ALL cases, including durable. This provides a secondary recovery path if the durable store is corrupted.

## Files Changed

| File | Change |
|------|--------|
| `runtime/.../recovery/DefaultWorkerExecutionRecoveryService.java` | Add `CaseDefinitionRegistry`, `StrategyResolver` injection. Add `resolveFactory(CaseInstance)`. Branch on `isDurable()` before calling strategy. |
| `runtime/.../context/CaseContextImpl.java` | Add private constructor with `load` flag. Add `loadFromStore(factory, caseId)` static method. Refactor existing factory constructor to delegate. |
| `runtime/.../engine/CaseHubRuntimeImpl.java` | Remove `isDurable()` guard from `resolveFactory()`. |
| `runtime/src/test/.../DefaultWorkerExecutionRecoveryServiceTest.java` | New: test durable factory path, volatile factory path, fallback on resolution failure. |
| `runtime/src/test/.../CaseContextImplTest.java` | Test `loadFromStore()` creates layers with loaded stores. |
| `runtime/src/test/.../CaseHubRuntimeImplTest.java` | Update: durable factory no longer throws. |

## References

- `DefaultWorkerExecutionRecoveryService.java:63` — current recovery entry point
- `CaseContextImpl.java:52` — final storeFactory field (the driving constraint)
- `CaseContextImpl.java:81` — factory constructor (createStore path)
- `CaseContextImpl.java:129,135` — layer()/writableLayer() use storeFactory
- `CaseContextImpl.java:630` — fromLayerDocument pattern (static factory method precedent)
- `CaseContextStoreFactory.java:35` — loadStore() default method
- `CaseContextStoreFactory.java:44` — isDurable() default method
- `CaseHubRuntimeImpl.java:116-131` — resolveFactory with durable guard
- `CaseDefinitionRegistry.java:67` — getCaseDefinition(CaseMetaModel)
- Design spec: `docs/specs/2026-07-13-case-context-store-design.md` §Recovery Model
- Design spec: `docs/specs/2026-07-14-context-store-factory-wiring-design.md` §7
- Issue #725 — CaseContextStoreFactory wiring through startCase
- decisions.md — D1-D4 for this issue
