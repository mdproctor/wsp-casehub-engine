# SPEC: AgentCapabilityGuard — Per-Agent Decision Scope Enforcement

**Issue:** casehubio/engine#1283
**Parent:** casehubio/agency#377
**Depends on:** engine#1282 (coordination vocabulary types)
**Date:** 2026-11-02

## Summary

Add mechanical enforcement of per-agent decision scope to the engine. When an agent attempts an action, the guard checks whether the agent's current `DecisionScope` permits that category of action. Also adds `DecisionAuthority` derivation from topology+role, and validates structurally incoherent topology×role combinations.

Five concerns (all deferred from #1282):
1. `AgentCapabilityGuard` SPI
2. Permission matrix enforcement
3. `DecisionAuthority` derivation function
4. Invalid `TopologyType × StructuralRole` rejection
5. Agent identity type resolution (→ String)

## Vocabulary Enrichment

### DecisionScope — permission matrix method

Add `isPermitted(DecisionType)` to the `DecisionScope` enum. Encoded as exhaustive switch expressions — adding a `DecisionType` value forces every `DecisionScope` override to handle it or the build breaks. This compiler-enforces the #1282 spec's total-function invariant.

```java
public enum DecisionScope {
  STIMULUS_DRIVEN {
    @Override public boolean isPermitted(DecisionType type) {
      return switch (type) {
        case SIGNALING -> true;
        case TASK_SELECTION, COLLABORATION, PROVISIONING, CONTEXT_MUTATION -> false;
      };
    }
  },
  TASK_ADAPTATION {
    @Override public boolean isPermitted(DecisionType type) {
      return switch (type) {
        case COLLABORATION, SIGNALING, CONTEXT_MUTATION -> true;
        case TASK_SELECTION, PROVISIONING -> false;
      };
    }
  },
  FULL_AUTONOMY {
    @Override public boolean isPermitted(DecisionType type) {
      return switch (type) {
        case TASK_SELECTION, COLLABORATION, SIGNALING, PROVISIONING, CONTEXT_MUTATION -> true;
      };
    }
  };

  public abstract boolean isPermitted(DecisionType type);
}
```

**Matrix (for reference — encoded in the switch expressions above):**

| | TASK_SELECTION | COLLABORATION | SIGNALING | PROVISIONING | CONTEXT_MUTATION |
|---|---|---|---|---|---|
| STIMULUS_DRIVEN | DENY | DENY | PERMIT | DENY | DENY |
| TASK_ADAPTATION | DENY | PERMIT | PERMIT | DENY | PERMIT |
| FULL_AUTONOMY | PERMIT | PERMIT | PERMIT | PERMIT | PERMIT |

### DecisionAuthority — new enum

New enum in `io.casehub.api.model.coordination`:

```java
public enum DecisionAuthority {
  LOCAL,
  COORDINATED,
  CENTRAL
}
```

- **LOCAL** — agent decisions affect only itself (subordinate in centralised/hybrid)
- **COORDINATED** — agent participates in shared decision-making (peer in distributed/hybrid)
- **CENTRAL** — agent decisions affect the entire swarm (initiator in centralised/hybrid)

This is a *derived* type — never directly configured. Always produced by `TopologyType.deriveAuthority(StructuralRole)`.

**Intended consumers:** `DecisionAuthority` is consumed by #1284 (TopologyDescriptor) where the topology lifecycle needs authority levels to determine which agent can make swarm-wide decisions vs local-only decisions. The `AgentCapabilityGuard` itself does not use `DecisionAuthority` — it works with `DecisionScope` and `DecisionType`, which are independent of topology. Authority and scope are orthogonal enforcement dimensions: authority constrains *structural reach* (who delegates to whom), scope constrains *operational freedom* (what categories of action).

### TopologyType — authority derivation and validation

Add `deriveAuthority(StructuralRole)` to the `TopologyType` enum:

```java
public enum TopologyType {
  CENTRALISED {
    @Override public DecisionAuthority deriveAuthority(StructuralRole role) {
      return switch (role) {
        case INITIATOR -> DecisionAuthority.CENTRAL;
        case CONTRACTOR -> DecisionAuthority.LOCAL;
        case PEER -> throw new IllegalArgumentException(
            "CENTRALISED topology does not support PEER — centralised structure requires coordinator/subordinate relationships");
      };
    }
  },
  DISTRIBUTED {
    @Override public DecisionAuthority deriveAuthority(StructuralRole role) {
      return switch (role) {
        case PEER -> DecisionAuthority.COORDINATED;
        case INITIATOR -> throw new IllegalArgumentException(
            "DISTRIBUTED topology does not support INITIATOR — peer-to-peer structure has no central coordinator");
        case CONTRACTOR -> throw new IllegalArgumentException(
            "DISTRIBUTED topology does not support CONTRACTOR — peer-to-peer structure has no delegation hierarchy");
      };
    }
  },
  HYBRID {
    @Override public DecisionAuthority deriveAuthority(StructuralRole role) {
      return switch (role) {
        case INITIATOR -> DecisionAuthority.CENTRAL;
        case PEER -> DecisionAuthority.COORDINATED;
        case CONTRACTOR -> DecisionAuthority.LOCAL;
      };
    }
  };

  public abstract DecisionAuthority deriveAuthority(StructuralRole role);
}
```

**Derivation mapping (total function):**

| TopologyType | INITIATOR | PEER | CONTRACTOR |
|---|---|---|---|
| CENTRALISED | CENTRAL | *invalid* | LOCAL |
| DISTRIBUTED | *invalid* | COORDINATED | *invalid* |
| HYBRID | CENTRAL | COORDINATED | LOCAL |

Null `StructuralRole` (symbolic roles) is not accepted — callers must handle null before calling `deriveAuthority`. Symbolic roles cannot participate in topology contexts that require authority derivation.

## AgentAction — sealed interface

```java
public sealed interface AgentAction {

  DecisionType decisionType();

  record InitiateMessage(String targetAgentId) implements AgentAction {
    @Override public DecisionType decisionType() { return DecisionType.COLLABORATION; }
  }

  record DepositSignal(String signalName) implements AgentAction {
    @Override public DecisionType decisionType() { return DecisionType.SIGNALING; }
  }

  record SelectTask(String taskId) implements AgentAction {
    @Override public DecisionType decisionType() { return DecisionType.TASK_SELECTION; }
  }

  record RequestProvisioning() implements AgentAction {
    @Override public DecisionType decisionType() { return DecisionType.PROVISIONING; }
  }

  record MutateContext(String contextKey) implements AgentAction {
    @Override public DecisionType decisionType() { return DecisionType.CONTEXT_MUTATION; }
  }
}
```

**Note:** `InitiateMessage` maps to `COLLABORATION`, not a separate MESSAGING type. Initiating a message to another agent is a collaboration decision — the agent is choosing to interact. This aligns with the #1282 spec's definition of COLLABORATION ("Deciding to collaborate with another agent").

**Note:** `MutateContext` is added here — the #1282 spec defines `CONTEXT_MUTATION` as a DecisionType but the original issue omitted the corresponding action. Writing to CaseContext keys that affect other agents' observations is a distinct action category.

## AgentCapabilityGuard SPI

```java
public sealed interface GuardDecision {
  record Permitted() implements GuardDecision {}
  record Denied(DecisionScope agentScope, DecisionType attemptedType) implements GuardDecision {}
}

public interface AgentCapabilityGuard {
  GuardDecision check(UUID caseId, String agentId, AgentAction action);
}
```

**Package:** `io.casehub.api.spi` (both `AgentCapabilityGuard` and `GuardDecision`)

**Semantics:** Returns `Permitted` if the agent may perform the action, or `Denied` with the agent's scope and the attempted decision type for structured logging. The guard resolves the agent's current `DecisionScope` internally — callers pass identifiers only.

**Why `GuardDecision` instead of `boolean`:** Follows the `ActionRiskClassifier` → `RiskDecision` precedent. The guard is the only component that knows the agent's scope — making callers separately look up scope to produce a log message is redundant computation and couples callers to enforcement internals. `Denied` carries the context needed for structured denial logging.

**Agent identity at integration points:** The SPI uses `agentId` because the guard enforces agent-level coordination constraints. At integration points (signal deposit, context mutation, provisioning), the available identity is `workerName`. Callers pass `workerName` as the `agentId` parameter — in coordination topologies, agent identity and worker identity are the same value. This mapping is intentional: the guard's purpose is agent-level enforcement, but the underlying identity infrastructure uses worker names.

**NoOp default:** `NoOpAgentCapabilityGuard` — a plain POJO in `runtime-core` (no CDI annotations). Returns `new GuardDecision.Permitted()` for all inputs. Wired via a `@Produces @DefaultBean @ApplicationScoped` method in `SpiDefaultBeans` (the `runtime` Quarkus-binding module). Agency displaces by providing a normal `@ApplicationScoped` bean — `@DefaultBean` yields automatically to any non-default qualifying bean, no `@Alternative` annotation needed.

**Call-site pattern:**

```java
@Inject AgentCapabilityGuard capabilityGuard;

GuardDecision decision = capabilityGuard.check(caseId, agentId, new AgentAction.DepositSignal(signalName));
if (decision instanceof GuardDecision.Denied denied) {
  log.warn("Agent {} denied: scope {} does not permit {}", agentId, denied.agentScope(), denied.attemptedType());
  return;
}
```

## Integration Points

The guard is checked at these enforcement boundaries. Each is a pre-check — if denied, the operation is skipped with a structured log message. No exceptions thrown on denial (soft enforcement — the caller decides the consequence).

| Operation | Call site | AgentAction | workerName → agentId |
|-----------|----------|-------------|---------------------|
| Signal deposit | `SignalSpace.deposit()` | `DepositSignal(signalName)` | `workerName` field |
| Message send | `CaseChannelProvider` dispatch | `InitiateMessage(targetAgentId)` | sender `workerName` |
| Task self-assignment | Worker scheduling pipeline | `SelectTask(taskId)` | scheduled `workerId` |
| Provisioning request | `WorkerProvisioner` | `RequestProvisioning()` | requesting `workerName` |
| Context write | `CaseContext` mutation | `MutateContext(contextKey)` | mutating `workerName` |

**Integration is deferred to downstream issues.** This issue defines the SPI, types, and NoOp default. Wiring the guard into each call site requires understanding the specific mechanics of each integration point — that work belongs in #1284+ where the topology lifecycle is defined.

**Rationale for deferring integration:** The guard SPI and the vocabulary enrichment (matrix, derivation, validation) are independently valuable and testable. Integration requires a scope-resolution mechanism (how does the guard look up an agent's DecisionScope?) which depends on the topology descriptor (#1284). Wiring the NoOp guard into call sites adds no value — it permits everything.

## Module Placement

| Artifact | Module | Package |
|----------|--------|---------|
| `AgentCapabilityGuard` | api | `io.casehub.api.spi` |
| `GuardDecision` | api | `io.casehub.api.spi` |
| `AgentAction` | api | `io.casehub.api.spi` |
| `DecisionAuthority` | api | `io.casehub.api.model.coordination` |
| `DecisionScope` (enriched) | api | `io.casehub.api.model.coordination` |
| `TopologyType` (enriched) | api | `io.casehub.api.model.coordination` |
| `NoOpAgentCapabilityGuard` | runtime-core | `io.casehub.engine.runtime.coordination` |
| Producer method | runtime | `SpiDefaultBeans` |

## Testing Strategy

### Exhaustive enum tests (api module)

- `DecisionScope.isPermitted()` — test all 15 cells of the 3×5 matrix
- `TopologyType.deriveAuthority()` — test all 9 cells of the 3×3 matrix (6 valid → correct authority, 3 invalid → IAE with descriptive message)
- `AgentAction.decisionType()` — test each variant returns the correct type

### SPI contract test (api module)

- `NoOpAgentCapabilityGuard` always returns `Permitted()` for any (caseId, agentId, action) triple

### Guard integration test (runtime-core module, @QuarkusTest)

- Verify `NoOpAgentCapabilityGuard` is injectable and active by default
- Verify it can be displaced by a normal `@ApplicationScoped` test implementation (no `@Alternative` needed)

## Resolved Questions

1. **Agent identity type → String.** Consistent with existing `workerName`/`workerId` usage throughout the engine. A typed wrapper adds no value when the underlying identity is always a worker name string. If String confusion becomes a real problem in the coordination layer, a typed wrapper can be introduced later without breaking the SPI contract (the wrapper would implement `toString()` → same wire value).

## What This Issue Does NOT Include

- **Integration wiring** — guard is not wired into call sites. Deferred to #1284+ (requires topology descriptor for scope resolution).
- **Activation guard** — the second enforcement layer for STIMULUS_DRIVEN (activation-time check). Deferred to #1284 (requires topology lifecycle).
- **YAML surface** — DSL parity for DecisionAuthority. Deferred to agency#385.
- **Scope resolution mechanism** — how the guard looks up an agent's current DecisionScope. Depends on topology descriptor (#1284).

## References

- `specs/issue-1282-coordination-vocabulary-types/SPEC.md` — vocabulary definitions, permission matrix, derivation mapping, deferral decomposition
- `api/src/main/java/io/casehub/api/spi/ActionRiskClassifier.java` — `RiskDecision` sealed interface precedent for guard return types
- `runtime/src/main/java/io/casehub/engine/runtime/quarkus/SpiDefaultBeans.java` — `@Produces @DefaultBean` producer pattern for NoOp defaults
- `runtime-core/src/main/java/io/casehub/engine/runtime/worker/NoOpCaseChannelProvider.java` — plain POJO NoOp default precedent
- PP-20260921-b7c277 — `@DefaultBean` for 1:1 SPI replacement
- PP-20260514-engine-spi-noops-defaultbean — engine SPI no-op defaults must use `@DefaultBean`
- PP-20260727-5267d2 — placement rules: plan-definition types in api, execution types in common
- casehubio/engine#1283 — parent issue
- casehubio/agency#377 — parent epic
