# Local Rule Evaluation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/engine#1109 — Local rule evaluation — per-agent decision rules
**Issue group:** casehubio/engine#1104 (Hive Mind epic), casehubio/engine#1105-#1108 (predecessors)

**Goal:** Add per-agent condition→action rules that turn observation into autonomous action, closing the stigmergy perceive→decide→act loop.

**Architecture:** Fourth WorkerRuntime coordination facet (`RuleSpace`) with engine-side evaluation after the observation pipeline. Rules are registered by agents at runtime via `RuleSpace.register()`. Conditions evaluate against a `RuleContext` (observations, signals, context snapshot). Actions modify coordination state (deposit signals, register interests) or domain state (write context keys). Batched context writes publish a single `CONTEXT_CHANGED` post-evaluation.

**Tech Stack:** Java 21, Quarkus 3.32.2, virtual threads, ConcurrentHashMap-based registries

## Global Constraints

- Follow established facet pattern from `SignalSpace`/`InterestSpace`/`NeighborSpace`
- No YAML rule declaration in v1 — runtime registration only
- BINDING scope rejected for rule registration — COMPOUND or CASE only
- All commits reference `Refs casehubio/engine#1109`
- Tests in the module where the code lives (`api`, `common-core`, `runtime-core`)

---

## Batch 1: Foundation types (engine-api)

### Task 1: RuleAction, RuleCondition, RuleContext, LocalRule, RuleFiring, RuleRegistration, RuleConfig

**Files:**
- Create: `api/src/main/java/io/casehub/api/spi/observation/RuleAction.java`
- Create: `api/src/main/java/io/casehub/api/spi/observation/RuleCondition.java`
- Create: `api/src/main/java/io/casehub/api/spi/observation/RuleContext.java`
- Create: `api/src/main/java/io/casehub/api/spi/observation/LocalRule.java`
- Create: `api/src/main/java/io/casehub/api/spi/observation/RuleFiring.java`
- Create: `api/src/main/java/io/casehub/api/spi/observation/RuleRegistration.java`
- Create: `api/src/main/java/io/casehub/api/spi/observation/RuleConfig.java`
- Test: `api/src/test/java/io/casehub/api/spi/observation/LocalRuleTest.java`

**Interfaces:**
- Consumes: `InterestDeclaration` (`api/spi/observation`), `Observation` (`api/spi/observation`), `PerceivedSignal` (`api/model/signal`), `InterestLandscape` (`api/spi/observation`), `ExpressionEvaluator` (`platform-api`)
- Produces: `RuleAction` (sealed: `DepositSignal`, `RegisterInterest`, `DeregisterInterest`, `WriteContext`), `RuleCondition` (sealed: `ExpressionCondition`, `PredicateCondition`), `RuleContext` (record), `LocalRule` (record), `RuleFiring` (record), `RuleRegistration` (record), `RuleConfig` (record with defaults)

- [ ] **Step 1: Write tests for the foundation types**

```java
package io.casehub.api.spi.observation;

import static org.junit.jupiter.api.Assertions.*;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.node.IntNode;
import com.fasterxml.jackson.databind.node.TextNode;
import io.casehub.api.model.signal.PerceivedSignal;
import io.casehub.platform.api.expression.ExpressionEvaluator;
import java.time.Duration;
import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.UUID;
import java.util.function.Predicate;
import org.junit.jupiter.api.Test;

class LocalRuleTest {

  @Test
  void depositSignalAction() {
    var action = new RuleAction.DepositSignal("danger", 0.8, null);
    assertEquals("danger", action.name());
    assertEquals(0.8, action.strength());
    assertNull(action.halfLife());
  }

  @Test
  void depositSignalActionWithHalfLife() {
    var action = new RuleAction.DepositSignal("danger", 0.8, Duration.ofMinutes(5));
    assertEquals(Duration.ofMinutes(5), action.halfLife());
  }

  @Test
  void writeContextAction() {
    var action = new RuleAction.WriteContext("status", new TextNode("alert"));
    assertEquals("status", action.key());
    assertEquals("alert", action.value().asText());
  }

  @Test
  void registerInterestAction() {
    var interest = new InterestDeclaration.KeyThreshold(
        "temperature", InterestDeclaration.ComparisonOperator.GT, 100.0);
    var action = new RuleAction.RegisterInterest(interest);
    assertEquals(interest, action.declaration());
  }

  @Test
  void deregisterInterestAction() {
    var action = new RuleAction.DeregisterInterest("interest-42");
    assertEquals("interest-42", action.interestId());
  }

  @Test
  void expressionCondition() {
    ExpressionEvaluator eval = new ExpressionEvaluator() {
      @Override public String type() { return "test"; }
      @Override public String expression() { return ".signals.danger.strength > 0.5"; }
    };
    var cond = new RuleCondition.ExpressionCondition(eval);
    assertEquals(eval, cond.evaluator());
  }

  @Test
  void predicateCondition() {
    Predicate<RuleContext> pred = ctx -> !ctx.observations().isEmpty();
    var cond = new RuleCondition.PredicateCondition(pred);
    assertEquals(pred, cond.predicate());
  }

  @Test
  void localRuleConstruction() {
    var condition = new RuleCondition.PredicateCondition(ctx -> true);
    var actions = List.<RuleAction>of(new RuleAction.DepositSignal("found", 1.0, null));
    var rule = new LocalRule("rule-1", condition, actions, 10);
    assertEquals("rule-1", rule.id());
    assertEquals(10, rule.priority());
    assertEquals(1, rule.actions().size());
  }

  @Test
  void localRuleActionsAreImmutable() {
    var actions = new java.util.ArrayList<RuleAction>();
    actions.add(new RuleAction.DepositSignal("a", 1.0, null));
    var rule = new LocalRule("rule-1", new RuleCondition.PredicateCondition(ctx -> true), actions, 0);
    assertThrows(UnsupportedOperationException.class, () -> rule.actions().add(
        new RuleAction.DepositSignal("b", 1.0, null)));
  }

  @Test
  void ruleContextConstruction() {
    var ctx = new RuleContext(
        List.of(), Map.of(), null, Set.of(), InterestLandscape.EMPTY,
        "agent-1", "tenant-1", UUID.randomUUID());
    assertEquals("agent-1", ctx.agentId());
    assertTrue(ctx.observations().isEmpty());
  }

  @Test
  void ruleFiringConstruction() {
    var actions = List.<RuleAction>of(new RuleAction.DepositSignal("x", 0.5, null));
    var firing = new RuleFiring("rule-1", actions, Instant.now());
    assertEquals("rule-1", firing.ruleId());
    assertEquals(1, firing.executedActions().size());
  }

  @Test
  void ruleRegistrationConstruction() {
    var rule = new LocalRule("r1",
        new RuleCondition.PredicateCondition(ctx -> true),
        List.of(), 0);
    var reg = new RuleRegistration("r1", rule, Instant.now());
    assertEquals("r1", reg.ruleId());
  }

  @Test
  void ruleConfigDefaults() {
    var config = RuleConfig.defaults();
    assertEquals(50, config.maxRulesPerCase());
    assertEquals(100, config.maxActionsPerCycle());
    assertEquals(100, config.ruleEvaluationTimeoutMs());
  }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl api -Dtest=LocalRuleTest -DfailIfNoTests=false -q`
