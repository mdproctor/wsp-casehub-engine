# Typed Composition + Context Isolation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> executing-plans to implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural editing.
> Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #693 — typed in-process composition for WorkerRuntime and sequence()
**Issue group:** #693, #698

**Goal:** Parameterize WorkerFunction<T,R>/WorkerResult<R>/WorkerOutcome<R>, eliminate all ThreadLocal from the engine, add typed in-process composition, and isolate per-task diagnostic context.

**Architecture:** Foundation types in casehub-worker-api gain a second type parameter R (output type). WorkerScope replaces ThreadLocal — the runtime is passed as an explicit BiFunction parameter. Context isolation namespaces diagnostic state under `_diagnostics.<taskId>` with input projection filtering.

**Tech Stack:** Java 21, Quarkus 3.32, CDI, jackson-databind, casehub-worker-api (tier 1), casehub-engine-api (tier 2)

## Global Constraints

- Pre-release: breaking changes are free. No backward compatibility shims.
- IntelliJ MCP mandatory for all .java operations.
- TDD: write failing test first, then implement, then verify.
- Zero ThreadLocal remaining in engine after this branch.
- Three-level DSL ceremony: Map→Map (no types), T→Map (contextType only), T→R (both declared).
- `WorkerScope` lives in casehub-worker-api (tier 1). `WorkerRuntime extends WorkerScope` stays in casehub-engine-api (tier 2).

---

### Task 1: Foundation types in casehub-worker-api

**This is the atomic foundation.** All three types change together — the code won't compile in any intermediate state. Worker-api must compile and all its tests pass before proceeding.

**Files:**
- Modify: `worker/src/main/java/io/casehub/worker/api/WorkerFunction.java`
- Modify: `worker/src/main/java/io/casehub/worker/api/WorkerResult.java`
- Modify: `worker/src/main/java/io/casehub/worker/api/WorkerOutcome.java`
- Modify: `worker/src/main/java/io/casehub/worker/api/Worker.java` (builder, TypedFunctionBuilder, TypedOutputBuilder)
- Create: `worker/src/main/java/io/casehub/worker/api/WorkerScope.java`
- Delete: `WorkerFunction.Async` (if it exists)
- Modify: All test files in worker module
- Test: `worker/src/test/java/io/casehub/worker/api/`

**Interfaces:**
- Produces: `WorkerFunction<T, R>` with `inputType()` + `outputType()`, `WorkerResult<R>` with top-level `output`, `WorkerOutcome<R>` parameterized, `WorkerScope` interface, `TypedOutputBuilder<T, R>` with `.returning(Class<R>)`

- [ ] **Step 1: Read current WorkerFunction, WorkerResult, WorkerOutcome, Worker**

Read all four files to understand exact current signatures, inner types, and test patterns.

- [ ] **Step 2: Create WorkerScope interface**

```java
package io.casehub.worker.api;

import java.util.Map;
import java.util.UUID;

public interface WorkerScope {
    UUID caseId();
    String taskId();
    <T, R> WorkerResult<R> execute(WorkerFunction<T, R> function, T input);
    WorkerResult<?> execute(String workerName, Map<String, Object> input);
}
```

- [ ] **Step 3: Update WorkerOutcome<R>**

Add type parameter R. Success loses output (moved to WorkerResult). All variants carry phantom R:

```java
public sealed interface WorkerOutcome<R> {
    record Success<R>(PlannedAction plannedAction) implements WorkerOutcome<R> {}
    record Declined<R>(String reason) implements WorkerOutcome<R> {}
    record Failed<R>(String reason) implements WorkerOutcome<R> {}
    record Expired<R>(String reason) implements WorkerOutcome<R> {}
}
```

- [ ] **Step 4: Update WorkerResult<R>**

Add type parameter R. Output becomes top-level record component:

