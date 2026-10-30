# Apply Triage Modifications to Playbook Re-Execution — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1264 — feat: apply triage modifications to playbook re-execution
**Issue group:** #1264

**Goal:** Close the gap between "triage decides what to change" and "the change actually takes effect" by applying playbook modifications before re-execution.

**Architecture:** A thin SPI (`TriageModificationProvider`) in `common-core` bridges the module boundary. `FlowWorkerFunctionHandler` queries it for modifications, merges them cumulatively via `PlaybookTriageModificationParser.mergeAll()`, and applies them to a copy of the `Workflow` via `PlaybookTriageModificationApplicator` before creating the workflow instance.

**Tech Stack:** Java 26, Quarkus 3.32.2, Serverless Workflow SDK 7.x, Mockito, AssertJ

## Global Constraints

- No cross-repo changes — everything in casehub-engine
- `TriageModificationProvider` SPI must be in `common-core` (not `engine-support-core` or `runtime-core`)
- Handler must use `Instance<TriageModificationProvider>` (optional CDI) — never hard dependency
- Original `Workflow` and `mapInput` must never be mutated — always produce copies
- Non-playbook execution paths (`FlowWorkerFunction`, MCP, ReAct, A2A) must be completely untouched
- `@ApplicationScoped` on new CDI beans (not `@DefaultBean` — per protocol PP-20260921-b7c277)

---

## Batch 1: SPI and cumulative merge

### Task 1: TriageModificationProvider SPI + DefaultTriageModificationProvider

**Files:**
- Create: `common-core/src/main/java/io/casehub/engine/common/spi/recovery/TriageModificationProvider.java`
- Create: `runtime-core/src/main/java/io/casehub/engine/runtime/recovery/DefaultTriageModificationProvider.java`
- Test: `runtime/src/test/java/io/casehub/engine/runtime/recovery/DefaultTriageModificationProviderTest.java`

**Interfaces:**
- Consumes: `CaseRecoveryStateRegistry.getOrCreate(UUID)`, `CaseRecoveryState.find(String)`, `BindingRecoveryState.previousTriageModifications()`
- Produces: `TriageModificationProvider.getModifications(UUID caseId, String bindingName) → List<Map<String, Object>>`

- [ ] **Step 1: Write the SPI interface**

```java
package io.casehub.engine.common.spi.recovery;

import java.util.List;
import java.util.Map;
import java.util.UUID;

public interface TriageModificationProvider {
    List<Map<String, Object>> getModifications(UUID caseId, String bindingName);
}
```

- [ ] **Step 2: Write failing tests for DefaultTriageModificationProvider**

Create `DefaultTriageModificationProviderTest.java` in `runtime/src/test/java/.../recovery/`:

```java
package io.casehub.engine.runtime.recovery;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.engine.common.spi.recovery.BindingRecoveryState;
import io.casehub.engine.common.spi.recovery.CaseRecoveryState;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class DefaultTriageModificationProviderTest {

    private CaseRecoveryStateRegistry stateRegistry;
    private DefaultTriageModificationProvider provider;

    @BeforeEach
    void setUp() {
        stateRegistry = new CaseRecoveryStateRegistry();
        provider = new DefaultTriageModificationProvider(stateRegistry);
    }

    @Test
    void returnsModificationsForKnownBinding() {
        UUID caseId = UUID.randomUUID();
        BindingRecoveryState bindingState = stateRegistry.getOrCreate(caseId).getOrCreate("worker-a");
        bindingState.recordTriageModifications(Map.of("playbook.skipSteps", List.of("step-1")));

        List<Map<String, Object>> result = provider.getModifications(caseId, "worker-a");

        assertThat(result).hasSize(1);
        assertThat(result.getFirst()).containsKey("playbook.skipSteps");
    }

    @Test
    void returnsEmptyForUnknownBinding() {
        UUID caseId = UUID.randomUUID();
        stateRegistry.getOrCreate(caseId);

        assertThat(provider.getModifications(caseId, "unknown")).isEmpty();
    }

    @Test
    void returnsEmptyWhenNoTriageState() {
        assertThat(provider.getModifications(UUID.randomUUID(), "any")).isEmpty();
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=DefaultTriageModificationProviderTest -DfailIfNoTests=false`
Expected: Compilation error — `DefaultTriageModificationProvider` does not exist

- [ ] **Step 4: Implement DefaultTriageModificationProvider**

