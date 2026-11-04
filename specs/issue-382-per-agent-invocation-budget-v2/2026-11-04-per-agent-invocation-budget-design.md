# Per-Agent Invocation Budget — Unified Pool Design

**Issue:** casehubio/agency#382
**Date:** 2026-11-04

## Problem

The budget system has a two-pool split. `SwarmCoordinatorBuilder.buildTopology()` creates one `AutonomyBudgetPool` and injects it into three direct consumers (SwarmRoutingStrategy, CollectiveReasoningAggregation, SwarmProvisioningAdvisorImpl). Then `CentralisedTopology.configure()` creates a second independent pool for the decision loops.

This causes:

1. **Split enforcement** — each pool tracks calls and tokens independently. Budget limits apply per-pool, not per-swarm.
2. **Double call-counting** — if both pools are merged naively, `BudgetAwareSummariser.tryAcquire()` and the consumer's `tryAcquire()` both increment the same pool's call count. One LLM call costs two budget units.
3. **Zero token recording** — `DecisionLoopFactory` uses the default `BudgetAwareSummariser` constructor with `r -> 0L` as the token estimator. Pool B's `tokenCount` stays at zero permanently. Token-based enforcement through the decision loop path never triggers.

## Design

### Single Pool

Remove pool creation and `setBudgetPool()` calls from `SwarmCoordinatorBuilder.buildTopology()`. The topology's pool in `configure()` is the only pool. The builder no longer touches budget infrastructure.

**Before:**
```
Builder creates Pool A → injects into consumers
Topology creates Pool B → injects into decision loop views
```

**After:**
```
Topology creates Pool → decision loop views use it
                       → consumers record tokens on it (no gating)
```

### Call Gating — Decision Loop Only

`BudgetAwareSummariser` remains the single call gate. It calls `view.tryAcquire()` → `pool.tryAcquire()`, incrementing the pool's call count once per LLM call. The consumer's own `tryAcquire()` is not triggered because no pool is injected via `setBudgetPool()` in the topology path.

The `setBudgetPool()` API remains on consumers for the YAML path (`PatternCompiler`), where consumers are used outside decision loops and need standalone call gating.

### Token Recording — Consumer Callback

Add `setTokenRecorder(LongConsumer)` to the three direct consumers. In the topology path, `CentralisedTopology.configure()` wires it to `pool.recordTokens()`. The consumer records token counts after each LLM call via the callback:

```java
// In each consumer (e.g., SwarmRoutingStrategy)
private @Nullable LongConsumer tokenRecorder;

public void setTokenRecorder(@Nullable LongConsumer recorder) {
    this.tokenRecorder = recorder;
}

// After successful LLM call:
long tokens = result.metadata().inputTokens() + result.metadata().outputTokens();
if (tokenRecorder != null) {
    tokenRecorder.accept(tokens);
} else if (budgetPool != null) {
    budgetPool.recordTokens(correlationId, tokens);
}
```

In `CentralisedTopology.configure()`:
```java
var pool = new AutonomyBudgetPool(context.budgetPolicy(), context.caseId() + "-CENTRALISED");

routingStrategy.setTokenRecorder(t -> pool.recordTokens("routing", t));
aggregation.setTokenRecorder(t -> pool.recordTokens("aggregation", t));
if (provisioningAdvisor instanceof SwarmProvisioningAdvisorImpl impl) {
    impl.setTokenRecorder(t -> pool.recordTokens("provisioning", t));
}
```

**Note:** `routingStrategy` and `aggregation` are concrete types in CentralisedTopology. The provisioner is held as the SPI interface `SwarmProvisioningAdvisor`, so `setTokenRecorder()` requires an `instanceof` check. This is consistent with the topology's existing tight coupling to the concrete swarm components.

**Recording path precedence:** `tokenRecorder` > `budgetPool`. When both are set (not expected, but defensive), `tokenRecorder` wins. The YAML path uses `setBudgetPool()` only; the topology path uses `setTokenRecorder()` only.

### Unchanged

- **`setBudgetPool()` API on consumers** — kept for the YAML path (`PatternCompiler`). The API is unchanged; it's just not called from the builder anymore.
- **All topology priority assignments** — already correct per the priority table. CentralisedTopology: all RESERVED. HierarchicalTopology: root RESERVED, others NORMAL. Etc.
- **`AutonomyBudgetPool`**, **`AgentBudgetView`**, **`BudgetPriority`**, **`BudgetGrant`**, **`BudgetPressure`**, **`BudgetAwareSummariser`**, **`BudgetTelemetryEvent`** — no changes.
- **`DecisionLoopFactory`**, **`DecisionLoopDescriptor`** — no changes.
- **`AutonomyBudgetPolicy`** — no changes to the policy record or its defaults.
- **Other topology implementations** — no direct consumers to wire.

## Files Changed

| File | Change |
|------|--------|
| `SwarmCoordinatorBuilder.java` | Remove pool creation + 3 `setBudgetPool()` calls (~5 lines removed) |
| `SwarmRoutingStrategy.java` | Add `setTokenRecorder(LongConsumer)`, use for token recording (~6 lines) |
| `CollectiveReasoningAggregation.java` | Add `setTokenRecorder(LongConsumer)`, use for token recording (~6 lines) |
| `SwarmProvisioningAdvisorImpl.java` | Add `setTokenRecorder(LongConsumer)`, use for token recording (~6 lines) |
| `CentralisedTopology.java` | Wire `setTokenRecorder()` on consumers after pool creation (~3 lines) |

## Testing

- **SwarmCoordinatorBuilder tests** — verify builder no longer creates pool or calls `setBudgetPool()` on consumers
- **CentralisedTopology tests** — verify single pool, token recording flows through `tokenRecorder` callback
- **Consumer unit tests** — verify `tokenRecorder` takes precedence over `budgetPool` for recording; verify `setBudgetPool()` still works for YAML path gating + recording
- Downstream issues cover remaining test gaps: #391 (priority assertions), #392 (pool assertions), #394 (setBudgetPool fallback tests)

## Scope Boundary

Out of scope:
- **YAML path pool unification** — `PatternCompiler` still creates its own pool. Addressed by #385 (YAML parity TopologySpec registries).
- **BudgetAwareSummariser token estimator** — remains `r -> 0L`. Token recording goes through consumers, not the estimator. The estimator is vestigial but harmless.
- **Other topologies' token recording** — they don't use direct consumers. Their loops use zero-estimator BudgetAwareSummariser. This is a broader concern beyond this issue.

## References

- `SwarmCoordinatorBuilder.java:83-121` — current two-pool creation
- `CentralisedTopology.java:70-124` — topology pool creation and loop wiring
- `SwarmRoutingStrategy.java:78-108` — consumer gating + token recording
- `BudgetAwareSummariser.java:22-24` — default zero token estimator
- `DecisionLoopFactory.java:32` — view creation from descriptor priority
- `AutonomyBudgetPool.java:36-38` — recordTokens ignores correlationId
- `PatternCompiler.java:152-154` — YAML path pool creation (out of scope)
- agency#391 — downstream: BudgetPriority test assertions
- agency#392 — downstream: CoordinationPlan.pool() test assertions
- agency#394 — downstream: setBudgetPool() fallback tests