```java
public record WorkerResult<R>(R output, WorkerOutcome<R> outcome) {
    public static <R> WorkerResult<R> of(R output) {
        return new WorkerResult<>(output, new WorkerOutcome.Success<>(null));
    }
    public static <R> WorkerResult<R> of(R output, PlannedAction action) {
        return new WorkerResult<>(output, new WorkerOutcome.Success<>(action));
    }
    public static <R> WorkerResult<R> failed(String reason) {
        return new WorkerResult<>(null, new WorkerOutcome.Failed<>(reason));
    }
    public static <R> WorkerResult<R> failed(String reason, R partialOutput) {
        return new WorkerResult<>(partialOutput, new WorkerOutcome.Failed<>(reason));
    }
    public static <R> WorkerResult<R> declined(String reason) {
        return new WorkerResult<>(null, new WorkerOutcome.Declined<>(reason));
    }
    public static <R> WorkerResult<R> declined(String reason, R partialOutput) {
        return new WorkerResult<>(partialOutput, new WorkerOutcome.Declined<>(reason));
    }
    public static <R> WorkerResult<R> expired(String reason) {
        return new WorkerResult<>(null, new WorkerOutcome.Expired<>(reason));
    }
    public static <R> WorkerResult<R> expired(String reason, R partialOutput) {
        return new WorkerResult<>(partialOutput, new WorkerOutcome.Expired<>(reason));
    }
}
```

- [ ] **Step 5: Update WorkerFunction<T, R>**

Add second type parameter R. Remove Async. Update Sync to use BiFunction with WorkerScope:

```java
public interface WorkerFunction<T, R> {
    WorkerFunction<Void, Void> NONE = new None();

    Class<T> inputType();
    Class<R> outputType();

    record Sync<T, R>(Class<T> inputType, Class<R> outputType,
                       BiFunction<T, WorkerScope, WorkerResult<R>> fn)
        implements WorkerFunction<T, R> {}

    record None() implements WorkerFunction<Void, Void> {
        public Class<Void> inputType() { return Void.class; }
        public Class<Void> outputType() { return Void.class; }
    }
}
```

- [ ] **Step 6: Update Worker builder**

Add `TypedOutputBuilder<T, R>` with `.returning(Class<R>)`. Update `.function()` to create `Sync<Map, Map>`. Update `TypedFunctionBuilder` to offer `.returning()` and a Map-output shortcut `.apply()`.

- [ ] **Step 7: Fix all worker-api tests**

Update every test that creates WorkerResult, WorkerFunction, or WorkerOutcome. The pattern: `WorkerResult.of(Map.of(...))` stays unchanged (R inferred as Map). `outcome instanceof WorkerOutcome.Success s` → `s` no longer has `.output()` — use `result.output()` instead.

- [ ] **Step 8: Verify worker module compiles and tests pass**

Run: `mvn test -pl worker -q`
Expected: all tests pass.

- [ ] **Step 9: Commit**

```
feat(#693): WorkerFunction<T,R>, WorkerResult<R>, WorkerScope — foundation types

Two type parameters on WorkerFunction (input T, output R).
WorkerResult<R> with top-level output, partial output on non-success.
WorkerScope interface in worker-api for explicit runtime parameter.
WorkerFunction.Async removed (superseded by virtual threads).
TypedOutputBuilder with .returning(Class<R>) for explicit output type.

Refs #693
```

---

### Task 2: Fix all engine module compilation

**Mechanical sweep.** Every module that references WorkerFunction, WorkerResult, or WorkerOutcome breaks after Task 1. Fix them all — add type parameters, update pattern matching, update casts.

**Files:**
- Modify: ALL files referencing WorkerFunction, WorkerResult, WorkerOutcome across api, common, runtime, blackboard, flow, resilience, scheduler-quartz, engine-ai, engine-inbound, queue, actor-state modules
- Modify: `api/src/main/java/io/casehub/api/model/AgentWorkerFunction.java`
- Modify: `flow/src/main/java/io/casehub/engine/flow/FlowWorkerFunction.java`
- Modify: `common/src/main/java/io/casehub/engine/common/internal/executor/WorkerExecutor.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/executor/SyncAgentWorkerFunctionHandler.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/WorkflowExecutionCompletedHandler.java`
- Plus ~40 other files (use `mvn compile test-compile` error list as the guide)