```java
package io.casehub.engine.runtime.recovery;

import io.casehub.engine.common.spi.recovery.BindingRecoveryState;
import io.casehub.engine.common.spi.recovery.CaseRecoveryState;
import io.casehub.engine.common.spi.recovery.TriageModificationProvider;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.List;
import java.util.Map;
import java.util.UUID;

@ApplicationScoped
public class DefaultTriageModificationProvider implements TriageModificationProvider {

    private final CaseRecoveryStateRegistry stateRegistry;

    @Inject
    public DefaultTriageModificationProvider(CaseRecoveryStateRegistry stateRegistry) {
        this.stateRegistry = stateRegistry;
    }

    @Override
    public List<Map<String, Object>> getModifications(UUID caseId, String bindingName) {
        CaseRecoveryState caseState = stateRegistry.getOrCreate(caseId);
        BindingRecoveryState bindingState = caseState.find(bindingName);
        if (bindingState == null) {
            return List.of();
        }
        return bindingState.previousTriageModifications();
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=DefaultTriageModificationProviderTest`
Expected: 3 tests PASS

- [ ] **Step 6: Commit**

```bash
git add common-core/src/main/java/io/casehub/engine/common/spi/recovery/TriageModificationProvider.java runtime-core/src/main/java/io/casehub/engine/runtime/recovery/DefaultTriageModificationProvider.java runtime/src/test/java/io/casehub/engine/runtime/recovery/DefaultTriageModificationProviderTest.java
git commit -m "feat(#1264): add TriageModificationProvider SPI and default impl

Refs #1264"
```

### Task 2: PlaybookTriageModificationParser.mergeAll()

**Files:**
- Modify: `engine-support-core/src/main/java/io/casehub/engine/flow/PlaybookTriageModificationParser.java`
- Modify: `engine-support-core/src/test/java/io/casehub/engine/flow/PlaybookTriageModificationParserTest.java`

**Interfaces:**
- Consumes: `PlaybookTriageModificationParser.parse(Map<String, Object>)` (existing)
- Produces: `PlaybookTriageModificationParser.mergeAll(List<Map<String, Object>>) → List<PlaybookTriageModification>`

- [ ] **Step 1: Write failing tests for mergeAll**

Add to `PlaybookTriageModificationParserTest.java`:

```java
@Test
void mergeAll_emptyListReturnsEmpty() {
    assertThat(PlaybookTriageModificationParser.mergeAll(List.of())).isEmpty();
}

@Test
void mergeAll_singleRoundSameAsParse() {
    Map<String, Object> round = Map.of("playbook.skipSteps", List.of("s1"));
    List<PlaybookTriageModification> merged = PlaybookTriageModificationParser.mergeAll(List.of(round));
    assertThat(merged).hasSize(1);
    assertThat(merged.getFirst()).isInstanceOf(PlaybookTriageModification.SkipStep.class);
}

@Test
void mergeAll_cumulativeSkipSteps() {
    var round1 = Map.<String, Object>of("playbook.skipSteps", List.of("step-a"));
    var round2 = Map.<String, Object>of("playbook.skipSteps", List.of("step-b"));
    List<PlaybookTriageModification> merged =
        PlaybookTriageModificationParser.mergeAll(List.of(round1, round2));

    List<String> skipped = merged.stream()
        .filter(m -> m instanceof PlaybookTriageModification.SkipStep)
        .map(m -> ((PlaybookTriageModification.SkipStep) m).stepId())
        .toList();
    assertThat(skipped).containsExactlyInAnyOrder("step-a", "step-b");
}

@Test
void mergeAll_duplicateSkipStepDeduplicates() {
    var round1 = Map.<String, Object>of("playbook.skipSteps", List.of("step-a"));
    var round2 = Map.<String, Object>of("playbook.skipSteps", List.of("step-a", "step-b"));
    List<PlaybookTriageModification> merged =
        PlaybookTriageModificationParser.mergeAll(List.of(round1, round2));

    List<String> skipped = merged.stream()
        .filter(m -> m instanceof PlaybookTriageModification.SkipStep)
        .map(m -> ((PlaybookTriageModification.SkipStep) m).stepId())
        .toList();
    assertThat(skipped).containsExactlyInAnyOrder("step-a", "step-b");
}

@Test
void mergeAll_replaceStepInputLastWriterWins() {
    var round1 = Map.<String, Object>of("playbook.replaceInput.fetch", Map.of("url", "/v1"));
    var round2 = Map.<String, Object>of("playbook.replaceInput.fetch", Map.of("url", "/v2"));
    List<PlaybookTriageModification> merged =
        PlaybookTriageModificationParser.mergeAll(List.of(round1, round2));

    var replace = merged.stream()
        .filter(m -> m instanceof PlaybookTriageModification.ReplaceStepInput)
        .map(m -> (PlaybookTriageModification.ReplaceStepInput) m)
        .findFirst().orElseThrow();
    assertThat(replace.newInput()).containsEntry("url", "/v2");
}

@Test
void mergeAll_updateParameterLastWriterWins() {
    var round1 = Map.<String, Object>of("playbook.updateParams.timeout", 1000);
    var round2 = Map.<String, Object>of("playbook.updateParams.timeout", 5000);
    List<PlaybookTriageModification> merged =
        PlaybookTriageModificationParser.mergeAll(List.of(round1, round2));

    var update = merged.stream()
        .filter(m -> m instanceof PlaybookTriageModification.UpdateParameter)
        .map(m -> (PlaybookTriageModification.UpdateParameter) m)
        .findFirst().orElseThrow();
    assertThat(update.value()).isEqualTo(5000);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl engine-support-core -Dtest=PlaybookTriageModificationParserTest`
