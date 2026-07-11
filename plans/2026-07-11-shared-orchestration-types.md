# Shared Orchestration Types Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #700 — unify orchestration model
**Issue group:** #700

**Goal:** Replace engine-internal `PlanItemStatus` with shared `TaskStatus`, add `TaskDescriptor` interface and `TaskSnapshot` read model, and thread `ExecutorRef` through `PlanItem` replacing bare `workerName`.

**Architecture:** Four new/relocated types in `api/model/`: `TaskStatus` (enum, replaces `PlanItemStatus`), `TaskDescriptor` (interface), `TaskSnapshot` (record). `PlanItem` implements `TaskDescriptor` and stores `ExecutorRef` instead of `String workerName`. Persistence layer threads `executorName`/`executorDescription` as flat strings.

**Tech Stack:** Java 21, Quarkus 3.32, Hibernate Reactive (Panache), JUnit 5, AssertJ

## Global Constraints

- `PlanItemStatus` is deleted — not wrapped, not aliased. `TaskStatus` IS the type.
- No Flyway migrations — this project uses `quarkus.hibernate-orm.schema-management.strategy=drop-and-create` with no installed instances. Update `@Entity` classes; Hibernate recreates the schema.
- IntelliJ MCP mandatory for all Java file operations. Use `ide_refactor_rename` for the `PlanItemStatus` → `TaskStatus` rename.
- `common` depends on `api` — `TaskStatus` in `api/model/` is visible from `common`.

---

### Task 1: Create TaskStatus, TaskSnapshot, TaskDescriptor in api/model

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/TaskStatus.java`
- Create: `api/src/main/java/io/casehub/api/model/TaskSnapshot.java`
- Create: `api/src/main/java/io/casehub/api/model/TaskDescriptor.java`
- Create: `api/src/test/java/io/casehub/api/model/TaskStatusTest.java`
- Create: `api/src/test/java/io/casehub/api/model/TaskSnapshotTest.java`

**Interfaces:**
- Produces: `TaskStatus` enum with `isTerminal()`, `isActive()` — same values as current `PlanItemStatus`
- Produces: `TaskDescriptor` interface with `id()`, `description()`, `executor()`, `status()`, `createdAt()`, `snapshot()`
- Produces: `TaskSnapshot` record with `id`, `description`, `executorName`, `executorDescription`, `status`, `createdAt`

- [ ] **Step 1: Write TaskStatus test**

```java
// api/src/test/java/io/casehub/api/model/TaskStatusTest.java
package io.casehub.api.model;

import static org.assertj.core.api.Assertions.assertThat;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.EnumSource;

class TaskStatusTest {

  @Test
  void terminalStates() {
    assertThat(TaskStatus.COMPLETED.isTerminal()).isTrue();
    assertThat(TaskStatus.FAULTED.isTerminal()).isTrue();
    assertThat(TaskStatus.REJECTED.isTerminal()).isTrue();
    assertThat(TaskStatus.OBSOLETE.isTerminal()).isTrue();
    assertThat(TaskStatus.CANCELLED.isTerminal()).isTrue();
  }

  @Test
  void activeStates() {
    assertThat(TaskStatus.PENDING.isActive()).isTrue();
    assertThat(TaskStatus.RUNNING.isActive()).isTrue();
    assertThat(TaskStatus.DELEGATED.isActive()).isTrue();
    assertThat(TaskStatus.SUSPENDED.isActive()).isTrue();
  }

  @ParameterizedTest
  @EnumSource(TaskStatus.class)
  void terminalAndActiveAreExhaustiveAndNonOverlapping(TaskStatus status) {
    assertThat(status.isTerminal() ^ status.isActive())
        .as("Every status must be either terminal or active, never both, never neither: %s", status)
        .isTrue();
  }

  @Test
  void allNineValuesPresent() {
    assertThat(TaskStatus.values()).hasSize(9);
  }
}
```

- [ ] **Step 2: Run test — expect FAIL (TaskStatus not found)**

```bash
mvn test -pl api -Dtest=TaskStatusTest -q
```

- [ ] **Step 3: Create TaskStatus enum**

```java
// api/src/main/java/io/casehub/api/model/TaskStatus.java
package io.casehub.api.model;

public enum TaskStatus {
  PENDING,
  RUNNING,
  DELEGATED,
  SUSPENDED,
  COMPLETED,
  FAULTED,
  REJECTED,
  OBSOLETE,
  CANCELLED;

  public boolean isTerminal() {
    return this == COMPLETED
        || this == FAULTED
        || this == REJECTED
        || this == OBSOLETE
        || this == CANCELLED;
  }

