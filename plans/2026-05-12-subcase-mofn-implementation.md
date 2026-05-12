# Sub-case M-of-N Coordination Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement parallel sub-case spawning with M-of-N completion coordination, PropagationContext propagation, and structural parent/child references in casehub-engine.

**Architecture:** A `SubCaseGroup` entity (persisted via SPI, in-memory for tests) tracks N child case IDs, completed/rejected counts, and a `policyTriggered` idempotency guard. `SubCaseGroupPolicy` is a pure-logic utility class that evaluates threshold arithmetic. `SubCaseCompletionListener` routes terminal child events through the policy and resumes the parent when the threshold is met. Ungrouped (single-child) sub-cases are unchanged.

**Tech Stack:** Java 21, Quarkus 3.x, Mutiny (Uni<>), CDI, Hibernate Reactive (JPA module), in-memory ConcurrentHashMap (test module), AssertJ + Awaitility for tests.

**Spec:** `specs/2026-05-12-subcase-mofn-coordination-design.md`

**Issue:** casehubio/engine#112

---

## File Map

**Create:**
- `api/src/main/java/io/casehub/api/model/OnThresholdReached.java`
- `common/src/main/java/io/casehub/engine/internal/model/GroupStatus.java`
- `common/src/main/java/io/casehub/engine/internal/model/SubCaseGroup.java`
- `common/src/main/java/io/casehub/engine/spi/SubCaseGroupRepository.java`
- `blackboard/src/main/java/io/casehub/blackboard/subcase/SubCaseGroupLifecycleEvent.java`
- `blackboard/src/main/java/io/casehub/blackboard/subcase/SubCaseGroupPolicy.java`
- `blackboard/src/test/java/io/casehub/blackboard/subcase/SubCaseGroupPolicyTest.java`
- `persistence-memory/src/main/java/io/casehub/persistence/memory/MemorySubCaseGroupRepository.java`
- `persistence-memory/src/test/java/io/casehub/persistence/memory/MemorySubCaseGroupRepositoryTest.java`
- `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/SubCaseGroupEntity.java`
- `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaSubCaseGroupRepository.java`
- `blackboard/src/test/java/io/casehub/blackboard/subcase/SubCaseParallelIntegrationTest.java`
- `blackboard/src/test/java/io/casehub/blackboard/subcase/SubCaseMofNIntegrationTest.java`
- `blackboard/src/test/java/io/casehub/blackboard/subcase/SubCasePropagationContextTest.java`

**Modify:**
- `api/src/main/java/io/casehub/api/model/SubCase.java` — add `groupId`, `totalInGroup`, `requiredCount`, `onThresholdReached`
- `api/src/main/java/io/casehub/api/engine/CaseHubRuntime.java` — add 4-arg `startCase` overload
- `common/src/main/java/io/casehub/engine/internal/model/CaseInstance.java` — add `parentCaseId`
- `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/CaseInstanceEntity.java` — add `parent_case_id` column
- `runtime/src/main/java/io/casehub/engine/internal/engine/CaseHubRuntimeImpl.java` — implement new overload
- `runtime/src/main/java/io/casehub/engine/internal/engine/CaseHubReactor.java` — add `startCaseInternal`
- `blackboard/src/main/java/io/casehub/blackboard/subcase/SubCaseExecutionHandler.java` — grouped path
- `blackboard/src/main/java/io/casehub/blackboard/subcase/SubCaseCompletionListener.java` — split paths
- `blackboard/src/test/resources/application.properties` — activate `MemorySubCaseGroupRepository`

---

## Task 1: `OnThresholdReached` enum + `GroupStatus` enum

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/OnThresholdReached.java`
- Create: `common/src/main/java/io/casehub/engine/internal/model/GroupStatus.java`

- [ ] **Step 1: Create `OnThresholdReached`**

```java
package io.casehub.api.model;

public enum OnThresholdReached {
  KEEP,
  CANCEL
}
```

- [ ] **Step 2: Create `GroupStatus`**

```java
package io.casehub.engine.internal.model;

public enum GroupStatus {
  IN_PROGRESS,
  COMPLETED,
  REJECTED
}
```

- [ ] **Step 3: Build to verify no compilation errors**

```bash
cd ~/claude/casehub/engine
mvn install -DskipTests -q -pl api,common
```

Expected: `BUILD SUCCESS`

- [ ] **Step 4: Commit**

```bash
git add api/src/main/java/io/casehub/api/model/OnThresholdReached.java \
        common/src/main/java/io/casehub/engine/internal/model/GroupStatus.java
git commit -m "feat(api): add OnThresholdReached and GroupStatus enums (Refs #112)"
```

---

## Task 2: Extend `SubCase` model

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/SubCase.java`

- [ ] **Step 1: Write failing test in existing `SubCaseTest`**

Add to `blackboard/src/test/java/io/casehub/blackboard/stage/SubCaseTest.java`:

```java
@Test
void groupId_and_totalInGroup_stored() {
  SubCase sc = SubCase.builder()
      .namespace("ns").name("n").version("v")
      .groupId("site-review").totalInGroup(3).requiredCount(2)
      .onThresholdReached(OnThresholdReached.CANCEL)
      .build();
  assertThat(sc.groupId()).isEqualTo("site-review");
  assertThat(sc.totalInGroup()).isEqualTo(3);
  assertThat(sc.requiredCount()).isEqualTo(2);
  assertThat(sc.onThresholdReached()).isEqualTo(OnThresholdReached.CANCEL);
}

@Test
void requiredCount_defaults_to_totalInGroup() {
  SubCase sc = SubCase.builder()
      .namespace("ns").name("n").version("v")
      .groupId("g").totalInGroup(3)
      .build();
  assertThat(sc.requiredCount()).isEqualTo(3);
}

@Test
void onThresholdReached_defaults_to_keep() {
  SubCase sc = SubCase.builder()
      .namespace("ns").name("n").version("v")
      .groupId("g").totalInGroup(2)
      .build();
  assertThat(sc.onThresholdReached()).isEqualTo(OnThresholdReached.KEEP);
}
```

- [ ] **Step 2: Run to verify tests fail**

```bash
cd ~/claude/casehub/engine
mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl blackboard \
  -Dtest=SubCaseTest -q 2>&1 | tail -10
```

Expected: compilation error — `groupId()` not found.

- [ ] **Step 3: Add fields to `SubCase`**

Replace the field block and builder in `api/src/main/java/io/casehub/api/model/SubCase.java`:

```java
package io.casehub.api.model;

import java.util.Objects;

public class SubCase {
  private final String namespace;
  private final String name;
  private final String version;
  private final SubCaseCompletionStrategy completionStrategy;
  private final boolean waitForCompletion;
  private final String inputMapping;
  private final String outputMapping;
  private final String groupId;
  private final int totalInGroup;
  private final int requiredCount;
  private final OnThresholdReached onThresholdReached;

  private SubCase(Builder b) {
    this.namespace = Objects.requireNonNull(b.namespace, "namespace");
    this.name = Objects.requireNonNull(b.name, "name");
    this.version = Objects.requireNonNull(b.version, "version");
    this.completionStrategy =
        b.completionStrategy != null ? b.completionStrategy : new DefaultSubCaseCompletionStrategy();
    this.waitForCompletion = b.waitForCompletion;
    this.inputMapping = b.inputMapping != null ? b.inputMapping : ".";
    this.outputMapping = b.outputMapping;
    this.groupId = b.groupId;
    this.totalInGroup = b.totalInGroup;
    this.requiredCount = b.requiredCount > 0 ? b.requiredCount : b.totalInGroup;
    this.onThresholdReached = b.onThresholdReached != null ? b.onThresholdReached : OnThresholdReached.KEEP;
  }

  public String namespace() { return namespace; }
  public String name() { return name; }
  public String version() { return version; }
  public SubCaseCompletionStrategy completionStrategy() { return completionStrategy; }
  public boolean waitForCompletion() { return waitForCompletion; }
  public String inputMapping() { return inputMapping; }
  public String outputMapping() { return outputMapping; }
  public String groupId() { return groupId; }
  public int totalInGroup() { return totalInGroup; }
  public int requiredCount() { return requiredCount; }
  public OnThresholdReached onThresholdReached() { return onThresholdReached; }

  public static Builder builder() { return new Builder(); }

  public static final class Builder {
    private String namespace, name, version, inputMapping, outputMapping, groupId;
    private SubCaseCompletionStrategy completionStrategy;
    private boolean waitForCompletion = true;
    private int totalInGroup = 0;
    private int requiredCount = 0;
    private OnThresholdReached onThresholdReached;

    public Builder namespace(String v) { namespace = v; return this; }
    public Builder name(String v) { name = v; return this; }
    public Builder version(String v) { version = v; return this; }
    public Builder completionStrategy(SubCaseCompletionStrategy s) { completionStrategy = s; return this; }
    public Builder waitForCompletion(boolean v) { waitForCompletion = v; return this; }
    public Builder inputMapping(String v) { inputMapping = v; return this; }
    public Builder outputMapping(String v) { outputMapping = v; return this; }
    public Builder groupId(String v) { groupId = v; return this; }
    public Builder totalInGroup(int v) { totalInGroup = v; return this; }
    public Builder requiredCount(int v) { requiredCount = v; return this; }
    public Builder onThresholdReached(OnThresholdReached v) { onThresholdReached = v; return this; }
    public SubCase build() { return new SubCase(this); }
  }
}
```