Expected: Compilation error — `mergeAll` method does not exist

- [ ] **Step 3: Implement mergeAll**

Add to `PlaybookTriageModificationParser.java`:

```java
public static List<PlaybookTriageModification> mergeAll(List<Map<String, Object>> rounds) {
    if (rounds.isEmpty()) {
        return List.of();
    }

    java.util.Set<String> skippedSteps = new java.util.LinkedHashSet<>();
    java.util.Map<String, Map<String, Object>> replacedInputs = new java.util.LinkedHashMap<>();
    java.util.Map<String, Object> updatedParams = new java.util.LinkedHashMap<>();

    for (Map<String, Object> round : rounds) {
        for (PlaybookTriageModification mod : parse(round)) {
            switch (mod) {
                case PlaybookTriageModification.SkipStep s -> skippedSteps.add(s.stepId());
                case PlaybookTriageModification.ReplaceStepInput r ->
                    replacedInputs.put(r.stepId(), r.newInput());
                case PlaybookTriageModification.UpdateParameter u ->
                    updatedParams.put(u.key(), u.value());
            }
        }
    }

    List<PlaybookTriageModification> result = new ArrayList<>();
    skippedSteps.forEach(id -> result.add(new PlaybookTriageModification.SkipStep(id)));
    replacedInputs.forEach(
        (id, input) -> result.add(new PlaybookTriageModification.ReplaceStepInput(id, input)));
    updatedParams.forEach(
        (k, v) -> result.add(new PlaybookTriageModification.UpdateParameter(k, v)));
    return List.copyOf(result);
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl engine-support-core -Dtest=PlaybookTriageModificationParserTest`
Expected: All tests PASS (original 5 + new 6)

- [ ] **Step 5: Commit**

```bash
git add engine-support-core/src/main/java/io/casehub/engine/flow/PlaybookTriageModificationParser.java engine-support-core/src/test/java/io/casehub/engine/flow/PlaybookTriageModificationParserTest.java
git commit -m "feat(#1264): add mergeAll for cumulative triage modification merge

Refs #1264"
```

## Batch 2: Workflow applicator

### Task 3: PlaybookTriageModificationApplicator

**Files:**
- Create: `engine-support-core/src/main/java/io/casehub/engine/flow/PlaybookTriageModificationApplicator.java`
- Test: `engine-support-core/src/test/java/io/casehub/engine/flow/PlaybookTriageModificationApplicatorTest.java`

**Interfaces:**
- Consumes: `PlaybookTriageModification` sealed hierarchy, SWF types (`Workflow`, `TaskItem`, `Task`, `SetTask`, `DoTask`, `TryTask`, `ForTask`, `ForkTask`)
- Produces: `PlaybookTriageModificationApplicator.apply(Workflow, List<PlaybookTriageModification>) → ModificationResult` where `ModificationResult` is `record ModificationResult(Workflow workflow, Map<String, Object> parameterOverrides)`

- [ ] **Step 1: Write failing tests**

Create `PlaybookTriageModificationApplicatorTest.java`. Tests compile a simple playbook, then apply modifications and verify the result.

