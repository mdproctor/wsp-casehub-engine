# SwarmTopologyProvider Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** engine#1284 — feat: SwarmTopologyProvider — topology-aware provisioning and lifecycle
**Issue group:** casehubio/agency#377 (parent epic)

**Goal:** Make the engine topology-aware by adding SwarmTopologyProvider SPI so SwarmProvisioner consults the topology before making central provisioning decisions.

**Architecture:** Three new types — TopologyDescriptor (immutable record in api/model/coordination), SwarmTopologyProvider (SPI in api/spi/coordination), CentralisedTopologyProvider (default impl in runtime-core/runtime/coordination). SwarmProvisioner gets a topology-type gate: DISTRIBUTED topologies skip central provisioning.

**Tech Stack:** Java 21, Quarkus 3.32.2, JUnit 5, AssertJ, Mockito

## Global Constraints

- All new types in `io.casehub.api.model.coordination` or `io.casehub.api.spi.coordination`
- Default impl in `io.casehub.engine.runtime.coordination`
- Agent identity is `String` (consistent with AgentCapabilityGuard)
- Apache 2.0 license header on all new files
- Test classes named `*Test.java` (never `*IT.java` — surefire, not failsafe)
- No DecisionType parameter on authorityFor — YAGNI
- Pre-release: no backward-compat shims

---

## Batch 1: TopologyDescriptor + SwarmTopologyProvider SPI

### Task 1: TopologyDescriptor record

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/coordination/TopologyDescriptor.java`
- Test: `api/src/test/java/io/casehub/api/model/coordination/TopologyDescriptorTest.java`

**Interfaces:**
- Consumes: `TopologyType.deriveAuthority(StructuralRole)`, `AgentRole`, `StructuralRole`, `DecisionAuthority`
- Produces: `TopologyDescriptor(TopologyType, Map<String, AgentRole>)`, `authorityFor(String)` → `DecisionAuthority`, `coordinatorAgentId()` → `Optional<String>`

- [ ] **Step 1: Write failing tests for construction and validation**

```java
package io.casehub.api.model.coordination;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatIllegalArgumentException;
import static org.assertj.core.api.Assertions.assertThatNullPointerException;

import java.util.HashMap;
import java.util.Map;
import org.junit.jupiter.api.Test;

class TopologyDescriptorTest {

  @Test
  void constructs_with_valid_centralised_topology() {
    var descriptor = new TopologyDescriptor(
        TopologyType.CENTRALISED,
        Map.of("coord", AgentRole.COORDINATOR, "worker-1", AgentRole.of("worker", StructuralRole.CONTRACTOR)));
    assertThat(descriptor.topologyType()).isEqualTo(TopologyType.CENTRALISED);
    assertThat(descriptor.agentRoles()).hasSize(2);
  }

  @Test
  void constructs_with_valid_distributed_topology() {
    var descriptor = new TopologyDescriptor(
        TopologyType.DISTRIBUTED,
        Map.of("peer-1", AgentRole.of("analyst", StructuralRole.PEER),
               "peer-2", AgentRole.of("reviewer", StructuralRole.PEER)));
    assertThat(descriptor.topologyType()).isEqualTo(TopologyType.DISTRIBUTED);
    assertThat(descriptor.agentRoles()).hasSize(2);
  }

  @Test
  void constructs_with_empty_role_map() {
    var descriptor = new TopologyDescriptor(TopologyType.CENTRALISED, Map.of());
    assertThat(descriptor.agentRoles()).isEmpty();
  }

  @Test
  void constructs_with_null_role_map_defaults_to_empty() {
    var descriptor = new TopologyDescriptor(TopologyType.CENTRALISED, null);
    assertThat(descriptor.agentRoles()).isEmpty();
  }

  @Test
  void rejects_null_topology_type() {
    assertThatNullPointerException()
        .isThrownBy(() -> new TopologyDescriptor(null, Map.of()))
        .withMessageContaining("topologyType");
  }