Expected: FAIL — classes not found

- [ ] **Step 3: Implement the foundation types**

Create `RuleAction.java`:
```java
package io.casehub.api.spi.observation;

import com.fasterxml.jackson.databind.JsonNode;
import jakarta.annotation.Nullable;
import java.time.Duration;

public sealed interface RuleAction {

  record DepositSignal(String name, double strength, @Nullable Duration halfLife)
      implements RuleAction {}

  record RegisterInterest(InterestDeclaration declaration) implements RuleAction {}

  record DeregisterInterest(String interestId) implements RuleAction {}

  record WriteContext(String key, JsonNode value) implements RuleAction {}
}
```

Create `RuleCondition.java`:
```java
package io.casehub.api.spi.observation;

import io.casehub.platform.api.expression.ExpressionEvaluator;
import java.util.function.Predicate;

public sealed interface RuleCondition {

  record ExpressionCondition(ExpressionEvaluator evaluator) implements RuleCondition {}

  record PredicateCondition(Predicate<RuleContext> predicate) implements RuleCondition {}
}
```

Create `RuleContext.java`:
```java
package io.casehub.api.spi.observation;

import com.fasterxml.jackson.databind.JsonNode;
import io.casehub.api.model.signal.PerceivedSignal;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.UUID;

public record RuleContext(
    List<Observation> observations,
    Map<String, PerceivedSignal> signals,
    JsonNode contextSnapshot,
    Set<String> changedKeys,
    InterestLandscape landscape,
    String agentId,
    String tenancyId,
    UUID caseId) {}
```

Create `LocalRule.java`:
```java
package io.casehub.api.spi.observation;

import java.util.List;

public record LocalRule(String id, RuleCondition condition, List<RuleAction> actions, int priority) {

  public LocalRule {
    if (id == null || id.isBlank()) {
      throw new IllegalArgumentException("rule id must not be null or blank");
    }
    if (condition == null) {
      throw new IllegalArgumentException("condition must not be null");
    }
    actions = List.copyOf(actions);
  }
}
```

Create `RuleFiring.java`:
```java
package io.casehub.api.spi.observation;

import java.time.Instant;
import java.util.List;

public record RuleFiring(String ruleId, List<RuleAction> executedActions, Instant firedAt) {

  public RuleFiring {
    executedActions = List.copyOf(executedActions);
  }
}
```

Create `RuleRegistration.java`:
```java
package io.casehub.api.spi.observation;

import java.time.Instant;

public record RuleRegistration(String ruleId, LocalRule rule, Instant registeredAt) {}
```

Create `RuleConfig.java`:
```java
package io.casehub.api.spi.observation;

public record RuleConfig(int maxRulesPerCase, int maxActionsPerCycle, int ruleEvaluationTimeoutMs) {

  public static final int DEFAULT_MAX_RULES = 50;
  public static final int DEFAULT_MAX_ACTIONS = 100;
  public static final int DEFAULT_TIMEOUT_MS = 100;

  public static RuleConfig defaults() {
    return new RuleConfig(DEFAULT_MAX_RULES, DEFAULT_MAX_ACTIONS, DEFAULT_TIMEOUT_MS);
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -pl api -Dtest=LocalRuleTest -q`
Expected: PASS — all tests green

- [ ] **Step 5: Add RuleConfig to CaseDefinition + RuleSpace to WorkerRuntime**

Modify `api/src/main/java/io/casehub/api/model/CaseDefinition.java`:
- Add field: `private RuleConfig ruleConfig;` (after `signalConfig` at line ~197)
- Add getter with defaults: `getRuleConfig()` returns `ruleConfig != null ? ruleConfig : RuleConfig.defaults()`
- Add setter: `setRuleConfig(RuleConfig)`
- Add builder field, method, and `build()` wiring (same pattern as `signalConfig`)

Create `api/src/main/java/io/casehub/api/engine/RuleSpace.java`:
```java
package io.casehub.api.engine;

import io.casehub.api.spi.observation.LocalRule;
import io.casehub.api.spi.observation.RuleFiring;
import io.casehub.api.spi.observation.RuleRegistration;
import java.time.Instant;
import java.util.List;

public interface RuleSpace {

  RuleRegistration register(LocalRule rule);

  void deregister(String ruleId);

  List<RuleRegistration> mine();

  List<RuleFiring> lastFired();

  RuleSpace NOOP =
      new RuleSpace() {
        @Override
        public RuleRegistration register(LocalRule rule) {
          return new RuleRegistration(rule.id(), rule, Instant.now());
        }

        @Override
        public void deregister(String ruleId) {}

        @Override
        public List<RuleRegistration> mine() {
          return List.of();
        }

        @Override
        public List<RuleFiring> lastFired() {
          return List.of();
        }
      };
}
```

Modify `api/src/main/java/io/casehub/api/engine/WorkerRuntime.java` — add accessor:
```java
default RuleSpace rules() {
    return RuleSpace.NOOP;
}
```

