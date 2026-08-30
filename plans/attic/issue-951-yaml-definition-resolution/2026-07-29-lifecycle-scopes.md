# Lifecycle Scopes Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #237 — feat: long-lived workers with lifecycle scopes (CASE / STAGE / BINDING)
**Issue group:** #237

**Goal:** Add first-class lifecycle scoping to workers — BINDING (current default), COMPOUND (lives for compound lifetime), and CASE (lives for case lifetime) — with persistent and reinvoked execution modes.

**Architecture:** Scope is declared on the Binding (the dispatch control point). A `ScopedWorkerRegistry` tracks active sessions per case. Scoped workers get one long-lived PlanItem that stays RUNNING for the scope's duration. Compound completion excludes COMPANION workers (sidecar pattern). Foundation-tier types (`WorkerOutcome.Completed`, `WorkerFunction.Persistent`, `PersistentScope`) are added to `casehubio/worker`.

**Tech Stack:** Java 21, Quarkus 3.32, virtual threads, `java.util.concurrent` (`BlockingQueue`, `ConcurrentHashMap`, `AtomicReference`), Vert.x event bus.

## Global Constraints

- Pre-release: breaking changes are free. Fix the design, never protect callers.
- All new enums/records default to current behavior when null (backward compatible).
- `casehubio/worker` changes must be installed as SNAPSHOT before engine compilation.
- IntelliJ MCP mandatory for all .java file operations. Open workspace with both repos.
- Cross-repo protocol: verify source repo before changing foundation-tier types (PP-20260722-60e519).
- TDD: write failing test first, then minimal implementation.

---

### Task 1: Foundation types in `casehubio/worker`

**Files:**
- Modify: `/Users/mdproctor/claude/casehub/worker/api/src/main/java/io/casehub/worker/api/WorkerOutcome.java`
- Modify: `/Users/mdproctor/claude/casehub/worker/api/src/main/java/io/casehub/worker/api/WorkerFunction.java`
- Modify: `/Users/mdproctor/claude/casehub/worker/api/src/main/java/io/casehub/worker/api/WorkerResult.java`
- Modify: `/Users/mdproctor/claude/casehub/worker/api/src/main/java/io/casehub/worker/api/WorkerScope.java`
- Create: `/Users/mdproctor/claude/casehub/worker/api/src/main/java/io/casehub/worker/api/PersistentScope.java`
- Create: `/Users/mdproctor/claude/casehub/worker/api/src/main/java/io/casehub/worker/api/ScopeTerminatedException.java`
- Test: `/Users/mdproctor/claude/casehub/worker/api/src/test/java/io/casehub/worker/api/WorkerOutcomeCompletedTest.java`
- Test: `/Users/mdproctor/claude/casehub/worker/api/src/test/java/io/casehub/worker/api/WorkerFunctionPersistentTest.java`
- Test: `/Users/mdproctor/claude/casehub/worker/api/src/test/java/io/casehub/worker/api/WorkerResultCompletedTest.java`

**Interfaces:**
- Consumes: nothing (foundation tier — no upstream)
- Produces:
  - `WorkerOutcome.Completed<R>()` — new permit in sealed hierarchy
  - `WorkerResult.completed(R output)` — factory returning `new WorkerResult<>(output, new Completed<>())`
  - `WorkerFunction.Persistent<T>(Class<T> inputType, Consumer<PersistentScope<T>> handler)` — new variant
  - `PersistentScope<T> extends WorkerScope` — `T nextEvent()`, `void emit(Map<String, Object> output)`
  - `ScopeTerminatedException extends RuntimeException` — thrown by `nextEvent()` on shutdown
  - `WorkerScope.accumulatedState()` — returns `Map<String, Object>`, default `Map.of()`

- [ ] **Step 1: Open workspace with worker repo**

```bash
ide_open_workspace({"modules": ["/Users/mdproctor/claude/casehub/worker", "/Users/mdproctor/claude/casehub/engine"]})
```

Wait for indexing to complete.

- [ ] **Step 2: Write failing test for `WorkerOutcome.Completed`**

Create test file `WorkerOutcomeCompletedTest.java`:

```java
package io.casehub.worker.api;

import static org.assertj.core.api.Assertions.assertThat;
import org.junit.jupiter.api.Test;

class WorkerOutcomeCompletedTest {

    @Test
    void completed_is_a_worker_outcome() {
        WorkerOutcome<String> outcome = new WorkerOutcome.Completed<>();
        assertThat(outcome).isInstanceOf(WorkerOutcome.class);
    }

    @Test
    void completed_factory_method() {
        WorkerOutcome<String> outcome = WorkerOutcome.completed();
        assertThat(outcome).isInstanceOf(WorkerOutcome.Completed.class);
    }

    @Test
    void exhaustive_switch_compiles() {
        WorkerOutcome<String> outcome = WorkerOutcome.completed();
        String result = switch (outcome) {
            case WorkerOutcome.Success<?> s -> "success";
            case WorkerOutcome.Declined<?> d -> "declined";
            case WorkerOutcome.Failed<?> f -> "failed";
            case WorkerOutcome.Expired<?> e -> "expired";
            case WorkerOutcome.Completed<?> c -> "completed";
        };
        assertThat(result).isEqualTo("completed");
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `mvn test -pl api -Dtest=WorkerOutcomeCompletedTest -f /Users/mdproctor/claude/casehub/worker/pom.xml`
Expected: compilation failure — `Completed` not found

- [ ] **Step 4: Implement `WorkerOutcome.Completed` and factory**

Add to `WorkerOutcome.java`:
```java
static <R> WorkerOutcome<R> completed() { return new Completed<>(); }

record Completed<R>() implements WorkerOutcome<R> {}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn test -pl api -Dtest=WorkerOutcomeCompletedTest -f /Users/mdproctor/claude/casehub/worker/pom.xml`
Expected: PASS

- [ ] **Step 6: Write failing test for `WorkerResult.completed()`**

Create `WorkerResultCompletedTest.java`:

```java
package io.casehub.worker.api;

import static org.assertj.core.api.Assertions.assertThat;
import org.junit.jupiter.api.Test;
import java.util.Map;

class WorkerResultCompletedTest {

    @Test
    void completed_factory_creates_completed_outcome() {
        WorkerResult<Map<String, Object>> result = WorkerResult.completed(Map.of("k", "v"));
        assertThat(result.outcome()).isInstanceOf(WorkerOutcome.Completed.class);
        assertThat(result.output()).containsEntry("k", "v");
    }

