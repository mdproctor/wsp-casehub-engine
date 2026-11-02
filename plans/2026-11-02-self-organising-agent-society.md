# Self-Organising Agent Society Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/eidos#216 — Self-organising agent society
**Issue group:** eidos#216, engine#1282, engine#1284, engine#1285, qhorus#509

**Goal:** Enable runtime self-forming collaborative groups with mutable
topologies, policy-governed roles, and desired-state reorganisation.

**Architecture:** Extend eidos-org-api with group lifecycle types
(GroupInstance, GroupPolicy, GroupLifecycleService). Add
SocietyLifecycleManager as the single entry point for all group
mutations. Extend the desiredstate/eidos bridge with org-specific
provisioner, drift policy, and actual-state adapter. Replace imperative
boot with unified desired-state-first reconciliation.

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI, casehub-desiredstate SPIs,
eidos-org-api/runtime/memory modules

## Global Constraints

- Quarkus version: `3.32.2`
- eidos cross-project version: `0.2-SNAPSHOT`
- All `OrgRegistry` methods require `tenancyId` — tenancy isolation invariant
- `OrganizationalUnit.MAX_MEMBERS = 50` — hard cap on distinct agents per group
- selfDetermination range: [0.0, 1.0] — validated at record construction
- GroupInstance is transient (in-memory) — desired-state graph is the recovery layer
- All OrgRegistry writes go through SocietyLifecycleManager — single entry point

---

## Batch 1: API Foundation (eidos-org-api)

New records, enums, and interfaces that define the group lifecycle model.
After this batch: the API compiles, all types are available for downstream
consumers, existing tests still pass.

### Task 1: Core Policy and Lifecycle Types

**Files:**
- Create: `org-api/src/main/java/io/casehub/eidos/org/api/AuthorityModel.java`
- Create: `org-api/src/main/java/io/casehub/eidos/org/api/GroupLifecycleState.java`
- Create: `org-api/src/main/java/io/casehub/eidos/org/api/FormationPath.java`
- Create: `org-api/src/main/java/io/casehub/eidos/org/api/RolePolicy.java`
- Create: `org-api/src/main/java/io/casehub/eidos/org/api/FormationRules.java`
- Create: `org-api/src/main/java/io/casehub/eidos/org/api/GroupPolicy.java`
- Test: `org-api/src/test/java/io/casehub/eidos/org/api/GroupPolicyTest.java`

**Interfaces:**
- Produces: `AuthorityModel` enum (CENTRALISED, DISTRIBUTED, HYBRID, HIERARCHICAL, GRADUATED)
- Produces: `GroupLifecycleState` enum (PROPOSED, FORMING, ACTIVE, REORGANISING, DISSOLVING, DISSOLVED)
- Produces: `FormationPath` enum (TOP_DOWN, EMERGENCE, BOTTOM_UP)
- Produces: `RolePolicy(List<String> obligations, List<String> permissions, int minAgents, int maxAgents)`
- Produces: `FormationRules(Set<FormationPath> allowedPaths, int minGroupSize, int maxGroupSize, boolean allowSubGroups, double proposalQuorum)`
- Produces: `GroupPolicy(Map<String, RolePolicy> roles, FormationRules formationRules, double topologyChangeThreshold, int normativePriority, NormSpecificity normSpecificity)`

- [ ] **Step 1: Write tests for AuthorityModel, GroupLifecycleState, FormationPath enums**