- [ ] **Step 6: Add RULE_FIRED and RULE_REGISTERED to CaseHubEventType**

Modify `api/src/main/java/io/casehub/api/model/event/CaseHubEventType.java` — add two new enum constants:
```java
RULE_REGISTERED,
RULE_FIRED,
```

- [ ] **Step 7: Run full api module tests**

Run: `mvn test -pl api -q`
Expected: PASS — no regressions

- [ ] **Step 8: Commit**

```bash
git add api/src/main/java/io/casehub/api/spi/observation/RuleAction.java \
  api/src/main/java/io/casehub/api/spi/observation/RuleCondition.java \
  api/src/main/java/io/casehub/api/spi/observation/RuleContext.java \
  api/src/main/java/io/casehub/api/spi/observation/LocalRule.java \
  api/src/main/java/io/casehub/api/spi/observation/RuleFiring.java \
  api/src/main/java/io/casehub/api/spi/observation/RuleRegistration.java \
  api/src/main/java/io/casehub/api/spi/observation/RuleConfig.java \
  api/src/main/java/io/casehub/api/engine/RuleSpace.java \
  api/src/main/java/io/casehub/api/model/CaseDefinition.java \
  api/src/main/java/io/casehub/api/model/event/CaseHubEventType.java \
  api/src/main/java/io/casehub/api/engine/WorkerRuntime.java \
  api/src/test/java/io/casehub/api/spi/observation/LocalRuleTest.java
git commit -m "feat: add local rule foundation types + RuleSpace facet + RuleConfig

RuleAction sealed (DepositSignal, RegisterInterest, DeregisterInterest,
WriteContext). RuleCondition sealed (ExpressionCondition, PredicateCondition).
RuleContext record for coordination fact space. RuleSpace 4th WorkerRuntime
facet. RuleConfig on CaseDefinition with defaults.

Refs casehubio/engine#1109"
```

## Batch 2: RuleRegistry + DefaultRuleSpace (common-core + runtime-core)

### Task 2: RuleRegistry

**Files:**
- Create: `common-core/src/main/java/io/casehub/engine/common/internal/observation/RuleRegistry.java`
- Test: `common-core/src/test/java/io/casehub/engine/common/internal/observation/RuleRegistryTest.java`

**Interfaces:**
- Consumes: `LocalRule`, `RuleFiring`, `Resettable` (`common/spi`)
- Produces: `RuleRegistry` — `@ApplicationScoped`, `Resettable`. Methods: `registerRule(UUID, String, String, LocalRule, int) → String`, `deregisterRule(UUID, String)`, `getRulesForCase(UUID) → Map<String, List<LocalRule>>`, `getRulesForAgent(UUID, String) → List<LocalRule>`, `storeFirings(UUID, String, List<RuleFiring>)`, `getFirings(UUID, String) → List<RuleFiring>`, `unregisterByAgent(UUID, String)`, `unregisterByBinding(UUID, Set<String>)`, `evictByCase(UUID)`, `ruleCount(UUID) → int`, `reset()`

- [ ] **Step 1: Write the failing tests**

```java
package io.casehub.engine.common.internal.observation;

import static org.junit.jupiter.api.Assertions.*;

import io.casehub.api.spi.observation.*;
import java.time.Instant;
import java.util.List;
import java.util.Set;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class RuleRegistryTest {

  private RuleRegistry registry;
  private UUID caseId;

  @BeforeEach
  void setUp() {
    registry = new RuleRegistry();
    caseId = UUID.randomUUID();
  }

  private LocalRule rule(String id, int priority) {
    return new LocalRule(id,
        new RuleCondition.PredicateCondition(ctx -> true),
        List.of(new RuleAction.DepositSignal("s", 1.0, null)),
        priority);
  }

  @Test
  void registerAndRetrieve() {
    String ruleId = registry.registerRule(caseId, "agent-1", "binding-a", rule("r1", 0), 50);
    assertNotNull(ruleId);
    var rules = registry.getRulesForAgent(caseId, "agent-1");
    assertEquals(1, rules.size());
    assertEquals("r1", rules.get(0).id());
  }

  @Test
  void respectsMaxPerCase() {
    for (int i = 0; i < 3; i++) {
      registry.registerRule(caseId, "agent-1", "binding-a", rule("r" + i, 0), 3);
    }
    String overflow = registry.registerRule(caseId, "agent-1", "binding-a", rule("r-overflow", 0), 3);
    assertNull(overflow);
    assertEquals(3, registry.ruleCount(caseId));
  }

  @Test
  void deduplicatesByRuleId() {
    registry.registerRule(caseId, "agent-1", "binding-a", rule("r1", 5), 50);
    registry.registerRule(caseId, "agent-1", "binding-a", rule("r1", 10), 50);
    var rules = registry.getRulesForAgent(caseId, "agent-1");
    assertEquals(1, rules.size());
    assertEquals(10, rules.get(0).priority());
  }

  @Test
  void deregisterByRuleId() {
    registry.registerRule(caseId, "agent-1", "binding-a", rule("r1", 0), 50);
    registry.deregisterRule(caseId, "r1");
    assertTrue(registry.getRulesForAgent(caseId, "agent-1").isEmpty());
  }

  @Test
  void getRulesForCaseGroupsByAgent() {
    registry.registerRule(caseId, "agent-1", "b", rule("r1", 0), 50);
    registry.registerRule(caseId, "agent-2", "b", rule("r2", 0), 50);
    var byAgent = registry.getRulesForCase(caseId);
    assertEquals(2, byAgent.size());
    assertTrue(byAgent.containsKey("agent-1"));
    assertTrue(byAgent.containsKey("agent-2"));
  }

  @Test
  void storeFiringsAndRetrieve() {
    var firings = List.of(new RuleFiring("r1",
        List.of(new RuleAction.DepositSignal("x", 0.5, null)), Instant.now()));
    registry.storeFirings(caseId, "agent-1", firings);
    var retrieved = registry.getFirings(caseId, "agent-1");
    assertEquals(1, retrieved.size());
    assertEquals("r1", retrieved.get(0).ruleId());
  }

  @Test
  void storeFiringsReplacesPerCycle() {
    registry.storeFirings(caseId, "agent-1",
        List.of(new RuleFiring("r1", List.of(), Instant.now())));
    registry.storeFirings(caseId, "agent-1",
        List.of(new RuleFiring("r2", List.of(), Instant.now())));
    var retrieved = registry.getFirings(caseId, "agent-1");
    assertEquals(1, retrieved.size());
    assertEquals("r2", retrieved.get(0).ruleId());
  }

  @Test
  void unregisterByAgent() {
    registry.registerRule(caseId, "agent-1", "b", rule("r1", 0), 50);
    registry.registerRule(caseId, "agent-2", "b", rule("r2", 0), 50);
    registry.unregisterByAgent(caseId, "agent-1");
    assertTrue(registry.getRulesForAgent(caseId, "agent-1").isEmpty());
    assertEquals(1, registry.getRulesForAgent(caseId, "agent-2").size());
  }

  @Test
  void unregisterByBinding() {
    registry.registerRule(caseId, "agent-1", "binding-a", rule("r1", 0), 50);
    registry.registerRule(caseId, "agent-1", "binding-b", rule("r2", 0), 50);
    registry.unregisterByBinding(caseId, Set.of("binding-a"));
    var rules = registry.getRulesForAgent(caseId, "agent-1");
    assertEquals(1, rules.size());
    assertEquals("r2", rules.get(0).id());
  }

  @Test
  void evictByCase() {
    registry.registerRule(caseId, "agent-1", "b", rule("r1", 0), 50);
    registry.storeFirings(caseId, "agent-1",
        List.of(new RuleFiring("r1", List.of(), Instant.now())));
    registry.evictByCase(caseId);
    assertTrue(registry.getRulesForAgent(caseId, "agent-1").isEmpty());
    assertTrue(registry.getFirings(caseId, "agent-1").isEmpty());
    assertEquals(0, registry.ruleCount(caseId));
  }

  @Test
  void resetClearsEverything() {
    registry.registerRule(caseId, "agent-1", "b", rule("r1", 0), 50);
    registry.reset();
    assertEquals(0, registry.ruleCount(caseId));
  }

  @Test
  void emptyReturnsForUnknownCase() {
    assertTrue(registry.getRulesForCase(UUID.randomUUID()).isEmpty());
    assertTrue(registry.getRulesForAgent(UUID.randomUUID(), "x").isEmpty());
    assertTrue(registry.getFirings(UUID.randomUUID(), "x").isEmpty());
    assertEquals(0, registry.ruleCount(UUID.randomUUID()));
  }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl common-core -Dtest=RuleRegistryTest -DfailIfNoTests=false -q`