- [ ] **Step 4: Run tests to verify pass**

```bash
cd ~/claude/casehub/engine
mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl blackboard \
  -Dtest=SubCaseTest -q 2>&1 | tail -5
```

Expected: `Tests run: 16, Failures: 0`

- [ ] **Step 5: Commit**

```bash
git add api/src/main/java/io/casehub/api/model/SubCase.java \
        blackboard/src/test/java/io/casehub/blackboard/stage/SubCaseTest.java
git commit -m "feat(api): add groupId/totalInGroup/requiredCount/onThresholdReached to SubCase (Refs #112)"
```

---

## Task 3: `CaseInstance.parentCaseId`

**Files:**
- Modify: `common/src/main/java/io/casehub/engine/internal/model/CaseInstance.java`
- Modify: `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/CaseInstanceEntity.java`

- [ ] **Step 1: Add `parentCaseId` to `CaseInstance`**

Add to the field list and getter/setter pair in `CaseInstance.java`:

```java
private UUID parentCaseId;

public UUID getParentCaseId() { return parentCaseId; }
public void setParentCaseId(UUID parentCaseId) { this.parentCaseId = parentCaseId; }
```

- [ ] **Step 2: Add column to `CaseInstanceEntity`**

Add to `CaseInstanceEntity.java`:

```java
@Column(name = "parent_case_id", nullable = true)
public UUID parentCaseId;
```

- [ ] **Step 3: Build to verify**

```bash
cd ~/claude/casehub/engine
mvn install -DskipTests -q -pl common,persistence-hibernate
```

Expected: `BUILD SUCCESS`

- [ ] **Step 4: Commit**

```bash
git add common/src/main/java/io/casehub/engine/internal/model/CaseInstance.java \
        persistence-hibernate/src/main/java/io/casehub/persistence/jpa/CaseInstanceEntity.java
git commit -m "feat(model): add parentCaseId to CaseInstance (Refs #112)"
```

---

## Task 4: `SubCaseGroup` POJO + `SubCaseGroupRepository` SPI

**Files:**
- Create: `common/src/main/java/io/casehub/engine/internal/model/SubCaseGroup.java`
- Create: `common/src/main/java/io/casehub/engine/spi/SubCaseGroupRepository.java`

- [ ] **Step 1: Create `SubCaseGroup` POJO**

```java
package io.casehub.engine.internal.model;

import io.casehub.api.model.OnThresholdReached;
import java.util.HashSet;
import java.util.Set;
import java.util.UUID;

public class SubCaseGroup {
  private UUID parentCaseId;
  private String groupId;
  private int instanceCount;
  private int requiredCount;
  private int completedCount;
  private int rejectedCount;
  private boolean policyTriggered;
  private OnThresholdReached onThresholdReached;
  private final Set<UUID> childCaseIds = new HashSet<>();

  public UUID getParentCaseId() { return parentCaseId; }
  public void setParentCaseId(UUID v) { parentCaseId = v; }
  public String getGroupId() { return groupId; }
  public void setGroupId(String v) { groupId = v; }
  public int getInstanceCount() { return instanceCount; }
  public void setInstanceCount(int v) { instanceCount = v; }
  public int getRequiredCount() { return requiredCount; }
  public void setRequiredCount(int v) { requiredCount = v; }
  public int getCompletedCount() { return completedCount; }
  public void setCompletedCount(int v) { completedCount = v; }
  public int getRejectedCount() { return rejectedCount; }
  public void setRejectedCount(int v) { rejectedCount = v; }
  public boolean isPolicyTriggered() { return policyTriggered; }
  public void setPolicyTriggered(boolean v) { policyTriggered = v; }
  public OnThresholdReached getOnThresholdReached() { return onThresholdReached; }
  public void setOnThresholdReached(OnThresholdReached v) { onThresholdReached = v; }
  public Set<UUID> getChildCaseIds() { return childCaseIds; }
}
```

- [ ] **Step 2: Create `SubCaseGroupRepository` SPI**

```java
package io.casehub.engine.spi;

import io.casehub.api.model.OnThresholdReached;
import io.casehub.engine.internal.model.SubCaseGroup;
import io.smallrye.mutiny.Uni;
import java.util.Optional;
import java.util.UUID;

public interface SubCaseGroupRepository {

  Uni<SubCaseGroup> getOrCreate(UUID parentCaseId, String groupId, int totalInGroup,
                                int requiredCount, OnThresholdReached onThresholdReached);

  Uni<SubCaseGroup> registerChild(UUID parentCaseId, String groupId, UUID childCaseId);

  Uni<SubCaseGroup> incrementCompleted(UUID parentCaseId, String groupId);

  Uni<SubCaseGroup> incrementRejected(UUID parentCaseId, String groupId);

  Uni<Void> markPolicyTriggered(UUID parentCaseId, String groupId);

  Uni<Optional<SubCaseGroup>> findByChildCaseId(UUID childCaseId);
}
```

- [ ] **Step 3: Build**

```bash
cd ~/claude/casehub/engine && mvn install -DskipTests -q -pl common
```

Expected: `BUILD SUCCESS`

- [ ] **Step 4: Commit**

```bash
git add common/src/main/java/io/casehub/engine/internal/model/SubCaseGroup.java \
        common/src/main/java/io/casehub/engine/spi/SubCaseGroupRepository.java
git commit -m "feat(common): SubCaseGroup POJO and SubCaseGroupRepository SPI (Refs #112)"
```

---

## Task 5: `SubCaseGroupPolicy` + `SubCaseGroupLifecycleEvent`

**Files:**
- Create: `blackboard/src/main/java/io/casehub/blackboard/subcase/SubCaseGroupLifecycleEvent.java`
- Create: `blackboard/src/main/java/io/casehub/blackboard/subcase/SubCaseGroupPolicy.java`
- Create: `blackboard/src/test/java/io/casehub/blackboard/subcase/SubCaseGroupPolicyTest.java`

- [ ] **Step 1: Write failing test**

```java
package io.casehub.blackboard.subcase;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.api.model.OnThresholdReached;
import io.casehub.engine.internal.model.GroupStatus;
import io.casehub.engine.internal.model.SubCaseGroup;
import java.util.UUID;
import org.junit.jupiter.api.Test;

class SubCaseGroupPolicyTest {

  private SubCaseGroup group(int instanceCount, int requiredCount, int completed, int rejected) {
    SubCaseGroup g = new SubCaseGroup();
    g.setParentCaseId(UUID.randomUUID());
    g.setGroupId("test-group");
    g.setInstanceCount(instanceCount);
    g.setRequiredCount(requiredCount);
    g.setCompletedCount(completed);
    g.setRejectedCount(rejected);
    g.setPolicyTriggered(false);
    g.setOnThresholdReached(OnThresholdReached.KEEP);
    return g;
  }

  @Test
  void allOf_allComplete_returnsCompleted() {
    assertThat(SubCaseGroupPolicy.evaluate(group(3, 3, 3, 0))).isEqualTo(GroupStatus.COMPLETED);
  }

  @Test
  void anyOf_firstCompletes_returnsCompleted() {
    assertThat(SubCaseGroupPolicy.evaluate(group(3, 1, 1, 0))).isEqualTo(GroupStatus.COMPLETED);
  }

  @Test
  void mOfN_mComplete_returnsCompleted() {
    assertThat(SubCaseGroupPolicy.evaluate(group(3, 2, 2, 0))).isEqualTo(GroupStatus.COMPLETED);
  }

  @Test
  void mOfN_belowThreshold_returnsInProgress() {
    assertThat(SubCaseGroupPolicy.evaluate(group(3, 2, 1, 0))).isEqualTo(GroupStatus.IN_PROGRESS);
  }

  @Test
  void rejected_thresholdUnreachable_returnsRejected() {
    // 2 of 3 needed; 2 rejected, 0 remaining → unreachable
    assertThat(SubCaseGroupPolicy.evaluate(group(3, 2, 0, 2))).isEqualTo(GroupStatus.REJECTED);
  }

  @Test
  void policyTriggered_returnsNull() {
    SubCaseGroup g = group(3, 3, 3, 0);
    g.setPolicyTriggered(true);
    assertThat(SubCaseGroupPolicy.evaluate(g)).isNull();
  }

  @Test
  void toEvent_mapsAllFields() {
    SubCaseGroup g = group(3, 2, 2, 1);
    SubCaseGroupLifecycleEvent evt = SubCaseGroupPolicy.toEvent(g, GroupStatus.COMPLETED);
    assertThat(evt.parentCaseId()).isEqualTo(g.getParentCaseId());
    assertThat(evt.groupId()).isEqualTo("test-group");
    assertThat(evt.groupStatus()).isEqualTo(GroupStatus.COMPLETED);
    assertThat(evt.completedCount()).isEqualTo(2);
    assertThat(evt.instanceCount()).isEqualTo(3);
  }

  @Test
  void oneRemaining_oneNeeded_stillInProgress() {
    // 3 total, 2 needed, 1 completed, 1 rejected → 1 remaining, 1 needed → still possible
    assertThat(SubCaseGroupPolicy.evaluate(group(3, 2, 1, 1))).isEqualTo(GroupStatus.IN_PROGRESS);
  }

  @Test
  void zeroRemaining_thresholdNotMet_returnsRejected() {
    // 3 total, 2 needed, 1 completed, 2 rejected → 0 remaining, still need 1 → rejected
    assertThat(SubCaseGroupPolicy.evaluate(group(3, 2, 1, 2))).isEqualTo(GroupStatus.REJECTED);
  }
}
```