    @Test
    void completed_factory_with_null_output() {
        WorkerResult<Map<String, Object>> result = WorkerResult.completed(null);
        assertThat(result.outcome()).isInstanceOf(WorkerOutcome.Completed.class);
        assertThat(result.output()).isNull();
    }
}
```

- [ ] **Step 7: Implement `WorkerResult.completed()`**

Add to `WorkerResult.java`:
```java
public static <R> WorkerResult<R> completed(R output) {
    return new WorkerResult<>(output, new WorkerOutcome.Completed<>());
}
```

- [ ] **Step 8: Run test — verify pass**

Run: `mvn test -pl api -Dtest=WorkerResultCompletedTest -f /Users/mdproctor/claude/casehub/worker/pom.xml`

- [ ] **Step 9: Create `ScopeTerminatedException`**

```java
package io.casehub.worker.api;

public class ScopeTerminatedException extends RuntimeException {
    public ScopeTerminatedException() {
        super("Scope terminated — engine-initiated shutdown");
    }
}
```

- [ ] **Step 10: Create `PersistentScope` interface**

```java
package io.casehub.worker.api;

import java.util.Map;

public interface PersistentScope<T> extends WorkerScope {
    T nextEvent() throws ScopeTerminatedException;
    void emit(Map<String, Object> output);
}
```

- [ ] **Step 11: Write failing test for `WorkerFunction.Persistent`**

Create `WorkerFunctionPersistentTest.java`:

```java
package io.casehub.worker.api;

import static org.assertj.core.api.Assertions.assertThat;
import org.junit.jupiter.api.Test;
import java.util.Map;
import java.util.function.Consumer;

class WorkerFunctionPersistentTest {

    @Test
    void persistent_captures_input_type() {
        WorkerFunction.Persistent<Map> fn = new WorkerFunction.Persistent<>(
            Map.class, scope -> {});
        assertThat(fn.inputType()).isEqualTo(Map.class);
    }

    @Test
    void persistent_output_type_is_void() {
        WorkerFunction.Persistent<String> fn = new WorkerFunction.Persistent<>(
            String.class, scope -> {});
        assertThat(fn.outputType()).isEqualTo(Void.class);
    }

    @Test
    void persistent_is_worker_function() {
        WorkerFunction<String, Void> fn = new WorkerFunction.Persistent<>(
            String.class, scope -> {});
        assertThat(fn).isInstanceOf(WorkerFunction.class);
    }
}
```

- [ ] **Step 12: Implement `WorkerFunction.Persistent`**

Add to `WorkerFunction.java`:
```java
record Persistent<T>(Class<T> inputType,
                     java.util.function.Consumer<PersistentScope<T>> handler)
        implements WorkerFunction<T, Void> {
    public Persistent {
        java.util.Objects.requireNonNull(inputType, "inputType must not be null");
        java.util.Objects.requireNonNull(handler, "handler must not be null");
    }
    @Override
    public Class<Void> outputType() { return Void.class; }
}
```

- [ ] **Step 13: Add `accumulatedState()` to `WorkerScope`**

Add default method to `WorkerScope.java`:
```java
default java.util.Map<String, Object> accumulatedState() {
    return java.util.Map.of();
}
```

- [ ] **Step 14: Run all worker-api tests**

Run: `mvn test -pl api -f /Users/mdproctor/claude/casehub/worker/pom.xml`
Expected: all pass (including existing tests — verify no breakage from sealed hierarchy change)

- [ ] **Step 15: Check for exhaustive switch breakage across repos**

Use `ide_find_references` on `WorkerOutcome.Success` to find all switch/instanceof sites. List them — they will need a `Completed` case in Task 7 (engine-side cascading changes).

- [ ] **Step 16: Install worker SNAPSHOT**

Run: `mvn install -DskipTests -q -f /Users/mdproctor/claude/casehub/worker/pom.xml`

- [ ] **Step 17: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/worker add api/src/main/java/io/casehub/worker/api/ api/src/test/java/io/casehub/worker/api/
git -C /Users/mdproctor/claude/casehub/worker commit -m "feat(engine#237): WorkerOutcome.Completed, WorkerFunction.Persistent, PersistentScope, accumulatedState()

Adds lifecycle scope foundation types:
- WorkerOutcome.Completed — new sealed permit for scoped worker completion
- WorkerResult.completed(output) — factory method
- WorkerFunction.Persistent<T> — long-running virtual thread variant
- PersistentScope<T> — mailbox-backed scope with nextEvent()/emit()
- ScopeTerminatedException — unchecked, thrown on engine shutdown
- WorkerScope.accumulatedState() — default Map.of(), overridden for REINVOKED

Breaking: exhaustive switches on WorkerOutcome need a Completed case.

Refs casehubio/engine#237

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 2: API types and Binding extensions (`engine-api`)

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/LifecycleScope.java`
- Create: `api/src/main/java/io/casehub/api/model/Participation.java`
- Create: `api/src/main/java/io/casehub/api/model/ExecutionMode.java`
- Create: `api/src/main/java/io/casehub/api/model/ScopeActivatedTrigger.java`
- Modify: `api/src/main/java/io/casehub/api/model/Binding.java`
- Test: `api/src/test/java/io/casehub/api/model/LifecycleScopeValidationTest.java`

**Interfaces:**
- Consumes: `Trigger` interface (existing), `CapabilityTarget` (existing)
- Produces:
  - `LifecycleScope.BINDING | COMPOUND | CASE`
  - `Participation.PARTICIPANT | COMPANION`
  - `ExecutionMode.TRANSIENT | PERSISTENT | REINVOKED`
  - `ScopeActivatedTrigger implements Trigger`
  - `Binding.lifecycleScope()`, `Binding.participation()`, `Binding.executionMode()`
  - Builder: `.lifecycleScope()`, `.participation()`, `.executionMode()`
  - Validation rules in `Binding.Builder.build()`

- [ ] **Step 1: Create the three enums**

`LifecycleScope.java`:
```java
package io.casehub.api.model;

public enum LifecycleScope {
    BINDING,
    COMPOUND,
    CASE
}
```

`Participation.java`:
```java
package io.casehub.api.model;

public enum Participation {
    PARTICIPANT,
    COMPANION
}
```

`ExecutionMode.java`:
```java
package io.casehub.api.model;

public enum ExecutionMode {
    TRANSIENT,
    PERSISTENT,
    REINVOKED
}
```

- [ ] **Step 2: Create `ScopeActivatedTrigger`**