Expected: FAIL — class not found

- [ ] **Step 3: Implement RuleRegistry**

```java
package io.casehub.engine.common.internal.observation;

import io.casehub.api.spi.observation.LocalRule;
import io.casehub.api.spi.observation.RuleFiring;
import io.casehub.engine.common.spi.Resettable;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

@ApplicationScoped
public class RuleRegistry implements Resettable {

  record RuleEntry(LocalRule rule, String agentId, String bindingName) {}

  private final ConcurrentHashMap<UUID, List<RuleEntry>> rules = new ConcurrentHashMap<>();
  private final ConcurrentHashMap<UUID, ConcurrentHashMap<String, List<RuleFiring>>> firings =
      new ConcurrentHashMap<>();

  public String registerRule(
      UUID caseId, String agentId, String bindingName, LocalRule rule, int maxPerCase) {
    var caseRules =
        rules.computeIfAbsent(caseId, k -> Collections.synchronizedList(new ArrayList<>()));
    synchronized (caseRules) {
      caseRules.removeIf(
          e -> e.agentId().equals(agentId)
              && e.bindingName().equals(bindingName)
              && e.rule().id().equals(rule.id()));
      if (caseRules.size() >= maxPerCase) {
        return null;
      }
      caseRules.add(new RuleEntry(rule, agentId, bindingName));
      return rule.id();
    }
  }

  public void deregisterRule(UUID caseId, String ruleId) {
    var caseRules = rules.get(caseId);
    if (caseRules != null) {
      synchronized (caseRules) {
        caseRules.removeIf(e -> e.rule().id().equals(ruleId));
      }
    }
  }

  public Map<String, List<LocalRule>> getRulesForCase(UUID caseId) {
    var caseRules = rules.get(caseId);
    if (caseRules == null) {
      return Map.of();
    }
    Map<String, List<LocalRule>> result = new LinkedHashMap<>();
    synchronized (caseRules) {
      for (var entry : caseRules) {
        result.computeIfAbsent(entry.agentId(), k -> new ArrayList<>()).add(entry.rule());
      }
    }
    return result;
  }

  public List<LocalRule> getRulesForAgent(UUID caseId, String agentId) {
    var caseRules = rules.get(caseId);
    if (caseRules == null) {
      return List.of();
    }
    synchronized (caseRules) {
      return caseRules.stream()
          .filter(e -> e.agentId().equals(agentId))
          .map(RuleEntry::rule)
          .toList();
    }
  }

  public void storeFirings(UUID caseId, String agentId, List<RuleFiring> agentFirings) {
    firings
        .computeIfAbsent(caseId, k -> new ConcurrentHashMap<>())
        .put(agentId, List.copyOf(agentFirings));
  }

  public List<RuleFiring> getFirings(UUID caseId, String agentId) {
    var caseFirings = firings.get(caseId);
    if (caseFirings == null) {
      return List.of();
    }
    return caseFirings.getOrDefault(agentId, List.of());
  }

  public void unregisterByAgent(UUID caseId, String agentId) {
    var caseRules = rules.get(caseId);
    if (caseRules != null) {
      synchronized (caseRules) {
        caseRules.removeIf(e -> e.agentId().equals(agentId));
      }
    }
  }

  public void unregisterByBinding(UUID caseId, Set<String> bindingNames) {
    var caseRules = rules.get(caseId);
    if (caseRules != null) {
      synchronized (caseRules) {
        caseRules.removeIf(e -> bindingNames.contains(e.bindingName()));
      }
    }
  }

  public void evictByCase(UUID caseId) {
    rules.remove(caseId);
    firings.remove(caseId);
  }

  public int ruleCount(UUID caseId) {
    var caseRules = rules.get(caseId);
    return caseRules == null ? 0 : caseRules.size();
  }

  @Override
  public void reset() {
    rules.clear();
    firings.clear();
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -pl common-core -Dtest=RuleRegistryTest -q`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add common-core/src/main/java/io/casehub/engine/common/internal/observation/RuleRegistry.java \
  common-core/src/test/java/io/casehub/engine/common/internal/observation/RuleRegistryTest.java