  public boolean isActive() {
    return this == PENDING || this == RUNNING || this == DELEGATED || this == SUSPENDED;
  }
}
```

- [ ] **Step 4: Run test — expect PASS**

- [ ] **Step 5: Write TaskSnapshot test**

```java
// api/src/test/java/io/casehub/api/model/TaskSnapshotTest.java
package io.casehub.api.model;

import static org.assertj.core.api.Assertions.assertThat;
import java.time.Instant;
import org.junit.jupiter.api.Test;

class TaskSnapshotTest {

  @Test
  void fieldsPreserved() {
    Instant now = Instant.now();
    TaskSnapshot snap = new TaskSnapshot("id-1", "do work", "worker-a", "desc", TaskStatus.RUNNING, now);
    assertThat(snap.id()).isEqualTo("id-1");
    assertThat(snap.description()).isEqualTo("do work");
    assertThat(snap.executorName()).isEqualTo("worker-a");
    assertThat(snap.executorDescription()).isEqualTo("desc");
    assertThat(snap.status()).isEqualTo(TaskStatus.RUNNING);
    assertThat(snap.createdAt()).isEqualTo(now);
  }

  @Test
  void nullableFieldsAllowed() {
    TaskSnapshot snap = new TaskSnapshot("id-1", null, null, null, TaskStatus.PENDING, Instant.now());
    assertThat(snap.description()).isNull();
    assertThat(snap.executorName()).isNull();
    assertThat(snap.executorDescription()).isNull();
  }
}
```

- [ ] **Step 6: Create TaskSnapshot record**

```java
// api/src/main/java/io/casehub/api/model/TaskSnapshot.java
package io.casehub.api.model;

import jakarta.annotation.Nullable;
import java.time.Instant;

public record TaskSnapshot(
    String id,
    @Nullable String description,
    @Nullable String executorName,
    @Nullable String executorDescription,
    TaskStatus status,
    Instant createdAt) {}
```

- [ ] **Step 7: Create TaskDescriptor interface**

```java
// api/src/main/java/io/casehub/api/model/TaskDescriptor.java
package io.casehub.api.model;

import jakarta.annotation.Nullable;
import java.time.Instant;

public interface TaskDescriptor {

  String id();

  @Nullable
  String description();

  @Nullable
  ExecutorRef executor();

  TaskStatus status();

  Instant createdAt();