```java
package io.casehub.api.model;

public record ScopeActivatedTrigger() implements Trigger {}
```

- [ ] **Step 3: Write failing tests for Binding validation**

Create `LifecycleScopeValidationTest.java`:

```java
package io.casehub.api.model;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import org.junit.jupiter.api.Test;

class LifecycleScopeValidationTest {

    private Capability cap() {
        return Capability.builder().name("test").build();
    }

    @Test
    void binding_defaults_to_binding_participant_transient() {
        Binding b = Binding.builder()
            .name("default")
            .capability(cap())
            .on(new ContextChangeTrigger(".x != null"))
            .build();
        assertThat(b.lifecycleScope()).isEqualTo(LifecycleScope.BINDING);
        assertThat(b.participation()).isEqualTo(Participation.PARTICIPANT);
        assertThat(b.executionMode()).isEqualTo(ExecutionMode.TRANSIENT);
    }

    @Test
    void binding_scope_rejects_non_transient_execution_mode() {
        assertThatThrownBy(() -> Binding.builder()
            .name("bad")
            .capability(cap())
            .on(new ContextChangeTrigger(".x"))
            .lifecycleScope(LifecycleScope.BINDING)
            .executionMode(ExecutionMode.PERSISTENT)
            .build())
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("BINDING scope requires TRANSIENT");
    }

    @Test
    void companion_requires_compound_or_case_scope() {
        assertThatThrownBy(() -> Binding.builder()
            .name("bad")
            .capability(cap())
            .on(new ContextChangeTrigger(".x"))
            .lifecycleScope(LifecycleScope.BINDING)
            .participation(Participation.COMPANION)
            .build())
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("COMPANION requires COMPOUND or CASE scope");
    }

    @Test
    void scope_activated_trigger_requires_compound_or_case_scope() {
        assertThatThrownBy(() -> Binding.builder()
            .name("bad")
            .capability(cap())
            .on(new ScopeActivatedTrigger())
            .lifecycleScope(LifecycleScope.BINDING)
            .build())
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("ScopeActivatedTrigger requires COMPOUND or CASE scope");
    }

    @Test
    void case_scope_requires_companion() {
        assertThatThrownBy(() -> Binding.builder()
            .name("bad")
            .capability(cap())
            .on(new ScopeActivatedTrigger())
            .lifecycleScope(LifecycleScope.CASE)
            .participation(Participation.PARTICIPANT)
            .build())
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("CASE scope requires COMPANION");
    }

    @Test
    void lifecycle_scope_requires_capability_target() {
        assertThatThrownBy(() -> Binding.builder()
            .name("bad")
            .subCase(SubCase.builder().name("child").build())
            .on(new ContextChangeTrigger(".x"))
            .lifecycleScope(LifecycleScope.COMPOUND)
            .executionMode(ExecutionMode.PERSISTENT)
            .build())
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("requires CapabilityTarget");
    }

    @Test
    void compound_scope_with_persistent_and_companion_is_valid() {
        Binding b = Binding.builder()
            .name("monitor")
            .capability(cap())
            .on(new ScopeActivatedTrigger())
            .lifecycleScope(LifecycleScope.COMPOUND)
            .participation(Participation.COMPANION)
            .executionMode(ExecutionMode.PERSISTENT)
            .build();
        assertThat(b.lifecycleScope()).isEqualTo(LifecycleScope.COMPOUND);
        assertThat(b.participation()).isEqualTo(Participation.COMPANION);
        assertThat(b.executionMode()).isEqualTo(ExecutionMode.PERSISTENT);
    }

    @Test
    void compound_scope_with_reinvoked_and_participant_is_valid() {
        Binding b = Binding.builder()
            .name("analyst")
            .capability(cap())
            .on(new ContextChangeTrigger(".request != null"))
            .lifecycleScope(LifecycleScope.COMPOUND)
            .participation(Participation.PARTICIPANT)
            .executionMode(ExecutionMode.REINVOKED)
            .build();
        assertThat(b.lifecycleScope()).isEqualTo(LifecycleScope.COMPOUND);
        assertThat(b.executionMode()).isEqualTo(ExecutionMode.REINVOKED);
    }
}
```

- [ ] **Step 4: Run tests — verify failures**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=LifecycleScopeValidationTest`
Expected: compilation failures — fields and validation not yet on Binding

- [ ] **Step 5: Add fields and builder to `Binding`**

Add three fields to `Binding`:
```java
private LifecycleScope lifecycleScope;
private Participation participation;
private ExecutionMode executionMode;
```

Add getters:
```java
public LifecycleScope lifecycleScope() {
    return lifecycleScope != null ? lifecycleScope : LifecycleScope.BINDING;
}
public Participation participation() {
    return participation != null ? participation : Participation.PARTICIPANT;
}
public ExecutionMode executionMode() {
    return executionMode != null ? executionMode : ExecutionMode.TRANSIENT;
}
```

Add setter methods (same pattern as `setOutcomePolicy`):
```java
public void setLifecycleScope(LifecycleScope lifecycleScope) { this.lifecycleScope = lifecycleScope; }
public void setParticipation(Participation participation) { this.participation = participation; }
public void setExecutionMode(ExecutionMode executionMode) { this.executionMode = executionMode; }
```

Add Builder fields and methods:
```java
private LifecycleScope lifecycleScope;
private Participation participation;
private ExecutionMode executionMode;

public Builder lifecycleScope(LifecycleScope lifecycleScope) {
    this.lifecycleScope = lifecycleScope; return this;
}
public Builder participation(Participation participation) {
    this.participation = participation; return this;
}
public Builder executionMode(ExecutionMode executionMode) {
    this.executionMode = executionMode; return this;
}
```

Add validation in `build()`:
```java
LifecycleScope ls = this.lifecycleScope != null ? this.lifecycleScope : LifecycleScope.BINDING;
ExecutionMode em = this.executionMode != null ? this.executionMode : ExecutionMode.TRANSIENT;
Participation p = this.participation != null ? this.participation : Participation.PARTICIPANT;

