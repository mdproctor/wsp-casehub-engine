# AgentCapabilityGuard Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1283 — feat: AgentCapabilityGuard — mechanical per-agent decision scope enforcement
**Issue group:** casehubio/agency#377 (parent epic)

**Goal:** Add per-agent decision scope enforcement to the engine — permission matrix on enums, authority derivation, guard SPI with NoOp default.

**Architecture:** Enrich existing vocabulary enums (`DecisionScope`, `TopologyType`) with total-function methods. Add `DecisionAuthority` enum, `AgentAction` sealed interface, `GuardDecision` sealed interface, `AgentCapabilityGuard` SPI, and `NoOpAgentCapabilityGuard` default. All in the `api` module except the NoOp POJO (`runtime-core`) and its CDI producer (`runtime`).

**Tech Stack:** Java 21+, JUnit 5, AssertJ, Quarkus CDI (`@DefaultBean`)

## Global Constraints

- Exhaustive switch expressions — every enum method must handle all values (compiler-enforced total function)
- License header on every new file (Apache 2.0, 2026-Present The Case Hub Authors)
- Test classes must be `*Test.java`, never `*IT.java` (surefire, not failsafe)
- NoOp defaults are plain POJOs in `runtime-core` — CDI wiring via `@Produces @DefaultBean @ApplicationScoped` in `SpiDefaultBeans`
- `spotless:check` runs on compile — format before committing

---

## Batch 1: Vocabulary Enrichment — Permission Matrix and Authority Derivation

### Task 1: DecisionScope.isPermitted() — permission matrix on enum

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/coordination/DecisionScope.java`
- Create: `api/src/test/java/io/casehub/api/model/coordination/DecisionScopeTest.java`

**Interfaces:**
- Consumes: `DecisionType` enum (existing, unchanged)
- Produces: `DecisionScope.isPermitted(DecisionType)` — returns `boolean`

- [ ] **Step 1: Write the failing tests — all 15 matrix cells**

```java
package io.casehub.api.model.coordination;

import static org.assertj.core.api.Assertions.assertThat;

import org.junit.jupiter.api.Test;

class DecisionScopeTest {

  @Test
  void stimulus_driven_permits_only_signaling() {
    assertThat(DecisionScope.STIMULUS_DRIVEN.isPermitted(DecisionType.SIGNALING)).isTrue();
    assertThat(DecisionScope.STIMULUS_DRIVEN.isPermitted(DecisionType.TASK_SELECTION)).isFalse();
    assertThat(DecisionScope.STIMULUS_DRIVEN.isPermitted(DecisionType.COLLABORATION)).isFalse();
    assertThat(DecisionScope.STIMULUS_DRIVEN.isPermitted(DecisionType.PROVISIONING)).isFalse();
    assertThat(DecisionScope.STIMULUS_DRIVEN.isPermitted(DecisionType.CONTEXT_MUTATION)).isFalse();
  }

  @Test
  void task_adaptation_permits_collaboration_signaling_context_mutation() {
    assertThat(DecisionScope.TASK_ADAPTATION.isPermitted(DecisionType.COLLABORATION)).isTrue();
    assertThat(DecisionScope.TASK_ADAPTATION.isPermitted(DecisionType.SIGNALING)).isTrue();
    assertThat(DecisionScope.TASK_ADAPTATION.isPermitted(DecisionType.CONTEXT_MUTATION)).isTrue();
    assertThat(DecisionScope.TASK_ADAPTATION.isPermitted(DecisionType.TASK_SELECTION)).isFalse();
    assertThat(DecisionScope.TASK_ADAPTATION.isPermitted(DecisionType.PROVISIONING)).isFalse();
  }