**Interfaces:**
- Consumes: Task 1 types
- Produces: compiling codebase with all existing tests passing

**Strategy:** Use `mvn compile test-compile -q` iteratively. Each error points to a file and line. Fix systematically by module:

1. api module (AgentWorkerFunction, Agent, handlers)
2. common module (WorkerExecutor, handler interfaces)
3. flow module (FlowWorkerFunction, FlowWorkerFunctionHandler)
4. runtime module (handlers, scheduler, recovery — largest)
5. blackboard module (PlanItem handlers)
6. remaining modules (resilience, scheduler-quartz, actor-state, queue, inbound)

**Common patterns:**
- `WorkerFunction<?>` → `WorkerFunction<?, ?>`
- `WorkerResult` → `WorkerResult<?>` (or `WorkerResult<Map<String, Object>>` where the context is clear)
- `result.outcome() instanceof WorkerOutcome.Success s` → `s.output()` becomes `result.output()`
- `AgentWorkerFunction implements WorkerFunction<Map>` → `implements WorkerFunction<Map<String, Object>, Map<String, Object>>`

- [ ] **Step 1: Install worker-api to local repo**

Run: `mvn install -pl worker -DskipTests -q`

- [ ] **Step 2: Compile all modules, collect errors**

Run: `mvn compile test-compile -q 2>&1 | grep "^\[ERROR\]" | head -50`

- [ ] **Step 3: Fix api module**

Update AgentWorkerFunction, Agent.execute(), CaseDefinitionYamlMapper worker construction. Add `outputType()` to AgentWorkerFunction and FlowWorkerFunction.

- [ ] **Step 4: Fix common module**

Update WorkerExecutor interface, WorkerFunctionHandler, handler interfaces. All use `WorkerFunction<?, ?>` and `WorkerResult<?>`.

- [ ] **Step 5: Fix flow module**

Update FlowWorkerFunction, FlowWorkerFunctionHandler, FlowWorkerFunctionProvider.

- [ ] **Step 6: Fix runtime module**

Largest module. Update SyncAgentWorkerFunctionHandler, DefaultWorkerRuntime, DefaultWorkerExecutor, WorkflowExecutionCompletedHandler, QuartzWorkerExecutionJob, all handler tests.

- [ ] **Step 7: Fix blackboard module**

Update PlanItemCompletionHandler, WorkerOutcomeResolvedHandler.

- [ ] **Step 8: Fix remaining modules**

resilience, scheduler-quartz, actor-state, queue, inbound — any that import the changed types.

- [ ] **Step 9: Verify full compilation**

Run: `mvn install -DskipTests -q`
Expected: clean compile across all modules.

- [ ] **Step 10: Run full test suite**

Run: `mvn test -pl worker,api,common,runtime,blackboard,flow,casehub-engine-inbound -q`
Expected: all existing tests pass (except pre-existing failures).

- [ ] **Step 11: Commit**

```
refactor(#693): update all engine modules for WorkerFunction<T,R> and WorkerResult<R>

Mechanical type parameter propagation across api, common, runtime,
blackboard, flow, and remaining modules. No behavioral changes —
all existing tests pass.

Refs #693
```

---

### Task 3: Remove WorkerExecutionContext + explicit runtime

**Files:**
- Delete: `api/src/main/java/io/casehub/api/model/WorkerExecutionContext.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/executor/SyncAgentWorkerFunctionHandler.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/executor/DefaultWorkerRuntime.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/executor/WorkerRuntimeFactory.java`
- Modify: `scheduler-quartz/src/main/java/io/casehub/engine/scheduler/quartz/QuartzWorkerExecutionJob.java`
- Modify: `api/src/main/java/io/casehub/api/engine/WorkerRuntime.java` (extends WorkerScope)
- Modify: `api/src/main/java/io/casehub/api/model/WorkerFunctions.java` (sequence uses runtime from BiFunction)
- Modify: All tests referencing WorkerExecutionContext
- Test: New tests for explicit runtime passing