  default TaskSnapshot snapshot() {
    return new TaskSnapshot(
        id(),
        description(),
        executor() != null ? executor().name() : null,
        executor() != null ? executor().description() : null,
        status(),
        createdAt());
  }
}
```

- [ ] **Step 8: Run all tests — expect PASS**

```bash
mvn test -pl api -Dtest='TaskStatusTest,TaskSnapshotTest' -q
```

- [ ] **Step 9: Commit**

```
feat(#700): add TaskStatus, TaskDescriptor, TaskSnapshot to api/model
```

---

### Task 2: Rename PlanItemStatus → TaskStatus across the codebase

**Files:**
- Delete: `common/src/main/java/io/casehub/engine/common/internal/model/PlanItemStatus.java`
- Delete: `common/src/test/java/io/casehub/engine/common/internal/model/PlanItemStatusTest.java`
- Modify: 34 files that reference `PlanItemStatus` (all updated by IntelliJ rename)

**Interfaces:**
- Consumes: `TaskStatus` from Task 1
- Produces: All `PlanItemStatus` references replaced with `TaskStatus` — every file in the codebase

**Strategy:** This is a mechanical rename. IntelliJ's `ide_refactor_rename` on the `PlanItemStatus` class renames it AND updates all 208 references across 34 files in one operation. Then move the file to `api/model/` — but since we already created `TaskStatus` in Task 1, we instead delete `PlanItemStatus` after verifying all references point to `TaskStatus`.

The actual approach: since `TaskStatus` already exists (Task 1) with identical values, we use `ide_structural_search_replace` to replace all `PlanItemStatus` references with `TaskStatus`, then delete `PlanItemStatus`.

- [ ] **Step 1: Replace all PlanItemStatus references with TaskStatus**

Use `ide_structural_search_replace` to replace `PlanItemStatus` → `TaskStatus` across the project. Then use `ide_optimize_imports` on each affected file to fix the import from `common.internal.model.PlanItemStatus` to `api.model.TaskStatus`.

Alternatively, since there are 34 files: change the import in each file from `io.casehub.engine.common.internal.model.PlanItemStatus` to `io.casehub.api.model.TaskStatus`, and the type name changes automatically since the simple name stays `TaskStatus` after import.

In practice: use `ide_search_text` to find all `PlanItemStatus` references, then for each file:
1. Replace the import line
2. Replace all `PlanItemStatus` occurrences with `TaskStatus`

- [ ] **Step 2: Delete PlanItemStatus.java and PlanItemStatusTest.java**

Use `ide_refactor_safe_delete` on `PlanItemStatus.java`. If any references remain, they indicate missed migration — fix them first.

- [ ] **Step 3: Build the full project (compile only)**

```bash
mvn compile -Dmaven.test.skip=true -q
```

- [ ] **Step 4: Run affected module tests**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api,common,blackboard -q
```

- [ ] **Step 5: Commit**

```
feat(#700): replace PlanItemStatus with TaskStatus — delete PlanItemStatus
```

---

### Task 3: Thread ExecutorRef through PlanItem, replacing workerName

**Files:**
- Modify: `blackboard/src/main/java/io/casehub/blackboard/plan/PlanItem.java` — replace `workerName` field with `ExecutorRef executor`, update `create()`/`restore()` signatures, add `executorName()` convenience, implement `TaskDescriptor`
- Modify: `blackboard/src/main/java/io/casehub/blackboard/control/PlanningStrategyLoopControl.java` — `resolveWorkerName()` → `resolveExecutor()` returning `ExecutorRef`
- Modify: `blackboard/src/main/java/io/casehub/blackboard/registry/PlanItemRestorer.java` — pass `ExecutorRef` to `restore()`
- Modify: `blackboard/src/test/java/io/casehub/blackboard/plan/PlanItemTest.java` — update all `create()` calls
- Modify: ~80 test files with `PlanItem.create("binding", "worker", ...)` calls — pass `ExecutorRef.of("worker")` instead
- Test: `blackboard/src/test/java/io/casehub/blackboard/plan/PlanItemTest.java`

**Interfaces:**
- Consumes: `TaskStatus` (Task 2), `TaskDescriptor` (Task 1), `ExecutorRef` (existing)
- Produces: `PlanItem implements TaskDescriptor` with `ExecutorRef executor` field, `executorName()` derived accessor

- [ ] **Step 1: Write failing test for PlanItem as TaskDescriptor**

Add to `PlanItemTest.java`:

```java
@Test
void implementsTaskDescriptor() {
  PlanItem item = PlanItem.create("binding-a", ExecutorRef.of("worker-a"), 0);
  TaskDescriptor td = item; // compile check
  assertThat(td.id()).isEqualTo(item.getPlanItemId());
  assertThat(td.description()).isNull();
  assertThat(td.executor()).isNotNull();
  assertThat(td.executor().name()).isEqualTo("worker-a");
  assertThat(td.status()).isEqualTo(TaskStatus.PENDING);
  assertThat(td.createdAt()).isNotNull();
}

@Test
void snapshotProjection() {
  PlanItem item = PlanItem.create("binding-a", ExecutorRef.of("worker-a", "does stuff"), 0);
  TaskSnapshot snap = item.snapshot();
  assertThat(snap.id()).isEqualTo(item.getPlanItemId());
  assertThat(snap.executorName()).isEqualTo("worker-a");
  assertThat(snap.executorDescription()).isEqualTo("does stuff");
  assertThat(snap.status()).isEqualTo(TaskStatus.PENDING);
}

@Test
void executorNameConvenience() {
  PlanItem item = PlanItem.create("binding-a", ExecutorRef.of("worker-a"), 0);
  assertThat(item.executorName()).isEqualTo("worker-a");
}

@Test
void restoreWithExecutorRef() {
  ExecutorRef exec = ExecutorRef.of("restored-worker", "desc");
  PlanItem item = PlanItem.restore(
      "plan-1", "binding-a", exec, null, TaskStatus.RUNNING, Instant.now(), null);
  assertThat(item.executor().name()).isEqualTo("restored-worker");
  assertThat(item.executor().description()).isEqualTo("desc");
}
```

- [ ] **Step 2: Run test — expect FAIL**

- [ ] **Step 3: Modify PlanItem**

Replace `workerName` field with `ExecutorRef executor`. Update constructors, `create()` overloads, `restore()` overloads. Implement `TaskDescriptor`. Add `executorName()` convenience. Deprecate `getPlanItemId()` with `@Deprecated` pointing to `id()`.

Key changes to PlanItem:
- Field: `private final String workerName` → `private final ExecutorRef executor`
- `create(String bindingName, String workerName, int priority)` → `create(String bindingName, ExecutorRef executor, int priority)`
- All three `create()` overloads: same pattern
- `restore()` overloads: `PlanItemStatus` already became `TaskStatus` in Task 2, now add `ExecutorRef` parameter
- `getWorkerName()` → `executorName()` (returns `executor != null ? executor.name() : null`)
- `getExecutor()` → returns `ExecutorRef`
- `implements Comparable<PlanItem>` → `implements Comparable<PlanItem>, TaskDescriptor`
- `id()` → returns `planItemId`
- `status()` → returns `getStatus()` (already returns `TaskStatus` after Task 2)

- [ ] **Step 4: Update PlanningStrategyLoopControl.resolveWorkerName → resolveExecutor**

Change `resolveWorkerName(Binding, PlanExecutionContext)` to `resolveExecutor(Binding, PlanExecutionContext)` returning `ExecutorRef`:

```java
private ExecutorRef resolveExecutor(Binding binding, PlanExecutionContext ctx) {
  return switch (binding.target()) {
    case null -> ExecutorRef.of("unknown");
    case CapabilityTarget ct -> {
      String capName = ct.capability().name();
      List<Worker> matching = ctx.definition().getWorkers().stream()
          .filter(w -> w.capabilityNames() != null && w.capabilityNames().contains(capName))
          .toList();
      // ... same multi-match warning ...
      yield matching.isEmpty() ? ExecutorRef.of(capName) : ExecutorRef.fromWorker(matching.get(0));
    }
    case SubCaseTarget st -> ExecutorRef.of("unknown");
    case HumanTaskTarget ht -> ExecutorRef.of("unknown");
    case ExtensionTarget et -> ExecutorRef.of("unknown");
  };
}
```

Update call sites at lines 142-144 and 202.

- [ ] **Step 5: Update PlanItemRestorer to pass ExecutorRef**

`PlanItemRestorer.restore()` currently doesn't have executor info from `PlanItemRecord`. After Task 4 adds executor columns, it will pass `ExecutorRef.of(r.executorName(), r.executorDescription())`. For now, pass `null` (executor info not yet in persistence).

- [ ] **Step 6: Update all test PlanItem.create() calls**

~80 test call sites change from `PlanItem.create("binding", "worker", 0)` to `PlanItem.create("binding", ExecutorRef.of("worker"), 0)`. Use `ide_structural_search_replace`:

```
Search:  PlanItem.create($bindingName$, $workerName$, $priority$)
Replace: PlanItem.create($bindingName$, ExecutorRef.of($workerName$), $priority$)
```

And for the 4-arg variant:
```
Search:  PlanItem.create($bindingName$, $workerName$, $priority$, $target$)
Replace: PlanItem.create($bindingName$, ExecutorRef.of($workerName$), $priority$, $target$)
```

Add `import io.casehub.api.model.ExecutorRef;` to each modified test file.

- [ ] **Step 7: Update PlanningStrategyLoopControl.indexForCompletion call**

Line 254: `pi.getWorkerName()` → `pi.executorName()`

- [ ] **Step 8: Build and run tests**

```bash
mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl blackboard -q
```

- [ ] **Step 9: Commit**

```
feat(#700): PlanItem implements TaskDescriptor, stores ExecutorRef — replace workerName
```

---

### Task 4: Thread ExecutorRef through persistence layer

**Files:**
- Modify: `common/src/main/java/io/casehub/engine/common/internal/model/PlanItemRecord.java` — add `executorName`, `executorDescription` fields
- Modify: `common/src/main/java/io/casehub/engine/common/internal/model/PlanItemSaveRequest.java` — add `executorName`, `executorDescription` fields
- Modify: `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/PlanItemEntity.java` — add `executor_name`, `executor_description` columns
- Modify: `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaReactivePlanItemStore.java` — map executor fields
- Modify: `persistence-memory/src/main/java/io/casehub/persistence/memory/InMemoryPlanItemStore.java` — map executor fields
- Modify: `blackboard/src/main/java/io/casehub/blackboard/registry/PlanItemRestorer.java` — pass `ExecutorRef.of(name, desc)` to `PlanItem.restore()`
- Modify: `common/src/test/java/io/casehub/engine/common/spi/PlanItemStoreContractTest.java`
- Modify: `common/src/test/java/io/casehub/engine/common/spi/ReactivePlanItemStoreContractTest.java`

**Interfaces:**
- Consumes: `PlanItem` with `ExecutorRef` (Task 3), `TaskStatus` (Task 2)
- Produces: Persistence round-trip preserves executor identity

- [ ] **Step 1: Update PlanItemRecord — add executor fields**

```java
public record PlanItemRecord(
    UUID caseId,
    String planItemId,
    String bindingName,
    TaskStatus status,
    Instant createdAt,
    TargetType targetType,
    String outputMappingExpression,
    String tenancyId,
    String description,
    String executorName,
    String executorDescription) {}
```

- [ ] **Step 2: Update PlanItemSaveRequest — add executor fields**

```java
public record PlanItemSaveRequest(
    UUID caseId,
    String planItemId,
    String bindingName,
    TaskStatus status,
    Instant createdAt,
    TargetType targetType,
    String outputMappingExpression,
    String tenancyId,
    String description,
    String executorName,
    String executorDescription) {}
```

- [ ] **Step 3: Update PlanItemEntity — add JPA columns**

Add two fields:

```java
@Column(name = "executor_name", length = 255)
public String executorName;

@Column(name = "executor_description", length = 1000)
public String executorDescription;
```

- [ ] **Step 4: Update JpaReactivePlanItemStore — map executor fields in save/load**

In the `save()` method, map `request.executorName()` and `request.executorDescription()` to entity fields. In `findByCaseId()`, include executor fields in the `PlanItemRecord` constructor.

- [ ] **Step 5: Update InMemoryPlanItemStore — map executor fields**

Same pattern: `save()` stores executor fields from request, `findByCaseId()` includes them in `PlanItemRecord`.

- [ ] **Step 6: Update PlanItemRestorer — create ExecutorRef from record**

```java
PlanItem restore(PlanItemRecord r) {
    BindingTarget target = r.targetType() == TargetType.HUMAN_TASK
        ? buildHumanTaskTarget(r.outputMappingExpression())
        : null;
    ExecutorRef executor = r.executorName() != null
        ? ExecutorRef.of(r.executorName(), r.executorDescription())
        : null;
    return PlanItem.restore(
        r.planItemId(), r.bindingName(), executor, target, r.status(), r.createdAt(), r.description());
}
```

- [ ] **Step 7: Update all PlanItemSaveRequest construction sites**

Search for `new PlanItemSaveRequest(` — add the two new trailing fields (`executorName`, `executorDescription`). The save request is constructed in `InMemoryPlanItemStore` and `NoOpPlanItemStore`/`NoOpReactivePlanItemStore` (if they build requests). Also in the blackboard save path.

- [ ] **Step 8: Update contract tests**

Update `PlanItemStoreContractTest` and `ReactivePlanItemStoreContractTest` to include executor fields in save requests and assert round-trip preservation:

```java
// In the contract test save request builder:
new PlanItemSaveRequest(caseId, planItemId, "binding-a", TaskStatus.RUNNING,
    Instant.now(), TargetType.CAPABILITY, null, "test-tenant", "desc",
    "worker-a", "worker description")
```

Assert in find assertions:
```java
assertThat(record.executorName()).isEqualTo("worker-a");
assertThat(record.executorDescription()).isEqualTo("worker description");
```

- [ ] **Step 9: Build and run tests**

```bash
mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl common,persistence-memory,blackboard -q
```

- [ ] **Step 10: Commit**

```
feat(#700): thread ExecutorRef through persistence — PlanItemRecord, PlanItemEntity, stores
```

---

### Task 5: Full build verification and file deferred issues

**Files:**
- No new files

- [ ] **Step 1: Full project compile**

```bash
mvn compile -Dmaven.test.skip=true -q
```

Fix any remaining compilation errors.

- [ ] **Step 2: Run all unit tests across affected modules**

```bash
mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api,common,blackboard,persistence-memory,ledger,engine-ai,runtime -q
```

- [ ] **Step 3: File deferred issues**

Create GitHub issues per the spec's "Issues to file" section:

| Repo | Title |
|------|-------|
| casehubio/blocks | `AgentRef extends ExecutorRef` — each variant implements `name()`/`description()` |
| casehubio/blocks | `PlannedTask implements TaskDescriptor` — gains `id`, `createdAt`; `status()` returns PENDING |
| casehubio/blocks | `SubTaskStatus` → `TaskStatus` — replace conversation SubTaskStatus |
| casehubio/engine | Event/handler `ExecutorRef` migration — WorkerScheduleEvent, QuartzWorkerExecutionJob, EventLog |

- [ ] **Step 4: Commit**

```
feat(#700): verify full build, file deferred tracking issues

Refs #700
```