  @Test
  void full_autonomy_permits_everything() {
    for (DecisionType type : DecisionType.values()) {
      assertThat(DecisionScope.FULL_AUTONOMY.isPermitted(type))
          .as("FULL_AUTONOMY should permit %s", type)
          .isTrue();
    }
  }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl api -Dtest=DecisionScopeTest -q -f /Users/mdproctor/claude/casehub/slots/220/engine/pom.xml`
Expected: compilation error — `isPermitted` method does not exist

- [ ] **Step 3: Implement DecisionScope.isPermitted()**

Replace the `DecisionScope` enum body with constant-specific method overrides using exhaustive switch expressions:

```java
public enum DecisionScope {
  STIMULUS_DRIVEN {
    @Override
    public boolean isPermitted(DecisionType type) {
      return switch (type) {
        case SIGNALING -> true;
        case TASK_SELECTION, COLLABORATION, PROVISIONING, CONTEXT_MUTATION -> false;
      };
    }
  },
  TASK_ADAPTATION {
    @Override
    public boolean isPermitted(DecisionType type) {
      return switch (type) {
        case COLLABORATION, SIGNALING, CONTEXT_MUTATION -> true;
        case TASK_SELECTION, PROVISIONING -> false;
      };
    }
  },
  FULL_AUTONOMY {
    @Override
    public boolean isPermitted(DecisionType type) {
      return switch (type) {
        case TASK_SELECTION, COLLABORATION, SIGNALING, PROVISIONING, CONTEXT_MUTATION -> true;
      };
    }
  };

  public abstract boolean isPermitted(DecisionType type);
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -pl api -Dtest=DecisionScopeTest -q -f /Users/mdproctor/claude/casehub/slots/220/engine/pom.xml`
Expected: 3 tests PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/220/engine add api/src/main/java/io/casehub/api/model/coordination/DecisionScope.java api/src/test/java/io/casehub/api/model/coordination/DecisionScopeTest.java
git -C /Users/mdproctor/claude/casehub/slots/220/engine commit -m "feat: add permission matrix to DecisionScope — exhaustive switch expressions

Refs #1283"
```

### Task 2: DecisionAuthority enum + TopologyType.deriveAuthority()

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/coordination/DecisionAuthority.java`
- Modify: `api/src/main/java/io/casehub/api/model/coordination/TopologyType.java`
- Create: `api/src/test/java/io/casehub/api/model/coordination/TopologyTypeTest.java`

**Interfaces:**
- Consumes: `StructuralRole` enum (existing), `DecisionAuthority` enum (created in this task)
- Produces: `TopologyType.deriveAuthority(StructuralRole)` — returns `DecisionAuthority`, throws `IllegalArgumentException` for invalid combinations

- [ ] **Step 1: Create DecisionAuthority enum**

```java
package io.casehub.api.model.coordination;

public enum DecisionAuthority {
  LOCAL,
  COORDINATED,
  CENTRAL
}
```

- [ ] **Step 2: Write failing tests — all 9 derivation cells**

```java
package io.casehub.api.model.coordination;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatIllegalArgumentException;

import org.junit.jupiter.api.Test;

class TopologyTypeTest {

  @Test
  void centralised_initiator_derives_central() {
    assertThat(TopologyType.CENTRALISED.deriveAuthority(StructuralRole.INITIATOR))
        .isEqualTo(DecisionAuthority.CENTRAL);
  }

  @Test
  void centralised_contractor_derives_local() {
    assertThat(TopologyType.CENTRALISED.deriveAuthority(StructuralRole.CONTRACTOR))
        .isEqualTo(DecisionAuthority.LOCAL);
  }

  @Test
  void centralised_peer_is_invalid() {
    assertThatIllegalArgumentException()
        .isThrownBy(() -> TopologyType.CENTRALISED.deriveAuthority(StructuralRole.PEER))
        .withMessageContaining("CENTRALISED");
  }

  @Test
  void distributed_peer_derives_coordinated() {
    assertThat(TopologyType.DISTRIBUTED.deriveAuthority(StructuralRole.PEER))
        .isEqualTo(DecisionAuthority.COORDINATED);
  }

  @Test
  void distributed_initiator_is_invalid() {
    assertThatIllegalArgumentException()
        .isThrownBy(() -> TopologyType.DISTRIBUTED.deriveAuthority(StructuralRole.INITIATOR))
        .withMessageContaining("DISTRIBUTED");
  }

  @Test
  void distributed_contractor_is_invalid() {
    assertThatIllegalArgumentException()
        .isThrownBy(() -> TopologyType.DISTRIBUTED.deriveAuthority(StructuralRole.CONTRACTOR))
        .withMessageContaining("DISTRIBUTED");
  }

  @Test
  void hybrid_initiator_derives_central() {
    assertThat(TopologyType.HYBRID.deriveAuthority(StructuralRole.INITIATOR))
        .isEqualTo(DecisionAuthority.CENTRAL);
  }