```java
package io.casehub.engine.flow;

import static org.assertj.core.api.Assertions.assertThat;

import io.serverlessworkflow.api.types.Document;
import io.serverlessworkflow.api.types.DoTask;
import io.serverlessworkflow.api.types.SetTask;
import io.serverlessworkflow.api.types.Task;
import io.serverlessworkflow.api.types.TaskItem;
import io.serverlessworkflow.api.types.TryTask;
import io.serverlessworkflow.api.types.TryTaskCatch;
import io.serverlessworkflow.api.types.Workflow;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import org.junit.jupiter.api.Test;

class PlaybookTriageModificationApplicatorTest {

    private static Workflow workflow(TaskItem... items) {
        var doc = new Document().withDsl("1.0.0").withNamespace("test").withName("test").withVersion("0.1.0");
        return new Workflow(doc, new ArrayList<>(List.of(items)));
    }

    private static TaskItem setTask(String name) {
        return new TaskItem(name, new Task().withSetTask(new SetTask()));
    }

    private static TaskItem doBlock(String name, TaskItem... children) {
        var doTask = new DoTask().withDo(new ArrayList<>(List.of(children)));
        return new TaskItem(name, new Task().withDoTask(doTask));
    }

    private static TaskItem tryBlock(String name, TaskItem... tryBody) {
        var tryTask = new TryTask()
            .withTry(new ArrayList<>(List.of(tryBody)))
            .withCatch(new TryTaskCatch().withDo(List.of()));
        return new TaskItem(name, new Task().withTryTask(tryTask));
    }

    @Test
    void emptyModificationsReturnsWorkflowUnchanged() {
        Workflow wf = workflow(setTask("step-1"), setTask("step-2"));
        var result = PlaybookTriageModificationApplicator.apply(wf, List.of());

        assertThat(result.workflow().getDo()).hasSize(2);
        assertThat(result.parameterOverrides()).isEmpty();
    }

    @Test
    void skipStepReplacesTopLevelTaskWithNoop() {
        Workflow wf = workflow(setTask("step-1"), setTask("step-2"), setTask("step-3"));
        var mods = List.<PlaybookTriageModification>of(
            new PlaybookTriageModification.SkipStep("step-2"));

        var result = PlaybookTriageModificationApplicator.apply(wf, mods);

        assertThat(result.workflow().getDo()).hasSize(3);
        TaskItem replaced = result.workflow().getDo().get(1);
        assertThat(replaced.getName()).isEqualTo("step-2");
        assertThat(replaced.getTask().getSetTask()).isNotNull();
    }

    @Test
    void skipStepInNestedDoBlock() {
        Workflow wf = workflow(doBlock("outer", setTask("inner-1"), setTask("inner-2")));
        var mods = List.<PlaybookTriageModification>of(
            new PlaybookTriageModification.SkipStep("inner-1"));

        var result = PlaybookTriageModificationApplicator.apply(wf, mods);

        DoTask outerDo = result.workflow().getDo().getFirst().getTask().getDoTask();
        assertThat(outerDo.getDo()).hasSize(2);
        assertThat(outerDo.getDo().getFirst().getName()).isEqualTo("inner-1");
        assertThat(outerDo.getDo().getFirst().getTask().getSetTask()).isNotNull();
    }

    @Test
    void skipStepInTryBlock() {
        Workflow wf = workflow(tryBlock("try-outer", setTask("try-step-1"), setTask("try-step-2")));
        var mods = List.<PlaybookTriageModification>of(
            new PlaybookTriageModification.SkipStep("try-step-1"));

        var result = PlaybookTriageModificationApplicator.apply(wf, mods);

        TryTask tryTask = result.workflow().getDo().getFirst().getTask().getTryTask();
        assertThat(tryTask.getTry().getFirst().getName()).isEqualTo("try-step-1");
        assertThat(tryTask.getTry().getFirst().getTask().getSetTask()).isNotNull();
    }

    @Test
    void unknownStepIdIsIgnored() {
        Workflow wf = workflow(setTask("step-1"));
        var mods = List.<PlaybookTriageModification>of(
            new PlaybookTriageModification.SkipStep("nonexistent"));

        var result = PlaybookTriageModificationApplicator.apply(wf, mods);

        assertThat(result.workflow().getDo()).hasSize(1);
        assertThat(result.workflow().getDo().getFirst().getName()).isEqualTo("step-1");
    }

    @Test
    void updateParameterCollectedInOverrides() {
        Workflow wf = workflow(setTask("step-1"));
        var mods = List.<PlaybookTriageModification>of(
            new PlaybookTriageModification.UpdateParameter("retryMode", "lenient"),
            new PlaybookTriageModification.UpdateParameter("timeout", 5000));

        var result = PlaybookTriageModificationApplicator.apply(wf, mods);

        assertThat(result.parameterOverrides())
            .containsEntry("retryMode", "lenient")
            .containsEntry("timeout", 5000);
    }

    @Test
    void replaceStepInputWrapsWithInputInjection() {
        Workflow wf = workflow(setTask("fetch-data"), setTask("process"));
        var mods = List.<PlaybookTriageModification>of(
            new PlaybookTriageModification.ReplaceStepInput(
                "fetch-data", Map.of("source", "backup", "format", "csv")));

        var result = PlaybookTriageModificationApplicator.apply(wf, mods);

        TaskItem modified = result.workflow().getDo().getFirst();
        assertThat(modified.getTask().getDoTask()).isNotNull();
        DoTask wrapper = modified.getTask().getDoTask();
        assertThat(wrapper.getDo()).hasSize(2);
        assertThat(wrapper.getDo().getFirst().getTask().getSetTask()).isNotNull();
    }

    @Test
    void multipleModificationsCombined() {
        Workflow wf = workflow(setTask("validate"), setTask("fetch"), setTask("process"));
        var mods = List.<PlaybookTriageModification>of(
            new PlaybookTriageModification.SkipStep("validate"),
            new PlaybookTriageModification.ReplaceStepInput("fetch", Map.of("url", "/backup")),
            new PlaybookTriageModification.UpdateParameter("mode", "safe"));

        var result = PlaybookTriageModificationApplicator.apply(wf, mods);

        assertThat(result.workflow().getDo().get(0).getTask().getSetTask()).isNotNull();
        assertThat(result.workflow().getDo().get(1).getTask().getDoTask()).isNotNull();
        assertThat(result.workflow().getDo().get(2).getName()).isEqualTo("process");
        assertThat(result.parameterOverrides()).containsEntry("mode", "safe");
    }

    @Test
    void originalWorkflowNotMutated() {
        Workflow original = workflow(setTask("step-1"), setTask("step-2"));
        int originalSize = original.getDo().size();
        String originalFirstName = original.getDo().getFirst().getName();

        PlaybookTriageModificationApplicator.apply(
            original,
            List.of(new PlaybookTriageModification.SkipStep("step-1")));

        assertThat(original.getDo()).hasSize(originalSize);
        assertThat(original.getDo().getFirst().getName()).isEqualTo(originalFirstName);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl engine-support-core -Dtest=PlaybookTriageModificationApplicatorTest -DfailIfNoTests=false`