if (ls == LifecycleScope.BINDING && em != ExecutionMode.TRANSIENT) {
    throw new IllegalArgumentException("BINDING scope requires TRANSIENT execution mode");
}
if (p == Participation.COMPANION && ls == LifecycleScope.BINDING) {
    throw new IllegalArgumentException("COMPANION requires COMPOUND or CASE scope");
}
if (on instanceof ScopeActivatedTrigger && ls == LifecycleScope.BINDING) {
    throw new IllegalArgumentException("ScopeActivatedTrigger requires COMPOUND or CASE scope");
}
if (ls == LifecycleScope.CASE && p != Participation.COMPANION) {
    throw new IllegalArgumentException("CASE scope requires COMPANION participation");
}
if (ls != LifecycleScope.BINDING && !(target instanceof CapabilityTarget)) {
    throw new IllegalArgumentException("Lifecycle scope " + ls + " requires CapabilityTarget");
}
```

Wire fields into the constructed Binding at the end of `build()`.

- [ ] **Step 6: Run tests — verify pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=LifecycleScopeValidationTest`
Expected: all pass

- [ ] **Step 7: Verify no existing Binding tests broken**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest="BindingTest,BindingProducedKeysTest"`
Expected: all pass

- [ ] **Step 8: Commit**

```bash
git add api/src/main/java/io/casehub/api/model/LifecycleScope.java \
  api/src/main/java/io/casehub/api/model/Participation.java \
  api/src/main/java/io/casehub/api/model/ExecutionMode.java \
  api/src/main/java/io/casehub/api/model/ScopeActivatedTrigger.java \
  api/src/main/java/io/casehub/api/model/Binding.java \
  api/src/test/java/io/casehub/api/model/LifecycleScopeValidationTest.java
git commit -m "feat(#237): LifecycleScope, Participation, ExecutionMode enums + Binding extensions

Refs casehubio/engine#237

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 3: Compound.scopedBindings → `Map<String, Participation>`

**Files:**
- Modify: `planning/src/main/java/io/casehub/engine/planning/plan/PlanItemDefinition.java` — Compound record + Builder
- Modify: `planning/src/main/java/io/casehub/engine/planning/plan/DefaultCasePlanModel.java` — evaluateCompletion
- Modify: `planning/src/test/java/io/casehub/engine/planning/plan/CompoundPlanModelTest.java`
- Modify: `planning/src/test/java/io/casehub/engine/planning/plan/PlanItemDefinitionTest.java`

**Interfaces:**
- Consumes: `Participation` enum (Task 2)
- Produces:
  - `Compound.scopedBindings()` returns `Map<String, Participation>` (was `Set<String>`)
  - `Compound.Builder.binding(String)` defaults to `PARTICIPANT`
  - `Compound.Builder.binding(String, Participation)` explicit participation
  - `evaluateCompletion()` excludes COMPANION bindings from completion count

- [ ] **Step 1: Write failing test for COMPANION exclusion from completion**

Add to `CompoundPlanModelTest.java`:

```java
@Test
void evaluateCompletion_companion_binding_excluded_from_completion() {
    var compound = PlanItemDefinition.Compound.builder("stage")
        .id("comp-1")
        .binding("required-binding")
        .binding("companion-binding", Participation.COMPANION)
        .build();
    var model = model();
    model.registerDefinition(compound);

    var piRequired = PlanItem.create("required-binding", ExecutorRef.of("w"), 0);
    model.addPlanItem(piRequired);
    piRequired.markRunning();
    piRequired.markCompleted();

    assertThat(model.evaluateCompletion("comp-1"))
        .as("COMPANION binding should not block completion")
        .isTrue();
}

@Test
void evaluateCompletion_participant_binding_blocks_completion() {
    var compound = PlanItemDefinition.Compound.builder("stage")
        .id("comp-1")
        .binding("binding-a")
        .binding("binding-b", Participation.PARTICIPANT)
        .build();
    var model = model();
    model.registerDefinition(compound);

    var piA = PlanItem.create("binding-a", ExecutorRef.of("w"), 0);
    model.addPlanItem(piA);
    piA.markRunning();
    piA.markCompleted();

    assertThat(model.evaluateCompletion("comp-1"))
        .as("PARTICIPANT binding-b not yet dispatched — blocks completion")
        .isFalse();
}
```

- [ ] **Step 2: Run test — verify failure**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl planning -Dtest=CompoundPlanModelTest`
Expected: compilation failure — `binding(String, Participation)` not found

- [ ] **Step 3: Change `Compound.scopedBindings` from `Set<String>` to `Map<String, Participation>`**

In `PlanItemDefinition.java`:

Change `Compound` record field:
```java
// was: java.util.Set<String> scopedBindings
java.util.Map<String, io.casehub.api.model.Participation> scopedBindings
```

Update compact constructor:
```java
scopedBindings = scopedBindings != null ? java.util.Map.copyOf(scopedBindings) : java.util.Map.of();
```

Update Builder:
```java
// was: private final java.util.LinkedHashSet<String> scopedBindings = new java.util.LinkedHashSet<>();
private final java.util.LinkedHashMap<String, io.casehub.api.model.Participation> scopedBindings = new java.util.LinkedHashMap<>();

public Builder binding(String bindingName) {
    return binding(bindingName, io.casehub.api.model.Participation.PARTICIPANT);
}

public Builder binding(String bindingName, io.casehub.api.model.Participation participation) {
    java.util.Objects.requireNonNull(bindingName, "bindingName required");
    java.util.Objects.requireNonNull(participation, "participation required");
    this.scopedBindings.put(bindingName, participation);
    return this;
}
```

- [ ] **Step 4: Fix `DefaultCasePlanModel.registerDefinition` — index map keys, not set elements**

Find where `scopedBindings` is iterated during registration. Change:
```java
// was: for (String binding : compound.scopedBindings()) { ... }
for (String binding : compound.scopedBindings().keySet()) { ... }
```

- [ ] **Step 5: Fix `evaluateCompletion` — filter COMPANION bindings**

In `DefaultCasePlanModel.evaluateCompletion()`, change the scoped binding terminal count to exclude COMPANIONs:

```java
// When counting scoped bindings toward completion, only count PARTICIPANT entries
Map<String, Participation> scoped = compound.scopedBindings();
long scopedParticipantCount = scoped.values().stream()
    .filter(p -> p == Participation.PARTICIPANT)
    .count();

// Only check terminal status for PARTICIPANT scoped bindings
long scopedTerminal = scoped.entrySet().stream()
    .filter(e -> e.getValue() == Participation.PARTICIPANT)
    .map(e -> /* look up PlanItem and check terminal */ ...)
    .filter(TaskStatus::isTerminal)
    .count();
```

- [ ] **Step 6: Fix compilation errors in existing tests**

Update test helpers that construct `Compound` with `Set.of()` for scopedBindings — change to `Map.of()`. Existing `binding(String)` builder calls still work (defaults to PARTICIPANT).

- [ ] **Step 7: Run tests — verify pass**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl planning`
Expected: all pass