```java
package io.casehub.eidos.org.api;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class GroupPolicyTest {

    @Test
    void authorityModelValues() {
        assertEquals(5, AuthorityModel.values().length);
        assertNotNull(AuthorityModel.CENTRALISED);
        assertNotNull(AuthorityModel.DISTRIBUTED);
        assertNotNull(AuthorityModel.GRADUATED);
    }

    @Test
    void groupLifecycleStateValues() {
        assertEquals(6, GroupLifecycleState.values().length);
        assertNotNull(GroupLifecycleState.PROPOSED);
        assertNotNull(GroupLifecycleState.ACTIVE);
        assertNotNull(GroupLifecycleState.DISSOLVED);
    }

    @Test
    void formationPathValues() {
        assertEquals(3, FormationPath.values().length);
    }

    @Test
    void rolePolicyDefaults() {
        var rp = new RolePolicy(List.of("review-output"), List.of("provision-agent"), 1, 3);
        assertEquals(1, rp.minAgents());
        assertEquals(3, rp.maxAgents());
        assertEquals(1, rp.obligations().size());
    }

    @Test
    void formationRulesDefaults() {
        var fr = new FormationRules(
            Set.of(FormationPath.TOP_DOWN, FormationPath.EMERGENCE),
            2, 10, true, 1.0);
        assertEquals(2, fr.minGroupSize());
        assertTrue(fr.allowSubGroups());
        assertEquals(1.0, fr.proposalQuorum());
    }

    @Test
    void groupPolicyWithRoles() {
        var coordinator = new RolePolicy(
            List.of("review-output", "approve-plan"),
            List.of("provision-agent", "change-topology"), 1, 1);
        var analyst = new RolePolicy(
            List.of("process-assigned-work"),
            List.of("signal-consensus"), 1, 5);
        var policy = new GroupPolicy(
            Map.of("coordinator", coordinator, "analyst", analyst),
            new FormationRules(Set.of(FormationPath.TOP_DOWN), 2, 10, true, 1.0),
            0.5, 0, null);
        assertEquals(2, policy.roles().size());
        assertEquals(0, policy.normativePriority());
        assertNull(policy.normSpecificity());
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl org-api -Dtest=GroupPolicyTest -f /Users/mdproctor/claude/casehub/eidos/pom.xml`
Expected: compilation errors — types don't exist yet

- [ ] **Step 3: Create AuthorityModel enum**

```java
package io.casehub.eidos.org.api;

public enum AuthorityModel {
    CENTRALISED, DISTRIBUTED, HYBRID, HIERARCHICAL, GRADUATED
}
```

- [ ] **Step 4: Create GroupLifecycleState enum**

```java
package io.casehub.eidos.org.api;

public enum GroupLifecycleState {
    PROPOSED, FORMING, ACTIVE, REORGANISING, DISSOLVING, DISSOLVED
}
```

- [ ] **Step 5: Create FormationPath enum**

```java
package io.casehub.eidos.org.api;

public enum FormationPath {
    TOP_DOWN, EMERGENCE, BOTTOM_UP
}
```

- [ ] **Step 6: Create RolePolicy record**

```java
package io.casehub.eidos.org.api;

import java.util.List;

public record RolePolicy(
    List<String> obligations,
    List<String> permissions,
    int minAgents,
    int maxAgents
) {
    public RolePolicy {
        obligations = List.copyOf(obligations);
        permissions = List.copyOf(permissions);
        if (minAgents < 0) throw new IllegalArgumentException("minAgents must be >= 0");
        if (maxAgents < minAgents) throw new IllegalArgumentException("maxAgents must be >= minAgents");
    }
}
```

- [ ] **Step 7: Create FormationRules record**

```java
package io.casehub.eidos.org.api;

import java.util.Set;

public record FormationRules(
    Set<FormationPath> allowedPaths,
    int minGroupSize,
    int maxGroupSize,
    boolean allowSubGroups,
    double proposalQuorum
) {
    public FormationRules {
        allowedPaths = Set.copyOf(allowedPaths);
        if (minGroupSize < 1) throw new IllegalArgumentException("minGroupSize must be >= 1");
        if (maxGroupSize < minGroupSize) throw new IllegalArgumentException("maxGroupSize must be >= minGroupSize");
        if (proposalQuorum < 0.0 || proposalQuorum > 1.0)
            throw new IllegalArgumentException("proposalQuorum must be in [0.0, 1.0]");
    }
}
```

- [ ] **Step 8: Create GroupPolicy record**

```java
package io.casehub.eidos.org.api;

import java.util.Map;

public record GroupPolicy(
    Map<String, RolePolicy> roles,
    FormationRules formationRules,
    double topologyChangeThreshold,
    int normativePriority,
    NormSpecificity normSpecificity
) {
    public GroupPolicy {
        roles = Map.copyOf(roles);
        if (topologyChangeThreshold < 0.0 || topologyChangeThreshold > 1.0)
            throw new IllegalArgumentException("topologyChangeThreshold must be in [0.0, 1.0]");
    }
}
```

Note: `NormSpecificity` is from agency-core's normative package. If not
on the eidos-org-api classpath, use a `String normSpecificity` with
well-known values instead — keep eidos decoupled from agency.

- [ ] **Step 9: Run tests to verify they pass**