  @Test
  void hybrid_peer_derives_coordinated() {
    assertThat(TopologyType.HYBRID.deriveAuthority(StructuralRole.PEER))
        .isEqualTo(DecisionAuthority.COORDINATED);
  }

  @Test
  void hybrid_contractor_derives_local() {
    assertThat(TopologyType.HYBRID.deriveAuthority(StructuralRole.CONTRACTOR))
        .isEqualTo(DecisionAuthority.LOCAL);
  }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn test -pl api -Dtest=TopologyTypeTest -q -f /Users/mdproctor/claude/casehub/slots/220/engine/pom.xml`
Expected: compilation error — `deriveAuthority` method does not exist

- [ ] **Step 4: Implement TopologyType.deriveAuthority()**

Replace the `TopologyType` enum body with constant-specific method overrides:

```java
public enum TopologyType {
  CENTRALISED {
    @Override
    public DecisionAuthority deriveAuthority(StructuralRole role) {
      return switch (role) {
        case INITIATOR -> DecisionAuthority.CENTRAL;
        case CONTRACTOR -> DecisionAuthority.LOCAL;
        case PEER -> throw new IllegalArgumentException(
            "CENTRALISED topology does not support PEER"
                + " — centralised structure requires coordinator/subordinate relationships");
      };
    }
  },
  DISTRIBUTED {
    @Override
    public DecisionAuthority deriveAuthority(StructuralRole role) {
      return switch (role) {
        case PEER -> DecisionAuthority.COORDINATED;
        case INITIATOR -> throw new IllegalArgumentException(
            "DISTRIBUTED topology does not support INITIATOR"
                + " — peer-to-peer structure has no central coordinator");
        case CONTRACTOR -> throw new IllegalArgumentException(
            "DISTRIBUTED topology does not support CONTRACTOR"
                + " — peer-to-peer structure has no delegation hierarchy");
      };
    }
  },
  HYBRID {
    @Override
    public DecisionAuthority deriveAuthority(StructuralRole role) {
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

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn test -pl api -Dtest=TopologyTypeTest -q -f /Users/mdproctor/claude/casehub/slots/220/engine/pom.xml`
Expected: 9 tests PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/220/engine add api/src/main/java/io/casehub/api/model/coordination/DecisionAuthority.java api/src/main/java/io/casehub/api/model/coordination/TopologyType.java api/src/test/java/io/casehub/api/model/coordination/TopologyTypeTest.java
git -C /Users/mdproctor/claude/casehub/slots/220/engine commit -m "feat: add DecisionAuthority enum + TopologyType.deriveAuthority()

Refs #1283"
```

## Batch 2: Guard SPI — AgentAction, GuardDecision, AgentCapabilityGuard, NoOp Default

### Task 3: AgentAction sealed interface + GuardDecision + AgentCapabilityGuard SPI

**Files:**
- Create: `api/src/main/java/io/casehub/api/spi/AgentAction.java`
- Create: `api/src/main/java/io/casehub/api/spi/GuardDecision.java`
- Create: `api/src/main/java/io/casehub/api/spi/AgentCapabilityGuard.java`
- Create: `api/src/test/java/io/casehub/api/spi/AgentActionTest.java`

**Interfaces:**
- Consumes: `DecisionType` enum (from api/model/coordination)
- Produces: `AgentAction` sealed interface with 5 record variants, `GuardDecision` sealed interface, `AgentCapabilityGuard` SPI interface

- [ ] **Step 1: Write failing tests for AgentAction.decisionType()**

```java
package io.casehub.api.spi;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.api.model.coordination.DecisionType;
import org.junit.jupiter.api.Test;

class AgentActionTest {

  @Test
  void initiate_message_is_collaboration() {
    assertThat(new AgentAction.InitiateMessage("target").decisionType())
        .isEqualTo(DecisionType.COLLABORATION);
  }

  @Test
  void deposit_signal_is_signaling() {
    assertThat(new AgentAction.DepositSignal("alert").decisionType())
        .isEqualTo(DecisionType.SIGNALING);
  }

  @Test
  void select_task_is_task_selection() {
    assertThat(new AgentAction.SelectTask("task-1").decisionType())
        .isEqualTo(DecisionType.TASK_SELECTION);
  }

  @Test
  void request_provisioning_is_provisioning() {
    assertThat(new AgentAction.RequestProvisioning().decisionType())
        .isEqualTo(DecisionType.PROVISIONING);
  }

  @Test
  void mutate_context_is_context_mutation() {
    assertThat(new AgentAction.MutateContext("key").decisionType())
        .isEqualTo(DecisionType.CONTEXT_MUTATION);
  }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl api -Dtest=AgentActionTest -q -f /Users/mdproctor/claude/casehub/slots/220/engine/pom.xml`
Expected: compilation error — classes do not exist

- [ ] **Step 3: Create GuardDecision sealed interface**

```java
package io.casehub.api.spi;

import io.casehub.api.model.coordination.DecisionScope;
import io.casehub.api.model.coordination.DecisionType;

public sealed interface GuardDecision {
  record Permitted() implements GuardDecision {}

  record Denied(DecisionScope agentScope, DecisionType attemptedType) implements GuardDecision {}
}
```

- [ ] **Step 4: Create AgentAction sealed interface**

```java
package io.casehub.api.spi;

import io.casehub.api.model.coordination.DecisionType;

public sealed interface AgentAction {

  DecisionType decisionType();

  record InitiateMessage(String targetAgentId) implements AgentAction {
    @Override
    public DecisionType decisionType() {
      return DecisionType.COLLABORATION;
    }
  }

  record DepositSignal(String signalName) implements AgentAction {
    @Override
    public DecisionType decisionType() {
      return DecisionType.SIGNALING;
    }
  }

  record SelectTask(String taskId) implements AgentAction {
    @Override
    public DecisionType decisionType() {
      return DecisionType.TASK_SELECTION;
    }
  }

  record RequestProvisioning() implements AgentAction {
    @Override
    public DecisionType decisionType() {
      return DecisionType.PROVISIONING;
    }
  }

  record MutateContext(String contextKey) implements AgentAction {
    @Override
    public DecisionType decisionType() {
      return DecisionType.CONTEXT_MUTATION;
    }
  }
}
```

- [ ] **Step 5: Create AgentCapabilityGuard SPI interface**

```java
package io.casehub.api.spi;

import java.util.UUID;

public interface AgentCapabilityGuard {
  GuardDecision check(UUID caseId, String agentId, AgentAction action);
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `mvn test -pl api -Dtest=AgentActionTest -q -f /Users/mdproctor/claude/casehub/slots/220/engine/pom.xml`
Expected: 5 tests PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/220/engine add api/src/main/java/io/casehub/api/spi/AgentAction.java api/src/main/java/io/casehub/api/spi/GuardDecision.java api/src/main/java/io/casehub/api/spi/AgentCapabilityGuard.java api/src/test/java/io/casehub/api/spi/AgentActionTest.java
git -C /Users/mdproctor/claude/casehub/slots/220/engine commit -m "feat: add AgentAction, GuardDecision, and AgentCapabilityGuard SPI

Refs #1283"
```

### Task 4: NoOpAgentCapabilityGuard + SpiDefaultBeans producer + integration test

**Files:**
- Create: `runtime-core/src/main/java/io/casehub/engine/runtime/coordination/NoOpAgentCapabilityGuard.java`
- Modify: `runtime/src/main/java/io/casehub/engine/runtime/quarkus/SpiDefaultBeans.java`
- Create: `runtime-core/src/test/java/io/casehub/engine/runtime/coordination/NoOpAgentCapabilityGuardTest.java`

**Interfaces:**
- Consumes: `AgentCapabilityGuard` (from api/spi), `GuardDecision` (from api/spi), `AgentAction` (from api/spi)
- Produces: `NoOpAgentCapabilityGuard` — plain POJO, always returns `Permitted()`

- [ ] **Step 1: Write failing test for NoOp**

```java
package io.casehub.engine.runtime.coordination;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.api.spi.AgentAction;
import io.casehub.api.spi.GuardDecision;
import java.util.UUID;
import org.junit.jupiter.api.Test;

class NoOpAgentCapabilityGuardTest {

  private final NoOpAgentCapabilityGuard guard = new NoOpAgentCapabilityGuard();

  @Test
  void always_permits_deposit_signal() {
    GuardDecision decision =
        guard.check(UUID.randomUUID(), "agent-1", new AgentAction.DepositSignal("alert"));
    assertThat(decision).isInstanceOf(GuardDecision.Permitted.class);
  }

  @Test
  void always_permits_select_task() {
    GuardDecision decision =
        guard.check(UUID.randomUUID(), "agent-1", new AgentAction.SelectTask("task-1"));
    assertThat(decision).isInstanceOf(GuardDecision.Permitted.class);
  }

  @Test
  void always_permits_request_provisioning() {
    GuardDecision decision =
        guard.check(UUID.randomUUID(), "agent-1", new AgentAction.RequestProvisioning());
    assertThat(decision).isInstanceOf(GuardDecision.Permitted.class);
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl runtime-core -Dtest=NoOpAgentCapabilityGuardTest -q -f /Users/mdproctor/claude/casehub/slots/220/engine/pom.xml`
Expected: compilation error — class does not exist

- [ ] **Step 3: Create NoOpAgentCapabilityGuard**

```java
package io.casehub.engine.runtime.coordination;

import io.casehub.api.spi.AgentAction;
import io.casehub.api.spi.AgentCapabilityGuard;
import io.casehub.api.spi.GuardDecision;
import java.util.UUID;

public class NoOpAgentCapabilityGuard implements AgentCapabilityGuard {

  @Override
  public GuardDecision check(UUID caseId, String agentId, AgentAction action) {
    return new GuardDecision.Permitted();
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl runtime-core -Dtest=NoOpAgentCapabilityGuardTest -q -f /Users/mdproctor/claude/casehub/slots/220/engine/pom.xml`
Expected: 3 tests PASS

- [ ] **Step 5: Add producer method to SpiDefaultBeans**

Add to `SpiDefaultBeans.java` (after the last existing producer method, before the closing brace). Use `ide_insert_member`:

```java
  // --- Coordination SPIs ---

  @Produces
  @DefaultBean
  @ApplicationScoped
  AgentCapabilityGuard noOpAgentCapabilityGuard() {
    return new NoOpAgentCapabilityGuard();
  }
```

Add the required imports to `SpiDefaultBeans.java`:
```java
import io.casehub.api.spi.AgentCapabilityGuard;
import io.casehub.engine.runtime.coordination.NoOpAgentCapabilityGuard;
```

- [ ] **Step 6: Run full api + runtime-core test suite to verify no regressions**

Run: `mvn test -pl api,runtime-core -q -f /Users/mdproctor/claude/casehub/slots/220/engine/pom.xml`
Expected: all tests PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/220/engine add runtime-core/src/main/java/io/casehub/engine/runtime/coordination/NoOpAgentCapabilityGuard.java runtime-core/src/test/java/io/casehub/engine/runtime/coordination/NoOpAgentCapabilityGuardTest.java runtime/src/main/java/io/casehub/engine/runtime/quarkus/SpiDefaultBeans.java
git -C /Users/mdproctor/claude/casehub/slots/220/engine commit -m "feat: add NoOpAgentCapabilityGuard + SpiDefaultBeans producer

Closes #1283"
```

## References

- `specs/issue-1283-agent-capability-guard/SPEC.md` — design spec
- `api/src/main/java/io/casehub/api/model/coordination/DecisionScope.java` — enriched with isPermitted()
- `api/src/main/java/io/casehub/api/model/coordination/TopologyType.java` — enriched with deriveAuthority()
- `api/src/main/java/io/casehub/api/model/coordination/DecisionType.java` — consumed by matrix
- `api/src/main/java/io/casehub/api/model/coordination/StructuralRole.java` — consumed by derivation
- `runtime-core/src/main/java/io/casehub/engine/runtime/worker/NoOpCaseChannelProvider.java` — NoOp POJO precedent
- `runtime/src/main/java/io/casehub/engine/runtime/quarkus/SpiDefaultBeans.java:82-280` — producer pattern
- `api/src/test/java/io/casehub/api/model/coordination/AgentRoleTest.java` — test pattern precedent
- PP-20260921-b7c277 — `@DefaultBean` for 1:1 SPI replacement
- PP-20260514-engine-spi-noops-defaultbean — NoOp defaults must use `@DefaultBean`
- casehubio/engine#1283
- casehubio/agency#377