- [ ] **Step 8: Check cross-module references to `scopedBindings`**

Use `ide_find_references` on `scopedBindings` to find all call sites. Fix any that call `.contains()` or iterate as `Set`.

- [ ] **Step 9: Commit**

```bash
git commit -m "feat(#237): Compound.scopedBindings carries Participation metadata

Changes scopedBindings from Set<String> to Map<String, Participation>.
COMPANION bindings excluded from evaluateCompletion.
Builder retains binding(String) overload defaulting to PARTICIPANT.

Refs casehubio/engine#237

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 4: ScopedWorkerRegistry and dispatch interception

**Files:**
- Create: `runtime/src/main/java/io/casehub/engine/internal/worker/scope/ContextEvent.java`
- Create: `runtime/src/main/java/io/casehub/engine/internal/worker/scope/ScopedWorkerSession.java`
- Create: `runtime/src/main/java/io/casehub/engine/internal/worker/scope/ScopedWorkerRegistry.java`
- Modify: `common/src/main/java/io/casehub/engine/common/internal/event/WorkerScheduleEvent.java` — add lifecycleScope, executionMode fields
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java` — registry check in publishWorkerSchedule
- Test: `runtime/src/test/java/io/casehub/engine/internal/worker/scope/ScopedWorkerRegistryTest.java`
- Test: `runtime/src/test/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandlerScopeTest.java`

**Interfaces:**
- Consumes: `LifecycleScope`, `Participation`, `ExecutionMode` (Task 2), `Binding` extensions (Task 2)
- Produces:
  - `ScopedWorkerRegistry.get(UUID caseId, String bindingName) → Optional<ScopedWorkerSession>`
  - `ScopedWorkerRegistry.register(ScopeKey, ScopedWorkerSession)`
  - `ScopedWorkerRegistry.terminateByScope(UUID caseId, String compoundId)`
  - `ScopedWorkerRegistry.terminateByCase(UUID caseId)`
  - `WorkerScheduleEvent` gains `lifecycleScope` and `executionMode` (nullable — null = TRANSIENT)

- [ ] **Step 1: Create `ContextEvent`**

```java
package io.casehub.engine.internal.worker.scope;

import com.fasterxml.jackson.databind.JsonNode;
import java.util.Map;

public record ContextEvent(JsonNode contextSnapshot, Map<String, Object> changeMetadata) {
    public static final ContextEvent SHUTDOWN = new ContextEvent(null, null);

    public boolean isShutdown() {
        return this == SHUTDOWN;
    }
}
```

- [ ] **Step 2: Create `ScopedWorkerSession`**

```java
package io.casehub.engine.internal.worker.scope;

import io.casehub.api.model.LifecycleScope;
import io.casehub.api.model.Participation;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.Future;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.atomic.AtomicReference;

public sealed interface ScopedWorkerSession
    permits ScopedWorkerSession.Persistent, ScopedWorkerSession.Reinvoked {

    String bindingName();
    UUID caseId();
    String planItemId();
    LifecycleScope scope();
    Participation participation();

    record Persistent(
        String bindingName, UUID caseId, String planItemId,
        LifecycleScope scope, Participation participation,
        BlockingQueue<ContextEvent> mailbox,
        Future<?> workerThread
    ) implements ScopedWorkerSession {}

    record Reinvoked(
        String bindingName, UUID caseId, String planItemId,
        LifecycleScope scope, Participation participation,
        AtomicReference<Map<String, Object>> accumulatedState
    ) implements ScopedWorkerSession {}
}
```

- [ ] **Step 3: Write failing test for `ScopedWorkerRegistry`**

Create `ScopedWorkerRegistryTest.java`:

```java
package io.casehub.engine.internal.worker.scope;

import static org.assertj.core.api.Assertions.assertThat;
import io.casehub.api.model.LifecycleScope;
import io.casehub.api.model.Participation;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.atomic.AtomicReference;
import org.junit.jupiter.api.Test;

class ScopedWorkerRegistryTest {

    private final ScopedWorkerRegistry registry = new ScopedWorkerRegistry();

    @Test
    void register_and_get() {
        UUID caseId = UUID.randomUUID();
        var session = new ScopedWorkerSession.Reinvoked(
            "binding-a", caseId, "pi-1",
            LifecycleScope.COMPOUND, Participation.PARTICIPANT,
            new AtomicReference<>(Map.of()));
        registry.register(new ScopedWorkerRegistry.ScopeKey(caseId, "binding-a"), session);

        assertThat(registry.get(caseId, "binding-a")).contains(session);
    }

    @Test
    void get_returns_empty_when_not_registered() {
        assertThat(registry.get(UUID.randomUUID(), "missing")).isEmpty();
    }

    @Test
    void terminateByCase_removes_all_sessions_for_case() {
        UUID caseId = UUID.randomUUID();
        var s1 = new ScopedWorkerSession.Reinvoked(
            "b1", caseId, "pi-1", LifecycleScope.COMPOUND,
            Participation.PARTICIPANT, new AtomicReference<>(Map.of()));
        var s2 = new ScopedWorkerSession.Reinvoked(
            "b2", caseId, "pi-2", LifecycleScope.CASE,
            Participation.COMPANION, new AtomicReference<>(Map.of()));
        registry.register(new ScopedWorkerRegistry.ScopeKey(caseId, "b1"), s1);
        registry.register(new ScopedWorkerRegistry.ScopeKey(caseId, "b2"), s2);

        registry.terminateByCase(caseId);

        assertThat(registry.get(caseId, "b1")).isEmpty();
        assertThat(registry.get(caseId, "b2")).isEmpty();
    }

    @Test
    void register_replaces_existing_session() {
        UUID caseId = UUID.randomUUID();
        var s1 = new ScopedWorkerSession.Reinvoked(
            "b1", caseId, "pi-1", LifecycleScope.COMPOUND,
            Participation.PARTICIPANT, new AtomicReference<>(Map.of()));
        var s2 = new ScopedWorkerSession.Reinvoked(
            "b1", caseId, "pi-2", LifecycleScope.COMPOUND,
            Participation.PARTICIPANT, new AtomicReference<>(Map.of()));
        var key = new ScopedWorkerRegistry.ScopeKey(caseId, "b1");
        registry.register(key, s1);
        registry.register(key, s2);

        assertThat(registry.get(caseId, "b1")).contains(s2);
    }
}
```