Run: `mvn test -pl org-api -Dtest=GroupPolicyTest -f /Users/mdproctor/claude/casehub/eidos/pom.xml`
Expected: PASS

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add org-api/src/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(org-api): AuthorityModel, GroupLifecycleState, GroupPolicy types Refs eidos#216"
```

### Task 2: GroupInstance and Runtime State Types

**Files:**
- Create: `org-api/src/main/java/io/casehub/eidos/org/api/ActualMembership.java`
- Create: `org-api/src/main/java/io/casehub/eidos/org/api/RoleAdoption.java`
- Create: `org-api/src/main/java/io/casehub/eidos/org/api/GroupInstance.java`
- Create: `org-api/src/main/java/io/casehub/eidos/org/api/GroupFormationRequest.java`
- Test: `org-api/src/test/java/io/casehub/eidos/org/api/GroupInstanceTest.java`

**Interfaces:**
- Consumes: `GroupLifecycleState`, `AuthorityModel`, `FormationPath`, `GroupPolicy`, `Membership` from Task 1
- Produces: `ActualMembership(String agentId, Instant joinedAt)`
- Produces: `RoleAdoption(String agentId, String role, String roleVocabulary, Instant adoptedAt)`
- Produces: `GroupInstance(String unitId, String tenancyId, GroupLifecycleState state, List<ActualMembership> members, String effectiveTopology, AuthorityModel effectiveAuthority, List<RoleAdoption> roleAdoptions, Instant stateChangedAt)`
- Produces: `GroupFormationRequest(String parentUnitId, String tenancyId, String name, String kind, String kindVocabulary, String topology, String topologyVocabulary, AuthorityModel authorityModel, GroupPolicy policy, double selfDetermination, List<Membership> initialMembers, FormationPath formationPath)`

- [ ] **Step 1: Write tests for GroupInstance invariants**

```java
package io.casehub.eidos.org.api;

import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class GroupInstanceTest {

    @Test
    void createGroupInstance() {
        var now = Instant.now();
        var member = new ActualMembership("agent-1", now);
        var role = new RoleAdoption("agent-1", "coordinator", null, now);
        var gi = new GroupInstance("unit-1", "tenant-1",
            GroupLifecycleState.ACTIVE,
            List.of(member), "hive", AuthorityModel.CENTRALISED,
            List.of(role), now);
        assertEquals("unit-1", gi.unitId());
        assertEquals(GroupLifecycleState.ACTIVE, gi.state());
        assertEquals(1, gi.members().size());
        assertEquals("hive", gi.effectiveTopology());
    }

    @Test
    void roleAdoptionRequiresMembership() {
        var now = Instant.now();
        var role = new RoleAdoption("agent-2", "analyst", null, now);
        var gi = new GroupInstance("unit-1", "tenant-1",
            GroupLifecycleState.ACTIVE,
            List.of(new ActualMembership("agent-1", now)),
            "swarm", AuthorityModel.DISTRIBUTED,
            List.of(role), now);
        // agent-2 has a role but is not a member — invariant violation
        assertFalse(gi.members().stream()
            .anyMatch(m -> m.agentId().equals("agent-2")));
        assertTrue(gi.roleAdoptions().stream()
            .anyMatch(r -> r.agentId().equals("agent-2")));
        // Note: invariant enforcement is in SocietyLifecycleManager, not the record
    }

    @Test
    void groupFormationRequestCarriesPath() {
        var request = new GroupFormationRequest(
            "parent-1", "tenant-1", "fraud-team",
            "team", null, "hive", null,
            AuthorityModel.CENTRALISED,
            new GroupPolicy(Map.of(), new FormationRules(
                Set.of(FormationPath.TOP_DOWN), 2, 10, true, 1.0),
                0.5, 0, null),
            0.5,
            List.of(new Membership("agent-1", "coordinator", null)),
            FormationPath.TOP_DOWN);
        assertEquals(FormationPath.TOP_DOWN, request.formationPath());
        assertEquals("hive", request.topology());
    }
}
```

- [ ] **Step 2: Run tests — verify compilation failure**

- [ ] **Step 3: Create ActualMembership, RoleAdoption, GroupInstance, GroupFormationRequest records**

Each as its own file following the spec's type definitions (§6.1).

- [ ] **Step 4: Run tests — verify pass**

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add org-api/src/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(org-api): GroupInstance, GroupFormationRequest runtime state types Refs eidos#216"
```

### Task 3: Extend OrganizationalUnit with Society Fields

**Files:**
- Modify: `org-api/src/main/java/io/casehub/eidos/org/api/OrganizationalUnit.java`
- Modify: tests and builders that construct OrganizationalUnit
- Test: `org-api/src/test/java/io/casehub/eidos/org/api/OrganizationalUnitSocietyTest.java`