Expected: Compilation error — `PlaybookTriageModificationApplicator` does not exist

- [ ] **Step 3: Implement PlaybookTriageModificationApplicator**

Create `engine-support-core/src/main/java/io/casehub/engine/flow/PlaybookTriageModificationApplicator.java`:

```java
package io.casehub.engine.flow;

import io.serverlessworkflow.api.types.DoTask;
import io.serverlessworkflow.api.types.ForTask;
import io.serverlessworkflow.api.types.ForkTask;
import io.serverlessworkflow.api.types.SetTask;
import io.serverlessworkflow.api.types.Task;
import io.serverlessworkflow.api.types.TaskItem;
import io.serverlessworkflow.api.types.TryTask;
import io.serverlessworkflow.api.types.Workflow;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.stream.Collectors;
import org.jboss.logging.Logger;

public final class PlaybookTriageModificationApplicator {

    private static final Logger LOG = Logger.getLogger(PlaybookTriageModificationApplicator.class);

    public record ModificationResult(Workflow workflow, Map<String, Object> parameterOverrides) {}

    private PlaybookTriageModificationApplicator() {}

    public static ModificationResult apply(
            Workflow workflow, List<PlaybookTriageModification> modifications) {
        if (modifications.isEmpty()) {
            return new ModificationResult(workflow, Map.of());
        }

        Set<String> skipSteps = modifications.stream()
                .filter(m -> m instanceof PlaybookTriageModification.SkipStep)
                .map(m -> ((PlaybookTriageModification.SkipStep) m).stepId())
                .collect(Collectors.toSet());

        Map<String, Map<String, Object>> replaceInputs = new LinkedHashMap<>();
        modifications.stream()
                .filter(m -> m instanceof PlaybookTriageModification.ReplaceStepInput)
                .map(m -> (PlaybookTriageModification.ReplaceStepInput) m)
                .forEach(r -> replaceInputs.put(r.stepId(), r.newInput()));

        Map<String, Object> paramOverrides = new LinkedHashMap<>();
        modifications.stream()
                .filter(m -> m instanceof PlaybookTriageModification.UpdateParameter)
                .map(m -> (PlaybookTriageModification.UpdateParameter) m)
                .forEach(u -> paramOverrides.put(u.key(), u.value()));

        List<TaskItem> modifiedTasks = transformTaskList(
                workflow.getDo(), skipSteps, replaceInputs);

        Workflow modified = new Workflow(workflow.getDocument(), modifiedTasks);
        if (workflow.getInput() != null) modified.setInput(workflow.getInput());
        if (workflow.getUse() != null) modified.setUse(workflow.getUse());
        if (workflow.getTimeout() != null) modified.setTimeout(workflow.getTimeout());
        if (workflow.getOutput() != null) modified.setOutput(workflow.getOutput());

        return new ModificationResult(modified, Map.copyOf(paramOverrides));
    }

    private static List<TaskItem> transformTaskList(
            List<TaskItem> tasks, Set<String> skipSteps, Map<String, Map<String, Object>> replaceInputs) {
        List<TaskItem> result = new ArrayList<>(tasks.size());
        for (TaskItem item : tasks) {
            result.add(transformTaskItem(item, skipSteps, replaceInputs));
        }
        return result;
    }

    private static TaskItem transformTaskItem(
            TaskItem item, Set<String> skipSteps, Map<String, Map<String, Object>> replaceInputs) {
        String name = item.getName();

        if (skipSteps.contains(name)) {
            LOG.debugf("Triage: skipping step '%s'", name);
            return new TaskItem(name, new Task().withSetTask(new SetTask()));
        }

        if (replaceInputs.containsKey(name)) {
            LOG.debugf("Triage: replacing input for step '%s'", name);
            Map<String, Object> newInput = replaceInputs.get(name);
            TaskItem setInput = new TaskItem(
                    name + "_triage_input",
                    new Task().withSetTask(new SetTask()));
            var doTask = new DoTask().withDo(new ArrayList<>(List.of(setInput, item)));
            var inputFrom = new io.serverlessworkflow.api.types.InputFrom().withObject(newInput);
            var input = new io.serverlessworkflow.api.types.Input().withFrom(inputFrom);
            doTask.setInput(input);
            return new TaskItem(name, new Task().withDoTask(doTask));
        }

        Task task = item.getTask();
        if (task == null) return item;

        Object active = task.get();
        if (active instanceof DoTask doTask && doTask.getDo() != null) {
            List<TaskItem> transformed = transformTaskList(doTask.getDo(), skipSteps, replaceInputs);
            var newDo = new DoTask().withDo(transformed);
            copyTaskBaseProperties(doTask, newDo);
            return new TaskItem(name, new Task().withDoTask(newDo));
        }
        if (active instanceof TryTask tryTask && tryTask.getTry() != null) {
            List<TaskItem> transformed = transformTaskList(tryTask.getTry(), skipSteps, replaceInputs);
            var newTry = new TryTask().withTry(transformed).withCatch(tryTask.getCatch());
            copyTaskBaseProperties(tryTask, newTry);
            return new TaskItem(name, new Task().withTryTask(newTry));
        }
        if (active instanceof ForTask forTask && forTask.getDo() != null) {
            List<TaskItem> transformed = transformTaskList(forTask.getDo(), skipSteps, replaceInputs);
            var newFor = new ForTask().withFor(forTask.getFor()).withDo(transformed);
            if (forTask.getWhile() != null) newFor.setWhile(forTask.getWhile());
            copyTaskBaseProperties(forTask, newFor);
            return new TaskItem(name, new Task().withForTask(newFor));
        }
        if (active instanceof ForkTask forkTask
                && forkTask.getFork() != null
                && forkTask.getFork().getBranches() != null) {
            List<TaskItem> transformed = transformTaskList(
                    forkTask.getFork().getBranches(), skipSteps, replaceInputs);
            var newConfig = new io.serverlessworkflow.api.types.ForkTaskConfiguration(transformed);
            newConfig.setCompete(forkTask.getFork().isCompete());
            var newFork = new ForkTask().withFork(newConfig);
            copyTaskBaseProperties(forkTask, newFork);
            return new TaskItem(name, new Task().withForkTask(newFork));
        }

        return item;
    }

    private static void copyTaskBaseProperties(
            io.serverlessworkflow.api.types.TaskBase source,
            io.serverlessworkflow.api.types.TaskBase target) {
        if (source.getIf() != null) target.setIf(source.getIf());
        if (source.getInput() != null) target.setInput(source.getInput());
        if (source.getOutput() != null) target.setOutput(source.getOutput());
        if (source.getExport() != null) target.setExport(source.getExport());
        if (source.getTimeout() != null) target.setTimeout(source.getTimeout());
        if (source.getThen() != null) target.setThen(source.getThen());
        if (source.getMetadata() != null) target.setMetadata(source.getMetadata());
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl engine-support-core -Dtest=PlaybookTriageModificationApplicatorTest`
Expected: All 9 tests PASS

