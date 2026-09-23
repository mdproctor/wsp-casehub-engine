# Conductor State Persistence Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1140 — feat: Conductor state persistence — repository SPIs and in-memory implementations
**Issue group:** #1140

**Goal:** Extract 4 repository SPIs from 3 domain beans, create @DefaultBean in-memory implementations, and refactor domain beans to inject SPIs with tenant-scoped methods.

**Architecture:** Four persistence SPIs (ConductorInboxRepository, WatchPatternStore, DenyPatternStore, ImprovementBlockStore) in common-core/spi. In-memory implementations in runtime-core with @DefaultBean @ApplicationScoped. Domain beans become stateless with respect to persistence — they inject SPIs and keep only business logic (and in ImprovementBudgetEnforcer's case, ephemeral runtime tracking state).

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI (@DefaultBean, @ApplicationScoped), JUnit 5, AssertJ

## Global Constraints

- All new SPI interfaces in `common-core/spi` package `io.casehub.engine.common.spi`
- All in-memory impls in `runtime-core` package `io.casehub.engine.internal.improvement`
- `tenancyId` as last parameter on every SPI method — accepted but unused by in-memory impls
- `@DefaultBean @ApplicationScoped` on all in-memory impls — yields when non-default impl registered
- Test classes must be `*Test.java`, never `*IT.java` (surefire, not failsafe)
- Copyright header: `Copyright 2026-Present The Case Hub Authors` (Apache-2.0)

---

## Batch 1: Model + SPI Foundation

### Task 1: Add caseId field to ConductorInboxEntry

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/stigmergy/ConductorInboxEntry.java`
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/ConductorInboxManager.java:77`
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/ResearchPipelineOrchestrator.java:148`
- Modify: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/ConductorInboxManagerTest.java:184`
- Modify: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/EvolutionApiTest.java:70,94`

**Interfaces:**
- Produces: `ConductorInboxEntry` record with `UUID caseId` as first field. All subsequent tasks depend on this field existing.

- [ ] **Step 1: Update the ConductorInboxEntry record**

Add `UUID caseId` as the first parameter of the record constructor. Use `ide_replace_member` to replace the record definition:

```java
public record ConductorInboxEntry(
    UUID caseId,
    String id,
    ImprovementStage stage,
    Status status,
    @Nullable String category,
    @Nullable String areaId,
    @Nullable UUID improvementCaseId,
    @Nullable String summary,
    List<EscalationTrigger> escalationTriggers,
    double confidence,
    Instant queuedAt,
    @Nullable Instant resolvedAt,
    @Nullable Integer timeoutMinutes,
    @Nullable ConductorDecision decision) {

  public enum Status {
    PENDING,
    APPROVED,
    REJECTED,
    REDIRECTED,
    TIMED_OUT,
    AUTO_APPROVED
  }
}
```

- [ ] **Step 2: Update ConductorInboxManager.resolve()**

In `ConductorInboxManager.java`, update the `resolve` method to pass `existing.caseId()` when constructing the resolved entry:

```java
var resolved =
    new ConductorInboxEntry(
        existing.caseId(),
        existing.id(),
        existing.stage(),
        decision.outcome(),
        existing.category(),
        existing.areaId(),
        existing.improvementCaseId(),
        existing.summary(),
        existing.escalationTriggers(),
        existing.confidence(),
        existing.queuedAt(),
        Instant.now(),
        existing.timeoutMinutes(),
        decision);
```

- [ ] **Step 3: Update ResearchPipelineOrchestrator.enqueueForApproval()**

In `ResearchPipelineOrchestrator.java`, update the `enqueueForApproval` method to pass `caseId` (already a parameter) as the first arg:

```java
private String enqueueForApproval(
    UUID caseId, ImprovementStage stage, GateCheckpoint checkpoint) {
  var entryId = UUID.randomUUID().toString();
  var entry =
      new ConductorInboxEntry(
          caseId,
          entryId,
          stage,
          ConductorInboxEntry.Status.PENDING,
          null,
          null,
          null,
          "Gate checkpoint at " + stage,
          List.of(),
          1.0,
          Instant.now(),
          null,
          null,
          null);
  inboxManager.enqueue(caseId, entry);
  return entryId;
}
```

- [ ] **Step 4: Update ConductorInboxManagerTest.makeEntry()**

Update the `makeEntry` helper in `ConductorInboxManagerTest.java` to accept and pass `caseId`. Since the test's `caseId` field is available, pass it through. Change the helper signature and update all call sites within the test:

```java
private ConductorInboxEntry makeEntry(UUID caseId, String id, ImprovementStage stage) {
  return new ConductorInboxEntry(
      caseId,
      id,
      stage,
      Status.PENDING,
      null,
      null,
      null,
      "test entry",
      List.of(),
      0.8,
      Instant.now(),
      null,
      null,
      null);
}
```

Update every call to `makeEntry` in the test class to pass `caseId`:
- `makeEntry("e1", ...)` → `makeEntry(caseId, "e1", ...)`
- For `perCaseIsolation`, the second case uses `case2`: `makeEntry(case2, "e2", ...)`

- [ ] **Step 5: Update EvolutionApiTest construction sites**

Update both `ConductorInboxEntry` constructions in `EvolutionApiTest.java` to include `caseId` as the first argument:

At line ~70 (`getInboxReturnsPendingEntries`):
```java
var entry =
    new ConductorInboxEntry(
        caseId,
        "e1",
        ImprovementStage.RESEARCH_SCOPE,
        Status.PENDING,
        null,
        null,
        null,
        "test",
        List.of(),
        0.8,
        Instant.now(),
        null,
        null,
        null);
```

At line ~94 (`resolveGateUpdatesEntryStatus`):
```java
var entry =
    new ConductorInboxEntry(
        caseId,
        "e1",
        ImprovementStage.RESEARCH_SCOPE,
        Status.PENDING,
        null,
        null,
        null,
        "test",
        List.of(),
        0.8,
        Instant.now(),
        null,
        null,
        null);
```

- [ ] **Step 6: Compile and run tests**

Run: `mvn install -DskipTests -q`
Expected: compilation succeeds (no construction site missed)

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime-core -Dtest="ConductorInboxManagerTest,EvolutionApiTest"`
Expected: all tests PASS

- [ ] **Step 7: Commit**

```bash
git add api/src/main/java/io/casehub/api/model/stigmergy/ConductorInboxEntry.java
git add runtime-core/src/main/java/io/casehub/engine/internal/improvement/ConductorInboxManager.java
git add runtime-core/src/main/java/io/casehub/engine/internal/improvement/ResearchPipelineOrchestrator.java
git add runtime-core/src/test/java/io/casehub/engine/internal/improvement/ConductorInboxManagerTest.java
git add runtime-core/src/test/java/io/casehub/engine/internal/improvement/EvolutionApiTest.java
git commit -m "feat(#1140): add caseId field to ConductorInboxEntry record

Refs #1140"
```

---

### Task 2: Create 4 SPI interfaces

**Files:**
- Create: `common-core/src/main/java/io/casehub/engine/common/spi/ConductorInboxRepository.java`
- Create: `common-core/src/main/java/io/casehub/engine/common/spi/WatchPatternStore.java`
- Create: `common-core/src/main/java/io/casehub/engine/common/spi/DenyPatternStore.java`
- Create: `common-core/src/main/java/io/casehub/engine/common/spi/ImprovementBlockStore.java`

**Interfaces:**
- Produces: Four SPI interfaces consumed by Tasks 3–6. Method signatures exactly as shown below.

- [ ] **Step 1: Create ConductorInboxRepository**

Create `common-core/src/main/java/io/casehub/engine/common/spi/ConductorInboxRepository.java`:

```java
/*
 * Copyright 2026-Present The Case Hub Authors
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package io.casehub.engine.common.spi;

import io.casehub.api.model.stigmergy.ConductorInboxEntry;
import jakarta.annotation.Nullable;
import java.util.List;
import java.util.UUID;

public interface ConductorInboxRepository {

  void save(ConductorInboxEntry entry, String tenancyId);

  @Nullable
  ConductorInboxEntry findById(UUID caseId, String entryId, String tenancyId);

  List<ConductorInboxEntry> findPending(UUID caseId, String tenancyId);

  int countPending(UUID caseId, String tenancyId);

  List<ConductorInboxEntry> findAll(UUID caseId, String tenancyId);
}
```

- [ ] **Step 2: Create WatchPatternStore**

Create `common-core/src/main/java/io/casehub/engine/common/spi/WatchPatternStore.java`:

```java
/*
 * Copyright 2026-Present The Case Hub Authors
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package io.casehub.engine.common.spi;

import io.casehub.api.model.stigmergy.WatchPattern;
import java.util.List;
import java.util.UUID;

public interface WatchPatternStore {

  void save(UUID caseId, WatchPattern pattern, String tenancyId);

  void remove(UUID caseId, String patternId, String tenancyId);

  List<WatchPattern> findActive(UUID caseId, String tenancyId);
}
```

- [ ] **Step 3: Create DenyPatternStore**

Create `common-core/src/main/java/io/casehub/engine/common/spi/DenyPatternStore.java`:

```java
/*
 * Copyright 2026-Present The Case Hub Authors
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package io.casehub.engine.common.spi;

import java.util.Set;
import java.util.UUID;

public interface DenyPatternStore {

  void save(UUID caseId, String pattern, String tenancyId);

  void remove(UUID caseId, String pattern, String tenancyId);

  Set<String> findAll(UUID caseId, String tenancyId);
}
```

- [ ] **Step 4: Create ImprovementBlockStore**

Create `common-core/src/main/java/io/casehub/engine/common/spi/ImprovementBlockStore.java`:

```java
/*
 * Copyright 2026-Present The Case Hub Authors
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package io.casehub.engine.common.spi;

import jakarta.annotation.Nullable;
import java.util.UUID;

public interface ImprovementBlockStore {

  void save(UUID caseId, UUID improvementCaseId, UUID blockerImprovementId, String tenancyId);

  void remove(UUID caseId, UUID improvementCaseId, String tenancyId);

  boolean isBlocked(UUID caseId, UUID improvementCaseId, String tenancyId);

  @Nullable
  UUID blockedBy(UUID caseId, UUID improvementCaseId, String tenancyId);
}
```

- [ ] **Step 5: Compile**

Run: `mvn install -DskipTests -q`
Expected: compilation succeeds

- [ ] **Step 6: Commit**

```bash
git add common-core/src/main/java/io/casehub/engine/common/spi/ConductorInboxRepository.java
git add common-core/src/main/java/io/casehub/engine/common/spi/WatchPatternStore.java
git add common-core/src/main/java/io/casehub/engine/common/spi/DenyPatternStore.java
git add common-core/src/main/java/io/casehub/engine/common/spi/ImprovementBlockStore.java
git commit -m "feat(#1140): add conductor state persistence SPI interfaces

ConductorInboxRepository, WatchPatternStore, DenyPatternStore,
ImprovementBlockStore — all tenant-scoped in common-core/spi.

Refs #1140"
```

---

## Batch 2: In-memory Implementations + Contract Tests

### Task 3: In-memory implementations with TDD contract tests

**Files:**
- Create: `common-core/src/test/java/io/casehub/engine/common/spi/ConductorInboxRepositoryContractTest.java`
- Create: `common-core/src/test/java/io/casehub/engine/common/spi/WatchPatternStoreContractTest.java`
- Create: `common-core/src/test/java/io/casehub/engine/common/spi/DenyPatternStoreContractTest.java`
- Create: `common-core/src/test/java/io/casehub/engine/common/spi/ImprovementBlockStoreContractTest.java`
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/InMemoryConductorInboxRepository.java`
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/InMemoryWatchPatternStore.java`
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/InMemoryDenyPatternStore.java`
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/InMemoryImprovementBlockStore.java`
- Create: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/InMemoryConductorInboxRepositoryContractTest.java`
- Create: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/InMemoryWatchPatternStoreContractTest.java`
- Create: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/InMemoryDenyPatternStoreContractTest.java`
- Create: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/InMemoryImprovementBlockStoreContractTest.java`

**Interfaces:**
- Consumes: The 4 SPI interfaces from Task 2
- Produces: 4 `@DefaultBean @ApplicationScoped` implementations consumed by Tasks 4–6

TDD cycle: write abstract contract test → write concrete test subclass → run (fails — no impl) → write impl → run (passes). Repeat for each SPI.

- [ ] **Step 1: Write ImprovementBlockStore contract test**

Create `common-core/src/test/java/io/casehub/engine/common/spi/ImprovementBlockStoreContractTest.java`:

```java
/*
 * Copyright 2026-Present The Case Hub Authors
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package io.casehub.engine.common.spi;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.UUID;
import org.junit.jupiter.api.Test;

public abstract class ImprovementBlockStoreContractTest {

  protected abstract ImprovementBlockStore store();

  protected abstract String tenancyId();

  @Test
  void saveAndIsBlocked() {
    var caseId = UUID.randomUUID();
    var improvementId = UUID.randomUUID();
    var blockerId = UUID.randomUUID();

    store().save(caseId, improvementId, blockerId, tenancyId());

    assertThat(store().isBlocked(caseId, improvementId, tenancyId())).isTrue();
  }

  @Test
  void blockedByReturnsBlocker() {
    var caseId = UUID.randomUUID();
    var improvementId = UUID.randomUUID();
    var blockerId = UUID.randomUUID();

    store().save(caseId, improvementId, blockerId, tenancyId());

    assertThat(store().blockedBy(caseId, improvementId, tenancyId())).isEqualTo(blockerId);
  }

  @Test
  void removeUnblocks() {
    var caseId = UUID.randomUUID();
    var improvementId = UUID.randomUUID();
    var blockerId = UUID.randomUUID();

    store().save(caseId, improvementId, blockerId, tenancyId());
    store().remove(caseId, improvementId, tenancyId());

    assertThat(store().isBlocked(caseId, improvementId, tenancyId())).isFalse();
    assertThat(store().blockedBy(caseId, improvementId, tenancyId())).isNull();
  }

  @Test
  void unblockedByDefault() {
    var caseId = UUID.randomUUID();
    assertThat(store().isBlocked(caseId, UUID.randomUUID(), tenancyId())).isFalse();
    assertThat(store().blockedBy(caseId, UUID.randomUUID(), tenancyId())).isNull();
  }

  @Test
  void perCaseIsolation() {
    var case1 = UUID.randomUUID();
    var case2 = UUID.randomUUID();
    var improvementId = UUID.randomUUID();
    var blockerId = UUID.randomUUID();

    store().save(case1, improvementId, blockerId, tenancyId());

    assertThat(store().isBlocked(case1, improvementId, tenancyId())).isTrue();
    assertThat(store().isBlocked(case2, improvementId, tenancyId())).isFalse();
  }
}
```

- [ ] **Step 2: Write concrete test and InMemoryImprovementBlockStore**

Create `runtime-core/src/test/java/io/casehub/engine/internal/improvement/InMemoryImprovementBlockStoreContractTest.java`:

```java
/*
 * Copyright 2026-Present The Case Hub Authors
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package io.casehub.engine.internal.improvement;

import io.casehub.engine.common.spi.ImprovementBlockStore;
import io.casehub.engine.common.spi.ImprovementBlockStoreContractTest;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import java.util.UUID;

class InMemoryImprovementBlockStoreContractTest extends ImprovementBlockStoreContractTest {

  private InMemoryImprovementBlockStore store;

  @BeforeEach
  void setUp() {
    store = new InMemoryImprovementBlockStore();
  }

  @Override
  protected ImprovementBlockStore store() {
    return store;
  }

  @Override
  protected String tenancyId() {
    return "test-tenant";
  }

  @Test
  void resetClearsAllBlocks() {
    var caseId = UUID.randomUUID();
    store.save(caseId, UUID.randomUUID(), UUID.randomUUID(), tenancyId());
    store.reset();
    assertThat(store.isBlocked(caseId, UUID.randomUUID(), tenancyId())).isFalse();
  }
}
```

Create `runtime-core/src/main/java/io/casehub/engine/internal/improvement/InMemoryImprovementBlockStore.java`:

```java
/*
 * Copyright 2026-Present The Case Hub Authors
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package io.casehub.engine.internal.improvement;

import io.casehub.engine.common.spi.ImprovementBlockStore;
import io.casehub.engine.common.spi.Resettable;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;

@DefaultBean
@ApplicationScoped
public class InMemoryImprovementBlockStore implements ImprovementBlockStore, Resettable {

  private final ConcurrentHashMap<UUID, ConcurrentHashMap<UUID, UUID>> blocks =
      new ConcurrentHashMap<>();

  @Override
  public void save(UUID caseId, UUID improvementCaseId, UUID blockerImprovementId,
      String tenancyId) {
    blocks.computeIfAbsent(caseId, k -> new ConcurrentHashMap<>())
        .put(improvementCaseId, blockerImprovementId);
  }

  @Override
  public void remove(UUID caseId, UUID improvementCaseId, String tenancyId) {
    var caseBlocks = blocks.get(caseId);
    if (caseBlocks != null) {
      caseBlocks.remove(improvementCaseId);
    }
  }

  @Override
  public boolean isBlocked(UUID caseId, UUID improvementCaseId, String tenancyId) {
    var caseBlocks = blocks.get(caseId);
    return caseBlocks != null && caseBlocks.containsKey(improvementCaseId);
  }

  @Override
  public UUID blockedBy(UUID caseId, UUID improvementCaseId, String tenancyId) {
    var caseBlocks = blocks.get(caseId);
    return caseBlocks != null ? caseBlocks.get(improvementCaseId) : null;
  }

  @Override
  public void reset() {
    blocks.clear();
  }
}
```

- [ ] **Step 3: Run ImprovementBlockStore tests**

Run: `mvn install -DskipTests -q && mvn test -pl runtime-core -Dtest=InMemoryImprovementBlockStoreContractTest`
Expected: all 6 tests PASS

- [ ] **Step 4: Write DenyPatternStore contract test**

Create `common-core/src/test/java/io/casehub/engine/common/spi/DenyPatternStoreContractTest.java`:

```java
/*
 * Copyright 2026-Present The Case Hub Authors
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package io.casehub.engine.common.spi;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.UUID;
import org.junit.jupiter.api.Test;

public abstract class DenyPatternStoreContractTest {

  protected abstract DenyPatternStore store();

  protected abstract String tenancyId();

  @Test
  void saveAndFindAll() {
    var caseId = UUID.randomUUID();
    store().save(caseId, "ImprovementBudget", tenancyId());
    store().save(caseId, "EvolutionTicker", tenancyId());

    var patterns = store().findAll(caseId, tenancyId());
    assertThat(patterns).containsExactlyInAnyOrder("ImprovementBudget", "EvolutionTicker");
  }

  @Test
  void removePattern() {
    var caseId = UUID.randomUUID();
    store().save(caseId, "pattern-a", tenancyId());
    store().save(caseId, "pattern-b", tenancyId());

    store().remove(caseId, "pattern-a", tenancyId());

    assertThat(store().findAll(caseId, tenancyId())).containsExactly("pattern-b");
  }

  @Test
  void findAllEmptyForUnknownCase() {
    assertThat(store().findAll(UUID.randomUUID(), tenancyId())).isEmpty();
  }

  @Test
  void perCaseIsolation() {
    var case1 = UUID.randomUUID();
    var case2 = UUID.randomUUID();
    store().save(case1, "pattern-a", tenancyId());

    assertThat(store().findAll(case1, tenancyId())).containsExactly("pattern-a");
    assertThat(store().findAll(case2, tenancyId())).isEmpty();
  }

  @Test
  void saveDuplicateIsIdempotent() {
    var caseId = UUID.randomUUID();
    store().save(caseId, "pattern-a", tenancyId());
    store().save(caseId, "pattern-a", tenancyId());

    assertThat(store().findAll(caseId, tenancyId())).hasSize(1);
  }
}
```

- [ ] **Step 5: Write concrete test and InMemoryDenyPatternStore**

Create `runtime-core/src/test/java/io/casehub/engine/internal/improvement/InMemoryDenyPatternStoreContractTest.java`:

```java
/*
 * Copyright 2026-Present The Case Hub Authors
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package io.casehub.engine.internal.improvement;

import io.casehub.engine.common.spi.DenyPatternStore;
import io.casehub.engine.common.spi.DenyPatternStoreContractTest;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import java.util.UUID;

class InMemoryDenyPatternStoreContractTest extends DenyPatternStoreContractTest {

  private InMemoryDenyPatternStore store;

  @BeforeEach
  void setUp() {
    store = new InMemoryDenyPatternStore();
  }

  @Override
  protected DenyPatternStore store() {
    return store;
  }

  @Override
  protected String tenancyId() {
    return "test-tenant";
  }

  @Test
  void resetClearsAllPatterns() {
    var caseId = UUID.randomUUID();
    store.save(caseId, "pattern-a", tenancyId());
    store.reset();
    assertThat(store.findAll(caseId, tenancyId())).isEmpty();
  }
}
```

Create `runtime-core/src/main/java/io/casehub/engine/internal/improvement/InMemoryDenyPatternStore.java`:

```java
/*
 * Copyright 2026-Present The Case Hub Authors
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package io.casehub.engine.internal.improvement;

import io.casehub.engine.common.spi.DenyPatternStore;
import io.casehub.engine.common.spi.Resettable;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.Set;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;

@DefaultBean
@ApplicationScoped
public class InMemoryDenyPatternStore implements DenyPatternStore, Resettable {

  private final ConcurrentHashMap<UUID, Set<String>> patterns = new ConcurrentHashMap<>();

  @Override
  public void save(UUID caseId, String pattern, String tenancyId) {
    patterns.computeIfAbsent(caseId, k -> ConcurrentHashMap.newKeySet()).add(pattern);
  }

  @Override
  public void remove(UUID caseId, String pattern, String tenancyId) {
    var casePatterns = patterns.get(caseId);
    if (casePatterns != null) {
      casePatterns.remove(pattern);
    }
  }

  @Override
  public Set<String> findAll(UUID caseId, String tenancyId) {
    return Set.copyOf(patterns.getOrDefault(caseId, Set.of()));
  }

  @Override
  public void reset() {
    patterns.clear();
  }
}
```

- [ ] **Step 6: Run DenyPatternStore tests**

Run: `mvn install -DskipTests -q && mvn test -pl runtime-core -Dtest=InMemoryDenyPatternStoreContractTest`
Expected: all 6 tests PASS

- [ ] **Step 7: Write WatchPatternStore contract test**

Create `common-core/src/test/java/io/casehub/engine/common/spi/WatchPatternStoreContractTest.java`:

```java
/*
 * Copyright 2026-Present The Case Hub Authors
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package io.casehub.engine.common.spi;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.api.model.stigmergy.WatchPattern;
import java.time.Instant;
import java.util.UUID;
import org.junit.jupiter.api.Test;

public abstract class WatchPatternStoreContractTest {

  protected abstract WatchPatternStore store();

  protected abstract String tenancyId();

  @Test
  void saveAndFindActive() {
    var caseId = UUID.randomUUID();
    var pattern = new WatchPattern("w1", "security", null, null, null, Instant.now());
    store().save(caseId, pattern, tenancyId());

    var active = store().findActive(caseId, tenancyId());
    assertThat(active).hasSize(1);
    assertThat(active.get(0).id()).isEqualTo("w1");
  }

  @Test
  void removeByPatternId() {
    var caseId = UUID.randomUUID();
    store().save(caseId, new WatchPattern("w1", "security", null, null, null, Instant.now()),
        tenancyId());
    store().save(caseId, new WatchPattern("w2", "architecture", null, null, null, Instant.now()),
        tenancyId());

    store().remove(caseId, "w1", tenancyId());

    var active = store().findActive(caseId, tenancyId());
    assertThat(active).hasSize(1);
    assertThat(active.get(0).id()).isEqualTo("w2");
  }

  @Test
  void findActiveEmptyForUnknownCase() {
    assertThat(store().findActive(UUID.randomUUID(), tenancyId())).isEmpty();
  }

  @Test
  void perCaseIsolation() {
    var case1 = UUID.randomUUID();
    var case2 = UUID.randomUUID();
    store().save(case1, new WatchPattern("w1", "security", null, null, null, Instant.now()),
        tenancyId());

    assertThat(store().findActive(case1, tenancyId())).hasSize(1);
    assertThat(store().findActive(case2, tenancyId())).isEmpty();
  }
}
```

- [ ] **Step 8: Write concrete test and InMemoryWatchPatternStore**

Create `runtime-core/src/test/java/io/casehub/engine/internal/improvement/InMemoryWatchPatternStoreContractTest.java`:

```java
/*
 * Copyright 2026-Present The Case Hub Authors
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package io.casehub.engine.internal.improvement;

import io.casehub.api.model.stigmergy.WatchPattern;
import io.casehub.engine.common.spi.WatchPatternStore;
import io.casehub.engine.common.spi.WatchPatternStoreContractTest;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import java.time.Instant;
import java.util.UUID;

class InMemoryWatchPatternStoreContractTest extends WatchPatternStoreContractTest {

  private InMemoryWatchPatternStore store;

  @BeforeEach
  void setUp() {
    store = new InMemoryWatchPatternStore();
  }

  @Override
  protected WatchPatternStore store() {
    return store;
  }

  @Override
  protected String tenancyId() {
    return "test-tenant";
  }

  @Test
  void resetClearsAllPatterns() {
    var caseId = UUID.randomUUID();
    store.save(caseId, new WatchPattern("w1", "security", null, null, null, Instant.now()),
        tenancyId());
    store.reset();
    assertThat(store.findActive(caseId, tenancyId())).isEmpty();
  }
}
```

Create `runtime-core/src/main/java/io/casehub/engine/internal/improvement/InMemoryWatchPatternStore.java`:

```java
/*
 * Copyright 2026-Present The Case Hub Authors
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package io.casehub.engine.internal.improvement;

import io.casehub.api.model.stigmergy.WatchPattern;
import io.casehub.engine.common.spi.Resettable;
import io.casehub.engine.common.spi.WatchPatternStore;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.List;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.CopyOnWriteArrayList;

@DefaultBean
@ApplicationScoped
public class InMemoryWatchPatternStore implements WatchPatternStore, Resettable {

  private final ConcurrentHashMap<UUID, CopyOnWriteArrayList<WatchPattern>> patterns =
      new ConcurrentHashMap<>();

  @Override
  public void save(UUID caseId, WatchPattern pattern, String tenancyId) {
    patterns.computeIfAbsent(caseId, k -> new CopyOnWriteArrayList<>()).add(pattern);
  }

  @Override
  public void remove(UUID caseId, String patternId, String tenancyId) {
    var casePatterns = patterns.get(caseId);
    if (casePatterns != null) {
      casePatterns.removeIf(p -> p.id().equals(patternId));
    }
  }

  @Override
  public List<WatchPattern> findActive(UUID caseId, String tenancyId) {
    var casePatterns = patterns.get(caseId);
    return casePatterns != null ? List.copyOf(casePatterns) : List.of();
  }

  @Override
  public void reset() {
    patterns.clear();
  }
}
```

- [ ] **Step 9: Run WatchPatternStore tests**

Run: `mvn install -DskipTests -q && mvn test -pl runtime-core -Dtest=InMemoryWatchPatternStoreContractTest`
Expected: all 5 tests PASS

- [ ] **Step 10: Write ConductorInboxRepository contract test**

Create `common-core/src/test/java/io/casehub/engine/common/spi/ConductorInboxRepositoryContractTest.java`:

```java
/*
 * Copyright 2026-Present The Case Hub Authors
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package io.casehub.engine.common.spi;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.api.model.stigmergy.ConductorDecision;
import io.casehub.api.model.stigmergy.ConductorInboxEntry;
import io.casehub.api.model.stigmergy.ConductorInboxEntry.Status;
import io.casehub.api.model.stigmergy.ImprovementStage;
import java.time.Instant;
import java.util.List;
import java.util.UUID;
import org.junit.jupiter.api.Test;

public abstract class ConductorInboxRepositoryContractTest {

  protected abstract ConductorInboxRepository repository();

  protected abstract String tenancyId();

  protected ConductorInboxEntry makeEntry(UUID caseId, String id, ImprovementStage stage) {
    return new ConductorInboxEntry(
        caseId, id, stage, Status.PENDING, null, null, null, "test entry",
        List.of(), 0.8, Instant.now(), null, null, null);
  }

  @Test
  void saveAndFindById() {
    var caseId = UUID.randomUUID();
    var entry = makeEntry(caseId, "e1", ImprovementStage.RESEARCH_SCOPE);
    repository().save(entry, tenancyId());

    var found = repository().findById(caseId, "e1", tenancyId());
    assertThat(found).isNotNull();
    assertThat(found.id()).isEqualTo("e1");
    assertThat(found.caseId()).isEqualTo(caseId);
  }

  @Test
  void findByIdReturnsNullForUnknown() {
    assertThat(repository().findById(UUID.randomUUID(), "nope", tenancyId())).isNull();
  }

  @Test
  void findPendingFiltersStatus() {
    var caseId = UUID.randomUUID();
    var pending = makeEntry(caseId, "e1", ImprovementStage.RESEARCH_SCOPE);
    var resolved = new ConductorInboxEntry(
        caseId, "e2", ImprovementStage.HYPOTHESIS_APPROVAL, Status.APPROVED,
        null, null, null, "resolved", List.of(), 0.9, Instant.now(), Instant.now(), null,
        new ConductorDecision(Status.APPROVED, null, "ok", null));

    repository().save(pending, tenancyId());
    repository().save(resolved, tenancyId());

    var result = repository().findPending(caseId, tenancyId());
    assertThat(result).hasSize(1);
    assertThat(result.get(0).id()).isEqualTo("e1");
  }

  @Test
  void countPendingMatchesPendingSize() {
    var caseId = UUID.randomUUID();
    repository().save(makeEntry(caseId, "e1", ImprovementStage.RESEARCH_SCOPE), tenancyId());
    repository().save(makeEntry(caseId, "e2", ImprovementStage.HYPOTHESIS_APPROVAL), tenancyId());

    assertThat(repository().countPending(caseId, tenancyId())).isEqualTo(2);
  }

  @Test
  void findAllReturnsAllEntries() {
    var caseId = UUID.randomUUID();
    repository().save(makeEntry(caseId, "e1", ImprovementStage.RESEARCH_SCOPE), tenancyId());
    var resolved = new ConductorInboxEntry(
        caseId, "e2", ImprovementStage.HYPOTHESIS_APPROVAL, Status.REJECTED,
        null, null, null, "rejected", List.of(), 0.5, Instant.now(), Instant.now(), null,
        new ConductorDecision(Status.REJECTED, null, "no", null));
    repository().save(resolved, tenancyId());

    assertThat(repository().findAll(caseId, tenancyId())).hasSize(2);
  }

  @Test
  void perCaseIsolation() {
    var case1 = UUID.randomUUID();
    var case2 = UUID.randomUUID();
    repository().save(makeEntry(case1, "e1", ImprovementStage.RESEARCH_SCOPE), tenancyId());
    repository().save(makeEntry(case2, "e2", ImprovementStage.RESEARCH_SCOPE), tenancyId());

    assertThat(repository().findPending(case1, tenancyId())).hasSize(1);
    assertThat(repository().findPending(case1, tenancyId()).get(0).id()).isEqualTo("e1");
    assertThat(repository().findPending(case2, tenancyId())).hasSize(1);
    assertThat(repository().findPending(case2, tenancyId()).get(0).id()).isEqualTo("e2");
  }

  @Test
  void saveIdempotentOverwrites() {
    var caseId = UUID.randomUUID();
    repository().save(makeEntry(caseId, "e1", ImprovementStage.RESEARCH_SCOPE), tenancyId());
    repository().save(makeEntry(caseId, "e1", ImprovementStage.HYPOTHESIS_APPROVAL), tenancyId());

    assertThat(repository().findAll(caseId, tenancyId())).hasSize(1);
    assertThat(repository().findById(caseId, "e1", tenancyId()).stage())
        .isEqualTo(ImprovementStage.HYPOTHESIS_APPROVAL);
  }
}
```

- [ ] **Step 11: Write concrete test and InMemoryConductorInboxRepository**

Create `runtime-core/src/test/java/io/casehub/engine/internal/improvement/InMemoryConductorInboxRepositoryContractTest.java`:

```java
/*
 * Copyright 2026-Present The Case Hub Authors
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package io.casehub.engine.internal.improvement;

import io.casehub.engine.common.spi.ConductorInboxRepository;
import io.casehub.engine.common.spi.ConductorInboxRepositoryContractTest;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import java.util.UUID;

class InMemoryConductorInboxRepositoryContractTest extends ConductorInboxRepositoryContractTest {

  private InMemoryConductorInboxRepository repo;

  @BeforeEach
  void setUp() {
    repo = new InMemoryConductorInboxRepository();
  }

  @Override
  protected ConductorInboxRepository repository() {
    return repo;
  }

  @Override
  protected String tenancyId() {
    return "test-tenant";
  }

  @Test
  void resetClearsAllEntries() {
    var caseId = UUID.randomUUID();
    repo.save(makeEntry(caseId, "e1", io.casehub.api.model.stigmergy.ImprovementStage.RESEARCH_SCOPE), tenancyId());
    repo.reset();
    assertThat(repo.findAll(caseId, tenancyId())).isEmpty();
  }
}
```

Create `runtime-core/src/main/java/io/casehub/engine/internal/improvement/InMemoryConductorInboxRepository.java`:

```java
/*
 * Copyright 2026-Present The Case Hub Authors
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package io.casehub.engine.internal.improvement;

import io.casehub.api.model.stigmergy.ConductorInboxEntry;
import io.casehub.engine.common.spi.ConductorInboxRepository;
import io.casehub.engine.common.spi.Resettable;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.List;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;

@DefaultBean
@ApplicationScoped
public class InMemoryConductorInboxRepository implements ConductorInboxRepository, Resettable {

  private final ConcurrentHashMap<UUID, ConcurrentHashMap<String, ConductorInboxEntry>> entries =
      new ConcurrentHashMap<>();

  @Override
  public void save(ConductorInboxEntry entry, String tenancyId) {
    entries.computeIfAbsent(entry.caseId(), k -> new ConcurrentHashMap<>())
        .put(entry.id(), entry);
  }

  @Override
  public ConductorInboxEntry findById(UUID caseId, String entryId, String tenancyId) {
    var caseEntries = entries.get(caseId);
    return caseEntries != null ? caseEntries.get(entryId) : null;
  }

  @Override
  public List<ConductorInboxEntry> findPending(UUID caseId, String tenancyId) {
    var caseEntries = entries.get(caseId);
    if (caseEntries == null) {
      return List.of();
    }
    return caseEntries.values().stream()
        .filter(e -> e.status() == ConductorInboxEntry.Status.PENDING)
        .toList();
  }

  @Override
  public int countPending(UUID caseId, String tenancyId) {
    var caseEntries = entries.get(caseId);
    if (caseEntries == null) {
      return 0;
    }
    return (int) caseEntries.values().stream()
        .filter(e -> e.status() == ConductorInboxEntry.Status.PENDING)
        .count();
  }

  @Override
  public List<ConductorInboxEntry> findAll(UUID caseId, String tenancyId) {
    var caseEntries = entries.get(caseId);
    if (caseEntries == null) {
      return List.of();
    }
    return List.copyOf(caseEntries.values());
  }

  @Override
  public void reset() {
    entries.clear();
  }
}
```

- [ ] **Step 12: Run ConductorInboxRepository tests**

Run: `mvn install -DskipTests -q && mvn test -pl runtime-core -Dtest=InMemoryConductorInboxRepositoryContractTest`
Expected: all 8 tests PASS

- [ ] **Step 13: Run all contract tests together**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl common-core,runtime-core`
Expected: all tests PASS (existing + new contract tests)

- [ ] **Step 14: Commit**

```bash
git add common-core/src/test/java/io/casehub/engine/common/spi/ImprovementBlockStoreContractTest.java
git add common-core/src/test/java/io/casehub/engine/common/spi/DenyPatternStoreContractTest.java
git add common-core/src/test/java/io/casehub/engine/common/spi/WatchPatternStoreContractTest.java
git add common-core/src/test/java/io/casehub/engine/common/spi/ConductorInboxRepositoryContractTest.java
git add runtime-core/src/main/java/io/casehub/engine/internal/improvement/InMemoryImprovementBlockStore.java
git add runtime-core/src/main/java/io/casehub/engine/internal/improvement/InMemoryDenyPatternStore.java
git add runtime-core/src/main/java/io/casehub/engine/internal/improvement/InMemoryWatchPatternStore.java
git add runtime-core/src/main/java/io/casehub/engine/internal/improvement/InMemoryConductorInboxRepository.java
git add runtime-core/src/test/java/io/casehub/engine/internal/improvement/InMemoryImprovementBlockStoreContractTest.java
git add runtime-core/src/test/java/io/casehub/engine/internal/improvement/InMemoryDenyPatternStoreContractTest.java
git add runtime-core/src/test/java/io/casehub/engine/internal/improvement/InMemoryWatchPatternStoreContractTest.java
git add runtime-core/src/test/java/io/casehub/engine/internal/improvement/InMemoryConductorInboxRepositoryContractTest.java
git commit -m "feat(#1140): add in-memory implementations with TDD contract tests

4 contract tests in common-core/spi, 4 @DefaultBean impls in runtime-core,
4 concrete test subclasses — all green.

Refs #1140"
```

---

## Batch 3: Domain Bean Refactoring

### Task 4: Refactor ImprovementCoordinator

**Files:**
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/ImprovementCoordinator.java`
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/DefaultEngineEvolutionApi.java:120-126`
- Modify: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/ImprovementCoordinatorTest.java`
- Modify: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/EvolutionApiTest.java`

**Interfaces:**
- Consumes: `ImprovementBlockStore` from Task 2, `InMemoryImprovementBlockStore` from Task 3
- Produces: Refactored `ImprovementCoordinator` with tenancyId-scoped methods

- [ ] **Step 1: Refactor ImprovementCoordinator**

Replace the entire class body of `ImprovementCoordinator.java`:

```java
/*
 * Copyright 2026-Present The Case Hub Authors
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package io.casehub.engine.internal.improvement;

import io.casehub.engine.common.spi.ImprovementBlockStore;
import jakarta.annotation.Nullable;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.UUID;

@ApplicationScoped
public class ImprovementCoordinator {

  private final ImprovementBlockStore blockStore;

  @Inject
  public ImprovementCoordinator(ImprovementBlockStore blockStore) {
    this.blockStore = blockStore;
  }

  public void block(UUID caseId, UUID improvementCaseId, UUID blockerImprovementId,
      String tenancyId) {
    blockStore.save(caseId, improvementCaseId, blockerImprovementId, tenancyId);
  }

  public void unblock(UUID caseId, UUID improvementCaseId, String tenancyId) {
    blockStore.remove(caseId, improvementCaseId, tenancyId);
  }

  public boolean isBlocked(UUID caseId, UUID improvementCaseId, String tenancyId) {
    return blockStore.isBlocked(caseId, improvementCaseId, tenancyId);
  }

  @Nullable
  public UUID blockedBy(UUID caseId, UUID improvementCaseId, String tenancyId) {
    return blockStore.blockedBy(caseId, improvementCaseId, tenancyId);
  }
}
```

- [ ] **Step 2: Update DefaultEngineEvolutionApi block/unblock methods**

In `DefaultEngineEvolutionApi.java`, update `blockImprovement` and `unblockImprovement` to pass `tenancyId`. These methods don't currently have `tenancyId` — add it as a parameter:

```java
public void blockImprovement(UUID caseId, String tenancyId, UUID improvementId,
    UUID blockedBy) {
  coordinator.block(caseId, improvementId, blockedBy, tenancyId);
}

public void unblockImprovement(UUID caseId, String tenancyId, UUID improvementId) {
  coordinator.unblock(caseId, improvementId, tenancyId);
}
```

- [ ] **Step 3: Update ImprovementCoordinatorTest**

Replace the entire test class:

```java
/*
 * Copyright 2026-Present The Case Hub Authors
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package io.casehub.engine.internal.improvement;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class ImprovementCoordinatorTest {

  private static final String TENANT = "test-tenant";

  private ImprovementCoordinator coordinator;
  private UUID caseId;

  @BeforeEach
  void setUp() {
    coordinator = new ImprovementCoordinator(new InMemoryImprovementBlockStore());
    caseId = UUID.randomUUID();
  }

  @Test
  void blockAndCheckBlocked() {
    var improvementId = UUID.randomUUID();
    var blockerId = UUID.randomUUID();
    coordinator.block(caseId, improvementId, blockerId, TENANT);

    assertThat(coordinator.isBlocked(caseId, improvementId, TENANT)).isTrue();
  }

  @Test
  void unblockReleasesBlock() {
    var improvementId = UUID.randomUUID();
    var blockerId = UUID.randomUUID();
    coordinator.block(caseId, improvementId, blockerId, TENANT);
    coordinator.unblock(caseId, improvementId, TENANT);

    assertThat(coordinator.isBlocked(caseId, improvementId, TENANT)).isFalse();
  }

  @Test
  void unblockedByDefault() {
    assertThat(coordinator.isBlocked(caseId, UUID.randomUUID(), TENANT)).isFalse();
  }

  @Test
  void blockedByReturnsBlocker() {
    var improvementId = UUID.randomUUID();
    var blockerId = UUID.randomUUID();
    coordinator.block(caseId, improvementId, blockerId, TENANT);

    assertThat(coordinator.blockedBy(caseId, improvementId, TENANT)).isEqualTo(blockerId);
  }

  @Test
  void blockedByReturnsNullWhenNotBlocked() {
    assertThat(coordinator.blockedBy(caseId, UUID.randomUUID(), TENANT)).isNull();
  }

  @Test
  void perCaseIsolation() {
    var case2 = UUID.randomUUID();
    var improvementId = UUID.randomUUID();
    var blockerId = UUID.randomUUID();
    coordinator.block(caseId, improvementId, blockerId, TENANT);

    assertThat(coordinator.isBlocked(caseId, improvementId, TENANT)).isTrue();
    assertThat(coordinator.isBlocked(case2, improvementId, TENANT)).isFalse();
  }
}
```

- [ ] **Step 4: Update EvolutionApiTest setUp and block/unblock tests**

In `EvolutionApiTest.java`, update `setUp()` to construct `ImprovementCoordinator` with `InMemoryImprovementBlockStore`:

```java
coordinator = new ImprovementCoordinator(new InMemoryImprovementBlockStore());
```

Update any tests that call `blockImprovement`/`unblockImprovement` to pass tenancyId (add `"test-tenant"` after `caseId`).

- [ ] **Step 5: Compile and run tests**

Run: `mvn install -DskipTests -q`
Expected: compilation succeeds

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime-core -Dtest="ImprovementCoordinatorTest,EvolutionApiTest"`
Expected: all tests PASS

- [ ] **Step 6: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/engine/internal/improvement/ImprovementCoordinator.java
git add runtime-core/src/main/java/io/casehub/engine/internal/improvement/DefaultEngineEvolutionApi.java
git add runtime-core/src/test/java/io/casehub/engine/internal/improvement/ImprovementCoordinatorTest.java
git add runtime-core/src/test/java/io/casehub/engine/internal/improvement/EvolutionApiTest.java
git commit -m "feat(#1140): refactor ImprovementCoordinator to inject ImprovementBlockStore

Domain bean is now stateless — delegates to ImprovementBlockStore SPI.
All methods gain tenancyId parameter.

Refs #1140"
```

---

### Task 5: Refactor ImprovementBudgetEnforcer

**Files:**
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/ImprovementBudgetEnforcer.java`
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/DefaultEngineEvolutionApi.java:58-83`
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/ImprovementGoalFormationStrategy.java:119`
- Modify: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/ImprovementBudgetEnforcerTest.java`
- Modify: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/EvolutionApiTest.java`

**Interfaces:**
- Consumes: `DenyPatternStore` from Task 2, `InMemoryDenyPatternStore` from Task 3
- Produces: Refactored `ImprovementBudgetEnforcer` with tenancyId on deny-pattern methods and `check()`

- [ ] **Step 1: Refactor ImprovementBudgetEnforcer**

Remove `dynamicDenyPatterns` field. Add `DenyPatternStore` injection. Add `tenancyId` to `check()`, `addDenyPattern()`, `removeDenyPattern()`, `dynamicDenyPatterns()`, `isDenied()`. Update `reset()` to not clear dynamicDenyPatterns. Keep remaining state fields unchanged.

Key changes to the class:

1. Remove field: `private final ConcurrentHashMap<UUID, Set<String>> dynamicDenyPatterns = new ConcurrentHashMap<>();`
2. Add field: `private final DenyPatternStore denyPatternStore;`
3. Add constructor: `@Inject public ImprovementBudgetEnforcer(DenyPatternStore denyPatternStore)`
4. `check()` signature: add `String tenancyId` as last parameter, use `denyPatternStore.findAll(caseId, tenancyId)` instead of `dynamicDenyPatterns.getOrDefault(caseId, Set.of())`
5. `addDenyPattern()`: `(UUID caseId, String pattern, String tenancyId)` → `denyPatternStore.save(caseId, pattern, tenancyId)`
6. `removeDenyPattern()`: `(UUID caseId, String pattern, String tenancyId)` → `denyPatternStore.remove(caseId, pattern, tenancyId)`
7. `dynamicDenyPatterns()`: `(UUID caseId, String tenancyId)` → `return denyPatternStore.findAll(caseId, tenancyId)`
8. `isDenied()`: `(UUID caseId, ImprovementRequest request, String tenancyId)` → use `denyPatternStore.findAll(caseId, tenancyId)`
9. `reset()`: remove `dynamicDenyPatterns.clear()` line

- [ ] **Step 2: Update DefaultEngineEvolutionApi deny pattern methods**

In `DefaultEngineEvolutionApi.java`, update:

```java
public DenyPatternView getDenyPatterns(UUID caseId, String tenancyId) {
  return new DenyPatternView(
      List.copyOf(ImprovementBudgetEnforcer.staticDenyPatterns()),
      budgetEnforcer.dynamicDenyPatterns(caseId, tenancyId).stream()
          .map(p -> new DenyPatternView.DynamicDenyEntry(p, "operator", Instant.now()))
          .toList());
}

public void addDenyPattern(UUID caseId, String tenancyId, String pattern) {
  budgetEnforcer.addDenyPattern(caseId, pattern, tenancyId);
}

public void removeDenyPattern(UUID caseId, String tenancyId, String pattern) {
  budgetEnforcer.removeDenyPattern(caseId, pattern, tenancyId);
}
```

- [ ] **Step 3: Update ImprovementGoalFormationStrategy**

In `ImprovementGoalFormationStrategy.java`, update `proposeImprovements` to accept and pass `tenancyId`:

```java
public GoalFormationProposal proposeImprovements(UUID caseId, String tenancyId,
    ImprovementConfig config) {
```

Update the `check()` call at line ~119:
```java
var budgetCheck = budgetEnforcer.check(caseId, config.effectiveBudget(), request, tenancyId);
```

Check all callers of `proposeImprovements` to pass `tenancyId` (likely `EvolutionTicker` — use `ide_find_references` to locate).

- [ ] **Step 4: Update ImprovementBudgetEnforcerTest**

Update `setUp()`:
```java
enforcer = new ImprovementBudgetEnforcer(new InMemoryDenyPatternStore());
```

Add `private static final String TENANT = "test-tenant";`

Update all calls to `check()` to include `TENANT` as the last argument:
```java
var result = enforcer.check(caseId, budget, request, TENANT);
```

Update the `userDenyListBlocksMatchingPath` test to use `addDenyPattern(caseId, pattern, TENANT)`.

- [ ] **Step 5: Update EvolutionApiTest**

Update `setUp()`:
```java
var budgetEnforcer = new ImprovementBudgetEnforcer(new InMemoryDenyPatternStore());
```

Update deny pattern test calls to include tenancyId where the API now requires it.

- [ ] **Step 6: Compile and run tests**

Run: `mvn install -DskipTests -q`
Expected: compilation succeeds

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime-core -Dtest="ImprovementBudgetEnforcerTest,EvolutionApiTest"`
Expected: all tests PASS

- [ ] **Step 7: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/engine/internal/improvement/ImprovementBudgetEnforcer.java
git add runtime-core/src/main/java/io/casehub/engine/internal/improvement/DefaultEngineEvolutionApi.java
git add runtime-core/src/main/java/io/casehub/engine/internal/improvement/ImprovementGoalFormationStrategy.java
git add runtime-core/src/test/java/io/casehub/engine/internal/improvement/ImprovementBudgetEnforcerTest.java
git add runtime-core/src/test/java/io/casehub/engine/internal/improvement/EvolutionApiTest.java
git commit -m "feat(#1140): refactor ImprovementBudgetEnforcer to inject DenyPatternStore

Dynamic deny patterns extracted to DenyPatternStore SPI. Ephemeral
runtime state (activeImprovements, dailyCounts, lastCompletionTime)
stays in the domain bean. All deny-pattern methods gain tenancyId.

Refs #1140"
```

---

### Task 6: Refactor ConductorInboxManager

**Files:**
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/ConductorInboxManager.java`
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/DefaultEngineEvolutionApi.java:85-118`
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/ResearchPipelineOrchestrator.java:162`
- Modify: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/ConductorInboxManagerTest.java`
- Modify: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/EvolutionApiTest.java`

**Interfaces:**
- Consumes: `ConductorInboxRepository` and `WatchPatternStore` from Task 2, `InMemoryConductorInboxRepository` and `InMemoryWatchPatternStore` from Task 3
- Produces: Refactored `ConductorInboxManager` with tenancyId-scoped methods

- [ ] **Step 1: Refactor ConductorInboxManager**

Replace the entire class:

```java
/*
 * Copyright 2026-Present The Case Hub Authors
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package io.casehub.engine.internal.improvement;

import io.casehub.api.model.stigmergy.ConductorDecision;
import io.casehub.api.model.stigmergy.ConductorInboxEntry;
import io.casehub.api.model.stigmergy.WatchPattern;
import io.casehub.engine.common.spi.ConductorInboxRepository;
import io.casehub.engine.common.spi.WatchPatternStore;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.time.Instant;
import java.util.List;
import java.util.UUID;

@ApplicationScoped
public class ConductorInboxManager {

  private final ConductorInboxRepository inboxRepository;
  private final WatchPatternStore watchPatternStore;

  @Inject
  public ConductorInboxManager(ConductorInboxRepository inboxRepository,
      WatchPatternStore watchPatternStore) {
    this.inboxRepository = inboxRepository;
    this.watchPatternStore = watchPatternStore;
  }

  public String enqueue(UUID caseId, ConductorInboxEntry entry, String tenancyId) {
    inboxRepository.save(entry, tenancyId);
    return entry.id();
  }

  public List<ConductorInboxEntry> pending(UUID caseId, String tenancyId) {
    return inboxRepository.findPending(caseId, tenancyId);
  }

  public int pendingCount(UUID caseId, String tenancyId) {
    return inboxRepository.countPending(caseId, tenancyId);
  }

  public List<ConductorInboxEntry> allEntries(UUID caseId, String tenancyId) {
    return inboxRepository.findAll(caseId, tenancyId);
  }

  public void resolve(UUID caseId, String entryId, ConductorDecision decision,
      String tenancyId) {
    var existing = inboxRepository.findById(caseId, entryId, tenancyId);
    if (existing == null) {
      return;
    }
    var resolved = new ConductorInboxEntry(
        existing.caseId(),
        existing.id(),
        existing.stage(),
        decision.outcome(),
        existing.category(),
        existing.areaId(),
        existing.improvementCaseId(),
        existing.summary(),
        existing.escalationTriggers(),
        existing.confidence(),
        existing.queuedAt(),
        Instant.now(),
        existing.timeoutMinutes(),
        decision);
    inboxRepository.save(resolved, tenancyId);
  }

  public List<WatchPattern> activeWatchPatterns(UUID caseId, String tenancyId) {
    return watchPatternStore.findActive(caseId, tenancyId);
  }

  public void addWatchPattern(UUID caseId, WatchPattern pattern, String tenancyId) {
    watchPatternStore.save(caseId, pattern, tenancyId);
  }

  public void removeWatchPattern(UUID caseId, String patternId, String tenancyId) {
    watchPatternStore.remove(caseId, patternId, tenancyId);
  }
}
```

- [ ] **Step 2: Update DefaultEngineEvolutionApi inbox/watch methods**

Update all inbox and watch pattern methods to pass tenancyId:

```java
public List<ConductorInboxEntry> getInbox(UUID caseId, String tenancyId) {
  return inboxManager.pending(caseId, tenancyId);
}

public void resolveGate(UUID caseId, String tenancyId, String entryId,
    ConductorInboxEntry.Status outcome, @Nullable String reason,
    @Nullable String feedback) {
  var decision = new ConductorDecision(outcome, null, reason, feedback);
  inboxManager.resolve(caseId, entryId, decision, tenancyId);
}

public void addWatchPattern(UUID caseId, String tenancyId, @Nullable String category,
    @Nullable String areaId, @Nullable String targetPattern,
    @Nullable Integer minEstimatedSize) {
  var pattern = new WatchPattern(UUID.randomUUID().toString(), category, areaId,
      targetPattern, minEstimatedSize, Instant.now());
  inboxManager.addWatchPattern(caseId, pattern, tenancyId);
}

public void removeWatchPattern(UUID caseId, String tenancyId, String patternId) {
  inboxManager.removeWatchPattern(caseId, patternId, tenancyId);
}
```

- [ ] **Step 3: Update ResearchPipelineOrchestrator.enqueueForApproval()**

Add `tenancyId` to the `enqueueForApproval` method and pass it to `inboxManager.enqueue()`:

```java
private String enqueueForApproval(UUID caseId, String tenancyId,
    ImprovementStage stage, GateCheckpoint checkpoint) {
  var entryId = UUID.randomUUID().toString();
  var entry = new ConductorInboxEntry(
      caseId, entryId, stage, ConductorInboxEntry.Status.PENDING,
      null, null, null, "Gate checkpoint at " + stage,
      List.of(), 1.0, Instant.now(), null, null, null);
  inboxManager.enqueue(caseId, entry, tenancyId);
  return entryId;
}
```

Update all callers of `enqueueForApproval` within ResearchPipelineOrchestrator to pass `tenancyId` (already available in `execute()` and `resume()` method parameters).

- [ ] **Step 4: Update ConductorInboxManagerTest**

Replace the entire test class to use the new constructor and tenancyId:

```java
/*
 * Copyright 2026-Present The Case Hub Authors
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package io.casehub.engine.internal.improvement;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.api.model.stigmergy.ConductorDecision;
import io.casehub.api.model.stigmergy.ConductorInboxEntry;
import io.casehub.api.model.stigmergy.ConductorInboxEntry.Status;
import io.casehub.api.model.stigmergy.ImprovementStage;
import io.casehub.api.model.stigmergy.WatchPattern;
import java.time.Instant;
import java.util.List;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class ConductorInboxManagerTest {

  private static final String TENANT = "test-tenant";

  private ConductorInboxManager manager;
  private UUID caseId;

  @BeforeEach
  void setUp() {
    manager = new ConductorInboxManager(
        new InMemoryConductorInboxRepository(),
        new InMemoryWatchPatternStore());
    caseId = UUID.randomUUID();
  }

  @Test
  void enqueueAndRetrievePending() {
    var entry = makeEntry(caseId, "e1", ImprovementStage.RESEARCH_SCOPE);
    manager.enqueue(caseId, entry, TENANT);

    var pending = manager.pending(caseId, TENANT);
    assertThat(pending).hasSize(1);
    assertThat(pending.get(0).id()).isEqualTo("e1");
    assertThat(pending.get(0).status()).isEqualTo(Status.PENDING);
  }

  @Test
  void resolveGateApproved() {
    manager.enqueue(caseId, makeEntry(caseId, "e1", ImprovementStage.RESEARCH_SCOPE), TENANT);
    var decision = new ConductorDecision(Status.APPROVED, null, "looks good", null);

    manager.resolve(caseId, "e1", decision, TENANT);

    assertThat(manager.pending(caseId, TENANT)).isEmpty();
    assertThat(manager.pendingCount(caseId, TENANT)).isZero();
  }

  @Test
  void resolveGateRejected() {
    manager.enqueue(caseId, makeEntry(caseId, "e1", ImprovementStage.HYPOTHESIS_APPROVAL),
        TENANT);
    var decision = new ConductorDecision(Status.REJECTED, null, "not viable", null);

    manager.resolve(caseId, "e1", decision, TENANT);

    assertThat(manager.pending(caseId, TENANT)).isEmpty();
  }

  @Test
  void resolveGateRedirected() {
    manager.enqueue(caseId, makeEntry(caseId, "e1", ImprovementStage.IMPLEMENTATION_PLAN),
        TENANT);
    var decision = new ConductorDecision(Status.REDIRECTED, null, "try different approach", null);

    manager.resolve(caseId, "e1", decision, TENANT);

    assertThat(manager.pending(caseId, TENANT)).isEmpty();
  }

  @Test
  void pendingCountMatchesPendingEntries() {
    manager.enqueue(caseId, makeEntry(caseId, "e1", ImprovementStage.RESEARCH_SCOPE), TENANT);
    manager.enqueue(caseId, makeEntry(caseId, "e2", ImprovementStage.HYPOTHESIS_APPROVAL),
        TENANT);

    assertThat(manager.pendingCount(caseId, TENANT)).isEqualTo(2);

    manager.resolve(caseId, "e1", new ConductorDecision(Status.APPROVED, null, null, null),
        TENANT);

    assertThat(manager.pendingCount(caseId, TENANT)).isEqualTo(1);
  }

  @Test
  void perCaseIsolation() {
    var case2 = UUID.randomUUID();
    manager.enqueue(caseId, makeEntry(caseId, "e1", ImprovementStage.RESEARCH_SCOPE), TENANT);
    manager.enqueue(case2, makeEntry(case2, "e2", ImprovementStage.RESEARCH_SCOPE), TENANT);

    assertThat(manager.pending(caseId, TENANT)).hasSize(1);
    assertThat(manager.pending(case2, TENANT)).hasSize(1);
    assertThat(manager.pending(caseId, TENANT).get(0).id()).isEqualTo("e1");
    assertThat(manager.pending(case2, TENANT).get(0).id()).isEqualTo("e2");
  }

  @Test
  void resolveNonexistentEntryIsNoOp() {
    manager.resolve(caseId, "nonexistent",
        new ConductorDecision(Status.APPROVED, null, null, null), TENANT);

    assertThat(manager.pending(caseId, TENANT)).isEmpty();
  }

  @Test
  void resolvedEntryRetainsDecision() {
    manager.enqueue(caseId, makeEntry(caseId, "e1", ImprovementStage.PR_REVIEW), TENANT);
    var decision = new ConductorDecision(Status.APPROVED, null, "LGTM", "nice work");

    manager.resolve(caseId, "e1", decision, TENANT);

    var resolved = manager.allEntries(caseId, TENANT).stream()
        .filter(e -> e.id().equals("e1"))
        .findFirst()
        .orElseThrow();
    assertThat(resolved.status()).isEqualTo(Status.APPROVED);
    assertThat(resolved.decision()).isEqualTo(decision);
    assertThat(resolved.resolvedAt()).isNotNull();
  }

  @Test
  void watchPatternAddAndList() {
    var pattern = new WatchPattern("w1", "security", null, null, null, Instant.now());
    manager.addWatchPattern(caseId, pattern, TENANT);

    assertThat(manager.activeWatchPatterns(caseId, TENANT)).hasSize(1);
    assertThat(manager.activeWatchPatterns(caseId, TENANT).get(0).id()).isEqualTo("w1");
  }

  @Test
  void watchPatternRemove() {
    manager.addWatchPattern(caseId,
        new WatchPattern("w1", "security", null, null, null, Instant.now()), TENANT);
    manager.addWatchPattern(caseId,
        new WatchPattern("w2", "architecture", null, null, null, Instant.now()), TENANT);

    manager.removeWatchPattern(caseId, "w1", TENANT);

    assertThat(manager.activeWatchPatterns(caseId, TENANT)).hasSize(1);
    assertThat(manager.activeWatchPatterns(caseId, TENANT).get(0).id()).isEqualTo("w2");
  }

  @Test
  void watchPatternPerCaseIsolation() {
    var case2 = UUID.randomUUID();
    manager.addWatchPattern(caseId,
        new WatchPattern("w1", "security", null, null, null, Instant.now()), TENANT);

    assertThat(manager.activeWatchPatterns(caseId, TENANT)).hasSize(1);
    assertThat(manager.activeWatchPatterns(case2, TENANT)).isEmpty();
  }

  @Test
  void emptyPendingForUnknownCase() {
    assertThat(manager.pending(UUID.randomUUID(), TENANT)).isEmpty();
    assertThat(manager.pendingCount(UUID.randomUUID(), TENANT)).isZero();
  }

  private ConductorInboxEntry makeEntry(UUID caseId, String id, ImprovementStage stage) {
    return new ConductorInboxEntry(
        caseId, id, stage, Status.PENDING, null, null, null, "test entry",
        List.of(), 0.8, Instant.now(), null, null, null);
  }
}
```

- [ ] **Step 5: Update EvolutionApiTest**

Update `setUp()` to construct ConductorInboxManager with in-memory repos:

```java
inboxManager = new ConductorInboxManager(
    new InMemoryConductorInboxRepository(),
    new InMemoryWatchPatternStore());
```

Update all test calls to pass tenancyId where API signatures changed.

- [ ] **Step 6: Compile and run all tests**

Run: `mvn install -DskipTests -q`
Expected: compilation succeeds

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime-core`
Expected: all tests PASS

- [ ] **Step 7: Run full test suite**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test`
Expected: all tests PASS across all modules

- [ ] **Step 8: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/engine/internal/improvement/ConductorInboxManager.java
git add runtime-core/src/main/java/io/casehub/engine/internal/improvement/DefaultEngineEvolutionApi.java
git add runtime-core/src/main/java/io/casehub/engine/internal/improvement/ResearchPipelineOrchestrator.java
git add runtime-core/src/test/java/io/casehub/engine/internal/improvement/ConductorInboxManagerTest.java
git add runtime-core/src/test/java/io/casehub/engine/internal/improvement/EvolutionApiTest.java
git commit -m "feat(#1140): refactor ConductorInboxManager to inject ConductorInboxRepository + WatchPatternStore

Domain bean is now stateless — inbox entries and watch patterns
delegated to separate SPIs per their different lifecycle semantics.
All methods gain tenancyId parameter.

Closes #1140"
```

---

## References

- `2026-09-22-conductor-state-persistence-design.md` — design spec this plan implements
- `CaseInstanceRepository.java:34` — SPI interface pattern
- `CaseInstanceRepositoryContractTest.java:28` — contract test pattern
- `InMemoryCaseInstanceRepository.java:36` — in-memory implementation pattern
- `DefaultSummarizationProvider.java` — @DefaultBean pattern
- `ConductorInboxManager.java:31-118` — current implementation
- `ImprovementBudgetEnforcer.java:32-206` — current implementation
- `ImprovementCoordinator.java:25-58` — current implementation
- `DefaultEngineEvolutionApi.java:33-127` — caller of all three domain beans
- `ResearchPipelineOrchestrator.java:49,144-164` — ConductorInboxManager caller
- `ImprovementGoalFormationStrategy.java:38,119` — ImprovementBudgetEnforcer caller
- D1–D6 in `decisions.md` — design decisions
- PP-20260921-b7c277 — no-defaultbean-multi-instance-spi protocol
- casehubio/engine#1140 — focal issue
- casehubio/engine#1149 — parent epic
- casehubio/engine#1150 — persistence layer coherence (platform direction)