**Interfaces:**
- Consumes: WorkerScope from Task 1
- Produces: Zero ThreadLocal in WorkerExecutionContext (deleted), WorkerRuntime extends WorkerScope

- [ ] **Step 1: Update WorkerRuntime to extend WorkerScope**

Add `taskId()` and `context()`. Move `execute()` signatures to align with WorkerScope.

- [ ] **Step 2: Update DefaultWorkerRuntime**

Constructor gains `taskId` and `context` params. `executeSync()` passes `this` as runtime — no ThreadLocal save/restore.

- [ ] **Step 3: Update WorkerRuntimeFactory**

`create()` gains `taskId` and `context` parameters.

- [ ] **Step 4: Update SyncAgentWorkerFunctionHandler**

Remove `WorkerExecutionContext.set()/clear()`. Pass runtime as second arg to `sync.fn().apply(input, runtime)`.

- [ ] **Step 5: Update QuartzWorkerExecutionJob**

Remove `WorkerExecutionContext.set()/clear()`. Pass runtime through to handler.

- [ ] **Step 6: Update WorkerFunctions.sequence()**

Receives runtime from its own BiFunction parameter instead of `WorkerExecutionContext.currentRuntime()`.

- [ ] **Step 7: Delete WorkerExecutionContext**

Use `ide_refactor_safe_delete`. Fix any remaining references.

- [ ] **Step 8: Update all tests**

Replace `WorkerExecutionContext.set(ctx)` / `WorkerExecutionContext.setRuntime(rt)` with direct runtime passing.

- [ ] **Step 9: Verify zero ThreadLocal in engine**

Run: `ide_search_text` for `ThreadLocal` in `*.java` — should return only `CasehubCallableTaskBuilder` (already using CallableTaskFactory, no ThreadLocal).

- [ ] **Step 10: Run tests**

Run: `mvn test -pl worker,api,common,runtime,blackboard,flow -q`

- [ ] **Step 11: Commit**

```
refactor(#693): remove WorkerExecutionContext ThreadLocal — explicit runtime

WorkerRuntime extends WorkerScope. Runtime passed as BiFunction second
parameter. DefaultWorkerRuntime passes this — no save/restore.
WorkerExecutionContext deleted. Zero ThreadLocal in engine.

Refs #693
```

---

### Task 4: Typed execute + typed sequence + output conversion

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/executor/DefaultWorkerRuntime.java`
- Modify: `api/src/main/java/io/casehub/api/model/WorkerFunctions.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/WorkflowExecutionCompletedHandler.java` (output conversion)
- Modify: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java` (outputType parsing)
- Test: `runtime/src/test/java/io/casehub/engine/internal/executor/DefaultWorkerRuntimeTypedTest.java`
- Test: `api/src/test/java/io/casehub/api/model/WorkerFunctionsTypedSequenceTest.java`

**Interfaces:**
- Consumes: WorkerFunction<T, R> with outputType(), WorkerScope, WorkerRuntime
- Produces: Typed `execute()`, typed `sequence()`, POJO→Map output conversion in handler, YAML outputType parsing

- [ ] **Step 1: Write failing tests for typed execute**

Tests: DefaultWorkerRuntime.execute() with typed function receives POJO input directly. execute(String workerName, Map) converts Map→POJO via Jackson when inputType != Map.

- [ ] **Step 2: Implement typed execute on DefaultWorkerRuntime**

`executeSync()` calls `sync.fn().apply(input, this)` — direct typed pass. `execute(String, Map)` resolves worker, checks `inputType()`, converts via `objectMapper.convertValue()` when needed.

- [ ] **Step 3: Write failing tests for typed sequence**

Tests: multi-step sequence with different POJO types at each step. Bridge converts between steps. Non-success short-circuits. MAPPER lenient config.

