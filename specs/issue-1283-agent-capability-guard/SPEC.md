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

Add `isPermitted(DecisionType)` to the `DecisionScope` enum. This is the reference matrix from the #1282 spec, encoded as a total function:

```java
public enum DecisionScope {
  STIMULUS_DRIVEN {
    @Override public boolean isPermitted(DecisionType type) {
      return type == DecisionType.SIGNALING;
    }
  },
  TASK_ADAPTATION {
    @Override public boolean isPermitted(DecisionType type) {
      return type != DecisionType.TASK_SELECTION && type != DecisionType.PROVISIONING;
    }
  },
  FULL_AUTONOMY {
    @Override public boolean isPermitted(DecisionType type) {
      return true;
    }
  };

  public abstract boolean isPermitted(DecisionType type);
}
```

**Matrix (for reference — encoded in the overrides above):**

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
public interface AgentCapabilityGuard {
  boolean isPermitted(UUID caseId, String agentId, AgentAction action);
}
```

**Package:** `io.casehub.api.spi`

**Semantics:** Returns `true` if the agent is permitted to perform the action, `false` otherwise. The guard resolves the agent's current `DecisionScope` internally — callers pass identifiers only.

**NoOp default:** `NoOpAgentCapabilityGuard` in `runtime-core/internal/` — permits everything (`return true`). Annotated `@DefaultBean @ApplicationScoped`. Agency displaces with a scope-aware implementation via `@Alternative @Priority(1)`.

**Call-site pattern:**

```java
@Inject AgentCapabilityGuard capabilityGuard;

if (!capabilityGuard.isPermitted(caseId, agentId, new AgentAction.DepositSignal(signalName))) {
  // log denial and skip
  return;
}
```

## Integration Points

The guard is checked at these enforcement boundaries. Each is a pre-check — if denied, the operation is skipped with a log message. No exceptions thrown on denial (soft enforcement — the caller decides the consequence).

| Operation | Call site | AgentAction |
|-----------|----------|-------------|
| Signal deposit | `SignalSpace.deposit()` | `DepositSignal(signalName)` |
| Message send | `CaseChannelProvider` dispatch | `InitiateMessage(targetAgentId)` |
| Task self-assignment | Worker scheduling pipeline | `SelectTask(taskId)` |
| Provisioning request | `WorkerProvisioner` | `RequestProvisioning()` |
| Context write | `CaseContext` mutation | `MutateContext(contextKey)` |

**Integration is deferred to downstream issues.** This issue defines the SPI, types, and NoOp default. Wiring the guard into each call site requires understanding the specific mechanics of each integration point — that work belongs in #1284+ where the topology lifecycle is defined.

**Rationale for deferring integration:** The guard SPI and the vocabulary enrichment (matrix, derivation, validation) are independently valuable and testable. Integration requires a scope-resolution mechanism (how does the guard look up an agent's DecisionScope?) which depends on the topology descriptor (#1284). Wiring the NoOp guard into call sites adds no value — it permits everything.

## Module Placement

| Artifact | Module | Package |
|----------|--------|---------|
| `AgentCapabilityGuard` | api | `io.casehub.api.spi` |
| `AgentAction` | api | `io.casehub.api.spi` |
| `DecisionAuthority` | api | `io.casehub.api.model.coordination` |
| `DecisionScope` (enriched) | api | `io.casehub.api.model.coordination` |
| `TopologyType` (enriched) | api | `io.casehub.api.model.coordination` |
| `NoOpAgentCapabilityGuard` | runtime-core | `io.casehub.engine.internal.coordination` |

## Testing Strategy

### Exhaustive enum tests (api module)

- `DecisionScope.isPermitted()` — test all 15 cells of the 3×5 matrix
- `TopologyType.deriveAuthority()` — test all 9 cells of the 3×3 matrix (6 valid → correct authority, 3 invalid → IAE with descriptive message)
- `AgentAction.decisionType()` — test each variant returns the correct type

### SPI contract test (api module)

- `NoOpAgentCapabilityGuard` always returns `true` for any (caseId, agentId, action) triple

### Guard integration test (runtime-core module, @QuarkusTest)

- Verify `NoOpAgentCapabilityGuard` is injectable and active by default
- Verify it can be displaced by an `@Alternative @Priority(1)` test implementation

## What This Issue Does NOT Include

- **Integration wiring** — guard is not wired into call sites. Deferred to #1284+ (requires topology descriptor for scope resolution).
- **Activation guard** — the second enforcement layer for STIMULUS_DRIVEN (activation-time check). Deferred to #1284 (requires topology lifecycle).
- **YAML surface** — DSL parity for DecisionAuthority. Deferred to agency#385.
- **Scope resolution mechanism** — how the guard looks up an agent's current DecisionScope. Depends on topology descriptor (#1284).

## References

- `specs/issue-1282-coordination-vocabulary-types/SPEC.md` — vocabulary definitions, permission matrix, derivation mapping, deferral decomposition
- `api/src/main/java/io/casehub/api/spi/WorkerExecutionGuard.java` — precedent guard SPI pattern
- `runtime-core/src/main/java/io/casehub/engine/internal/worker/AllowAllWorkerExecutionGuard.java` — precedent NoOp default
- `api/src/main/java/io/casehub/api/spi/ActionRiskClassifier.java` — related action-level gating pattern
- PP-20260921-b7c277 — `@DefaultBean` for 1:1 SPI replacement
- PP-20260727-5267d2 — placement rules: plan-definition types in api, execution types in common
- casehubio/engine#1283 — parent issue
- casehubio/agency#377 — parent epic