- [ ] **Step 5: Commit**

```bash
git add engine-support-core/src/main/java/io/casehub/engine/flow/PlaybookTriageModificationApplicator.java engine-support-core/src/test/java/io/casehub/engine/flow/PlaybookTriageModificationApplicatorTest.java
git commit -m "feat(#1264): add PlaybookTriageModificationApplicator

Transforms a compiled Workflow by applying SkipStep (no-op replacement),
ReplaceStepInput (input injection wrapper), and UpdateParameter
(parameter overrides). Recursive traversal handles DoTask, TryTask,
ForTask, and ForkTask containers.

Refs #1264"
```

## Batch 3: Handler integration

### Task 4: Wire FlowWorkerFunctionHandler to apply triage modifications

**Files:**
- Modify: `engine-support-core/src/main/java/io/casehub/engine/flow/FlowWorkerFunctionHandler.java`
- Modify: `engine-support-core/src/test/java/io/casehub/engine/flow/FlowWorkerFunctionHandlerTest.java`

**Interfaces:**
- Consumes: `TriageModificationProvider.getModifications(UUID, String)`, `PlaybookTriageModificationParser.mergeAll(List)`, `PlaybookTriageModificationApplicator.apply(Workflow, List)`
- Produces: Modified handler that applies triage modifications for `PlaybookWorkerFunction` executions

- [ ] **Step 1: Write failing tests**

Add to `FlowWorkerFunctionHandlerTest.java`. The handler constructor changes to accept an optional `Instance<TriageModificationProvider>`, so update `setUp()` and add new tests:

```java
// Update imports — add:
import io.casehub.engine.common.spi.recovery.TriageModificationProvider;
import jakarta.enterprise.inject.Instance;
import io.casehub.platform.compiler.CompilationResult;
import io.casehub.platform.compiler.CoordinationSpec;
import io.casehub.platform.compiler.SourceMap;
import java.util.List;
import java.util.UUID;

// Update setUp():
@BeforeEach
void setUp() {
    app = mock(WorkflowApplication.class);
    registry = mock(FlowExecutionRegistry.class);
    @SuppressWarnings("unchecked")
    Instance<TriageModificationProvider> noProvider = mock(Instance.class);
    when(noProvider.isResolvable()).thenReturn(false);
    handler = new FlowWorkerFunctionHandler(
        app, registry, java.util.concurrent.Executors.newSingleThreadExecutor(), noProvider);
}

// Add new test methods:

@Test
void no_modification_when_provider_not_resolvable() {
    // Provider is not resolvable (set up in setUp) — handler uses original workflow
    var wfInstance = mockWorkflowInstance("inst-1");
    var model = mock(WorkflowModel.class);
    when(model.asMap()).thenReturn(Optional.of(Map.of("result", "ok")));
    when(wfInstance.start()).thenReturn(CompletableFuture.completedFuture(model));
    stubApp(wfInstance);

    Workflow wf = mock(Workflow.class);
    var compilation = new CompilationResult(wf, CoordinationSpec.empty(), SourceMap.empty());
    var function = new PlaybookWorkerFunction(wf, compilation);
    var context = new WorkerContext("task", UUID.randomUUID(), List.of(),
        List.of(), PropagationContext.empty(), Map.of(), List.of(), List.of());
    var metadata = new ExecutionMetadata("worker-1", "hash-1", "binding-a", null);

    handler.execute(function, Map.of("key", "val"), context, 5000, metadata);

    // Verify app.workflowDefinition was called with the ORIGINAL workflow
    verify(app).workflowDefinition(eq(wf));
}

@Test
void applies_modifications_when_provider_returns_data() {
    TriageModificationProvider provider = mock(TriageModificationProvider.class);
    @SuppressWarnings("unchecked")
    Instance<TriageModificationProvider> providerInstance = mock(Instance.class);
    when(providerInstance.isResolvable()).thenReturn(true);
    when(providerInstance.get()).thenReturn(provider);

    var modHandler = new FlowWorkerFunctionHandler(
        app, registry, java.util.concurrent.Executors.newSingleThreadExecutor(), providerInstance);

    UUID caseId = UUID.randomUUID();
    when(provider.getModifications(caseId, "binding-a"))
        .thenReturn(List.of(Map.of("playbook.updateParams.mode", "safe")));

    var wfInstance = mockWorkflowInstance("inst-2");
    var model = mock(WorkflowModel.class);
    when(model.asMap()).thenReturn(Optional.of(Map.of("result", "ok")));
    when(wfInstance.start()).thenReturn(CompletableFuture.completedFuture(model));

    // Accept any Workflow (will be a modified copy)
    var wfDef = mock(WorkflowDefinition.class);
    when(app.workflowDefinition(any(Workflow.class))).thenReturn(wfDef);
    when(wfDef.instance(any())).thenReturn(wfInstance);

    var doc = new io.serverlessworkflow.api.types.Document()
        .withDsl("1.0.0").withNamespace("test").withName("test").withVersion("0.1.0");
    var taskItem = new io.serverlessworkflow.api.types.TaskItem(
        "step-1", new io.serverlessworkflow.api.types.Task()
            .withSetTask(new io.serverlessworkflow.api.types.SetTask()));
    Workflow wf = new Workflow(doc, List.of(taskItem));
    var compilation = new CompilationResult(wf, CoordinationSpec.empty(), SourceMap.empty());
    var function = new PlaybookWorkerFunction(wf, compilation);

    var context = new WorkerContext("task", caseId, List.of(),
        List.of(), PropagationContext.empty(), Map.of(), List.of(), List.of());
    var metadata = new ExecutionMetadata("worker-1", "hash-1", "binding-a", null);

    modHandler.execute(function, Map.of("key", "val"), context, 5000, metadata);

    // Verify provider was queried
    verify(provider).getModifications(caseId, "binding-a");
}

@Test
void no_modification_for_non_playbook_function() {
    TriageModificationProvider provider = mock(TriageModificationProvider.class);
    @SuppressWarnings("unchecked")
    Instance<TriageModificationProvider> providerInstance = mock(Instance.class);
    when(providerInstance.isResolvable()).thenReturn(true);
    when(providerInstance.get()).thenReturn(provider);

    var modHandler = new FlowWorkerFunctionHandler(
        app, registry, java.util.concurrent.Executors.newSingleThreadExecutor(), providerInstance);

    var wfInstance = mockWorkflowInstance("inst-3");
    var model = mock(WorkflowModel.class);
    when(model.asMap()).thenReturn(Optional.of(Map.of("result", "ok")));
    when(wfInstance.start()).thenReturn(CompletableFuture.completedFuture(model));
    stubApp(wfInstance);

    Workflow wf = mock(Workflow.class);
    var function = new FlowWorkerFunction(wf, null);
    var context = new WorkerContext("task", UUID.randomUUID(), List.of(),
        List.of(), PropagationContext.empty(), Map.of(), List.of(), List.of());
    var metadata = new ExecutionMetadata("worker-1", "hash-1", "binding-a", null);

    modHandler.execute(function, Map.of(), context, 5000, metadata);

    // Provider should NOT be queried for non-playbook functions
    verify(provider, org.mockito.Mockito.never()).getModifications(any(), any());
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl engine-support-core -Dtest=FlowWorkerFunctionHandlerTest`
Expected: Compilation error — constructor signature mismatch (no `Instance<TriageModificationProvider>` parameter)