- [ ] **Step 4: Implement typed sequence**

`sequence()` uses `convertIfNeeded()` between steps. Receives runtime from BiFunction. Overlay accumulation with `toMap()` for POJO outputs.

- [ ] **Step 5: Implement output conversion in WorkflowExecutionCompletedHandler**

When `worker.function().outputType() != Map.class`, convert POJO output to Map via `objectMapper.convertValue()` before applying to CaseContext. Error → route through semantic failure path.

- [ ] **Step 6: Add outputType to YAML mapper**

Parse `outputType:` field. Default to Map.class when absent.

- [ ] **Step 7: Run tests**

Run: `mvn test -pl api,runtime -q`

- [ ] **Step 8: Commit**

```
feat(#693): typed execute, typed sequence, output conversion

DefaultWorkerRuntime.execute() passes typed POJO input directly.
WorkerFunctions.sequence() with bridge-mediated type conversion.
WorkflowExecutionCompletedHandler converts POJO output to Map for
CaseContext. YAML outputType field parsed.

Refs #693
```

---

### Task 5: Context isolation — _diagnostics namespace

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/WorkflowExecutionCompletedHandler.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java`
- Modify: `blackboard/src/main/java/io/casehub/blackboard/handler/StageResetOutcomesCleaner.java`
- Modify: `scheduler-quartz/src/main/java/io/casehub/engine/scheduler/quartz/QuartzWorkerExecutionJob.java` (input projection filter)
- Test: Context isolation tests — sibling diagnostics not visible

**Interfaces:**
- Consumes: WorkerRuntime.taskId() from Task 3
- Produces: `_diagnostics.<taskId>.outcomes` namespace, input projection filter

- [ ] **Step 1: Write failing test for diagnostic namespace**

Test: worker writes outcome → stored at `_diagnostics.<bindingName>.outcomes`, NOT at `_outcomes.<bindingName>`.

- [ ] **Step 2: Rename _outcomes to _diagnostics in handlers**

Update `WorkflowExecutionCompletedHandler.handleSemanticFailure()` — write to `_diagnostics.<bindingName>.outcomes.*`.
Update `recordSuccessOutcome()` — same namespace.
Update `CaseContextChangedEventHandler.publishWorkerSchedule()` — read excludedAgents from new namespace.
Update `CaseContextChangedEventHandler.handleAllCandidatesExhausted()` — write REROUTES_EXHAUSTED to new namespace.
Update `StageResetOutcomesCleaner` — clear from `_diagnostics`.

- [ ] **Step 3: Write failing test for input projection filter**

Test: two parallel tasks, task A fails. Task B's input projection does NOT contain task A's diagnostics.

- [ ] **Step 4: Implement input projection filter**

Before JQ evaluation in QuartzWorkerExecutionJob, create filtered view of working layer that strips `_diagnostics.<X>` for all X ≠ own taskId.

- [ ] **Step 5: Run tests**

Run: `mvn test -pl runtime,blackboard,scheduler-quartz -q`

- [ ] **Step 6: Commit**

```
feat(#698): context isolation — _diagnostics namespace with input filter

Diagnostic state namespaced under _diagnostics.<taskId>. Input projection
filter strips sibling diagnostics before worker input construction.
LLM-driven workers see only their own diagnostic state.

Closes #698
```

---

### Task 6: CLAUDE.md + final verification

**Files:**
- Modify: `CLAUDE.md`

- [ ] **Step 1: Update CLAUDE.md**

Document: WorkerFunction<T,R>, WorkerResult<R>, WorkerScope, TypedOutputBuilder, outputType in YAML, three-level ceremony, _diagnostics namespace, WorkerExecutionContext removal.

- [ ] **Step 2: Final full test run**

Run: `mvn test -q` across all modules.

- [ ] **Step 3: Commit**

```
docs(#693,#698): CLAUDE.md — typed composition, context isolation, ThreadLocal elimination

Refs #693, Refs #698
```