**Interfaces:**
- Consumes: `AuthorityModel`, `GroupPolicy` from Task 1
- Produces: Extended `OrganizationalUnit` with `topology: String`, `topologyVocabulary: String`, `authorityModel: AuthorityModel`, `policy: GroupPolicy`, `selfDetermination: double`

- [ ] **Step 1: Write test for extended OrganizationalUnit**

```java
@Test
void organizationalUnitWithTopologyAndPolicy() {
    var ou = OrganizationalUnit.builder()
        .unitId("fraud-team").name("Fraud Team")
        .kind("team").tenancyId("tenant-1")
        .topology("hive").topologyVocabulary("casehub:topology")
        .authorityModel(AuthorityModel.CENTRALISED)
        .selfDetermination(0.7)
        .policy(new GroupPolicy(Map.of(), ...))
        .build();
    assertEquals("hive", ou.topology());
    assertEquals(0.7, ou.selfDetermination());
}

@Test
void selfDeterminationValidation() {
    assertThrows(IllegalArgumentException.class, () ->
        OrganizationalUnit.builder()
            .unitId("x").name("x").tenancyId("t")
            .selfDetermination(1.5).build());
}
```

- [ ] **Step 2: Run tests — verify fail**

- [ ] **Step 3: Add new fields to OrganizationalUnit record and Builder**

Use `ide_edit_member` to add fields to the record and update the Builder.
New fields default to sensible values: `topology=null`, `authorityModel=null`,
`policy=null`, `selfDetermination=0.0`.

Add validation in compact constructor: `selfDetermination` must be in [0.0, 1.0].

- [ ] **Step 4: Fix downstream compilation — update all OrganizationalUnit construction sites**

Use `ide_find_references` on `OrganizationalUnit` to find all construction
sites. Add default values for new fields. The Builder pattern means most
sites won't need changes — only direct record construction.

- [ ] **Step 5: Run full org-api tests**

Run: `mvn test -pl org-api -f /Users/mdproctor/claude/casehub/eidos/pom.xml`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add org-api/src/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(org-api): extend OrganizationalUnit with topology, policy, selfDetermination Refs eidos#216"
```

### Task 4: GroupLifecycleService Interface and OrgRegistry Extensions

**Files:**
- Create: `org-api/src/main/java/io/casehub/eidos/org/api/GroupLifecycleService.java`
- Modify: `org-api/src/main/java/io/casehub/eidos/org/api/OrgRegistry.java`
- Test: `org-api/src/test/java/io/casehub/eidos/org/api/GroupLifecycleServiceTest.java`

**Interfaces:**
- Consumes: `GroupInstance`, `GroupFormationRequest`, `OrgQuery` from Tasks 1-3
- Produces: `GroupLifecycleService` SPI with `formGroup`, `dissolveGroup`, `joinGroup`, `leaveGroup`, `adoptRole`, `releaseRole`, `changeTopology`
- Produces: Extended `OrgRegistry` with `findGroupInstance(unitId, tenancyId)`, `findGroupInstances(OrgQuery)`

- [ ] **Step 1: Write interface contract tests**

```java
@Test
void groupLifecycleServiceMethodSignatures() {
    // Verify the interface compiles with expected signatures
    GroupLifecycleService service = new GroupLifecycleService() {
        public GroupInstance formGroup(GroupFormationRequest request) { return null; }
        public void dissolveGroup(String unitId, String tenancyId) {}
        public void joinGroup(String unitId, String agentId, String role, String tenancyId) {}
        public void leaveGroup(String unitId, String agentId, String tenancyId) {}
        public void adoptRole(String unitId, String agentId, String role, String tenancyId) {}
        public void releaseRole(String unitId, String agentId, String role, String tenancyId) {}
        public void changeTopology(String unitId, String topology,
            String topologyVocabulary, AuthorityModel authority, String tenancyId) {}
    };
    assertNotNull(service);
}
```

- [ ] **Step 2: Create GroupLifecycleService interface**

- [ ] **Step 3: Add GroupInstance query methods to OrgRegistry as default methods**

```java
default Optional<GroupInstance> findGroupInstance(String unitId, String tenancyId) {
    return Optional.empty();
}
default List<GroupInstance> findGroupInstances(OrgQuery query) {
    return List.of();
}
```

- [ ] **Step 4: Run full org-api tests — verify pass**

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add org-api/src/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(org-api): GroupLifecycleService SPI, OrgRegistry GroupInstance queries Refs eidos#216"
```

---