- [ ] **Step 2: Run to verify test fails**

```bash
cd ~/claude/casehub/engine
mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl blackboard \
  -Dtest=SubCaseGroupPolicyTest 2>&1 | tail -10
```

Expected: compilation failure — `SubCaseGroupPolicy` and `SubCaseGroupLifecycleEvent` not found.

- [ ] **Step 3: Create `SubCaseGroupLifecycleEvent`**

```java
package io.casehub.blackboard.subcase;

import io.casehub.engine.internal.model.GroupStatus;
import java.util.UUID;

public record SubCaseGroupLifecycleEvent(
    UUID parentCaseId,
    String groupId,
    int instanceCount,
    int requiredCount,
    int completedCount,
    int rejectedCount,
    GroupStatus groupStatus) {}
```

- [ ] **Step 4: Create `SubCaseGroupPolicy`**

```java
package io.casehub.blackboard.subcase;

import io.casehub.engine.internal.model.GroupStatus;
import io.casehub.engine.internal.model.SubCaseGroup;

public final class SubCaseGroupPolicy {

  private SubCaseGroupPolicy() {}

  public static GroupStatus evaluate(SubCaseGroup group) {
    if (group.isPolicyTriggered()) return null;
    int remaining = group.getInstanceCount() - group.getCompletedCount() - group.getRejectedCount();
    int needed = group.getRequiredCount() - group.getCompletedCount();
    if (group.getCompletedCount() >= group.getRequiredCount()) return GroupStatus.COMPLETED;
    if (remaining < needed) return GroupStatus.REJECTED;
    return GroupStatus.IN_PROGRESS;
  }

  public static SubCaseGroupLifecycleEvent toEvent(SubCaseGroup group, GroupStatus status) {
    return new SubCaseGroupLifecycleEvent(
        group.getParentCaseId(), group.getGroupId(),
        group.getInstanceCount(), group.getRequiredCount(),
        group.getCompletedCount(), group.getRejectedCount(),
        status);
  }
}
```

- [ ] **Step 5: Run tests to verify pass**

```bash
cd ~/claude/casehub/engine
mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl blackboard \
  -Dtest=SubCaseGroupPolicyTest 2>&1 | tail -5
```

Expected: `Tests run: 9, Failures: 0`

- [ ] **Step 6: Commit**

```bash
git add blackboard/src/main/java/io/casehub/blackboard/subcase/SubCaseGroupLifecycleEvent.java \
        blackboard/src/main/java/io/casehub/blackboard/subcase/SubCaseGroupPolicy.java \
        blackboard/src/test/java/io/casehub/blackboard/subcase/SubCaseGroupPolicyTest.java
git commit -m "feat(blackboard): SubCaseGroupPolicy pure logic + SubCaseGroupLifecycleEvent (Refs #112)"
```

---

## Task 6: `MemorySubCaseGroupRepository`

**Files:**
- Create: `persistence-memory/src/main/java/io/casehub/persistence/memory/MemorySubCaseGroupRepository.java`
- Create: `persistence-memory/src/test/java/io/casehub/persistence/memory/MemorySubCaseGroupRepositoryTest.java`
- Modify: `blackboard/src/test/resources/application.properties`

- [ ] **Step 1: Write failing test**

```java
package io.casehub.persistence.memory;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.api.model.OnThresholdReached;
import io.casehub.engine.internal.model.SubCaseGroup;
import java.util.Optional;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class MemorySubCaseGroupRepositoryTest {

  private MemorySubCaseGroupRepository repo;
  private final UUID parentId = UUID.randomUUID();
  private final String groupId = "test-group";

  @BeforeEach
  void setUp() {
    repo = new MemorySubCaseGroupRepository();
  }

  @Test
  void getOrCreate_createsGroup() {
    SubCaseGroup g = repo.getOrCreate(parentId, groupId, 3, 2, OnThresholdReached.KEEP)
        .await().indefinitely();
    assertThat(g.getInstanceCount()).isEqualTo(3);
    assertThat(g.getRequiredCount()).isEqualTo(2);
    assertThat(g.getCompletedCount()).isZero();
  }

  @Test
  void getOrCreate_idempotent() {
    repo.getOrCreate(parentId, groupId, 3, 2, OnThresholdReached.KEEP).await().indefinitely();
    SubCaseGroup second = repo.getOrCreate(parentId, groupId, 3, 2, OnThresholdReached.KEEP)
        .await().indefinitely();
    assertThat(second.getInstanceCount()).isEqualTo(3); // not doubled
  }

  @Test
  void registerChild_and_findByChildCaseId() {
    UUID childId = UUID.randomUUID();
    repo.getOrCreate(parentId, groupId, 3, 2, OnThresholdReached.KEEP).await().indefinitely();
    repo.registerChild(parentId, groupId, childId).await().indefinitely();
    Optional<SubCaseGroup> found = repo.findByChildCaseId(childId).await().indefinitely();
    assertThat(found).isPresent();
    assertThat(found.get().getParentCaseId()).isEqualTo(parentId);
  }

  @Test
  void findByChildCaseId_unknownChild_returnsEmpty() {
    Optional<SubCaseGroup> found = repo.findByChildCaseId(UUID.randomUUID()).await().indefinitely();
    assertThat(found).isEmpty();
  }

  @Test
  void incrementCompleted_updatesCount() {
    repo.getOrCreate(parentId, groupId, 3, 2, OnThresholdReached.KEEP).await().indefinitely();
    SubCaseGroup g = repo.incrementCompleted(parentId, groupId).await().indefinitely();
    assertThat(g.getCompletedCount()).isEqualTo(1);
  }

  @Test
  void incrementRejected_updatesCount() {
    repo.getOrCreate(parentId, groupId, 3, 2, OnThresholdReached.KEEP).await().indefinitely();
    SubCaseGroup g = repo.incrementRejected(parentId, groupId).await().indefinitely();
    assertThat(g.getRejectedCount()).isEqualTo(1);
  }

  @Test
  void markPolicyTriggered_setsFlag() {
    repo.getOrCreate(parentId, groupId, 3, 2, OnThresholdReached.KEEP).await().indefinitely();
    repo.markPolicyTriggered(parentId, groupId).await().indefinitely();
    SubCaseGroup g = repo.getOrCreate(parentId, groupId, 3, 2, OnThresholdReached.KEEP)
        .await().indefinitely();
    assertThat(g.isPolicyTriggered()).isTrue();
  }
}
```

- [ ] **Step 2: Run to verify test fails**

```bash
cd ~/claude/casehub/engine
mvn install -DskipTests -q && mvn test -pl persistence-memory \
  -Dtest=MemorySubCaseGroupRepositoryTest 2>&1 | tail -10
```

Expected: compilation error — class not found.

- [ ] **Step 3: Create `MemorySubCaseGroupRepository`**