- [ ] **Step 3: Update FlowWorkerFunctionHandler**

Modify the handler to add the new dependency and modification block. Changes:

1. Add field: `private final Instance<TriageModificationProvider> triageModProvider;`
2. Update constructor to accept `Instance<TriageModificationProvider>` as the 4th parameter
3. Add the modification logic in `execute()` between `mapInput` extraction and `wfInstance` creation

Updated `execute()` method — insert after line 75 (`final Map<String, Object> mapInput = ...`):

```java
Workflow effectiveWorkflow = swf.workflow();
Map<String, Object> effectiveInput = mapInput;

if (swf instanceof PlaybookWorkerFunction
        && triageModProvider.isResolvable()
        && metadata.bindingName() != null) {
    java.util.List<java.util.Map<String, Object>> rawMods =
        triageModProvider.get().getModifications(context.caseId(), metadata.bindingName());
    if (!rawMods.isEmpty()) {
        java.util.List<PlaybookTriageModification> merged =
            PlaybookTriageModificationParser.mergeAll(rawMods);
        PlaybookTriageModificationApplicator.ModificationResult modResult =
            PlaybookTriageModificationApplicator.apply(effectiveWorkflow, merged);
        effectiveWorkflow = modResult.workflow();
        if (!modResult.parameterOverrides().isEmpty()) {
            effectiveInput = new java.util.LinkedHashMap<>(mapInput);
            effectiveInput.putAll(modResult.parameterOverrides());
        }
    }
}
```

Then change line 77 from:
```java
final WorkflowInstance wfInstance = app.workflowDefinition(swf.workflow()).instance(mapInput);
```
to:
```java
final WorkflowInstance wfInstance = app.workflowDefinition(effectiveWorkflow).instance(effectiveInput);
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl engine-support-core -Dtest=FlowWorkerFunctionHandlerTest`
Expected: All tests PASS (original + 3 new)

- [ ] **Step 5: Check for CDI wiring**

Verify that `FlowWorkerFunctionHandler`'s constructor injection is correct. The class uses constructor injection — adding `Instance<TriageModificationProvider>` as a CDI parameter will auto-resolve when the provider is on the classpath. Check that no `@Inject`-annotated constructor conflicts exist and that the class remains `@ApplicationScoped` (or is discovered via Jandex).

- [ ] **Step 6: Run full module test suite**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl engine-support-core`
Expected: All existing tests still pass — no regressions

- [ ] **Step 7: Commit**

```bash
git add engine-support-core/src/main/java/io/casehub/engine/flow/FlowWorkerFunctionHandler.java engine-support-core/src/test/java/io/casehub/engine/flow/FlowWorkerFunctionHandlerTest.java
git commit -m "feat(#1264): wire triage modifications into FlowWorkerFunctionHandler

FlowWorkerFunctionHandler now queries TriageModificationProvider for
playbook executions, merges modifications cumulatively, and applies
them to a copy of the Workflow before creating the instance.
Non-playbook execution is untouched.

Closes #1264"
```

## References

- [2026-10-30-apply-triage-mods-design.md] — design spec this plan implements
- `FlowWorkerFunctionHandler.java:46-130` — handler being modified
- `PlaybookTriageModification.java:20-31` — sealed modification hierarchy
- `PlaybookTriageModificationParser.java:22-56` — existing parser, extended with mergeAll
- `BindingRecoveryState.java:23-70` — stores raw modifications per binding
- `CaseRecoveryStateRegistry.java:22-33` — case-level registry (runtime-core)
- `DefaultRecoveryCoordinator.java:291-346` — records modifications and publishes CaseContextChangedEvent
- `PlaybookWorkerFunction.java:25-53` — carries CompilationResult
- `Workflow` (SWF SDK) — `getDo()` returns `List<TaskItem>`, constructor `Workflow(Document, List<TaskItem>)`
- `Task` (SWF SDK) — `getDoTask()`, `getTryTask()`, `getForTask()`, `getForkTask()`, `getSetTask()` for type dispatch
- Protocol PP-20260921-b7c277 — no `@DefaultBean` on multi-instance SPIs
- GitHub #1264
- GitHub #1262 — triage infrastructure (prerequisite)