## Batch 2: In-Memory Implementation (eidos org-memory + org-runtime)

Implement GroupInstance storage and the SocietyLifecycleManager — the
single entry point for all group lifecycle operations. After this batch:
groups can be formed, joined, dissolved, with roles adopted and released
— all in-memory, with full test coverage.

### Task 5: InMemoryOrgRegistry GroupInstance Storage

**Files:**
- Modify: `org-memory/src/main/java/io/casehub/eidos/org/memory/InMemoryOrgRegistry.java`
- Test: `org-memory/src/test/java/io/casehub/eidos/org/memory/InMemoryOrgRegistryGroupInstanceTest.java`

**Interfaces:**
- Consumes: `GroupInstance`, `OrgQuery` from Batch 1
- Produces: In-memory `GroupInstance` CRUD — `registerGroupInstance`, `updateGroupInstance`, `removeGroupInstance`, `findGroupInstance`, `findGroupInstances`

- [ ] **Step 1: Write tests for GroupInstance CRUD**

```java
@Test
void registerAndFindGroupInstance() {
    var gi = new GroupInstance("unit-1", "tenant-1",
        GroupLifecycleState.ACTIVE, List.of(), "swarm",
        AuthorityModel.DISTRIBUTED, List.of(), Instant.now());
    registry.registerGroupInstance(gi);
    var found = registry.findGroupInstance("unit-1", "tenant-1");
    assertTrue(found.isPresent());
    assertEquals(GroupLifecycleState.ACTIVE, found.get().state());
}

@Test
void findGroupInstanceRespectstenancyId() {
    var gi = new GroupInstance("unit-1", "tenant-1", ...);
    registry.registerGroupInstance(gi);
    assertTrue(registry.findGroupInstance("unit-1", "tenant-2").isEmpty());
}
```

- [ ] **Step 2: Implement GroupInstance storage in InMemoryOrgRegistry**

Add `ConcurrentHashMap<String, GroupInstance> groupInstances` keyed by
`tenancyId + ":" + unitId`. Implement the query methods.

- [ ] **Step 3: Run tests — verify pass**

- [ ] **Step 4: Commit**

### Task 6: SocietyLifecycleManager — Formation and Membership

**Files:**
- Create: `org-runtime/src/main/java/io/casehub/eidos/org/runtime/SocietyLifecycleManager.java`
- Test: `org-runtime/src/test/java/io/casehub/eidos/org/runtime/SocietyLifecycleManagerTest.java`

**Interfaces:**
- Consumes: `GroupLifecycleService`, `OrgRegistry`, `GroupFormationRequest`, `GroupInstance`
- Produces: CDI bean implementing `GroupLifecycleService` — `formGroup()`, `joinGroup()`, `leaveGroup()`, `adoptRole()`, `releaseRole()`

- [ ] **Step 1: Write test for group formation**

```java
@Test
void formGroupCreatesUnitAndInstance() {
    var request = new GroupFormationRequest("root", "tenant-1",
        "fraud-team", "team", null, "hive", null,
        AuthorityModel.CENTRALISED, defaultPolicy(),
        0.5, List.of(new Membership("agent-1", "coordinator", null)),
        FormationPath.TOP_DOWN);
    var instance = manager.formGroup(request);
    assertEquals(GroupLifecycleState.ACTIVE, instance.state());
    assertEquals("hive", instance.effectiveTopology());
    assertTrue(registry.findUnit(instance.unitId(), "tenant-1").isPresent());
}
```

- [ ] **Step 2: Write test for formation path validation**

```java
@Test
void formGroupRejectsDisallowedPath() {
    var policy = new GroupPolicy(Map.of(),
        new FormationRules(Set.of(FormationPath.TOP_DOWN), 2, 10, true, 1.0),
        0.5, 0, null);
    var request = new GroupFormationRequest(...,
        policy, 0.5, members, FormationPath.BOTTOM_UP);
    assertThrows(IllegalArgumentException.class, () -> manager.formGroup(request));
}
```

- [ ] **Step 3: Write test for membership operations**

```java
@Test
void joinAndLeaveGroup() {
    var instance = manager.formGroup(formRequest);
    manager.joinGroup(instance.unitId(), "agent-2", "analyst", "tenant-1");
    var updated = registry.findGroupInstance(instance.unitId(), "tenant-1").get();
    assertEquals(2, updated.members().size());

    manager.leaveGroup(instance.unitId(), "agent-2", "tenant-1");
    updated = registry.findGroupInstance(instance.unitId(), "tenant-1").get();
    assertEquals(1, updated.members().size());
}
```