```java
package io.casehub.persistence.memory;

import io.casehub.api.model.OnThresholdReached;
import io.casehub.engine.internal.model.SubCaseGroup;
import io.casehub.engine.spi.SubCaseGroupRepository;
import io.smallrye.mutiny.Uni;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import java.util.Optional;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;

@Alternative
@ApplicationScoped
public class MemorySubCaseGroupRepository implements SubCaseGroupRepository {

  private final ConcurrentHashMap<String, SubCaseGroup> groups = new ConcurrentHashMap<>();
  private final ConcurrentHashMap<UUID, String> childIndex = new ConcurrentHashMap<>();

  private static String key(UUID parentCaseId, String groupId) {
    return parentCaseId + ":" + groupId;
  }

  @Override
  public Uni<SubCaseGroup> getOrCreate(UUID parentCaseId, String groupId, int totalInGroup,
                                       int requiredCount, OnThresholdReached onThresholdReached) {
    String k = key(parentCaseId, groupId);
    SubCaseGroup g = groups.computeIfAbsent(k, __ -> {
      SubCaseGroup ng = new SubCaseGroup();
      ng.setParentCaseId(parentCaseId);
      ng.setGroupId(groupId);
      ng.setInstanceCount(totalInGroup);
      ng.setRequiredCount(requiredCount);
      ng.setOnThresholdReached(onThresholdReached != null ? onThresholdReached : OnThresholdReached.KEEP);
      return ng;
    });
    return Uni.createFrom().item(g);
  }

  @Override
  public Uni<SubCaseGroup> registerChild(UUID parentCaseId, String groupId, UUID childCaseId) {
    String k = key(parentCaseId, groupId);
    SubCaseGroup g = groups.get(k);
    if (g == null) return Uni.createFrom().failure(
        new IllegalStateException("Group not found: " + k));
    synchronized (g) { g.getChildCaseIds().add(childCaseId); }
    childIndex.put(childCaseId, k);
    return Uni.createFrom().item(g);
  }

  @Override
  public Uni<SubCaseGroup> incrementCompleted(UUID parentCaseId, String groupId) {
    String k = key(parentCaseId, groupId);
    SubCaseGroup g = groups.get(k);
    if (g == null) return Uni.createFrom().failure(
        new IllegalStateException("Group not found: " + k));
    synchronized (g) { g.setCompletedCount(g.getCompletedCount() + 1); }
    return Uni.createFrom().item(g);
  }

  @Override
  public Uni<SubCaseGroup> incrementRejected(UUID parentCaseId, String groupId) {
    String k = key(parentCaseId, groupId);
    SubCaseGroup g = groups.get(k);
    if (g == null) return Uni.createFrom().failure(
        new IllegalStateException("Group not found: " + k));
    synchronized (g) { g.setRejectedCount(g.getRejectedCount() + 1); }
    return Uni.createFrom().item(g);
  }

  @Override
  public Uni<Void> markPolicyTriggered(UUID parentCaseId, String groupId) {
    SubCaseGroup g = groups.get(key(parentCaseId, groupId));
    if (g != null) { synchronized (g) { g.setPolicyTriggered(true); } }
    return Uni.createFrom().voidItem();
  }

  @Override
  public Uni<Optional<SubCaseGroup>> findByChildCaseId(UUID childCaseId) {
    String k = childIndex.get(childCaseId);
    if (k == null) return Uni.createFrom().item(Optional.empty());
    return Uni.createFrom().item(Optional.ofNullable(groups.get(k)));
  }
}
```

- [ ] **Step 4: Run repository tests**

```bash
cd ~/claude/casehub/engine
mvn install -DskipTests -q && mvn test -pl persistence-memory \
  -Dtest=MemorySubCaseGroupRepositoryTest 2>&1 | tail -5
```

Expected: `Tests run: 7, Failures: 0`

- [ ] **Step 5: Activate in blackboard test `application.properties`**

In `blackboard/src/test/resources/application.properties`, append `MemorySubCaseGroupRepository` to the `quarkus.arc.selected-alternatives` list:

```properties
quarkus.arc.selected-alternatives=\
  io.casehub.persistence.memory.InMemoryCaseMetaModelRepository,\
  io.casehub.persistence.memory.InMemoryCaseInstanceRepository,\
  io.casehub.persistence.memory.InMemoryEventLogRepository,\
  io.casehub.persistence.memory.MemorySubCaseGroupRepository
```

- [ ] **Step 6: Commit**

```bash
git add persistence-memory/src/main/java/io/casehub/persistence/memory/MemorySubCaseGroupRepository.java \
        persistence-memory/src/test/java/io/casehub/persistence/memory/MemorySubCaseGroupRepositoryTest.java \
        blackboard/src/test/resources/application.properties
git commit -m "feat(persistence-memory): MemorySubCaseGroupRepository (Refs #112)"
```

---

## Task 7: `CaseHubRuntime` new overload + `CaseHubReactor`

**Files:**
- Modify: `api/src/main/java/io/casehub/api/engine/CaseHubRuntime.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/CaseHubRuntimeImpl.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/CaseHubReactor.java`

- [ ] **Step 1: Add overload to `CaseHubRuntime` interface**

Add after the existing `startCase(CaseDefinition, Map)` method:

```java
CompletionStage<UUID> startCase(CaseDefinition definition, Map<String, Object> inputData,
                                UUID parentCaseId, PropagationContext propagationContext);
```

Also add the import at the top:
```java
import io.casehub.api.context.PropagationContext;
```

- [ ] **Step 2: Implement in `CaseHubRuntimeImpl`**

Add after the existing 2-arg `startCase`:

```java
@Override
public CompletionStage<UUID> startCase(CaseDefinition definition, Map<String, Object> inputData,
                                       UUID parentCaseId, PropagationContext propagationContext) {
  return reactor.startCase(definition, new CaseContextImpl(inputData), parentCaseId, propagationContext);
}
```

- [ ] **Step 3: Refactor `CaseHubReactor`**

In `CaseHubReactor`, extract `startCaseInternal` and delegate both overloads to it. Replace the existing `startCase(CaseDefinition, CaseContext)` body and add the new overload:

```java
CompletionStage<UUID> startCase(CaseDefinition definition, CaseContext context) {
  return startCaseInternal(definition, context, null, null);
}

CompletionStage<UUID> startCase(CaseDefinition definition, CaseContext context,
                                UUID parentCaseId, PropagationContext propagationContext) {
  return startCaseInternal(definition, context, parentCaseId, propagationContext);
}

private CompletionStage<UUID> startCaseInternal(CaseDefinition definition, CaseContext context,
                                                 UUID parentCaseId, PropagationContext propagationContext) {
  return buildInstance(definition, context, parentCaseId, propagationContext)
      .flatMap(instance -> {
        eventBus.publish(CONTEXT_CHANGED,
            new CaseStateContextChangedEvent(instance.getUuid()));
        return Uni.createFrom().item(instance);
      })
      .transform(CaseInstance::getUuid)
      .subscribeAsCompletionStage();
}

private Uni<CaseInstance> buildInstance(CaseDefinition definition, CaseContext context,
                                         UUID parentCaseId, PropagationContext parentPropCtx) {
  CaseMetaModel model = caseDefinitionRegistry.getCaseMetaModel(definition);

  PropagationContext propagationContext;
  if (parentPropCtx != null) {
    propagationContext = parentPropCtx.createChild();
  } else {
    String traceId = traceIdProvider.currentTraceId()
        .filter(id -> !id.isBlank())
        .orElseGet(() -> UUID.randomUUID().toString());
    propagationContext = maxDuration
        .map(budget -> PropagationContext.createRoot(traceId, Map.<String, String>of(), budget))
        .orElse(PropagationContext.createRoot(traceId));
  }

  CaseInstance instance = new CaseInstance();
  instance.setUuid(UUID.randomUUID());
  instance.setCaseMetaModel(model);
  instance.setVersion(0L);
  instance.setState(CaseStatus.RUNNING);
  instance.setCaseContext(context);
  instance.setPropagationContext(propagationContext);
  instance.setParentCaseId(parentCaseId);

  caseInstanceCache.put(instance);
  return caseInstanceRepository.save(instance);
}
```

Note: rename `getCaseInstance` to `buildInstance` and remove it — `startCaseInternal` now calls `buildInstance` directly.

- [ ] **Step 4: Build all modules**

```bash
cd ~/claude/casehub/engine && mvn install -DskipTests -q
```

Expected: `BUILD SUCCESS`

- [ ] **Step 5: Run existing blackboard tests to verify nothing broke**

```bash
cd ~/claude/casehub/engine
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl blackboard 2>&1 | tail -8
```

Expected: `Tests run: 142, Failures: 0`

- [ ] **Step 6: Commit**

```bash
git add api/src/main/java/io/casehub/api/engine/CaseHubRuntime.java \
        runtime/src/main/java/io/casehub/engine/internal/engine/CaseHubRuntimeImpl.java \
        runtime/src/main/java/io/casehub/engine/internal/engine/CaseHubReactor.java
git commit -m "feat(runtime): startCase overload with parentCaseId + PropagationContext propagation (Refs #112)"
```

---

## Task 8: `JpaSubCaseGroupRepository` (persistence-hibernate)

**Files:**
- Create: `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/SubCaseGroupEntity.java`
- Create: `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaSubCaseGroupRepository.java`

- [ ] **Step 1: Create `SubCaseGroupEntity`**

```java
package io.casehub.persistence.jpa;

import io.casehub.api.model.OnThresholdReached;
import io.quarkus.hibernate.reactive.panache.PanacheEntity;
import jakarta.persistence.CollectionTable;
import jakarta.persistence.Column;
import jakarta.persistence.ElementCollection;
import jakarta.persistence.Entity;
import jakarta.persistence.EnumType;
import jakarta.persistence.Enumerated;
import jakarta.persistence.FetchType;
import jakarta.persistence.JoinColumn;
import jakarta.persistence.Table;
import jakarta.persistence.UniqueConstraint;
import java.util.HashSet;
import java.util.Set;
import java.util.UUID;

@Entity
@Table(name = "subcase_group",
    uniqueConstraints = @UniqueConstraint(columnNames = {"parent_case_id", "group_id"}))
public class SubCaseGroupEntity extends PanacheEntity {

  @Column(name = "parent_case_id", nullable = false)
  public UUID parentCaseId;

  @Column(name = "group_id", nullable = false, length = 255)
  public String groupId;

  @Column(name = "instance_count", nullable = false)
  public int instanceCount;

  @Column(name = "required_count", nullable = false)
  public int requiredCount;

  @Column(name = "completed_count", nullable = false)
  public int completedCount;

  @Column(name = "rejected_count", nullable = false)
  public int rejectedCount;

  @Column(name = "policy_triggered", nullable = false)
  public boolean policyTriggered;

  @Enumerated(EnumType.STRING)
  @Column(name = "on_threshold_reached", nullable = false, length = 50)
  public OnThresholdReached onThresholdReached;

  @ElementCollection(fetch = FetchType.EAGER)
  @CollectionTable(name = "subcase_group_children",
      joinColumns = @JoinColumn(name = "group_id_fk"))
  @Column(name = "child_case_id")
  public Set<UUID> childCaseIds = new HashSet<>();
}
```