  @Test
  void rejects_invalid_role_topology_combination() {
    assertThatIllegalArgumentException()
        .isThrownBy(() -> new TopologyDescriptor(
            TopologyType.CENTRALISED,
            Map.of("agent-1", AgentRole.of("analyst", StructuralRole.PEER))));
  }

  @Test
  void allows_symbolic_roles_without_validation() {
    var descriptor = new TopologyDescriptor(
        TopologyType.CENTRALISED,
        Map.of("observer", AgentRole.of("observer")));
    assertThat(descriptor.agentRoles()).hasSize(1);
  }

  @Test
  void role_map_is_immutable() {
    var mutable = new HashMap<String, AgentRole>();
    mutable.put("coord", AgentRole.COORDINATOR);
    var descriptor = new TopologyDescriptor(TopologyType.CENTRALISED, mutable);
    mutable.put("intruder", AgentRole.of("intruder", StructuralRole.CONTRACTOR));
    assertThat(descriptor.agentRoles()).hasSize(1);
  }

  @Test
  void authority_for_known_agent_with_structural_role() {
    var descriptor = new TopologyDescriptor(
        TopologyType.CENTRALISED,
        Map.of("coord", AgentRole.COORDINATOR, "worker-1", AgentRole.of("worker", StructuralRole.CONTRACTOR)));
    assertThat(descriptor.authorityFor("coord")).isEqualTo(DecisionAuthority.CENTRAL);
    assertThat(descriptor.authorityFor("worker-1")).isEqualTo(DecisionAuthority.LOCAL);
  }

  @Test
  void authority_for_distributed_peers() {
    var descriptor = new TopologyDescriptor(
        TopologyType.DISTRIBUTED,
        Map.of("peer-1", AgentRole.of("analyst", StructuralRole.PEER)));
    assertThat(descriptor.authorityFor("peer-1")).isEqualTo(DecisionAuthority.COORDINATED);
  }

  @Test
  void authority_for_unknown_agent_throws() {
    var descriptor = new TopologyDescriptor(TopologyType.CENTRALISED, Map.of());
    assertThatIllegalArgumentException()
        .isThrownBy(() -> descriptor.authorityFor("unknown"))
        .withMessageContaining("unknown");
  }

  @Test
  void authority_for_symbolic_role_throws() {
    var descriptor = new TopologyDescriptor(
        TopologyType.CENTRALISED,
        Map.of("observer", AgentRole.of("observer")));
    assertThatIllegalArgumentException()
        .isThrownBy(() -> descriptor.authorityFor("observer"))
        .withMessageContaining("symbolic");
  }

  @Test
  void coordinator_agent_id_returns_initiator() {
    var descriptor = new TopologyDescriptor(
        TopologyType.CENTRALISED,
        Map.of("coord", AgentRole.COORDINATOR, "worker-1", AgentRole.of("worker", StructuralRole.CONTRACTOR)));
    assertThat(descriptor.coordinatorAgentId()).contains("coord");
  }

  @Test
  void coordinator_agent_id_empty_for_distributed() {
    var descriptor = new TopologyDescriptor(
        TopologyType.DISTRIBUTED,
        Map.of("peer-1", AgentRole.of("analyst", StructuralRole.PEER)));
    assertThat(descriptor.coordinatorAgentId()).isEmpty();
  }

  @Test
  void coordinator_agent_id_empty_for_empty_map() {
    var descriptor = new TopologyDescriptor(TopologyType.CENTRALISED, Map.of());
    assertThat(descriptor.coordinatorAgentId()).isEmpty();
  }