- [ ] **Step 4: Write test for role adoption/release**

```java
@Test
void adoptAndReleaseRole() {
    var instance = manager.formGroup(formRequest);
    manager.joinGroup(instance.unitId(), "agent-2", null, "tenant-1");
    manager.adoptRole(instance.unitId(), "agent-2", "analyst", "tenant-1");
    var updated = registry.findGroupInstance(instance.unitId(), "tenant-1").get();
    assertTrue(updated.roleAdoptions().stream()
        .anyMatch(r -> r.agentId().equals("agent-2") && r.role().equals("analyst")));

    manager.releaseRole(instance.unitId(), "agent-2", "analyst", "tenant-1");
    updated = registry.findGroupInstance(instance.unitId(), "tenant-1").get();
    assertFalse(updated.roleAdoptions().stream()
        .anyMatch(r -> r.agentId().equals("agent-2")));
}
```

- [ ] **Step 5: Implement SocietyLifecycleManager**

CDI `@ApplicationScoped` bean. Injects `OrgRegistry`. Implements
`GroupLifecycleService`. Key internals:
- Per-society `ReentrantLock` (keyed by root group's unitId)
- Per-agent membership version counter (`ConcurrentHashMap<String, AtomicLong>`)
- `formGroup()`: validate path, validate selfDetermination, create
  `OrganizationalUnit` (spec), create `GroupInstance` (FORMING → ACTIVE),
  register both in OrgRegistry
- `joinGroup()` / `leaveGroup()`: membership version CAS, update GroupInstance
- `adoptRole()` / `releaseRole()`: verify membership, update GroupInstance

- [ ] **Step 6: Run tests — verify pass**

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add org-runtime/src/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(org-runtime): SocietyLifecycleManager — formation, membership, roles Refs eidos#216"
```

### Task 7: SocietyLifecycleManager — Dissolution and Topology Changes

**Files:**
- Modify: `org-runtime/src/main/java/io/casehub/eidos/org/runtime/SocietyLifecycleManager.java`
- Test: `org-runtime/src/test/java/io/casehub/eidos/org/runtime/SocietyLifecycleManagerDissolutionTest.java`

**Interfaces:**
- Consumes: `GroupLifecycleService.dissolveGroup()`, `changeTopology()` from Task 6
- Produces: Dissolution with dependency ordering, topology transitions with REORGANISING state

- [ ] **Step 1: Write test for dissolution**

```java
@Test
void dissolveGroupReleasesAllRolesAndMembers() {
    var instance = manager.formGroup(formRequest);
    manager.dissolveGroup(instance.unitId(), "tenant-1");
    var updated = registry.findGroupInstance(instance.unitId(), "tenant-1").get();
    assertEquals(GroupLifecycleState.DISSOLVED, updated.state());
    assertTrue(updated.members().isEmpty());
    assertTrue(updated.roleAdoptions().isEmpty());
}

@Test
void dissolveChildBeforeParent() {
    var parent = manager.formGroup(parentRequest);
    var child = manager.formGroup(childRequestUnder(parent));
    manager.dissolveGroup(parent.unitId(), "tenant-1");
    // child should be dissolved first
    assertEquals(GroupLifecycleState.DISSOLVED,
        registry.findGroupInstance(child.unitId(), "tenant-1").get().state());
}
```

- [ ] **Step 2: Write test for topology change**

```java
@Test
void changeTopologyTransitionsViaReorganising() {
    var instance = manager.formGroup(swarmRequest);
    assertEquals("swarm", instance.effectiveTopology());

    manager.changeTopology(instance.unitId(), "hive", null,
        AuthorityModel.CENTRALISED, "tenant-1");
    var updated = registry.findGroupInstance(instance.unitId(), "tenant-1").get();
    assertEquals("hive", updated.effectiveTopology());
    assertEquals(AuthorityModel.CENTRALISED, updated.effectiveAuthority());
    assertEquals(GroupLifecycleState.ACTIVE, updated.state());
}
```

- [ ] **Step 3: Implement dissolution and topology change**

- [ ] **Step 4: Run tests — verify pass**

- [ ] **Step 5: Commit**

---

## Batch 3: Desired-State Bridge (casehub-desiredstate/eidos)

Extend the existing desiredstate/eidos bridge with org-specific provisioner,
drift policy, fault policy, and actual-state adapter. After this batch:
org diagrams can be compiled to desired-state graphs, groups can be
provisioned/deprovisioned through reconciliation, and drift is detected.

### Task 8: Extend OrgGoalCompiler for Society Fields

**Files:**
- Modify: `desiredstate/eidos/.../OrgGoalCompiler.java`
- Modify: `desiredstate/eidos/.../OrgUnitNodeSpec.java`
- Test: `desiredstate/eidos/.../OrgGoalCompilerSocietyTest.java`

**Interfaces:**
- Consumes: Extended `OrganizationalUnit` from Task 3
- Produces: `OrgUnitNodeSpec` with topology, policy, selfDetermination fields; parent→child dependency edges in compiled graph

- [ ] **Step 1: Write test for compiler producing topology in node spec**

- [ ] **Step 2: Extend OrgUnitNodeSpec with new fields**

- [ ] **Step 3: Update OrgGoalCompiler to populate new fields**

- [ ] **Step 4: Run tests — verify pass**

- [ ] **Step 5: Commit**

### Task 9: OrgNodeProvisioner and DesiredStateOrgAdapter

**Files:**
- Create: `desiredstate/eidos/.../OrgNodeProvisioner.java`
- Create: `desiredstate/eidos/.../DesiredStateOrgAdapter.java`
- Test: `desiredstate/eidos/.../OrgNodeProvisionerTest.java`

**Interfaces:**
- Consumes: `NodeProvisioner` SPI (desiredstate), `GroupLifecycleService`, `OrgRegistry`
- Produces: `OrgNodeProvisioner.provision()` delegates to `SocietyLifecycleManager.formGroup()`; `deprovision()` delegates to `dissolveGroup()`
- Produces: `DesiredStateOrgAdapter` reads `GroupInstance` from OrgRegistry as actual state

- [ ] **Step 1: Write test for provision creating a group**

- [ ] **Step 2: Write test for DesiredStateOrgAdapter reading actual state**

- [ ] **Step 3: Implement OrgNodeProvisioner**

- [ ] **Step 4: Implement DesiredStateOrgAdapter**

- [ ] **Step 5: Run tests — verify pass**

- [ ] **Step 6: Commit**

### Task 10: OrgDriftPolicy and OrgFaultPolicy

**Files:**
- Create: `desiredstate/eidos/.../OrgDriftPolicy.java`
- Create: `desiredstate/eidos/.../OrgFaultPolicy.java`
- Test: `desiredstate/eidos/.../OrgDriftPolicyTest.java`

**Interfaces:**
- Consumes: `DriftPolicy` SPI, `FaultPolicy` SPI (desiredstate)
- Produces: Drift detection (member departure, role gap, autonomous topology change); fault recovery (re-provision, substitute, escalate)

- [ ] **Step 1: Write tests for drift scenarios**

- [ ] **Step 2: Implement OrgDriftPolicy**

- [ ] **Step 3: Write tests for fault recovery**

- [ ] **Step 4: Implement OrgFaultPolicy**

- [ ] **Step 5: Run tests — verify pass**

- [ ] **Step 6: Commit**

---

## Batch 4: YAML Schema and Unified Boot Path

Replace the existing `organization:` YAML schema with the new `society:`
schema. Replace imperative OrgBootstrap with desired-state-first
reconciliation. After this batch: groups can be declared in YAML and
boot through the desired-state reconciliation path.

### Task 11: New YAML Society Schema

**Files:**
- Create: `org-runtime/src/main/java/io/casehub/eidos/org/runtime/yaml/YamlSociety.java`
- Modify: `org-runtime/src/main/java/io/casehub/eidos/org/runtime/yaml/ClasspathYamlOrgRegistrar.java`
- Delete: `org-runtime/src/main/java/io/casehub/eidos/org/runtime/yaml/YamlOrganization.java` (use `ide_refactor_safe_delete`)
- Test: `org-runtime/src/test/java/io/casehub/eidos/org/runtime/yaml/YamlSocietyParsingTest.java`

**Interfaces:**
- Consumes: `OrganizationalUnit`, `GroupPolicy`, `AuthorityModel`, `Membership`
- Produces: `YamlSociety` that parses `society:` root key with nested `groups:`, `topology:`, `policy:`, `members:`

- [ ] **Step 1: Write test for YAML parsing**

```java
@Test
void parseSocietyYaml() {
    var yaml = """
        society:
          id: claims-ops
          name: Claims Operations
          selfDetermination: 0.5
          groups:
            - id: fraud-watch
              topology:
                pattern: swarm
                authority: distributed
              selfDetermination: 0.7
              members:
                - agentId: agent-f1
                  role: analyst
        """;
    var society = YamlSociety.parse(yaml);
    assertEquals("claims-ops", society.id());
    assertEquals(1, society.groups().size());
    assertEquals("swarm", society.groups().get(0).topology().pattern());
}
```

- [ ] **Step 2: Implement YamlSociety**

- [ ] **Step 3: Update ClasspathYamlOrgRegistrar to use new schema**

Change resource path, root key, and parser. Output
`List<OrganizationalUnit>` with all new fields populated.

- [ ] **Step 4: Run tests — verify pass**

- [ ] **Step 5: Commit**

### Task 12: Unified Boot Path — Desired-State-First

**Files:**
- Modify: `desiredstate/eidos/.../OrgBootstrap.java` (desiredstate-side)
- Modify or delete: `org-runtime/src/main/java/io/casehub/eidos/org/runtime/OrgBootstrap.java` (eidos-side)
- Test: `desiredstate/eidos/.../OrgBootstrapUnifiedTest.java`

**Interfaces:**
- Consumes: `ClasspathYamlOrgRegistrar`, `OrgGoalCompiler`, `ReconciliationLoop`, `DesiredStateHistoryStore`
- Produces: Unified boot: check persisted graph → if exists, reconcile from it; if not, compile from YAML → reconcile

- [ ] **Step 1: Write test for fresh deployment boot**

```java
@Test
void freshDeploymentCompilesYamlAndReconciles() {
    // No persisted graph → YAML compiled → reconciliation provisions all groups
    bootstrap.start();
    var instance = registry.findGroupInstance("claims-ops", "tenant-1");
    assertTrue(instance.isPresent());
    assertEquals(GroupLifecycleState.ACTIVE, instance.get().state());
}
```

- [ ] **Step 2: Write test for restart boot (persisted graph)**

- [ ] **Step 3: Implement unified boot path**

- [ ] **Step 4: Remove imperative OrgBootstrap**

- [ ] **Step 5: Run tests — verify pass**

- [ ] **Step 6: Commit**

---

## Batch 5: Engine Integration (casehub-engine)

New SPIs in engine-api that consume eidos org events. These are thin
interfaces — implementations will be wired when the engine consumes
the society model. After this batch: engine has the SPIs ready for
downstream consumers.

### Task 13: TopologyProvider and AgentWorkLifecycleListener SPIs

**Files:**
- Create: `engine/api/src/main/java/io/casehub/api/.../TopologyProvider.java`
- Create: `engine/api/src/main/java/io/casehub/api/.../AgentWorkLifecycleListener.java`
- Test: `engine/api/src/test/java/io/casehub/api/.../TopologyProviderTest.java`

**Interfaces:**
- Produces: `TopologyProvider` — `topologyFor(agentId, tenancyId)`, `effectiveSelfDetermination(agentId, tenancyId)`
- Produces: `AgentWorkLifecycleListener` — `onRoleAdopted`, `onRoleReleased`, `onTopologyChanged`

Note: these land in the engine repo (casehubio/engine), not eidos.
Refs engine#1284, engine#1285.

- [ ] **Step 1: Write interface contract tests**

- [ ] **Step 2: Create TopologyProvider interface**

- [ ] **Step 3: Create AgentWorkLifecycleListener interface**

- [ ] **Step 4: Run tests — verify pass**

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add api/src/
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(api): TopologyProvider, AgentWorkLifecycleListener SPIs Refs #1284 #1285"
```

---

## References

- [2026-11-02-self-organising-agent-society-design.md] — design spec this plan implements
- [2026-11-02-society-moise-fitgap-research.md] — MOISE+ fit-gap research
- [2026-11-02-society-moise-fitgap-research-decisions.md] — D1-D10 design decisions
- [eidos org-api/OrganizationalUnit.java] — existing org unit record
- [eidos org-api/OrgRegistry.java] — existing registry interface
- [eidos org-memory/InMemoryOrgRegistry.java] — existing in-memory implementation
- [eidos org-runtime/OrgBootstrap.java] — existing imperative bootstrap (superseded)
- [desiredstate/eidos/OrgGoalCompiler.java] — existing goal compiler
- [desiredstate/eidos/OrgUnitNodeSpec.java] — existing node spec
- casehubio/eidos#216 — focal issue
- casehubio/engine#1284 — TopologyProvider SPI
- casehubio/engine#1285 — AgentWorkLifecycleListener SPI
- casehubio/qhorus#509 — Channel visibility semantics