- [ ] **Step 2: Create `JpaSubCaseGroupRepository`**

```java
package io.casehub.persistence.jpa;

import io.casehub.api.model.OnThresholdReached;
import io.casehub.engine.internal.model.SubCaseGroup;
import io.casehub.engine.spi.SubCaseGroupRepository;
import io.quarkus.hibernate.reactive.panache.Panache;
import io.smallrye.mutiny.Uni;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.HashSet;
import java.util.Optional;
import java.util.UUID;

@ApplicationScoped
public class JpaSubCaseGroupRepository implements SubCaseGroupRepository {

  @Override
  public Uni<SubCaseGroup> getOrCreate(UUID parentCaseId, String groupId, int totalInGroup,
                                       int requiredCount, OnThresholdReached onThresholdReached) {
    return Panache.withTransaction(() ->
        SubCaseGroupEntity.<SubCaseGroupEntity>find(
            "parentCaseId = ?1 and groupId = ?2", parentCaseId, groupId)
            .firstResult()
            .flatMap(existing -> {
              if (existing != null) return Uni.createFrom().item(toDomain(existing));
              SubCaseGroupEntity e = new SubCaseGroupEntity();
              e.parentCaseId = parentCaseId;
              e.groupId = groupId;
              e.instanceCount = totalInGroup;
              e.requiredCount = requiredCount;
              e.onThresholdReached = onThresholdReached != null ? onThresholdReached : OnThresholdReached.KEEP;
              return e.<SubCaseGroupEntity>persist().map(this::toDomain);
            }));
  }

  @Override
  public Uni<SubCaseGroup> registerChild(UUID parentCaseId, String groupId, UUID childCaseId) {
    return Panache.withTransaction(() ->
        SubCaseGroupEntity.<SubCaseGroupEntity>find(
            "parentCaseId = ?1 and groupId = ?2", parentCaseId, groupId)
            .firstResult()
            .flatMap(e -> {
              if (e == null) return Uni.createFrom().failure(
                  new IllegalStateException("Group not found: " + parentCaseId + ":" + groupId));
              e.childCaseIds.add(childCaseId);
              return Uni.createFrom().item(toDomain(e));
            }));
  }

  @Override
  public Uni<SubCaseGroup> incrementCompleted(UUID parentCaseId, String groupId) {
    return Panache.withTransaction(() ->
        SubCaseGroupEntity.<SubCaseGroupEntity>find(
            "parentCaseId = ?1 and groupId = ?2", parentCaseId, groupId)
            .firstResult()
            .flatMap(e -> {
              if (e == null) return Uni.createFrom().failure(
                  new IllegalStateException("Group not found: " + parentCaseId + ":" + groupId));
              e.completedCount++;
              return Uni.createFrom().item(toDomain(e));
            }));
  }

  @Override
  public Uni<SubCaseGroup> incrementRejected(UUID parentCaseId, String groupId) {
    return Panache.withTransaction(() ->
        SubCaseGroupEntity.<SubCaseGroupEntity>find(
            "parentCaseId = ?1 and groupId = ?2", parentCaseId, groupId)
            .firstResult()
            .flatMap(e -> {
              if (e == null) return Uni.createFrom().failure(
                  new IllegalStateException("Group not found: " + parentCaseId + ":" + groupId));
              e.rejectedCount++;
              return Uni.createFrom().item(toDomain(e));
            }));
  }

  @Override
  public Uni<Void> markPolicyTriggered(UUID parentCaseId, String groupId) {
    return Panache.withTransaction(() ->
        SubCaseGroupEntity.<SubCaseGroupEntity>find(
            "parentCaseId = ?1 and groupId = ?2", parentCaseId, groupId)
            .firstResult()
            .flatMap(e -> {
              if (e != null) e.policyTriggered = true;
              return Uni.createFrom().voidItem();
            }));
  }

  @Override
  public Uni<Optional<SubCaseGroup>> findByChildCaseId(UUID childCaseId) {
    return SubCaseGroupEntity.<SubCaseGroupEntity>find(
            "?1 member of childCaseIds", childCaseId)
        .firstResult()
        .map(e -> Optional.ofNullable(e == null ? null : toDomain(e)));
  }

  private SubCaseGroup toDomain(SubCaseGroupEntity e) {
    SubCaseGroup g = new SubCaseGroup();
    g.setParentCaseId(e.parentCaseId);
    g.setGroupId(e.groupId);
    g.setInstanceCount(e.instanceCount);
    g.setRequiredCount(e.requiredCount);
    g.setCompletedCount(e.completedCount);
    g.setRejectedCount(e.rejectedCount);
    g.setPolicyTriggered(e.policyTriggered);
    g.setOnThresholdReached(e.onThresholdReached);
    g.getChildCaseIds().addAll(e.childCaseIds != null ? e.childCaseIds : new HashSet<>());
    return g;
  }
}
```

- [ ] **Step 3: Build**

```bash
cd ~/claude/casehub/engine && mvn install -DskipTests -q -pl persistence-hibernate
```

Expected: `BUILD SUCCESS`

- [ ] **Step 4: Commit**

```bash
git add persistence-hibernate/src/main/java/io/casehub/persistence/jpa/SubCaseGroupEntity.java \
        persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaSubCaseGroupRepository.java
git commit -m "feat(persistence-hibernate): JpaSubCaseGroupRepository (Refs #112)"
```

---

## Task 9: `SubCaseExecutionHandler` — grouped path

**Files:**
- Modify: `blackboard/src/main/java/io/casehub/blackboard/subcase/SubCaseExecutionHandler.java`

- [ ] **Step 1: Update handler**

Replace the full `SubCaseExecutionHandler.java` with this (preserving the existing ungrouped path and adding the grouped path):

