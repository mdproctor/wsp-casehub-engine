# Local Rule Evaluation — Per-Agent Decision Rules

**Issue:** casehubio/engine#1109
**Epic:** casehubio/engine#1104 (Hive Mind)
**Date:** 2026-09-16
**Status:** Draft

## Summary

Per-agent condition→action rules that turn observation into autonomous action. Rules evaluate against coordination state (observations, signals, neighbors) and produce actions that modify the shared environment (deposit signals, register interests, write context). This is the decision layer between perception (#1105-#1108) and coordination (#1110-#1112) — the intelligence that closes the stigmergy loop.

The existing `CaseContextChangedEventHandler` is already a forward-chaining rule system where bindings are rules, CaseContext is the fact space, and worker dispatch is the action. Local rules add a **second fact space** — coordination state — with its own condition→action mechanism. Bindings handle domain-state activation; local rules handle coordination-state activation.

Interim design. Expected to be revisited when Drools vol2 integration provides a more sophisticated rule evaluation engine.

## Architecture

### Rule Model

```java
public record LocalRule(
    String id,
    RuleCondition condition,
    List<RuleAction> actions,
    int priority
) {}
```

`RuleCondition` — sealed interface:
- `ExpressionCondition(ExpressionEvaluator evaluator)` — auditable, YAML-declarable, evaluated against a combined JSON view of coordination state
- `PredicateCondition(Predicate<RuleContext> predicate)` — Java DSL, full type safety, runtime-only

`priority` — higher value evaluates first. Determines execution order, not selection (all matching rules fire).

### RuleContext — The Coordination Fact Space

```java
public record RuleContext(
    List<Observation> observations,
    Map<String, PerceivedSignal> signals,
    JsonNode contextSnapshot,
    Set<String> changedKeys,
    InterestLandscape landscape,
    String agentId,
    String tenancyId,
    UUID caseId
) {}
```

For expression-based conditions, assembled into a combined JSON document:
```json
{
  "observations": [{"patternId": "...", "confidence": 0.9, "details": {...}}],
  "signals": {"danger": {"strength": 0.8, "reinforcements": 3, "age": "PT30S"}},
  "context": { ... working layer ... },
  "changedKeys": ["transaction", "entityResolution"],
  "landscape": {"keyObserverCounts": {"transaction": 3}, "totalObserverCount": 7}
}
```

Neighbors are deliberately excluded from the automatic context. The four `NeighborSpace` queries have different semantics and cost. Agents needing neighbor data in conditions use lambda conditions with explicit `NeighborSpace` calls.

### RuleAction — Sealed Hierarchy

```java
public sealed interface RuleAction {
    record DepositSignal(String name, double strength, @Nullable Duration halfLife)
        implements RuleAction {}
    record RegisterInterest(InterestDeclaration declaration)
        implements RuleAction {}
    record DeregisterInterest(String interestId)
        implements RuleAction {}
    record WriteContext(String key, JsonNode value)
        implements RuleAction {}
}
```

Four action types covering the stigmergy loop:
- **DepositSignal** — announce findings, reinforce coordination signals
- **RegisterInterest** — adapt perception, start watching new patterns
- **DeregisterInterest** — stop watching, reduce observation overhead
- **WriteContext** — bridge coordination state to domain state, triggering `CONTEXT_CHANGED` → binding dispatch

Best practice: use coordination-only actions (DepositSignal, RegisterInterest, DeregisterInterest) by default. Use WriteContext only when a coordination pattern needs to influence case progression directly.

### Firing Semantics

- **Per-agent isolation:** each agent's rules evaluated independently, no cross-agent interaction
- **All-fire:** ALL matching rules fire per cycle (not just the highest-priority one)
- **Priority = execution order:** higher priority rules execute first; within one agent, lower-priority WriteContext overwrites higher-priority on the same key. Cross-agent WriteContext on the same key has undefined ordering — agents should coordinate via signals to avoid key contention
- **One-shot per cycle:** rules evaluate once per evaluation cycle, no intra-cycle chaining
- **Implicit refraction:** the one-cycle delay between rule evaluation and the resulting `CONTEXT_CHANGED` means the environment has changed by the next evaluation — rules don't re-fire on the same unchanged state

This differs from Drools' match-resolve-act model. In swarm systems, conflict resolution happens at the environment level (signal reinforcement/decay), not at the rule engine level. Multiple simultaneous behaviors are desirable (forage AND avoid danger AND follow gradient).

## Pipeline Integration

### Evaluation Cycle Position

Local rule evaluation runs as the fourth phase in `CaseContextChangedEventHandler.evaluateAndDispatch()`:

```
rules()         → binding dispatch (domain state → worker execution)
goals()         → goal condition evaluation
observations()  → observer notification (produces observations)
localRules()    → local rule evaluation (consumes observations, produces actions)
```

### Batched Context Writes

Within `localRules()`:
1. Build `RuleContext` per agent from registries (observations, signals, landscape, context snapshot)
2. Evaluate all agents' rules independently — per-agent timeout 100ms via `CompletableFuture.orTimeout()` on virtual threads
3. Collect all `RuleAction`s from all agents
4. Execute coordination actions immediately (signal deposits, interest changes)
5. Batch all `WriteContext` actions
6. Apply batched writes in a single pass after all rules complete
7. If any writes occurred, publish one `CONTEXT_CHANGED` event

The `CaseEvaluationSerializer`'s `drainPending()` queues the resulting re-evaluation for the next cycle — no re-entrant evaluation.

### Error Isolation

Per-agent try-catch. A failing agent's rule set loses its contribution for that cycle only — other agents and the rest of the pipeline are unaffected. WARN-level logging. Same pattern as observer error isolation (D8).

## RuleSpace Facet

Fourth WorkerRuntime coordination facet, alongside SignalSpace, InterestSpace, and NeighborSpace.

```java
public interface RuleSpace {
    RuleRegistration register(LocalRule rule);
    void deregister(String ruleId);
    List<RuleRegistration> mine();
    List<RuleFiring> lastFired();

    RuleSpace NOOP = new RuleSpace() { /* no-op implementations */ };
}
```

### Supporting Types

```java
public record RuleRegistration(String ruleId, LocalRule rule, Instant registeredAt) {}

public record RuleFiring(String ruleId, List<RuleAction> executedActions, Instant firedAt) {}
```

`lastFired()` returns the rules that fired in the most recent evaluation cycle — gives agents visibility into their own rule behavior for adaptive decision-making.

### WorkerRuntime Accessor

```java
public interface WorkerRuntime extends WorkerScope {
    // ... existing methods ...
    default RuleSpace rules() { return RuleSpace.NOOP; }
}
```

### DefaultRuleSpace

`DefaultRuleSpace` in `runtime-core/internal/observation/` — wraps `RuleRegistry` with case/agent/binding scoping. Created per-invocation by `WorkerRuntimeFactory`. Scope enforcement: BINDING scope rejected with `IllegalStateException`, only COMPOUND or CASE scope allowed.

## RuleRegistry

`RuleRegistry` in `common-core/internal/observation/`, `@ApplicationScoped`, `Resettable`.

### Storage Model

```java
// Per-case, per-agent rule storage
ConcurrentHashMap<UUID, Map<String, List<RuleEntry>>> rules;   // caseId → agentId → rules

// Per-cycle firing results (replaced each cycle)
ConcurrentHashMap<UUID, Map<String, List<RuleFiring>>> firings; // caseId → agentId → firings
```

`RuleEntry` (internal record): `(LocalRule rule, String agentId, String bindingName, Instant registeredAt)`.

### Key Methods

- `registerRule(UUID caseId, String agentId, String bindingName, LocalRule rule, int maxPerCase)` — deduplication by `(caseId, agentId, bindingName, ruleId)`
- `deregisterRule(UUID caseId, String ruleId)`
- `getRulesForCase(UUID caseId)` → `Map<String, List<LocalRule>>` (agentId → rules)
- `getRulesForAgent(UUID caseId, String agentId)` → `List<LocalRule>`
- `storeFirings(UUID caseId, String agentId, List<RuleFiring> firings)` — per-cycle replacement
- `getFirings(UUID caseId, String agentId)` → `List<RuleFiring>`
- `unregisterByAgent(UUID caseId, String agentId)`
- `unregisterByBinding(UUID caseId, Set<String> bindingNames)`
- `evictByCase(UUID caseId)`
- `ruleCount(UUID caseId)` — for cap enforcement

### Per-Cycle Firing Replacement

`storeFirings()` uses `put()` semantics — each evaluation cycle overwrites the previous cycle's firings. Same pattern as `ObservationRegistry.storeObservations()`. A rule that fires in cycle N and doesn't fire in cycle N+1 has no residual trace.

## Configuration

`RuleConfig` record on `CaseDefinition`:

```java
public record RuleConfig(int maxRulesPerCase, int maxActionsPerCycle, int ruleEvaluationTimeoutMs) {
    public static final int DEFAULT_MAX_RULES = 50;
    public static final int DEFAULT_MAX_ACTIONS = 100;
    public static final int DEFAULT_TIMEOUT_MS = 100;
}
```

YAML:
```yaml
spec:
  ruleConfig:
    maxRulesPerCase: 50
    maxActionsPerCycle: 100
    ruleEvaluationTimeoutMs: 100
```

`CaseDefinition.getRuleConfig()` returns defaults when null (same pattern as `getObservationConfig()` and `getSignalConfig()`).

## Lifecycle

### Scope Enforcement

Same as observers (D26): BINDING scope rejected with `IllegalStateException`. Only COMPOUND or CASE scope allowed. Rules need to persist across evaluation cycles to be useful.

### Cleanup

- `CaseStatusChangedHandler` calls `ruleRegistry.evictByCase(caseId)` on terminal case status (COMPLETED, FAULTED, CANCELLED)
- `ScopedWorkerTerminationHandler` calls `ruleRegistry.unregisterByBinding(caseId, bindingNames)` on `COMPOUND_COMPLETED`
- `RuleRegistry implements Resettable` for demo/test replay

### Deduplication

Registration deduplicates by `(caseId, agentId, bindingName, ruleId)`. Re-registration replaces the existing rule (same semantics as observer deduplication, D30). COMPOUND-scoped workers that re-register rules on each dispatch get replace semantics, not accumulation.

## Audit

Two new `CaseHubEventType` values:

- `RULE_REGISTERED` — metadata: `agentId`, `ruleId`, `conditionType` (expression type or "predicate"), `actionCount`, `priority`
- `RULE_FIRED` — metadata: `agentId`, `ruleId`, `actions[]` (action type + parameters), `priority`

Published per firing, not per cycle. EventLog publishing follows the same wiring pattern as `PHEROMONE_DEPOSITED` and `INTEREST_REGISTERED`.

## Module Placement

| Type | Package | Module |
|------|---------|--------|
| `RuleSpace` | `io.casehub.api.engine` | engine-api |
| `LocalRule`, `RuleAction`, `RuleCondition`, `RuleContext`, `RuleFiring`, `RuleConfig`, `RuleRegistration` | `io.casehub.api.spi.observation` | engine-api |
| `RuleRegistry` | `io.casehub.engine.common.internal.observation` | engine-common (common-core) |
| `DefaultRuleSpace` | `io.casehub.engine.internal.observation` | runtime-core |
| `localRules()` handler integration | `io.casehub.engine.internal.engine.handler` | runtime-core |

Follows D23/D36 pattern: facet interfaces in `api/engine`, domain types in `api/spi/observation`, mutable state management in `common-core`, handler integration in `runtime-core`.

## Not In Scope (v1)

- **YAML rule declaration** — rules are runtime-registered by agents via `RuleSpace.register()`. Static YAML rules are a natural extension for v2.
- **Learned rules from CBR** — CBR-generated rules would be dynamically registered. The mechanism exists via `RuleSpace.register()` but the CBR → rule generation pipeline is out of scope.
- **Rule groups / agenda groups** — no partitioning of rules within an agent. All registered rules evaluate every cycle. Extension point for Drools vol2.
- **Cross-agent rule visibility** — agents cannot see other agents' rules or firings. Per-agent isolation.
- **Rule chaining within a cycle** — one-shot evaluation. Chaining is deferred to Drools vol2.

## Relationship to Drools (#445)

Local rules (#1109) and Drools (#445) are complementary, not overlapping:

| Aspect | Local Rules (#1109) | Drools (#445) |
|--------|-------------------|---------------|
| Fact space | Coordination state (observations, signals, neighbors) | Typed domain facts (Java POJOs in working memory) |
| Evaluation | Per-agent, all-fire, one-shot | Rete network, conflict resolution, agenda |
| Conditions | JQ/MVEL expressions + lambdas | DRL rules with pattern matching |
| Actions | Deposit signals, register interests, write context | Modify working memory facts, trigger consequences |
| Scope | Engine-internal coordination | Full production rule system |

When Drools vol2 arrives, it could serve as a `RuleEvaluationStrategy` that replaces the simple all-fire evaluator with Rete-based match-resolve-act for domains that need it. The `RuleSpace` facet, `RuleRegistry`, and pipeline integration would remain — only the evaluation engine changes.

## References

- `CaseContextChangedEventHandler.java:243-253` — existing evaluation pipeline (rules → goals → observations)
- `CaseEvaluationSerializer.java:35-66` — per-case serialization gate with drainPending
- `ObservationRegistry.java` — registry pattern, per-cycle replacement semantics
- `SignalRegistry.java` — deposit/perceive API, reinforcement semantics
- `WorkerRuntime.java` — existing three facets (SignalSpace, InterestSpace, NeighborSpace)
- `WorkerRuntimeFactory.java:82` — facet creation and wiring
- `ExpressionEngine.java:36` — pluggable expression evaluation SPI
- `DefaultExpressionEngineRegistry.java` — CDI-discovered expression engines
- Blog: "The Data Store Drools Actually Needs" (2026-06-08) — Drools vs expression evaluation distinction
- ADR-0009 — expressionLang granularity (per-definition vs per-expression)
- D37-D44 — design decisions for this issue
- SwarmSys (arXiv:2510.10047) — Explorer/Worker/Validator roles, pheromone-inspired reinforcement
- engine#1111 — stigmergy execution model (downstream consumer)
- engine#1112 — swarm execution model (downstream consumer)
- engine#445 — Drools integration epic (complementary, not overlapping)