- [ ] **Step 4: Implement `ScopedWorkerRegistry`**

```java
package io.casehub.engine.internal.worker.scope;

import jakarta.enterprise.context.ApplicationScoped;
import java.util.List;
import java.util.Optional;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;

@ApplicationScoped
public class ScopedWorkerRegistry {

    private final ConcurrentHashMap<ScopeKey, ScopedWorkerSession> sessions = new ConcurrentHashMap<>();

    public Optional<ScopedWorkerSession> get(UUID caseId, String bindingName) {
        return Optional.ofNullable(sessions.get(new ScopeKey(caseId, bindingName)));
    }

    public void register(ScopeKey key, ScopedWorkerSession session) {
        ScopedWorkerSession previous = sessions.put(key, session);
        if (previous instanceof ScopedWorkerSession.Persistent p) {
            p.mailbox().offer(ContextEvent.SHUTDOWN);
        }
    }

    public void terminateByCase(UUID caseId) {
        sessions.entrySet().removeIf(e -> {
            if (e.getKey().caseId().equals(caseId)) {
                terminateSession(e.getValue());
                return true;
            }
            return false;
        });
    }

    public void terminateByScope(UUID caseId, String compoundId, java.util.Set<String> ownedBindings) {
        for (String bindingName : ownedBindings) {
            ScopedWorkerSession removed = sessions.remove(new ScopeKey(caseId, bindingName));
            if (removed != null) {
                terminateSession(removed);
            }
        }
    }

    private void terminateSession(ScopedWorkerSession session) {
        if (session instanceof ScopedWorkerSession.Persistent p) {
            p.mailbox().offer(ContextEvent.SHUTDOWN);
        }
    }

    public record ScopeKey(UUID caseId, String bindingName) {}
}
```

- [ ] **Step 5: Run registry tests — verify pass**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=ScopedWorkerRegistryTest`

- [ ] **Step 6: Add lifecycleScope and executionMode to `WorkerScheduleEvent`**

Add two nullable fields to the record:
```java
private final LifecycleScope lifecycleScope;   // null = TRANSIENT
private final ExecutionMode executionMode;     // null = TRANSIENT
```

Add a new full constructor. Existing convenience constructors pass `null` for both.

- [ ] **Step 7: Modify `CaseContextChangedEventHandler.publishWorkerSchedule()` — add registry check**

Before creating a new PlanItem, insert the scoped worker interception:

```java
if (binding.lifecycleScope() != LifecycleScope.BINDING) {
    Optional<ScopedWorkerSession> existing = scopedWorkerRegistry.get(
        caseInstance.getUuid(), binding.getName());
    if (existing.isPresent()) {
        routeToExistingSession(existing.get(), caseInstance);
        return;
    }
}
```

Add `routeToExistingSession()` method:
```java
private void routeToExistingSession(ScopedWorkerSession session, CaseInstance caseInstance) {
    if (session instanceof ScopedWorkerSession.Persistent p) {
        JsonNode snapshot = caseInstance.getCaseContext().layer(ContextLayer.WORKING).asJsonNode();
        p.mailbox().offer(new ContextEvent(snapshot, Map.of()));
    } else if (session instanceof ScopedWorkerSession.Reinvoked r) {
        // Publish WorkerScheduleEvent with executionMode=REINVOKED — Quartz handles re-invocation
        // (Implemented in Task 5)
    }
}
```

Inject `ScopedWorkerRegistry` into the handler constructor.

- [ ] **Step 8: Run full runtime tests to check for breakage**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime`

- [ ] **Step 9: Commit**

```bash
git commit -m "feat(#237): ScopedWorkerRegistry, dispatch interception, WorkerScheduleEvent extensions

Refs casehubio/engine#237

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 5: QuartzWorkerExecutionJob completion suppression

**Files:**
- Modify: `scheduler-quartz/src/main/java/io/casehub/engine/scheduler/quartz/QuartzWorkerExecutionJob.java`
- Test: `scheduler-quartz/src/test/java/io/casehub/engine/scheduler/quartz/QuartzWorkerExecutionJobScopeTest.java`

**Interfaces:**
- Consumes: `WorkerScheduleEvent.lifecycleScope()`, `WorkerScheduleEvent.executionMode()` (Task 4), `ScopedWorkerRegistry` (Task 4), `WorkerFunction.Persistent` (Task 1), `WorkerOutcome.Completed` (Task 1)
- Produces: For non-TRANSIENT executionMode, suppresses `WorkflowExecutionCompleted` on `Success`. Publishes it only on `Completed`.

- [ ] **Step 1: Write failing test for completion suppression**

```java
@Test
void reinvoked_success_does_not_publish_completion_event() {
    // Setup: WorkerScheduleEvent with executionMode=REINVOKED
    // Worker returns WorkerResult.of(output) (Success)
    // Assert: WorkflowExecutionCompleted NOT published
    // Assert: PlanItem stays RUNNING
}

@Test
void reinvoked_completed_publishes_completion_event() {
    // Setup: WorkerScheduleEvent with executionMode=REINVOKED
    // Worker returns WorkerResult.completed(output) (Completed)
    // Assert: WorkflowExecutionCompleted published with Completed outcome
}

@Test
void transient_success_publishes_completion_event_as_before() {
    // Setup: WorkerScheduleEvent with executionMode=null (TRANSIENT)
    // Worker returns WorkerResult.of(output) (Success)
    // Assert: WorkflowExecutionCompleted published normally (no behavior change)
}
```

- [ ] **Step 2: Implement completion suppression in `QuartzWorkerExecutionJob`**

In the `onSuccess` path (after worker function returns), add:

```java
ExecutionMode mode = /* read from EventLog metadata or job data */;
if (mode != null && mode != ExecutionMode.TRANSIENT) {
    if (result.outcome() instanceof WorkerOutcome.Completed) {
        // Publish WorkflowExecutionCompleted — PlanItem will be completed
        eventBus.publish(EventBusAddresses.WORKER_EXECUTION_FINISHED,
            new WorkflowExecutionCompleted(caseInstance, worker, idempotency,
                output, bindingName, result.outcome()));
    } else {
        // Success on scoped worker — apply output but suppress completion
        // Output is applied via signal to case context
        // PlanItem stays RUNNING
        LOG.debugf("Scoped worker %s returned Success — output applied, PlanItem stays RUNNING", bindingName);
    }
    return;
}
// Existing TRANSIENT path — unchanged
```

- [ ] **Step 3: Store executionMode and lifecycleScope in EventLog metadata**

In `WorkerScheduleEventHandler`, when creating the EventLog entry, add metadata:
```java
if (event.lifecycleScope() != null) {
    metadata.put("lifecycleScope", event.lifecycleScope().name());
}
if (event.executionMode() != null) {
    metadata.put("executionMode", event.executionMode().name());
}
```

- [ ] **Step 4: Run tests — verify pass**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl scheduler-quartz`