```java
package io.casehub.blackboard.subcase;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import io.casehub.api.engine.CaseHubRuntime;
import io.casehub.api.model.CaseStatus;
import io.casehub.api.model.SubCase;
import io.casehub.api.model.event.CaseHubEventType;
import io.casehub.api.model.event.EventStreamType;
import io.casehub.engine.internal.event.EventBusAddresses;
import io.casehub.engine.internal.event.SubCaseScheduleEvent;
import io.casehub.engine.internal.history.EventLog;
import io.casehub.engine.internal.model.CaseInstance;
import io.casehub.engine.internal.model.CaseMetaModel;
import io.casehub.engine.internal.work.PendingWorkRegistry;
import io.casehub.engine.spi.CaseDefinitionRegistry;
import io.casehub.engine.spi.CaseInstanceRepository;
import io.casehub.engine.spi.EventLogRepository;
import io.casehub.engine.spi.SubCaseGroupRepository;
import io.quarkus.vertx.ConsumeEvent;
import io.smallrye.mutiny.Uni;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.time.Instant;
import java.util.UUID;
import java.util.concurrent.CompletionStage;
import org.jboss.logging.Logger;

@ApplicationScoped
public class SubCaseExecutionHandler {

  private static final Logger LOG = Logger.getLogger(SubCaseExecutionHandler.class);
  private static final ObjectMapper OBJECT_MAPPER = new ObjectMapper();

  @Inject CaseHubRuntime caseHubRuntime;
  @Inject CaseDefinitionRegistry caseDefinitionRegistry;
  @Inject CaseInstanceRepository caseInstanceRepository;
  @Inject EventLogRepository eventLogRepository;
  @Inject PendingWorkRegistry pendingWorkRegistry;
  @Inject SubCaseGroupRepository subCaseGroupRepository;

  @ConsumeEvent(value = EventBusAddresses.SUBCASE_SCHEDULE, blocking = true)
  public Uni<Void> onSubCaseSchedule(SubCaseScheduleEvent event) {
    CaseInstance parent = event.parentInstance();
    SubCase subCase = event.subCase();

    CaseMetaModel parentMeta = parent.getCaseMetaModel();
    if (parentMeta != null
        && subCase.namespace().equals(parentMeta.getNamespace())
        && subCase.name().equals(parentMeta.getName())
        && subCase.version().equals(parentMeta.getVersion())) {
      LOG.errorf("SubCase circular dependency: case %s cannot spawn itself (%s/%s/%s)",
          parent.getUuid(), subCase.namespace(), subCase.name(), subCase.version());
      return Uni.createFrom().voidItem();
    }

    CaseMetaModel childMeta = new CaseMetaModel();
    childMeta.setNamespace(subCase.namespace());
    childMeta.setName(subCase.name());
    childMeta.setVersion(subCase.version());

    var childDefinition = caseDefinitionRegistry.getCaseDefinition(childMeta);
    if (childDefinition == null) {
      LOG.errorf("SubCaseExecutionHandler: no CaseDefinition for %s/%s/%s",
          subCase.namespace(), subCase.name(), subCase.version());
      return Uni.createFrom().voidItem();
    }

    CompletionStage<UUID> childFuture = caseHubRuntime.startCase(
        childDefinition,
        event.childInitialContext(),
        parent.getUuid(),
        parent.getPropagationContext());
    UUID childCaseId = childFuture.toCompletableFuture().join();

    LOG.infof("SubCase spawned: parentCaseId=%s childCaseId=%s grouped=%s",
        parent.getUuid(), childCaseId, subCase.groupId() != null);

    if (subCase.groupId() != null) {
      return handleGrouped(parent, subCase, childCaseId);
    } else {
      return handleUngrouped(parent, subCase, childCaseId);
    }
  }

  private Uni<Void> handleGrouped(CaseInstance parent, SubCase subCase, UUID childCaseId) {
    String groupId = subCase.groupId();

    return subCaseGroupRepository
        .getOrCreate(parent.getUuid(), groupId, subCase.totalInGroup(),
            subCase.requiredCount(), subCase.onThresholdReached())
        .flatMap(group -> subCaseGroupRepository.registerChild(parent.getUuid(), groupId, childCaseId))
        .flatMap(group -> {
          EventLog log = new EventLog();
          log.setCaseId(parent.getUuid());
          log.setWorkerId(childCaseId.toString());
          log.setEventType(CaseHubEventType.SUBCASE_STARTED);
          log.setStreamType(EventStreamType.CASE);
          log.setTimestamp(Instant.now());
          ObjectNode meta = OBJECT_MAPPER.createObjectNode();
          meta.put("childCaseId", childCaseId.toString());
          meta.put("groupId", groupId);
          meta.put("waitForCompletion", true);
          if (subCase.outputMapping() != null) meta.put("outputMapping", subCase.outputMapping());
          log.setMetadata(meta);

          if (parent.getState() != CaseStatus.WAITING
              || !groupId.equals(parent.getWaitingForWorkId())) {
            parent.setState(CaseStatus.WAITING);
            parent.setWaitingForWorkId(groupId);
            return caseInstanceRepository.updateStateAndAppendEvent(parent, log).replaceWithVoid();
          } else {
            return eventLogRepository.append(log).replaceWithVoid();
          }
        });
  }

  private Uni<Void> handleUngrouped(CaseInstance parent, SubCase subCase, UUID childCaseId) {
    EventLog log = new EventLog();
    log.setCaseId(parent.getUuid());
    log.setWorkerId(childCaseId.toString());
    log.setEventType(CaseHubEventType.SUBCASE_STARTED);
    log.setStreamType(EventStreamType.CASE);
    log.setTimestamp(Instant.now());
    ObjectNode meta = OBJECT_MAPPER.createObjectNode();
    meta.put("childCaseId", childCaseId.toString());
    meta.put("waitForCompletion", subCase.waitForCompletion());
    if (subCase.outputMapping() != null) meta.put("outputMapping", subCase.outputMapping());
    log.setMetadata(meta);

    if (subCase.waitForCompletion()) {
      pendingWorkRegistry.register(childCaseId.toString());
      parent.setState(CaseStatus.WAITING);
      parent.setWaitingForWorkId(childCaseId.toString());
      return caseInstanceRepository.updateStateAndAppendEvent(parent, log).replaceWithVoid();
    } else {
      return eventLogRepository.append(log).replaceWithVoid();
    }
  }
}
```

- [ ] **Step 2: Build**

```bash
cd ~/claude/casehub/engine && mvn install -DskipTests -q
```

Expected: `BUILD SUCCESS`

- [ ] **Step 3: Run existing blackboard tests**

```bash
cd ~/claude/casehub/engine
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl blackboard 2>&1 | tail -8
```

Expected: `Tests run: 142+, Failures: 0`

- [ ] **Step 4: Commit**

```bash
git add blackboard/src/main/java/io/casehub/blackboard/subcase/SubCaseExecutionHandler.java
git commit -m "feat(blackboard): SubCaseExecutionHandler grouped path with PropagationContext (Refs #112)"
```

---

## Task 10: `SubCaseCompletionListener` — split grouped/ungrouped paths

**Files:**
- Modify: `blackboard/src/main/java/io/casehub/blackboard/subcase/SubCaseCompletionListener.java`

- [ ] **Step 1: Replace `SubCaseCompletionListener`**

```java
package io.casehub.blackboard.subcase;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import io.casehub.api.engine.CaseHubRuntime;
import io.casehub.api.model.CaseStatus;
import io.casehub.api.model.DefaultSubCaseCompletionStrategy;
import io.casehub.api.model.SubCaseCompletionStrategy;
import io.casehub.api.model.event.CaseHubEventType;
import io.casehub.api.model.event.EventStreamType;
import io.casehub.engine.internal.event.CaseLifecycleEvent;
import io.casehub.engine.internal.history.EventLog;
import io.casehub.engine.internal.model.CaseInstance;
import io.casehub.engine.internal.model.GroupStatus;
import io.casehub.engine.internal.model.SubCaseGroup;
import io.casehub.engine.internal.work.CaseResumptionService;
import io.casehub.engine.spi.EventLogRepository;
import io.casehub.engine.spi.SubCaseGroupRepository;
import io.casehub.engine.spi.cache.CaseInstanceCache;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;
import java.time.Duration;
import java.time.Instant;
import java.util.Map;
import java.util.Optional;
import java.util.UUID;
import org.jboss.logging.Logger;

@ApplicationScoped
public class SubCaseCompletionListener {

  private static final Logger LOG = Logger.getLogger(SubCaseCompletionListener.class);
  private static final ObjectMapper OBJECT_MAPPER = new ObjectMapper();

  @Inject EventLogRepository eventLogRepository;
  @Inject CaseInstanceCache caseInstanceCache;
  @Inject CaseResumptionService caseResumptionService;
  @Inject SubCaseGroupRepository subCaseGroupRepository;
  @Inject CaseHubRuntime caseHubRuntime;

  public void onCaseLifecycle(@ObservesAsync CaseLifecycleEvent event) {
    if (!isTerminal(event.commandType())) return;

    UUID childCaseId = event.caseId();

    EventLog startedEntry = eventLogRepository
        .findByWorkerAndType(childCaseId.toString(), CaseHubEventType.SUBCASE_STARTED)
        .await().atMost(Duration.ofSeconds(10))
        .stream().findFirst().orElse(null);

    if (startedEntry == null) return;

    String groupId = startedEntry.getMetadata().has("groupId")
        ? startedEntry.getMetadata().get("groupId").asText(null)
        : null;

    if (groupId != null) {
      handleGroupedCompletion(childCaseId, event, startedEntry, groupId);
    } else {
      handleUngroupedCompletion(childCaseId, event, startedEntry);
    }
  }

  private void handleGroupedCompletion(UUID childCaseId, CaseLifecycleEvent event,
                                        EventLog startedEntry, String groupId) {
    UUID parentCaseId = startedEntry.getCaseId();
    CaseStatus childStatus = event.caseStatus() != null
        ? CaseStatus.valueOf(event.caseStatus()) : CaseStatus.FAULTED;
    boolean childCompleted = childStatus == CaseStatus.COMPLETED;

    SubCaseGroup group;
    if (childCompleted) {
      group = subCaseGroupRepository.incrementCompleted(parentCaseId, groupId)
          .await().atMost(Duration.ofSeconds(10));
    } else {
      group = subCaseGroupRepository.incrementRejected(parentCaseId, groupId)
          .await().atMost(Duration.ofSeconds(10));
    }

    GroupStatus groupStatus = SubCaseGroupPolicy.evaluate(group);
    if (groupStatus == null) return; // policyTriggered — already handled

    SubCaseGroupLifecycleEvent groupEvent = SubCaseGroupPolicy.toEvent(group, groupStatus);

    if (groupStatus == GroupStatus.COMPLETED || groupStatus == GroupStatus.REJECTED) {
      subCaseGroupRepository.markPolicyTriggered(parentCaseId, groupId)
          .await().atMost(Duration.ofSeconds(10));

      if (groupStatus == GroupStatus.COMPLETED
          && group.getOnThresholdReached() != null
          && group.getOnThresholdReached().name().equals("CANCEL")) {
        cancelRemainingChildren(group, childCaseId);
      }

      writeGroupCompletedLog(parentCaseId, childCaseId, groupId, groupStatus);

      CaseInstance parent = caseInstanceCache.get(parentCaseId);
      if (parent == null) {
        LOG.warnf("SubCaseCompletionListener: parent %s not in cache", parentCaseId);
        return;
      }

      if (groupStatus == GroupStatus.COMPLETED) {
        applyOutputMapping(startedEntry, childCaseId, parent);
        caseResumptionService.resumeIfWaiting(
            parent, groupId, childCaseId.toString(), Map.of(), CaseHubEventType.SUBCASE_COMPLETED)
            .await().atMost(Duration.ofSeconds(10));
      } else {
        LOG.warnf("SubCaseGroup REJECTED: parentCaseId=%s groupId=%s", parentCaseId, groupId);
        // Parent stays WAITING — no automatic fault; caller can monitor groupEvent
      }
    }

    LOG.infof("SubCaseGroup event: parentCaseId=%s groupId=%s status=%s completed=%d/%d",
        parentCaseId, groupId, groupStatus, group.getCompletedCount(), group.getRequiredCount());
  }

  private void handleUngroupedCompletion(UUID childCaseId, CaseLifecycleEvent event,
                                          EventLog startedEntry) {
    UUID parentCaseId = startedEntry.getCaseId();
    String outputMapping = startedEntry.getMetadata().has("outputMapping")
        ? startedEntry.getMetadata().get("outputMapping").asText() : null;

    CaseStatus childStatus = event.caseStatus() != null
        ? CaseStatus.valueOf(event.caseStatus()) : CaseStatus.FAULTED;

    CaseInstance parent = caseInstanceCache.get(parentCaseId);
    if (parent == null) {
      LOG.warnf("SubCaseCompletionListener: parent %s not in cache", parentCaseId);
      return;
    }

    if (outputMapping != null) {
      applyOutputMappingDirect(childCaseId, parent, outputMapping);
    }

    SubCaseCompletionStrategy strategy = new DefaultSubCaseCompletionStrategy();
    LOG.infof("SubCaseCompletionListener (ungrouped): child %s (%s) → parent %s",
        childCaseId, childStatus, parentCaseId);

    writeGroupCompletedLog(parentCaseId, childCaseId, null, null);

    caseResumptionService.resumeIfWaiting(
        parent, childCaseId.toString(), childCaseId.toString(), Map.of(),
        CaseHubEventType.SUBCASE_COMPLETED)
        .await().atMost(Duration.ofSeconds(10));
  }

  private void applyOutputMapping(EventLog startedEntry, UUID childCaseId, CaseInstance parent) {
    String outputMapping = startedEntry.getMetadata().has("outputMapping")
        ? startedEntry.getMetadata().get("outputMapping").asText() : null;
    if (outputMapping != null) applyOutputMappingDirect(childCaseId, parent, outputMapping);
  }

  private void applyOutputMappingDirect(UUID childCaseId, CaseInstance parent, String outputMapping) {
    CaseInstance child = caseInstanceCache.get(childCaseId);
    if (child != null) {
      Map<String, Object> mapped = child.getCaseContext().evalObjectTemplate(outputMapping);
      if (mapped != null) mapped.forEach((k, v) -> parent.getCaseContext().set(k, v));
    } else {
      LOG.warnf("SubCaseCompletionListener: child %s not in cache — outputMapping skipped", childCaseId);
    }
  }

  private void cancelRemainingChildren(SubCaseGroup group, UUID justCompletedChildId) {
    group.getChildCaseIds().stream()
        .filter(id -> !id.equals(justCompletedChildId))
        .forEach(id -> {
          try { caseHubRuntime.cancelCase(id); }
          catch (Exception e) {
            LOG.warnf("SubCaseCompletionListener: could not cancel child %s: %s", id, e.getMessage());
          }
        });
  }

  private void writeGroupCompletedLog(UUID parentCaseId, UUID childCaseId,
                                       String groupId, GroupStatus groupStatus) {
    EventLog log = new EventLog();
    log.setCaseId(parentCaseId);
    log.setWorkerId(childCaseId.toString());
    log.setEventType(CaseHubEventType.SUBCASE_COMPLETED);
    log.setStreamType(EventStreamType.CASE);
    log.setTimestamp(Instant.now());
    ObjectNode meta = OBJECT_MAPPER.createObjectNode();
    meta.put("childCaseId", childCaseId.toString());
    if (groupId != null) meta.put("groupId", groupId);
    if (groupStatus != null) meta.put("groupStatus", groupStatus.name());
    log.setMetadata(meta);
    eventLogRepository.append(log).await().atMost(Duration.ofSeconds(10));
  }

  private static boolean isTerminal(String commandType) {
    return "CompleteCase".equals(commandType)
        || "FaultCase".equals(commandType)
        || "CancelCase".equals(commandType);
  }
}
```