  @Test
  void hybrid_topology_all_roles_valid() {
    var descriptor = new TopologyDescriptor(
        TopologyType.HYBRID,
        Map.of("coord", AgentRole.COORDINATOR,
               "peer-1", AgentRole.of("analyst", StructuralRole.PEER),
               "worker-1", AgentRole.of("worker", StructuralRole.CONTRACTOR)));
    assertThat(descriptor.authorityFor("coord")).isEqualTo(DecisionAuthority.CENTRAL);
    assertThat(descriptor.authorityFor("peer-1")).isEqualTo(DecisionAuthority.COORDINATED);
    assertThat(descriptor.authorityFor("worker-1")).isEqualTo(DecisionAuthority.LOCAL);
  }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl api -Dtest=TopologyDescriptorTest -q`
Expected: FAIL — `TopologyDescriptor` class does not exist

- [ ] **Step 3: Implement TopologyDescriptor**

Use `ide_create_file` for the new file.

```java
package io.casehub.api.model.coordination;

import java.util.Map;
import java.util.Objects;
import java.util.Optional;

public record TopologyDescriptor(
    TopologyType topologyType,
    Map<String, AgentRole> agentRoles) {

  public TopologyDescriptor {
    Objects.requireNonNull(topologyType, "topologyType must not be null");
    agentRoles = agentRoles != null ? Map.copyOf(agentRoles) : Map.of();
    for (var entry : agentRoles.entrySet()) {
      Objects.requireNonNull(entry.getKey(), "agent ID must not be null");
      Objects.requireNonNull(entry.getValue(), "agent role must not be null");
      AgentRole role = entry.getValue();
      if (role.structuralRole() != null) {
        topologyType.deriveAuthority(role.structuralRole());
      }
    }
  }

  public DecisionAuthority authorityFor(String agentId) {
    AgentRole role = agentRoles.get(agentId);
    if (role == null) {
      throw new IllegalArgumentException(
          "Unknown agent: " + agentId + " — not present in topology descriptor");
    }
    if (role.structuralRole() == null) {
      throw new IllegalArgumentException(
          "Agent " + agentId + " has symbolic role '" + role.name()
              + "' — symbolic roles have no structural position and cannot derive authority");
    }
    return topologyType.deriveAuthority(role.structuralRole());
  }

  public Optional<String> coordinatorAgentId() {
    return agentRoles.entrySet().stream()
        .filter(e -> e.getValue().structuralRole() == StructuralRole.INITIATOR)
        .map(Map.Entry::getKey)
        .findFirst();
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -pl api -Dtest=TopologyDescriptorTest -q`
Expected: All 16 tests PASS

- [ ] **Step 5: Commit**

```bash
git add api/src/main/java/io/casehub/api/model/coordination/TopologyDescriptor.java \
       api/src/test/java/io/casehub/api/model/coordination/TopologyDescriptorTest.java
git commit -m "feat: add TopologyDescriptor — immutable topology state record

Refs engine#1284"
```

### Task 2: SwarmTopologyProvider SPI + CentralisedTopologyProvider + DefaultBean

**Files:**
- Create: `api/src/main/java/io/casehub/api/spi/coordination/SwarmTopologyProvider.java`
- Create: `runtime-core/src/main/java/io/casehub/engine/runtime/coordination/CentralisedTopologyProvider.java`
- Create: `runtime-core/src/test/java/io/casehub/engine/runtime/coordination/CentralisedTopologyProviderTest.java`
- Modify: `runtime/src/main/java/io/casehub/engine/runtime/quarkus/SpiDefaultBeans.java` — add producer

**Interfaces:**
- Consumes: `TopologyDescriptor` from Task 1
- Produces: `SwarmTopologyProvider.describeTopology(UUID)` → `TopologyDescriptor`, `CentralisedTopologyProvider` (default impl)

- [ ] **Step 1: Write failing test for CentralisedTopologyProvider**

```java
package io.casehub.engine.runtime.coordination;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.api.model.coordination.TopologyType;
import java.util.UUID;
import org.junit.jupiter.api.Test;

class CentralisedTopologyProviderTest {

  private final CentralisedTopologyProvider provider = new CentralisedTopologyProvider();

  @Test
  void returns_centralised_topology() {
    var descriptor = provider.describeTopology(UUID.randomUUID());
    assertThat(descriptor.topologyType()).isEqualTo(TopologyType.CENTRALISED);
  }

  @Test
  void returns_empty_role_map() {
    var descriptor = provider.describeTopology(UUID.randomUUID());
    assertThat(descriptor.agentRoles()).isEmpty();
  }

  @Test
  void returns_same_shape_for_different_cases() {
    var d1 = provider.describeTopology(UUID.randomUUID());
    var d2 = provider.describeTopology(UUID.randomUUID());
    assertThat(d1.topologyType()).isEqualTo(d2.topologyType());
    assertThat(d1.agentRoles()).isEqualTo(d2.agentRoles());
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl runtime-core -Dtest=CentralisedTopologyProviderTest -q`
Expected: FAIL — classes do not exist

- [ ] **Step 3: Create SwarmTopologyProvider SPI interface**

Use `ide_create_file` for the new file.

```java
package io.casehub.api.spi.coordination;

import io.casehub.api.model.coordination.TopologyDescriptor;
import java.util.UUID;

public interface SwarmTopologyProvider {
  TopologyDescriptor describeTopology(UUID caseId);
}
```

- [ ] **Step 4: Create CentralisedTopologyProvider**

Use `ide_create_file` for the new file.

```java
package io.casehub.engine.runtime.coordination;

import io.casehub.api.model.coordination.TopologyDescriptor;
import io.casehub.api.model.coordination.TopologyType;
import io.casehub.api.spi.coordination.SwarmTopologyProvider;
import java.util.Map;
import java.util.UUID;

public class CentralisedTopologyProvider implements SwarmTopologyProvider {

  @Override
  public TopologyDescriptor describeTopology(UUID caseId) {
    return new TopologyDescriptor(TopologyType.CENTRALISED, Map.of());
  }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn test -pl runtime-core -Dtest=CentralisedTopologyProviderTest -q`
Expected: All 3 tests PASS

- [ ] **Step 6: Add DefaultBean producer to SpiDefaultBeans**

Use `ide_insert_member` to add the producer method to `SpiDefaultBeans` in the
`// --- Coordination SPIs ---` section, after the existing `noOpAgentCapabilityGuard()` method.

```java
@Produces
@DefaultBean
@ApplicationScoped
SwarmTopologyProvider centralisedTopologyProvider() {
    return new CentralisedTopologyProvider();
}
```

Add the required imports:
- `io.casehub.api.spi.coordination.SwarmTopologyProvider`
- `io.casehub.engine.runtime.coordination.CentralisedTopologyProvider`

- [ ] **Step 7: Run module compile to verify wiring**

Run: `mvn compile -pl runtime -q`
Expected: Compiles without errors

- [ ] **Step 8: Commit**

```bash
git add api/src/main/java/io/casehub/api/spi/coordination/SwarmTopologyProvider.java \
       runtime-core/src/main/java/io/casehub/engine/runtime/coordination/CentralisedTopologyProvider.java \
       runtime-core/src/test/java/io/casehub/engine/runtime/coordination/CentralisedTopologyProviderTest.java \
       runtime/src/main/java/io/casehub/engine/runtime/quarkus/SpiDefaultBeans.java
git commit -m "feat: add SwarmTopologyProvider SPI + CentralisedTopologyProvider default

Refs engine#1284"
```

## Batch 2: SwarmProvisioner Integration

### Task 3: SwarmProvisioner topology gate

**Files:**
- Modify: `runtime-core/src/main/java/io/casehub/engine/runtime/stigmergy/SwarmProvisioner.java` — add topology provider, topology-type gate
- Modify: `runtime/src/main/java/io/casehub/engine/runtime/quarkus/RuntimeBeans.java:1366` — add SwarmTopologyProvider param to swarmProvisioner producer
- Modify: `runtime-core/src/test/java/io/casehub/engine/runtime/stigmergy/SwarmProvisionerTest.java` — add topology gate tests
- Modify: `runtime-core/src/test/java/io/casehub/engine/runtime/stigmergy/IntegrationDelayTest.java:66` — add topology provider param
- Modify: `runtime-core/src/test/java/io/casehub/engine/runtime/stigmergy/SelfProvisioningIntegrationTest.java:72` — add topology provider param

**Interfaces:**
- Consumes: `SwarmTopologyProvider.describeTopology(UUID)` from Task 2, `TopologyDescriptor.topologyType()` from Task 1
- Produces: topology-aware provisioning behavior (DISTRIBUTED → skip)

- [ ] **Step 1: Write failing tests for topology gate**

Add these tests to the existing `SwarmProvisionerTest.java`:

```java
@Test
void distributed_topology_skips_central_provisioning() {
    var swarmConfig = new SwarmConfig(10, null, null, null, null, null, null, null, null, null, null);
    var config = new StigmergyConfig(null, null, swarmConfig);
    coordinator.initializeCase(caseId, List.of("a1"), config, "test-tenant");
    coordinator.agentJoined(caseId, "a1", "b-a1");
    coordinator.agentActivated(caseId, "a1");

    var events = swarmProvisioner.evaluateAndProvision(caseId, swarmConfig, config);
    assertTrue(events.isEmpty(), "DISTRIBUTED topology should skip central provisioning");
}

@Test
void centralised_topology_allows_provisioning() {
    var budget = new ProvisionBudget(50, 10, 0, null, null);
    var policy = IntegrationPolicy.BALANCED;
    var swarmConfig = new SwarmConfig(10, null, null, null, null, null, null, null, null, budget, policy);
    var config = new StigmergyConfig(null, null, swarmConfig);
    coordinator.initializeCase(caseId, List.of("a1"), config, "test-tenant");
    coordinator.agentJoined(caseId, "a1", "b-a1");
    coordinator.agentActivated(caseId, "a1");

    var events = swarmProvisioner.evaluateAndProvision(caseId, swarmConfig, config);
    assertFalse(events.isEmpty(), "CENTRALISED topology should allow provisioning");
}

@Test
void distributed_topology_skips_deprovisioning() {
    var swarmConfig = new SwarmConfig(10, null, null, null, null, null, null, null, null, null, null);
    var events = swarmProvisioner.evaluateDeprovisioning(caseId, swarmConfig);
    assertTrue(events.isEmpty(), "DISTRIBUTED topology should skip deprovisioning");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl runtime-core -Dtest=SwarmProvisionerTest#distributed_topology_skips_central_provisioning -q`
Expected: FAIL — SwarmProvisioner constructor doesn't accept SwarmTopologyProvider

- [ ] **Step 3: Add SwarmTopologyProvider to SwarmProvisioner**

Modify `SwarmProvisioner`:

1. Add field: `private final SwarmTopologyProvider topologyProvider;`
2. Add parameter to the main constructor (after `channelProvider`): `SwarmTopologyProvider topologyProvider`
3. Assign in main constructor: `this.topologyProvider = topologyProvider;`
4. In no-arg constructor, set: `this.topologyProvider = null;`
5. At the top of `evaluateAndProvision`, before the `if (!active)` check, add the topology gate:

```java
if (topologyProvider != null) {
    var descriptor = topologyProvider.describeTopology(caseId);
    if (descriptor.topologyType() == TopologyType.DISTRIBUTED) {
        return List.of();
    }
}
```

6. At the top of `evaluateDeprovisioning`, after `if (!active) return List.of();`, add the same gate:

```java
if (topologyProvider != null) {
    var descriptor = topologyProvider.describeTopology(caseId);
    if (descriptor.topologyType() == TopologyType.DISTRIBUTED) {
        return List.of();
    }
}
```

Add imports:
- `io.casehub.api.model.coordination.TopologyType`
- `io.casehub.api.spi.coordination.SwarmTopologyProvider`

- [ ] **Step 4: Update SwarmProvisionerTest setUp to pass topology provider**

In `setUp()`, update the SwarmProvisioner constructor call. The first
test (`distributed_topology_skips_central_provisioning`) needs a
DISTRIBUTED provider. Use a field-level provider that defaults to
CENTRALISED for backward-compat:

Add a field to the test class:

```java
private SwarmTopologyProvider topologyProvider;
```

In `setUp()`, create a CentralisedTopologyProvider:

```java
topologyProvider = new CentralisedTopologyProvider();
```

Add `topologyProvider` as the last argument to the SwarmProvisioner constructor call.

For the `distributed_topology_skips_central_provisioning` test, create a local SwarmProvisioner
with a DISTRIBUTED provider:

```java
var distributedProvider = new SwarmTopologyProvider() {
    @Override
    public TopologyDescriptor describeTopology(UUID id) {
        return new TopologyDescriptor(TopologyType.DISTRIBUTED, Map.of());
    }
};
var distributedProvisioner = new SwarmProvisioner(
    workerProvisioner, coordinator, signalRegistry, activityTracker,
    roleTracker, teamDetector, progressTracker,
    new NoOpDispatchBudget(), null, List.of(listener), null, null,
    distributedProvider);
```

Then call `distributedProvisioner.evaluateAndProvision(...)`.

Apply same pattern for `distributed_topology_skips_deprovisioning`.

- [ ] **Step 5: Update all other callers of SwarmProvisioner constructor**

Three additional call sites need the new `topologyProvider` parameter:

**Production — RuntimeBeans.java:1366:**
Add `SwarmTopologyProvider topologyProvider` parameter to the `swarmProvisioner()` producer method.
Pass it as the last arg to `new SwarmProvisioner(...)` at line 1382.
The no-arg constructor call at line 1380 (inactive path) needs no change — it already sets `topologyProvider = null`.
Add import: `io.casehub.api.spi.coordination.SwarmTopologyProvider`.

**Test — IntegrationDelayTest.java:66:**
Add `new CentralisedTopologyProvider()` as the last argument to `new SwarmProvisioner(...)`.
Add import: `io.casehub.engine.runtime.coordination.CentralisedTopologyProvider`.

**Test — SelfProvisioningIntegrationTest.java:72:**
Add `new CentralisedTopologyProvider()` as the last argument to `new SwarmProvisioner(...)`.
Add import: `io.casehub.engine.runtime.coordination.CentralisedTopologyProvider`.

- [ ] **Step 6: Run all tests to verify everything passes**

Run: `mvn test -pl runtime-core -Dtest=SwarmProvisionerTest -q`
Expected: All tests PASS (existing + 3 new)

Run: `mvn test -pl runtime-core -q`
Expected: Full module passes — no regressions

- [ ] **Step 7: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/engine/runtime/stigmergy/SwarmProvisioner.java \
       runtime/src/main/java/io/casehub/engine/runtime/quarkus/RuntimeBeans.java \
       runtime-core/src/test/java/io/casehub/engine/runtime/stigmergy/SwarmProvisionerTest.java \
       runtime-core/src/test/java/io/casehub/engine/runtime/stigmergy/IntegrationDelayTest.java \
       runtime-core/src/test/java/io/casehub/engine/runtime/stigmergy/SelfProvisioningIntegrationTest.java
git commit -m "feat: SwarmProvisioner consults topology before central provisioning

DISTRIBUTED topology skips central provisioning and deprovisioning.
CENTRALISED and HYBRID proceed with existing behavior.

Refs engine#1284"
```

## References

- [2026-11-02-swarm-topology-provider-design.md] — design spec this plan implements
- [api/model/coordination/TopologyType.java] — deriveAuthority method
- [api/model/coordination/AgentRole.java] — role record with structural/platform facets
- [api/spi/AgentCapabilityGuard.java] — established agent identity as String
- [runtime-core/runtime/coordination/NoOpAgentCapabilityGuard.java] — default impl pattern
- [runtime/quarkus/SpiDefaultBeans.java] — DefaultBean producer pattern
- [runtime-core/runtime/stigmergy/SwarmProvisioner.java] — integration target
- [runtime-core/test/stigmergy/SwarmProvisionerTest.java] — existing test patterns
- [specs/issue-1282-coordination-vocabulary-types/SPEC.md] — vocabulary spec, deferral table
- engine#1284 — requirements
- casehubio/agency#377 — parent epic