git commit -m "feat: implement RuleRegistry with per-case per-agent rule storage

ConcurrentHashMap-based, Resettable, deduplication by (agentId, bindingName,
ruleId), per-cycle firing replacement, maxPerCase cap enforcement.

Refs casehubio/engine#1109"
```

### Task 3: DefaultRuleSpace + WorkerRuntime wiring

**Files:**
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/observation/DefaultRuleSpace.java`
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/executor/DefaultWorkerRuntime.java` — add `ruleSpace` field + constructor param + accessor
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/executor/WorkerRuntimeFactory.java` — inject `RuleRegistry`, create `DefaultRuleSpace`, pass to `DefaultWorkerRuntime`
- Test: `runtime-core/src/test/java/io/casehub/engine/internal/observation/DefaultRuleSpaceTest.java`

**Interfaces:**
- Consumes: `RuleSpace` (`api/engine`), `RuleRegistry` (`common-core`), `LocalRule`, `RuleConfig`, `RuleRegistration`, `RuleFiring`
- Produces: `DefaultRuleSpace` — implements `RuleSpace`, delegates to `RuleRegistry` with case/agent/binding scoping and scope enforcement

- [ ] **Step 1: Write the failing tests**

```java
package io.casehub.engine.internal.observation;

import static org.junit.jupiter.api.Assertions.*;

import io.casehub.api.spi.observation.*;
import io.casehub.engine.common.internal.observation.RuleRegistry;
import java.util.List;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class DefaultRuleSpaceTest {

  private RuleRegistry registry;
  private DefaultRuleSpace ruleSpace;
  private UUID caseId;

  @BeforeEach
  void setUp() {
    registry = new RuleRegistry();
    caseId = UUID.randomUUID();
    ruleSpace = new DefaultRuleSpace(
        registry, caseId, "agent-1", "binding-a",
        RuleConfig.defaults());
  }

  private LocalRule rule(String id) {
    return new LocalRule(id,
        new RuleCondition.PredicateCondition(ctx -> true),
        List.of(new RuleAction.DepositSignal("s", 1.0, null)), 0);
  }

  @Test
  void registerAndMine() {
    var reg = ruleSpace.register(rule("r1"));
    assertNotNull(reg);
    assertEquals("r1", reg.ruleId());
    var mine = ruleSpace.mine();
    assertEquals(1, mine.size());
  }

  @Test
  void deregister() {
    ruleSpace.register(rule("r1"));
    ruleSpace.deregister("r1");
    assertTrue(ruleSpace.mine().isEmpty());
  }

  @Test
  void lastFiredInitiallyEmpty() {
    assertTrue(ruleSpace.lastFired().isEmpty());
  }

  @Test
  void lastFiredReturnsFiringsForThisAgent() {
    registry.storeFirings(caseId, "agent-1",
        List.of(new RuleFiring("r1", List.of(), java.time.Instant.now())));
    assertEquals(1, ruleSpace.lastFired().size());
  }

  @Test
  void lastFiredDoesNotReturnOtherAgentFirings() {
    registry.storeFirings(caseId, "agent-2",
        List.of(new RuleFiring("r1", List.of(), java.time.Instant.now())));
    assertTrue(ruleSpace.lastFired().isEmpty());
  }

  @Test
  void respectsMaxRulesPerCase() {
    var smallConfig = new RuleConfig(2, 100, 100);
    var space = new DefaultRuleSpace(registry, caseId, "agent-1", "binding-a", smallConfig);
    space.register(rule("r1"));
    space.register(rule("r2"));
    var reg = space.register(rule("r3"));
    assertNull(reg);
  }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl runtime-core -Dtest=DefaultRuleSpaceTest -DfailIfNoTests=false -q`
Expected: FAIL — class not found

- [ ] **Step 3: Implement DefaultRuleSpace**

```java
package io.casehub.engine.internal.observation;

import io.casehub.api.engine.RuleSpace;
import io.casehub.api.spi.observation.*;
import io.casehub.engine.common.internal.observation.RuleRegistry;
import java.time.Instant;
import java.util.List;
import java.util.UUID;

public class DefaultRuleSpace implements RuleSpace {

  private final RuleRegistry registry;
  private final UUID caseId;
  private final String agentId;
  private final String bindingName;
  private final RuleConfig config;

  public DefaultRuleSpace(
      RuleRegistry registry, UUID caseId, String agentId, String bindingName, RuleConfig config) {
    this.registry = registry;
    this.caseId = caseId;
    this.agentId = agentId;
    this.bindingName = bindingName;
    this.config = config;
  }

  @Override
  public RuleRegistration register(LocalRule rule) {
    String ruleId =
        registry.registerRule(caseId, agentId, bindingName, rule, config.maxRulesPerCase());
    if (ruleId == null) {
      return null;
    }
    return new RuleRegistration(ruleId, rule, Instant.now());
  }

  @Override
  public void deregister(String ruleId) {
    registry.deregisterRule(caseId, ruleId);
  }

  @Override
  public List<RuleRegistration> mine() {
    return registry.getRulesForAgent(caseId, agentId).stream()
        .map(r -> new RuleRegistration(r.id(), r, Instant.now()))
        .toList();
  }

  @Override
  public List<RuleFiring> lastFired() {
    return registry.getFirings(caseId, agentId);
  }
}
```

- [ ] **Step 4: Wire into DefaultWorkerRuntime and WorkerRuntimeFactory**