- [ ] **Step 2: Build**

```bash
cd ~/claude/casehub/engine && mvn install -DskipTests -q
```

Expected: `BUILD SUCCESS`

- [ ] **Step 3: Run all blackboard tests**

```bash
cd ~/claude/casehub/engine
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl blackboard 2>&1 | tail -8
```

Expected: all pass.

- [ ] **Step 4: Commit**

```bash
git add blackboard/src/main/java/io/casehub/blackboard/subcase/SubCaseCompletionListener.java
git commit -m "feat(blackboard): SubCaseCompletionListener grouped/ungrouped split (Refs #112)"
```

---

## Task 11: Integration tests

**Files:**
- Create: `blackboard/src/test/java/io/casehub/blackboard/subcase/SubCaseParallelIntegrationTest.java`
- Create: `blackboard/src/test/java/io/casehub/blackboard/subcase/SubCaseMofNIntegrationTest.java`
- Create: `blackboard/src/test/java/io/casehub/blackboard/subcase/SubCasePropagationContextTest.java`

The child case used in all three tests completes immediately when signalled with `done=true`.

- [ ] **Step 1: Write `SubCasePropagationContextTest`**

```java
package io.casehub.blackboard.subcase;

import static org.assertj.core.api.Assertions.assertThat;
import static org.awaitility.Awaitility.await;

import io.casehub.api.engine.CaseHub;
import io.casehub.api.engine.CaseHubRuntime;
import io.casehub.api.model.Binding;
import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.CaseStatus;
import io.casehub.api.model.ContextChangeTrigger;
import io.casehub.api.model.Goal;
import io.casehub.api.model.SubCase;
import io.casehub.engine.internal.model.CaseInstance;
import io.casehub.engine.spi.cache.CaseInstanceCache;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.TimeUnit;
import org.junit.jupiter.api.Test;

@QuarkusTest
class SubCasePropagationContextTest {

  @Inject ParentForPropTest parentCase;
  @Inject CaseInstanceCache caseInstanceCache;
  @Inject CaseHubRuntime caseHubRuntime;

  @Test
  void child_inherits_traceId_and_parentCaseId() {
    UUID parentId = parentCase.startCase(Map.of("trigger", "go"))
        .toCompletableFuture().join();

    // Wait for parent to go WAITING (child spawned)
    await().atMost(10, TimeUnit.SECONDS).until(() -> {
      CaseInstance p = caseInstanceCache.get(parentId);
      return p != null && p.getState() == CaseStatus.WAITING;
    });

    CaseInstance parent = caseInstanceCache.get(parentId);
    String parentTraceId = parent.getPropagationContext().getTraceId();

    // Find child via waitingForWorkId (ungrouped — childCaseId as string)
    UUID childId = UUID.fromString(parent.getWaitingForWorkId());
    CaseInstance child = caseInstanceCache.get(childId);

    assertThat(child).isNotNull();
    assertThat(child.getParentCaseId()).isEqualTo(parentId);
    assertThat(child.getPropagationContext().getTraceId()).isEqualTo(parentTraceId);
  }

  @ApplicationScoped
  public static class PropTestChildBean extends CaseHub {
    @Override
    public CaseDefinition getDefinition() {
      return CaseDefinition.builder()
          .namespace("test").name("prop-child").version("1.0.0")
          .goals(Goal.builder()
              .description("completes when done=true")
              .successWhen(ctx -> Boolean.TRUE.equals(ctx.get("done", Boolean.class).orElse(false)))
              .build())
          .build();
    }
  }

  @ApplicationScoped
  public static class ParentForPropTest extends CaseHub {
    @Override
    public CaseDefinition getDefinition() {
      SubCase child = SubCase.builder()
          .namespace("test").name("prop-child").version("1.0.0")
          .waitForCompletion(true).build();
      return CaseDefinition.builder()
          .namespace("test").name("prop-parent").version("1.0.0")
          .bindings(Binding.builder()
              .name("spawn-prop-child").subCase(child)
              .on(new ContextChangeTrigger(".trigger == \"go\"")).build())
          .build();
    }
  }
}
```

- [ ] **Step 2: Write `SubCaseParallelIntegrationTest`**