- [ ] **Step 5: Commit**

```bash
git commit -m "feat(#237): completion suppression for scoped workers in QuartzWorkerExecutionJob

Non-TRANSIENT workers: Success outcome applies output but does not
publish WorkflowExecutionCompleted. Only Completed outcome triggers
PlanItem completion. Backward compatible — null executionMode = TRANSIENT.

Refs casehubio/engine#237

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 6: Lifecycle event handlers

**Files:**
- Create: `planning/src/main/java/io/casehub/engine/planning/event/CompoundActivatedEvent.java`
- Modify: `planning/src/main/java/io/casehub/engine/planning/control/CompoundLifecycleEvaluator.java` — publish activated event
- Create: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CompoundActivatedEventHandler.java`
- Create: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/ScopedWorkerTerminationHandler.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseStatusChangedHandler.java` — inject registry
- Test: `runtime/src/test/java/io/casehub/engine/internal/engine/handler/CompoundActivatedEventHandlerTest.java`
- Test: `runtime/src/test/java/io/casehub/engine/internal/engine/handler/ScopedWorkerTerminationHandlerTest.java`

**Interfaces:**
- Consumes: `ScopedWorkerRegistry` (Task 4), `CompoundCompletedEvent` (existing), `CaseDefinition.getBindings()` (existing)
- Produces:
  - `CompoundActivatedEvent(caseId, tenancyId, compoundId, compoundName)` — published on compound PENDING→RUNNING
  - `CompoundActivatedEventHandler` — dispatches scope-activated bindings
  - `ScopedWorkerTerminationHandler` — terminates companions on compound completion
  - `CaseStatusChangedHandler` calls `terminateByCase()` on terminal state

- [ ] **Step 1: Create `CompoundActivatedEvent`**

```java
package io.casehub.engine.planning.event;

import java.util.UUID;

public record CompoundActivatedEvent(UUID caseId, String tenancyId, String compoundId, String compoundName) {}
```

Add `COMPOUND_ACTIVATED` constant to `BlackboardEventBusAddresses`.

- [ ] **Step 2: Publish `CompoundActivatedEvent` from `CompoundLifecycleEvaluator`**

In `activatePendingCompounds()`, after a successful transition to RUNNING, publish the event:

```java
if (plan.tryDefinitionTransition(compound.id(), TaskStatus.PENDING, TaskStatus.RUNNING)) {
    LOG.debugf("Compound '%s' activated for case %s", compound.name(), ctx.caseId());
    eventBus.publish(BlackboardEventBusAddresses.COMPOUND_ACTIVATED,
        new CompoundActivatedEvent(ctx.caseId(), ctx.tenancyId(), compound.id(), compound.name()));
}
```

Inject `EventBus` into `CompoundLifecycleEvaluator`.

- [ ] **Step 3: Write failing test for `CompoundActivatedEventHandler`**

Test that when a compound activates, scope-activated bindings are dispatched as `WorkerScheduleEvent`.

- [ ] **Step 4: Implement `CompoundActivatedEventHandler`**

```java
@ApplicationScoped
public class CompoundActivatedEventHandler {
    // Inject: CaseDefinitionRegistry, CaseInstanceCache, EventBus
    // @ConsumeEvent(COMPOUND_ACTIVATED, blocking = true)
    // For each binding in the definition with ScopeActivatedTrigger
    //   that is owned by the activated compound:
    //   evaluate when condition, publish WorkerScheduleEvent
}
```

- [ ] **Step 5: Write failing test for `ScopedWorkerTerminationHandler`**

Test that when `CompoundCompletedEvent` fires, COMPANION sessions are terminated.

- [ ] **Step 6: Implement `ScopedWorkerTerminationHandler`**

```java
@ApplicationScoped
public class ScopedWorkerTerminationHandler {
    // Inject: ScopedWorkerRegistry, CaseDefinitionRegistry
    // @ConsumeEvent(COMPOUND_COMPLETED, blocking = true)
    // Look up compound's scopedBindings from definition
    // Call registry.terminateByScope(caseId, compoundId, bindingNames)
}
```

- [ ] **Step 7: Add `terminateByCase` call to `CaseStatusChangedHandler`**

Inject `ScopedWorkerRegistry`. In the terminal-state block, add:
```java
scopedWorkerRegistry.terminateByCase(caseInstance.getUuid());
```

- [ ] **Step 8: Run tests**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest="CompoundActivatedEventHandlerTest,ScopedWorkerTerminationHandlerTest"`

- [ ] **Step 9: Commit**

```bash
git commit -m "feat(#237): compound lifecycle event handlers + case termination

CompoundActivatedEvent published on compound activation.
CompoundActivatedEventHandler dispatches scope-activated bindings.
ScopedWorkerTerminationHandler terminates companions on compound completion.
CaseStatusChangedHandler terminates all scoped workers on case terminal state.

Refs casehubio/engine#237

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 7: OutcomeKind.COMPLETED + PlanItemCompletionHandler cascading changes

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/OutcomeKind.java` — add COMPLETED
- Modify: `planning/src/main/java/io/casehub/engine/planning/handler/PlanItemCompletionHandler.java` — handle Completed
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/WorkflowExecutionCompletedHandler.java` — handle Completed in switch
- Fix: all exhaustive switch sites on `WorkerOutcome` identified in Task 1 Step 15

**Interfaces:**
- Consumes: `WorkerOutcome.Completed` (Task 1)
- Produces: `OutcomeKind.COMPLETED`, exhaustive switch coverage across codebase

- [ ] **Step 1: Add `COMPLETED` to `OutcomeKind`**

```java
COMPLETED,  // after EXPIRED, before ESCALATED
```

Update `fromWorkerOutcome()`:
```java
case WorkerOutcome.Completed<?> c -> COMPLETED;
```

Update `isTerminal()`:
```java
// COMPLETED is not terminal (same as SUCCESS — it signals lifecycle done, not failure)
return this != SUCCESS && this != COMPLETED;
```

- [ ] **Step 2: Update `PlanItemCompletionHandler.onWorkerFinished()`**

Change the gate to accept both `Success` and `Completed`:
```java
if (!(event.outcome() instanceof WorkerOutcome.Success)
    && !(event.outcome() instanceof WorkerOutcome.Completed)) {
    return;
}
```

- [ ] **Step 3: Update `WorkflowExecutionCompletedHandler`**

Add `Completed` handling in the switch — treat same as `Success` for output application:
```java
case WorkerOutcome.Completed<?> c -> {
    // Apply output, same as Success path
    // No PlannedAction on Completed (no action gate)
}
```

In `handleSemanticFailure()`:
```java
case WorkerOutcome.Completed<?> c ->
    throw new IllegalStateException("Completed outcome should not reach failure handling");