Modify `DefaultWorkerRuntime.java`:
- Add field: `private final RuleSpace ruleSpace;` (after `neighborSpace` at line 60)
- Add 14-arg constructor that accepts `RuleSpace ruleSpace` as the last param (after the existing 13-arg constructor). The 13-arg and 10-arg constructors pass `RuleSpace.NOOP`.
- Add accessor: `@Override public RuleSpace rules() { return ruleSpace; }`

Modify `WorkerRuntimeFactory.java`:
- Add field: `private final RuleRegistry ruleRegistry;` (after `planItemStore` at line 36)
- Add `RuleRegistry ruleRegistry` to the constructor (after `PlanItemStore planItemStore`) and assign
- In the 6-arg `create()` method (line 82), after creating `DefaultNeighborSpace`:
  - Add `resolveRuleConfig(caseId)` call (private method, same pattern as `resolveSignalConfig`)
  - Create `new DefaultRuleSpace(ruleRegistry, caseId, workerName, bindingName, resolvedRuleConfig)`
  - Pass to the new 14-arg `DefaultWorkerRuntime` constructor
- Add `private RuleConfig resolveRuleConfig(UUID caseId)` method (same pattern as `resolveSignalConfig`)

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn test -pl runtime-core -Dtest=DefaultRuleSpaceTest -q`
Expected: PASS

- [ ] **Step 6: Run full common-core and runtime-core tests**

Run: `mvn test -pl common-core,runtime-core -q`
Expected: PASS — no regressions

- [ ] **Step 7: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/engine/internal/observation/DefaultRuleSpace.java \
  runtime-core/src/main/java/io/casehub/engine/internal/executor/DefaultWorkerRuntime.java \
  runtime-core/src/main/java/io/casehub/engine/internal/executor/WorkerRuntimeFactory.java \
  runtime-core/src/test/java/io/casehub/engine/internal/observation/DefaultRuleSpaceTest.java
git commit -m "feat: implement DefaultRuleSpace + wire into WorkerRuntime pipeline

DefaultRuleSpace delegates to RuleRegistry with case/agent/binding scoping.
WorkerRuntimeFactory creates DefaultRuleSpace per invocation. DefaultWorkerRuntime
gains 14-arg constructor with RuleSpace as 4th facet.

Refs casehubio/engine#1109"
```

## Batch 3: Pipeline integration + lifecycle

### Task 4: localRules() in CaseContextChangedEventHandler + lifecycle cleanup

**Files:**
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java` — add `localRules()` method, inject `RuleRegistry`, call after `observations()`
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/engine/handler/CaseStatusChangedHandler.java` — call `ruleRegistry.evictByCase()` on terminal status
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/engine/handler/ScopedWorkerTerminationHandler.java` — call `ruleRegistry.unregisterByBinding()` on compound completion
- Test: `runtime-core/src/test/java/io/casehub/engine/internal/observation/LocalRuleEvaluationTest.java`

**Interfaces:**
- Consumes: `RuleRegistry`, `ObservationRegistry`, `SignalRegistry`, `ExpressionEngineRegistry`, `LocalRule`, `RuleAction`, `RuleCondition`, `RuleContext`, `RuleFiring`, `RuleConfig`
- Produces: `localRules()` method in `CaseContextChangedEventHandler` — evaluates per-agent rules, executes coordination actions immediately, batches WriteContext actions, publishes single CONTEXT_CHANGED if writes occurred

- [ ] **Step 1: Write the failing test for rule evaluation**

```java
package io.casehub.engine.internal.observation;

import static org.junit.jupiter.api.Assertions.*;