```java
package io.casehub.blackboard.subcase;

import static org.assertj.core.api.Assertions.assertThat;
import static org.awaitility.Awaitility.await;

import io.casehub.api.engine.CaseHub;
import io.casehub.api.engine.CaseHubRuntime;
import io.casehub.api.model.Binding;
import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.CaseStatus;
import io.casehub.api.model.ContextChangeTrigger;
import io.casehub.api.model.Goal;
import io.casehub.api.model.OnThresholdReached;
import io.casehub.api.model.SubCase;
import io.casehub.api.model.event.CaseHubEventType;
import io.casehub.engine.internal.model.CaseInstance;
import io.casehub.engine.spi.EventLogRepository;
import io.casehub.engine.spi.cache.CaseInstanceCache;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.TimeUnit;
import org.junit.jupiter.api.Test;

@QuarkusTest
class SubCaseParallelIntegrationTest {

  @Inject ParentAllOfBean parentCase;
  @Inject CaseInstanceCache caseInstanceCache;
  @Inject CaseHubRuntime caseHubRuntime;
  @Inject EventLogRepository eventLogRepository;

  @Test
  void allOf3_allComplete_parentResumes() throws Exception {
    UUID parentId = parentCase.startCase(Map.of("trigger", "go"))
        .toCompletableFuture().join();

    // Wait for parent to go WAITING (all 3 children spawned)
    await().atMost(15, TimeUnit.SECONDS).until(() -> {
      CaseInstance p = caseInstanceCache.get(parentId);
      return p != null && p.getState() == CaseStatus.WAITING;
    });

    // Find child IDs from EventLog
    List<UUID> childIds = eventLogRepository
        .findByWorkerAndType(null, CaseHubEventType.SUBCASE_STARTED)
        .await().indefinitely()
        .stream()
        .filter(e -> e.getCaseId().equals(parentId))
        .map(e -> UUID.fromString(e.getMetadata().get("childCaseId").asText()))
        .toList();

    assertThat(childIds).hasSize(3);

    // Complete all 3 children by signalling done=true
    childIds.forEach(childId -> caseHubRuntime.signal(childId, "done", true));

    // Parent should resume to RUNNING (or COMPLETED if goals met)
    await().atMost(15, TimeUnit.SECONDS).until(() -> {
      CaseInstance p = caseInstanceCache.get(parentId);
      return p != null && (p.getState() == CaseStatus.RUNNING || p.getState() == CaseStatus.COMPLETED);
    });

    assertThat(caseInstanceCache.get(parentId).getState())
        .isNotEqualTo(CaseStatus.WAITING);
  }

  @ApplicationScoped
  public static class AllOfChildBean extends CaseHub {
    @Override
    public CaseDefinition getDefinition() {
      return CaseDefinition.builder()
          .namespace("test").name("allof-child").version("1.0.0")
          .goals(Goal.builder()
              .description("completes when done=true")
              .successWhen(ctx -> Boolean.TRUE.equals(ctx.get("done", Boolean.class).orElse(false)))
              .build())
          .build();
    }
  }

  @ApplicationScoped
  public static class ParentAllOfBean extends CaseHub {
    @Override
    public CaseDefinition getDefinition() {
      SubCase child1 = SubCase.builder()
          .namespace("test").name("allof-child").version("1.0.0")
          .groupId("sites").totalInGroup(3).requiredCount(3)
          .onThresholdReached(OnThresholdReached.KEEP).build();
      SubCase child2 = SubCase.builder()
          .namespace("test").name("allof-child").version("1.0.0")
          .groupId("sites").totalInGroup(3).requiredCount(3)
          .onThresholdReached(OnThresholdReached.KEEP).build();
      SubCase child3 = SubCase.builder()
          .namespace("test").name("allof-child").version("1.0.0")
          .groupId("sites").totalInGroup(3).requiredCount(3)
          .onThresholdReached(OnThresholdReached.KEEP).build();
      return CaseDefinition.builder()
          .namespace("test").name("allof-parent").version("1.0.0")
          .bindings(
              Binding.builder().name("spawn-site-a").subCase(child1)
                  .on(new ContextChangeTrigger(".trigger == \"go\"")).build(),
              Binding.builder().name("spawn-site-b").subCase(child2)
                  .on(new ContextChangeTrigger(".trigger == \"go\"")).build(),
              Binding.builder().name("spawn-site-c").subCase(child3)
                  .on(new ContextChangeTrigger(".trigger == \"go\"")).build())
          .build();
    }
  }
}
```

- [ ] **Step 3: Write `SubCaseMofNIntegrationTest`**

```java
package io.casehub.blackboard.subcase;

import static org.assertj.core.api.Assertions.assertThat;
import static org.awaitility.Awaitility.await;

import io.casehub.api.engine.CaseHub;
import io.casehub.api.engine.CaseHubRuntime;
import io.casehub.api.model.Binding;
import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.CaseStatus;
import io.casehub.api.model.ContextChangeTrigger;
import io.casehub.api.model.Goal;
import io.casehub.api.model.OnThresholdReached;
import io.casehub.api.model.SubCase;
import io.casehub.api.model.event.CaseHubEventType;
import io.casehub.engine.internal.model.CaseInstance;
import io.casehub.engine.spi.EventLogRepository;
import io.casehub.engine.spi.cache.CaseInstanceCache;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.TimeUnit;
import org.junit.jupiter.api.Test;

@QuarkusTest
class SubCaseMofNIntegrationTest {

  @Inject ParentMofNBean parentCase;
  @Inject CaseInstanceCache caseInstanceCache;
  @Inject CaseHubRuntime caseHubRuntime;
  @Inject EventLogRepository eventLogRepository;

  @Test
  void twoOfThree_firstTwoComplete_parentResumes_thirdKeepRunning() throws Exception {
    UUID parentId = parentCase.startCase(Map.of("trigger", "go"))
        .toCompletableFuture().join();

    await().atMost(15, TimeUnit.SECONDS).until(() -> {
      CaseInstance p = caseInstanceCache.get(parentId);
      return p != null && p.getState() == CaseStatus.WAITING;
    });

    List<UUID> childIds = eventLogRepository
        .findByWorkerAndType(null, CaseHubEventType.SUBCASE_STARTED)
        .await().indefinitely()
        .stream()
        .filter(e -> e.getCaseId().equals(parentId))
        .map(e -> UUID.fromString(e.getMetadata().get("childCaseId").asText()))
        .toList();

    assertThat(childIds).hasSize(3);

    // Complete only the first 2
    caseHubRuntime.signal(childIds.get(0), "done", true);
    caseHubRuntime.signal(childIds.get(1), "done", true);

    // Parent resumes after 2 complete (requiredCount=2)
    await().atMost(15, TimeUnit.SECONDS).until(() -> {
      CaseInstance p = caseInstanceCache.get(parentId);
      return p != null && p.getState() != CaseStatus.WAITING;
    });

    assertThat(caseInstanceCache.get(parentId).getState()).isNotEqualTo(CaseStatus.WAITING);

    // Third child is still RUNNING (KEEP policy)
    CaseInstance thirdChild = caseInstanceCache.get(childIds.get(2));
    assertThat(thirdChild).isNotNull();
    assertThat(thirdChild.getState()).isEqualTo(CaseStatus.RUNNING);
  }

  @ApplicationScoped
  public static class MofNChildBean extends CaseHub {
    @Override
    public CaseDefinition getDefinition() {
      return CaseDefinition.builder()
          .namespace("test").name("mofn-child").version("1.0.0")
          .goals(Goal.builder()
              .description("completes when done=true")
              .successWhen(ctx -> Boolean.TRUE.equals(ctx.get("done", Boolean.class).orElse(false)))
              .build())
          .build();
    }
  }

  @ApplicationScoped
  public static class ParentMofNBean extends CaseHub {
    @Override
    public CaseDefinition getDefinition() {
      SubCase child = SubCase.builder()
          .namespace("test").name("mofn-child").version("1.0.0")
          .groupId("mofn-sites").totalInGroup(3).requiredCount(2)
          .onThresholdReached(OnThresholdReached.KEEP).build();
      return CaseDefinition.builder()
          .namespace("test").name("mofn-parent").version("1.0.0")
          .bindings(
              Binding.builder().name("spawn-a").subCase(child)
                  .on(new ContextChangeTrigger(".trigger == \"go\"")).build(),
              Binding.builder().name("spawn-b").subCase(child)
                  .on(new ContextChangeTrigger(".trigger == \"go\"")).build(),
              Binding.builder().name("spawn-c").subCase(child)
                  .on(new ContextChangeTrigger(".trigger == \"go\"")).build())
          .build();
    }
  }
}
```

- [ ] **Step 4: Run all blackboard tests**

```bash
cd ~/claude/casehub/engine
mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl blackboard 2>&1 | tail -15
```

Expected: all tests pass including the 3 new integration tests.

- [ ] **Step 5: Commit**

```bash
git add blackboard/src/test/java/io/casehub/blackboard/subcase/SubCasePropagationContextTest.java \
        blackboard/src/test/java/io/casehub/blackboard/subcase/SubCaseParallelIntegrationTest.java \
        blackboard/src/test/java/io/casehub/blackboard/subcase/SubCaseMofNIntegrationTest.java
git commit -m "test(blackboard): parallel sub-case, M-of-N, and PropagationContext integration tests (Refs #112)"
```

---

## Task 12: Full suite verification

- [ ] **Step 1: Install all modules then run blackboard tests**

```bash
cd ~/claude/casehub/engine
mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl blackboard 2>&1 | tail -10
```

Expected: all tests pass, no failures.

- [ ] **Step 2: Close issue and final commit**

```bash
git commit --allow-empty -m "feat: sub-case M-of-N coordination complete (Closes #112)"
```