```

- [ ] **Step 4: Fix all remaining exhaustive switch sites**

Use the list from Task 1 Step 15. Each site needs:
```java
case WorkerOutcome.Completed<?> c -> /* appropriate handling */;
```

- [ ] **Step 5: Run all affected module tests**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api,runtime,planning`

- [ ] **Step 6: Commit**

```bash
git commit -m "feat(#237): OutcomeKind.COMPLETED + exhaustive switch coverage

Adds COMPLETED to OutcomeKind. PlanItemCompletionHandler accepts both
Success and Completed outcomes. All exhaustive switches updated.

Refs casehubio/engine#237

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 8: PlanItem persistence and YAML schema

**Files:**
- Modify: `common/src/main/java/io/casehub/engine/common/internal/model/PlanItemRecord.java` (or equivalent) — add lifecycle_scope
- Modify: `persistence-hibernate/...` (if PlanItemEntity exists) — add column
- Modify: `api/src/main/java/io/casehub/api/model/CaseDefinitionYamlMapper.java` (or wherever YAML parsing lives) — parse lifecycleScope, participation, executionMode, scope-activated trigger
- Test: YAML parsing test

**Interfaces:**
- Consumes: all types from Tasks 1-7
- Produces: `PlanItemRecord.lifecycleScope` field, YAML schema support

- [ ] **Step 1: Find PlanItemRecord and PlanItemEntity**

Use `ide_find_class` for `PlanItemRecord` and `PlanItemEntity`.

- [ ] **Step 2: Add `lifecycleScope` field to PlanItemRecord**

```java
private String lifecycleScope; // nullable — null = BINDING
```

Add to `PlanItemSaveRequest` if separate.

- [ ] **Step 3: Add column to PlanItemEntity (if JPA)**

```java
@Column(name = "lifecycle_scope")
private String lifecycleScope;
```

- [ ] **Step 4: Write failing test for YAML parsing**

```java
@Test
void yaml_parses_lifecycle_scope_on_binding() {
    // YAML with:
    //   lifecycleScope: COMPOUND
    //   participation: COMPANION
    //   executionMode: PERSISTENT
    //   trigger: scope-activated
    // Assert: parsed Binding has correct enum values
}
```

- [ ] **Step 5: Implement YAML parsing in CaseDefinitionYamlMapper**

Parse `lifecycleScope`, `participation`, `executionMode` from binding nodes. `trigger: scope-activated` creates `ScopeActivatedTrigger`.

- [ ] **Step 6: Run tests**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api,common,persistence-hibernate`

- [ ] **Step 7: Commit**

```bash
git commit -m "feat(#237): PlanItem persistence + YAML schema for lifecycle scopes

PlanItemRecord gains lifecycle_scope field. YAML parser supports
lifecycleScope, participation, executionMode on bindings and
scope-activated trigger type.

Refs casehubio/engine#237

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 9: Integration test — full lifecycle round-trip

**Files:**
- Test: `runtime/src/test/java/io/casehub/engine/LifecycleScopeIntegrationTest.java`

**Interfaces:**
- Consumes: all tasks
- Produces: end-to-end validation

- [ ] **Step 1: Write integration test for COMPOUND-scoped COMPANION with PERSISTENT execution**

```java
@QuarkusTest
class LifecycleScopeIntegrationTest {

    @Test
    void compound_scoped_companion_receives_events_and_terminates_on_compound_completion() {
        // 1. Define CaseHub with a Compound containing:
        //    - A normal BINDING-scoped worker (processes .request)
        //    - A COMPOUND-scoped COMPANION PERSISTENT worker (monitors case)
        // 2. Start case with initial context
        // 3. Signal context change — verify companion receives event via mailbox
        // 4. Signal context that satisfies normal worker
        // 5. Normal worker completes → compound completes
        // 6. Verify companion was terminated (session removed from registry)
        // 7. Verify case completed normally (companion did not block)
    }

    @Test
    void compound_scoped_participant_with_reinvoked_blocks_completion() {
        // 1. Define CaseHub with COMPOUND-scoped PARTICIPANT REINVOKED worker
        // 2. Trigger binding — worker invoked, returns Success (stays RUNNING)
        // 3. Trigger again — worker re-invoked with accumulated state
        // 4. Worker returns Completed — PlanItem transitions to COMPLETED
        // 5. Compound completes
    }

    @Test
    void case_scoped_companion_terminates_on_case_completion() {
        // 1. Define CaseHub with CASE-scoped COMPANION
        // 2. Start case — companion activated
        // 3. Complete case via goal
        // 4. Verify companion terminated
    }
}
```

- [ ] **Step 2: Implement tests — iterative red-green cycle**

Each test follows the pattern: define case → start → signal → assert state.

- [ ] **Step 3: Run integration tests**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=LifecycleScopeIntegrationTest`

- [ ] **Step 4: Commit**

```bash
git commit -m "test(#237): lifecycle scope integration tests — full round-trip

Tests COMPOUND-scoped COMPANION/PARTICIPANT and CASE-scoped COMPANION
with both PERSISTENT and REINVOKED execution modes.

Refs casehubio/engine#237

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Task Dependency Graph

```
Task 1 (worker foundation) ─────────────────────────────────────────────┐
Task 2 (api enums + binding) ─────────────────────┐                     │
Task 3 (compound scopedBindings) ─────────────────┤                     │
                                                   ├── Task 4 (registry + dispatch)
                                                   │     │
                                                   │     ├── Task 5 (Quartz suppression)
                                                   │     ├── Task 6 (lifecycle handlers)
                                                   │     └── Task 8 (persistence + YAML)
                                                   │
                                                   └── Task 7 (OutcomeKind cascading)
                                                          │
                                                          └── Task 9 (integration test)
```

Tasks 1, 2, 3 can run in parallel. Tasks 4-8 depend on earlier tasks. Task 9 depends on all.