import com.fasterxml.jackson.databind.node.IntNode;
import com.fasterxml.jackson.databind.node.JsonNodeFactory;
import io.casehub.api.spi.observation.*;
import io.casehub.engine.common.internal.observation.ObservationRegistry;
import io.casehub.engine.common.internal.observation.RuleRegistry;
import io.casehub.engine.common.internal.signal.SignalRegistry;
import java.time.Duration;
import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class LocalRuleEvaluationTest {

  private RuleRegistry ruleRegistry;
  private SignalRegistry signalRegistry;
  private ObservationRegistry observationRegistry;
  private UUID caseId;

  @BeforeEach
  void setUp() {
    ruleRegistry = new RuleRegistry();
    signalRegistry = new SignalRegistry();
    observationRegistry = new ObservationRegistry();
    caseId = UUID.randomUUID();
  }

  @Test
  void predicateConditionMatchesAndFiresDepositSignal() {
    var rule = new LocalRule("r1",
        new RuleCondition.PredicateCondition(ctx -> !ctx.observations().isEmpty()),
        List.of(new RuleAction.DepositSignal("alert", 0.9, null)),
        0);
    ruleRegistry.registerRule(caseId, "agent-1", "binding-a", rule, 50);

    observationRegistry.storeObservations(caseId, "agent-1",
        List.of(new Observation("pattern-1", 0.8, Map.of(), Instant.now())));

    var evaluator = new LocalRuleEvaluator(signalRegistry);
    var ruleContext = new RuleContext(
        observationRegistry.getObservations(caseId, "agent-1"),
        Map.of(), JsonNodeFactory.instance.objectNode(), Set.of(),
        InterestLandscape.EMPTY, "agent-1", "tenant-1", caseId);

    var firings = evaluator.evaluate("agent-1", ruleRegistry.getRulesForAgent(caseId, "agent-1"),
        ruleContext, RuleConfig.defaults());

    assertEquals(1, firings.size());
    assertEquals("r1", firings.get(0).ruleId());
    var perceived = signalRegistry.perceive(caseId, 0.01);
    assertTrue(perceived.containsKey("alert"));
  }

  @Test
  void predicateConditionNoMatchNoFiring() {
    var rule = new LocalRule("r1",
        new RuleCondition.PredicateCondition(ctx -> !ctx.observations().isEmpty()),
        List.of(new RuleAction.DepositSignal("alert", 0.9, null)),
        0);
    ruleRegistry.registerRule(caseId, "agent-1", "binding-a", rule, 50);

    var ruleContext = new RuleContext(
        List.of(), Map.of(), JsonNodeFactory.instance.objectNode(), Set.of(),
        InterestLandscape.EMPTY, "agent-1", "tenant-1", caseId);

    var evaluator = new LocalRuleEvaluator(signalRegistry);
    var firings = evaluator.evaluate("agent-1", ruleRegistry.getRulesForAgent(caseId, "agent-1"),
        ruleContext, RuleConfig.defaults());

    assertTrue(firings.isEmpty());
    assertTrue(signalRegistry.perceive(caseId, 0.01).isEmpty());
  }

  @Test
  void priorityDeterminesExecutionOrder() {
    var lowPriority = new LocalRule("low",
        new RuleCondition.PredicateCondition(ctx -> true),
        List.of(new RuleAction.DepositSignal("low-signal", 0.3, null)),
        1);
    var highPriority = new LocalRule("high",
        new RuleCondition.PredicateCondition(ctx -> true),
        List.of(new RuleAction.DepositSignal("high-signal", 0.9, null)),
        10);
    ruleRegistry.registerRule(caseId, "agent-1", "b", lowPriority, 50);
    ruleRegistry.registerRule(caseId, "agent-1", "b", highPriority, 50);

    var ruleContext = new RuleContext(
        List.of(), Map.of(), JsonNodeFactory.instance.objectNode(), Set.of(),
        InterestLandscape.EMPTY, "agent-1", "tenant-1", caseId);

    var evaluator = new LocalRuleEvaluator(signalRegistry);
    var firings = evaluator.evaluate("agent-1", ruleRegistry.getRulesForAgent(caseId, "agent-1"),
        ruleContext, RuleConfig.defaults());

    assertEquals(2, firings.size());
    assertEquals("high", firings.get(0).ruleId());
    assertEquals("low", firings.get(1).ruleId());
  }

  @Test
  void writeContextActionCollected() {
    var rule = new LocalRule("r1",
        new RuleCondition.PredicateCondition(ctx -> true),
        List.of(new RuleAction.WriteContext("status", new IntNode(42))),
        0);
    ruleRegistry.registerRule(caseId, "agent-1", "b", rule, 50);

    var ruleContext = new RuleContext(
        List.of(), Map.of(), JsonNodeFactory.instance.objectNode(), Set.of(),
        InterestLandscape.EMPTY, "agent-1", "tenant-1", caseId);

    var evaluator = new LocalRuleEvaluator(signalRegistry);
    var firings = evaluator.evaluate("agent-1", ruleRegistry.getRulesForAgent(caseId, "agent-1"),
        ruleContext, RuleConfig.defaults());

    assertEquals(1, firings.size());
    var writeActions = firings.get(0).executedActions().stream()
        .filter(a -> a instanceof RuleAction.WriteContext)
        .toList();
    assertEquals(1, writeActions.size());
  }

  @Test
  void maxActionsPerCycleEnforced() {
    var rule = new LocalRule("r1",
        new RuleCondition.PredicateCondition(ctx -> true),
        List.of(
            new RuleAction.DepositSignal("s1", 1.0, null),
            new RuleAction.DepositSignal("s2", 1.0, null),
            new RuleAction.DepositSignal("s3", 1.0, null)),
        0);
    ruleRegistry.registerRule(caseId, "agent-1", "b", rule, 50);

    var ruleContext = new RuleContext(
        List.of(), Map.of(), JsonNodeFactory.instance.objectNode(), Set.of(),
        InterestLandscape.EMPTY, "agent-1", "tenant-1", caseId);

    var config = new RuleConfig(50, 2, 100);
    var evaluator = new LocalRuleEvaluator(signalRegistry);
    var firings = evaluator.evaluate("agent-1", ruleRegistry.getRulesForAgent(caseId, "agent-1"),
        ruleContext, config);

    long totalActions = firings.stream().mapToLong(f -> f.executedActions().size()).sum();
    assertTrue(totalActions <= 2);
  }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl runtime-core -Dtest=LocalRuleEvaluationTest -DfailIfNoTests=false -q`
Expected: FAIL — `LocalRuleEvaluator` not found

- [ ] **Step 3: Implement LocalRuleEvaluator**

Create `runtime-core/src/main/java/io/casehub/engine/internal/observation/LocalRuleEvaluator.java`:

```java
package io.casehub.engine.internal.observation;

import io.casehub.api.spi.observation.*;
import io.casehub.engine.common.internal.signal.SignalRegistry;
import java.time.Instant;
import java.util.ArrayList;
import java.util.Comparator;
import java.util.List;

public class LocalRuleEvaluator {

  private final SignalRegistry signalRegistry;

  public LocalRuleEvaluator(SignalRegistry signalRegistry) {
    this.signalRegistry = signalRegistry;
  }

  public List<RuleFiring> evaluate(
      String agentId,
      List<LocalRule> rules,
      RuleContext context,
      RuleConfig config) {
    List<LocalRule> sorted = rules.stream()
        .sorted(Comparator.comparingInt(LocalRule::priority).reversed())
        .toList();

    List<RuleFiring> firings = new ArrayList<>();
    int totalActions = 0;

    for (LocalRule rule : sorted) {
      if (totalActions >= config.maxActionsPerCycle()) {
        break;
      }

      boolean matches = evaluateCondition(rule.condition(), context);
      if (!matches) {
        continue;
      }

      List<RuleAction> executed = new ArrayList<>();
      for (RuleAction action : rule.actions()) {
        if (totalActions >= config.maxActionsPerCycle()) {
          break;
        }
        executeCoordinationAction(action, context.caseId(), agentId, config);
        executed.add(action);
        totalActions++;
      }

      if (!executed.isEmpty()) {
        firings.add(new RuleFiring(rule.id(), List.copyOf(executed), Instant.now()));
      }
    }

    return firings;
  }

  private boolean evaluateCondition(RuleCondition condition, RuleContext context) {
    return switch (condition) {
      case RuleCondition.PredicateCondition pc -> {
        try {
          yield pc.predicate().test(context);
        } catch (Exception e) {
          yield false;
        }
      }
      case RuleCondition.ExpressionCondition ec -> false;
    };
  }

  private void executeCoordinationAction(RuleAction action, java.util.UUID caseId, String agentId,
      RuleConfig config) {
    switch (action) {
      case RuleAction.DepositSignal ds ->
          signalRegistry.deposit(caseId, ds.name(), ds.strength(),
              ds.halfLife() != null ? ds.halfLife() : java.time.Duration.ofMinutes(5),
              agentId, 100);
      case RuleAction.WriteContext wc -> {} // collected, not executed here
      case RuleAction.RegisterInterest ri -> {} // deferred to handler context
      case RuleAction.DeregisterInterest di -> {} // deferred to handler context
    }
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -pl runtime-core -Dtest=LocalRuleEvaluationTest -q`
Expected: PASS

- [ ] **Step 5: Wire localRules() into CaseContextChangedEventHandler**

Modify `CaseContextChangedEventHandler.java`:
- Add field: `private final RuleRegistry ruleRegistry;` — inject via constructor
- In `evaluateAndDispatch()` (around line 253), after the `observations()` call, add: `localRules(caseInstance, contextSnapshot, caseDefinition);`
- Add `private void localRules(CaseInstance, CaseContext, CaseDefinition)` method:
  1. Check `ruleRegistry.ruleCount(caseId) == 0` → early return
  2. Get `RuleConfig` from definition
  3. Get perceived signals from `signalRegistry.perceive()`
  4. Get changed keys from `contextHistoryBuffer`
  5. Get landscape from `observationRegistry.computeLandscape()`
  6. Get working layer snapshot
  7. For each agent in `ruleRegistry.getRulesForCase(caseId)`:
     - Build `RuleContext` with agent's observations from `observationRegistry.getObservations()`
     - Call `localRuleEvaluator.evaluate()` with timeout on virtual thread
     - Store firings via `ruleRegistry.storeFirings()`
     - Collect `WriteContext` actions
  8. After all agents: apply batched `WriteContext` via `writableLayer.set(key, value)`
  9. If writes occurred, publish `CONTEXT_CHANGED`

- [ ] **Step 6: Wire lifecycle cleanup**

Modify `CaseStatusChangedHandler.java` — after existing `observationRegistry.unregisterByCase()` and `signalRegistry.evictByCase()` calls, add: `ruleRegistry.evictByCase(caseId);`

Modify `ScopedWorkerTerminationHandler.java` — after existing `observationRegistry.unregisterByBinding()` call, add: `ruleRegistry.unregisterByBinding(caseId, bindingNames);`

- [ ] **Step 7: Run full test suite**

Run: `mvn test -pl runtime-core -q`
Expected: PASS — no regressions

- [ ] **Step 8: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/engine/internal/observation/LocalRuleEvaluator.java \
  runtime-core/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java \
  runtime-core/src/main/java/io/casehub/engine/internal/engine/handler/CaseStatusChangedHandler.java \
  runtime-core/src/main/java/io/casehub/engine/internal/engine/handler/ScopedWorkerTerminationHandler.java \
  runtime-core/src/test/java/io/casehub/engine/internal/observation/LocalRuleEvaluationTest.java
git commit -m "feat: integrate local rule evaluation into engine pipeline

localRules() runs after observations() in the evaluation cycle. Per-agent
rule evaluation with timeout. Batched WriteContext actions with single
CONTEXT_CHANGED. Lifecycle cleanup on case terminal status and compound
completion.

Refs casehubio/engine#1109"
```

- [ ] **Step 9: Update CLAUDE.md with local rule evaluation documentation**

Add a new section `## Local Rule Evaluation` to `CLAUDE.md` documenting:
- `LocalRule`, `RuleAction` sealed hierarchy, `RuleCondition` sealed hierarchy
- `RuleContext` record fields
- `RuleSpace` facet on `WorkerRuntime`
- `RuleRegistry` in `common-core`
- Pipeline placement (after `observations()`)
- `RuleConfig` on `CaseDefinition`
- Event types: `RULE_REGISTERED`, `RULE_FIRED`
- Scope: COMPOUND or CASE only

- [ ] **Step 10: Commit CLAUDE.md update**

```bash
git add CLAUDE.md
git commit -m "docs: add local rule evaluation to CLAUDE.md

Refs casehubio/engine#1109"
```

## References

- `wsp/specs/issue-1104-hive-mind/2026-09-16-local-rule-evaluation-design.md` — design spec
- `api/src/main/java/io/casehub/api/engine/WorkerRuntime.java:24` — existing facet accessors
- `api/src/main/java/io/casehub/api/engine/InterestSpace.java:25` — facet pattern reference
- `api/src/main/java/io/casehub/api/engine/SignalSpace.java:22` — facet pattern reference
- `api/src/main/java/io/casehub/api/engine/NeighborSpace.java:18` — facet pattern reference
- `api/src/main/java/io/casehub/api/spi/observation/ObservationConfig.java:20` — config record pattern
- `api/src/main/java/io/casehub/api/model/signal/SignalConfig.java:20` — config record pattern
- `common-core/src/main/java/io/casehub/engine/common/internal/observation/ObservationRegistry.java:29` — registry pattern
- `common-core/src/main/java/io/casehub/engine/common/internal/signal/SignalRegistry.java` — signal deposit API
- `runtime-core/src/main/java/io/casehub/engine/internal/observation/DefaultInterestSpace.java:31` — facet impl pattern
- `runtime-core/src/main/java/io/casehub/engine/internal/observation/DefaultNeighborSpace.java:34` — facet impl pattern
- `runtime-core/src/main/java/io/casehub/engine/internal/executor/DefaultWorkerRuntime.java:43` — constructor pattern
- `runtime-core/src/main/java/io/casehub/engine/internal/executor/WorkerRuntimeFactory.java:25` — factory wiring
- `runtime-core/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java:243` — pipeline integration point
- `runtime-core/src/main/java/io/casehub/engine/internal/engine/handler/CaseStatusChangedHandler.java:53` — lifecycle cleanup
- `runtime-core/src/main/java/io/casehub/engine/internal/engine/handler/ScopedWorkerTerminationHandler.java:24` — binding cleanup
- `api/src/main/java/io/casehub/api/model/event/CaseHubEventType.java:18` — event type enum
- D37-D44 — design decisions
- casehubio/engine#1109 — focal issue
- casehubio/engine#1104 — Hive Mind epic
