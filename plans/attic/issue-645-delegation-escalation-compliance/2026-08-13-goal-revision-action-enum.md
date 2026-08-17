# GoalRevisionAction Enum Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #903 — Add action enum to GoalRevisionProposal.RevisedGoal
**Issue group:** #903

**Goal:** Add a `GoalRevisionAction` enum (REVISE, ABANDON, COMPLETE) to
`RevisedGoal` so goal lifecycle transitions are type-safe instead of
encoded as string conventions.

**Architecture:** New enum in engine-api. `RevisedGoal` gains an `action`
field (second component). `GoalRevisionEvaluator.mergeDescriptions()`
renamed to `applyRevisions()` with a switch on the action — REVISE
updates the description, ABANDON/COMPLETE exclude the goal from the
descriptor. `LlmGoalRevisionStrategy` prompt and parsing updated to
produce all three actions.

**Tech Stack:** Java 21 records, Quarkus CDI, Jackson JSON parsing

## Global Constraints

- No cross-repo changes (eidos-api, neocortex, etc.)
- No backward-compatible constructors — all callers specify action explicitly
- LLM parser tolerates absent/invalid action fields (defaults to REVISE)
- Pre-release stage — no backward compat required

---

### Task 1: GoalRevisionAction enum + RevisedGoal modification (engine-api)

**Files:**
- Create: `api/src/main/java/io/casehub/api/spi/routing/GoalRevisionAction.java`
- Modify: `api/src/main/java/io/casehub/api/spi/routing/GoalRevisionProposal.java`
- Test: `api/src/test/java/io/casehub/api/spi/routing/GoalRevisionProposalTest.java`

**Interfaces:**
- Consumes: nothing
- Produces: `GoalRevisionAction` enum with `REVISE`, `ABANDON`, `COMPLETE`.
  `RevisedGoal(String goalName, GoalRevisionAction action, String revisedDescription, String revisionReason)` — compact constructor validates non-null `goalName`, `action`, `revisionReason`; throws `IllegalArgumentException` when `action == REVISE && revisedDescription == null`.

- [ ] **Step 1: Write failing tests for the new action field**

Replace the entire `GoalRevisionProposalTest` with tests covering the new `action` field:

```java
package io.casehub.api.spi.routing;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertNull;
import static org.junit.jupiter.api.Assertions.assertThrows;
import static org.junit.jupiter.api.Assertions.assertTrue;

import java.util.List;
import org.junit.jupiter.api.Test;

class GoalRevisionProposalTest {

  @Test
  void constructsWithReviseAction() {
    var revision = new GoalRevisionProposal.RevisedGoal(
        "g1", GoalRevisionAction.REVISE, "new desc", "better fit");
    var proposal = new GoalRevisionProposal(List.of(revision), "test rationale");
    assertEquals(1, proposal.revisions().size());
    assertEquals("g1", proposal.revisions().get(0).goalName());
    assertEquals(GoalRevisionAction.REVISE, proposal.revisions().get(0).action());
    assertEquals("new desc", proposal.revisions().get(0).revisedDescription());
  }

  @Test
  void reviseActionRequiresDescription() {
    assertThrows(
        IllegalArgumentException.class,
        () -> new GoalRevisionProposal.RevisedGoal(
            "g1", GoalRevisionAction.REVISE, null, "reason"));
  }

  @Test
  void abandonActionAllowsNullDescription() {
    var revision = new GoalRevisionProposal.RevisedGoal(
        "g1", GoalRevisionAction.ABANDON, null, "no longer relevant");
    assertNull(revision.revisedDescription());
    assertEquals(GoalRevisionAction.ABANDON, revision.action());
  }

  @Test
  void completeActionAllowsNullDescription() {
    var revision = new GoalRevisionProposal.RevisedGoal(
        "g1", GoalRevisionAction.COMPLETE, null, "goal achieved");
    assertNull(revision.revisedDescription());
    assertEquals(GoalRevisionAction.COMPLETE, revision.action());
  }

  @Test
  void abandonActionAcceptsInformationalDescription() {
    var revision = new GoalRevisionProposal.RevisedGoal(
        "g1", GoalRevisionAction.ABANDON, "was trying X", "unachievable");
    assertEquals("was trying X", revision.revisedDescription());
  }

  @Test
  void nullActionThrows() {
    assertThrows(
        NullPointerException.class,
        () -> new GoalRevisionProposal.RevisedGoal("g1", null, "desc", "reason"));
  }

  @Test
  void nullGoalNameThrows() {
    assertThrows(
        NullPointerException.class,
        () -> new GoalRevisionProposal.RevisedGoal(
            null, GoalRevisionAction.REVISE, "desc", "reason"));
  }

  @Test
  void nullRevisionReasonThrows() {
    assertThrows(
        NullPointerException.class,
        () -> new GoalRevisionProposal.RevisedGoal(
            "g1", GoalRevisionAction.REVISE, "desc", null));
  }

  @Test
  void revisionsListIsImmutable() {
    var proposal = new GoalRevisionProposal(List.of(), "rationale");
    assertThrows(
        UnsupportedOperationException.class,
        () -> proposal.revisions().add(new GoalRevisionProposal.RevisedGoal(
            "g1", GoalRevisionAction.ABANDON, null, "reason")));
  }

  @Test
  void nullRevisionsThrows() {
    assertThrows(NullPointerException.class, () -> new GoalRevisionProposal(null, "rationale"));
  }

  @Test
  void emptyRevisionsAllowed() {
    var proposal = new GoalRevisionProposal(List.of(), "no changes needed");
    assertTrue(proposal.revisions().isEmpty());
  }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl api -Dtest=GoalRevisionProposalTest -Dsurefire.failIfNoSpecifiedTests=false -q`
Expected: Compilation failure (GoalRevisionAction not found, constructor signature mismatch)

- [ ] **Step 3: Create GoalRevisionAction enum**

Create `api/src/main/java/io/casehub/api/spi/routing/GoalRevisionAction.java`:

```java
package io.casehub.api.spi.routing;

public enum GoalRevisionAction {
    REVISE,
    ABANDON,
    COMPLETE
}
```

- [ ] **Step 4: Modify RevisedGoal record**

Replace the `RevisedGoal` inner record in `GoalRevisionProposal.java`:

```java
public record RevisedGoal(
    String goalName,
    GoalRevisionAction action,
    String revisedDescription,
    String revisionReason) {
  public RevisedGoal {
    Objects.requireNonNull(goalName, "goalName must not be null");
    Objects.requireNonNull(action, "action must not be null");
    Objects.requireNonNull(revisionReason, "revisionReason must not be null");
    if (action == GoalRevisionAction.REVISE && revisedDescription == null) {
      throw new IllegalArgumentException(
          "revisedDescription is required for REVISE action");
    }
  }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn test -pl api -Dtest=GoalRevisionProposalTest -q`
Expected: All 11 tests PASS

- [ ] **Step 6: Commit**

```
git add api/src/main/java/io/casehub/api/spi/routing/GoalRevisionAction.java \
       api/src/main/java/io/casehub/api/spi/routing/GoalRevisionProposal.java \
       api/src/test/java/io/casehub/api/spi/routing/GoalRevisionProposalTest.java
git commit -m "feat(#903): add GoalRevisionAction enum to RevisedGoal"
```

---

### Task 2: GoalRevisionEvaluator — applyRevisions + audit enrichment (runtime)

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/routing/GoalRevisionEvaluator.java`
- Test: `runtime/src/test/java/io/casehub/engine/internal/routing/GoalRevisionEvaluatorTest.java`

**Interfaces:**
- Consumes: `GoalRevisionAction` enum, `RevisedGoal(goalName, action, revisedDescription, revisionReason)` from Task 1
- Produces: `applyRevisions(List<AgentGoal>, GoalRevisionProposal) → List<AgentGoal>` — REVISE updates description, ABANDON/COMPLETE exclude goal. `writeAuditLog()` gains `abandonedGoals` and `completedGoals` parameters.

- [ ] **Step 1: Write failing tests for ABANDON and COMPLETE actions**

Add the following tests to `GoalRevisionEvaluatorTest`. These test `applyRevisions()` indirectly through the full `handleEvolved()` flow — the evaluator triggers on outcome threshold, calls GoalEvolution, then applies revisions from the strategy.

First, add a helper method to set up a mock strategy that returns a specific proposal:

```java
import io.casehub.api.spi.routing.GoalRevisionAction;
import io.casehub.api.spi.routing.GoalRevisionProposal;
import io.casehub.api.spi.routing.GoalRevisionStrategy;

// Add to setUp() or as needed per test:
private void setupStrategyReturning(GoalRevisionProposal proposal) {
  GoalRevisionStrategy strategy = mock(GoalRevisionStrategy.class);
  when(strategy.revise(any())).thenReturn(proposal);
  when(strategyResolver.resolve(GoalRevisionStrategy.class, "llm")).thenReturn(strategy);
}
```

Then add these tests:

```java
@Test
void abandonActionRemovesGoalFromDescriptor() throws Exception {
  CaseInstance instance = buildCaseInstance("tenant-1");
  AgentGoal g1 = goal("g1", GoalPriority.SECONDARY);
  AgentGoal g2 = goal("g2", GoalPriority.PRIMARY);
  setupDefinition(instance, "worker-1", g1, g2);

  for (int i = 0; i < 9; i++) {
    goalSignalStore.recordOutcome("agent-1", "tenant-1", "g1", GoalOutcome.SUCCESS);
  }
  goalSignalStore.recordOutcome("agent-1", "tenant-1", "g1", GoalOutcome.FAILURE);
  goalSignalStore.recordOutcome("agent-1", "tenant-1", "g2", GoalOutcome.SUCCESS);

  AgentDescriptor current = descriptorWithGoals(g1, g2);
  when(agentRegistry.findById("agent-1", "tenant-1")).thenReturn(Optional.of(current));

  GoalRevisionProposal proposal = new GoalRevisionProposal(
      List.of(new GoalRevisionProposal.RevisedGoal(
          "g2", GoalRevisionAction.ABANDON, null, "no longer relevant")),
      "dropping g2");
  setupStrategyReturning(proposal);

  CountDownLatch latch = new CountDownLatch(1);
  doAnswer(inv -> {
    registeredDescriptors.add(inv.getArgument(0));
    latch.countDown();
    return null;
  }).when(agentRegistry).register(any());

  for (int i = 0; i < 3; i++) {
    evaluator.record(instance, "worker-1", "cap-x", WorkerOutcome.success());
  }

  assertThat(latch.await(5, TimeUnit.SECONDS)).isTrue();
  AgentDescriptor registered = registeredDescriptors.get(0);
  assertThat(registered.goals()).hasSize(1);
  assertThat(registered.goals().get(0).name()).isEqualTo("g1");
}

@Test
void completeActionRemovesGoalFromDescriptor() throws Exception {
  CaseInstance instance = buildCaseInstance("tenant-1");
  AgentGoal g1 = goal("g1", GoalPriority.SECONDARY);
  AgentGoal g2 = goal("g2", GoalPriority.PRIMARY);
  setupDefinition(instance, "worker-1", g1, g2);

  for (int i = 0; i < 9; i++) {
    goalSignalStore.recordOutcome("agent-1", "tenant-1", "g1", GoalOutcome.SUCCESS);
  }
  goalSignalStore.recordOutcome("agent-1", "tenant-1", "g1", GoalOutcome.FAILURE);
  goalSignalStore.recordOutcome("agent-1", "tenant-1", "g2", GoalOutcome.SUCCESS);

  AgentDescriptor current = descriptorWithGoals(g1, g2);
  when(agentRegistry.findById("agent-1", "tenant-1")).thenReturn(Optional.of(current));

  GoalRevisionProposal proposal = new GoalRevisionProposal(
      List.of(new GoalRevisionProposal.RevisedGoal(
          "g2", GoalRevisionAction.COMPLETE, null, "goal achieved")),
      "completing g2");
  setupStrategyReturning(proposal);

  CountDownLatch latch = new CountDownLatch(1);
  doAnswer(inv -> {
    registeredDescriptors.add(inv.getArgument(0));
    latch.countDown();
    return null;
  }).when(agentRegistry).register(any());

  for (int i = 0; i < 3; i++) {
    evaluator.record(instance, "worker-1", "cap-x", WorkerOutcome.success());
  }

  assertThat(latch.await(5, TimeUnit.SECONDS)).isTrue();
  AgentDescriptor registered = registeredDescriptors.get(0);
  assertThat(registered.goals()).hasSize(1);
  assertThat(registered.goals().get(0).name()).isEqualTo("g1");
}

@Test
void mixedActionsAppliedCorrectly() throws Exception {
  CaseInstance instance = buildCaseInstance("tenant-1");
  AgentGoal g1 = goal("g1", GoalPriority.SECONDARY);
  AgentGoal g2 = goal("g2", GoalPriority.PRIMARY);
  AgentGoal g3 = goal("g3", GoalPriority.SECONDARY);
  setupDefinition(instance, "worker-1", g1, g2, g3);

  for (int i = 0; i < 9; i++) {
    goalSignalStore.recordOutcome("agent-1", "tenant-1", "g1", GoalOutcome.SUCCESS);
  }
  goalSignalStore.recordOutcome("agent-1", "tenant-1", "g1", GoalOutcome.FAILURE);

  AgentDescriptor current = descriptorWithGoals(g1, g2, g3);
  when(agentRegistry.findById("agent-1", "tenant-1")).thenReturn(Optional.of(current));

  GoalRevisionProposal proposal = new GoalRevisionProposal(
      List.of(
          new GoalRevisionProposal.RevisedGoal(
              "g1", GoalRevisionAction.REVISE, "updated desc", "refined"),
          new GoalRevisionProposal.RevisedGoal(
              "g2", GoalRevisionAction.ABANDON, null, "unachievable"),
          new GoalRevisionProposal.RevisedGoal(
              "g3", GoalRevisionAction.COMPLETE, null, "achieved")),
      "mixed actions");
  setupStrategyReturning(proposal);

  CountDownLatch latch = new CountDownLatch(1);
  doAnswer(inv -> {
    registeredDescriptors.add(inv.getArgument(0));
    latch.countDown();
    return null;
  }).when(agentRegistry).register(any());

  for (int i = 0; i < 3; i++) {
    evaluator.record(instance, "worker-1", "cap-x", WorkerOutcome.success());
  }

  assertThat(latch.await(5, TimeUnit.SECONDS)).isTrue();
  AgentDescriptor registered = registeredDescriptors.get(0);
  assertThat(registered.goals()).hasSize(1);
  assertThat(registered.goals().get(0).name()).isEqualTo("g1");
  assertThat(registered.goals().get(0).description()).isEqualTo("updated desc");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn install -pl api -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=GoalRevisionEvaluatorTest -q`
Expected: Compilation failure (`GoalRevisionAction` not imported, `mergeDescriptions` not matching)

- [ ] **Step 3: Implement applyRevisions and audit enrichment**

In `GoalRevisionEvaluator.java`:

1. Add import: `import io.casehub.api.spi.routing.GoalRevisionAction;`

2. Rename `mergeDescriptions` to `applyRevisions` and replace its body:

```java
private List<AgentGoal> applyRevisions(List<AgentGoal> goals, GoalRevisionProposal proposal) {
  Map<String, GoalRevisionProposal.RevisedGoal> revisionsByGoal = new HashMap<>();
  for (var revision : proposal.revisions()) {
    revisionsByGoal.put(revision.goalName(), revision);
  }
  if (revisionsByGoal.isEmpty()) {
    return goals;
  }

  List<AgentGoal> result = new ArrayList<>();
  for (AgentGoal goal : goals) {
    var revision = revisionsByGoal.get(goal.name());
    if (revision == null) {
      result.add(goal);
      continue;
    }
    switch (revision.action()) {
      case REVISE -> {
        try {
          result.add(goal.toBuilder().description(revision.revisedDescription()).build());
        } catch (Exception e) {
          LOG.warnf(
              "Invalid description for goal %s, keeping original: %s",
              goal.name(), e.getMessage());
          result.add(goal);
        }
      }
      case ABANDON, COMPLETE -> {}
    }
  }
  return result;
}
```

3. Update `handleEvolved()` to extract action lists before calling `applyRevisions()` and pass them to `writeAuditLog()`:

```java
private void handleEvolved(
    String agentId,
    String tenancyId,
    AgentDescriptor descriptor,
    GoalEvolutionResult.Evolved evolved,
    Map<String, GoalOutcomeCounts> counts,
    CaseDefinition definition) {

  List<AgentGoal> finalGoals = evolved.newGoals();
  List<String> abandonedGoals = List.of();
  List<String> completedGoals = List.of();

  GoalRevisionStrategy strategy = resolveStrategy();
  if (strategy != null) {
    try {
      GoalRevisionContext context =
          new GoalRevisionContext(agentId, tenancyId, finalGoals, counts);
      GoalRevisionProposal proposal = strategy.revise(context);
      if (proposal != null && !proposal.revisions().isEmpty()) {
        abandonedGoals = proposal.revisions().stream()
            .filter(r -> r.action() == GoalRevisionAction.ABANDON)
            .map(GoalRevisionProposal.RevisedGoal::goalName)
            .toList();
        completedGoals = proposal.revisions().stream()
            .filter(r -> r.action() == GoalRevisionAction.COMPLETE)
            .map(GoalRevisionProposal.RevisedGoal::goalName)
            .toList();
        finalGoals = applyRevisions(finalGoals, proposal);
      }
    } catch (Exception e) {
      LOG.warnf(e, "GoalRevisionStrategy failed for agent %s, applying priority-only", agentId);
    }
  }

  AgentDescriptor updated = descriptor.toBuilder().goals(finalGoals).build();
  agentRegistry.get().register(updated);
  goalSignalStore.get().clear(agentId, tenancyId);

  writeAuditLog(agentId, tenancyId, evolved, counts, abandonedGoals, completedGoals);
}
```

4. Update `writeAuditLog()` signature and add new metadata:

```java
private void writeAuditLog(
    String agentId,
    String tenancyId,
    GoalEvolutionResult.Evolved evolved,
    Map<String, GoalOutcomeCounts> counts,
    List<String> abandonedGoals,
    List<String> completedGoals) {
  try {
    com.fasterxml.jackson.databind.ObjectMapper mapper =
        new com.fasterxml.jackson.databind.ObjectMapper();
    Map<String, Object> metadata = new HashMap<>();
    metadata.put("agentId", agentId);
    metadata.put("evolutionResult", "EVOLVED");
    metadata.put("promotedGoals", evolved.promotedGoals());
    metadata.put("demotedGoals", evolved.demotedGoals());
    metadata.put("abandonedGoals", abandonedGoals);
    metadata.put("completedGoals", completedGoals);
    metadata.put(
        "totalGoalsRevised", evolved.promotedGoals().size() + evolved.demotedGoals().size()
            + abandonedGoals.size() + completedGoals.size());
    metadata.put("totalGoalsEvaluated", evolved.newGoals().size());

    EventLog eventLog = new EventLog();
    eventLog.setEventType(CaseHubEventType.GOAL_REVISED);
    eventLog.setPayload(mapper.valueToTree(metadata));
    eventLog.setTimestamp(Instant.now());
    eventLogRepository.append(eventLog, tenancyId);
  } catch (Exception e) {
    LOG.warnf(e, "Failed to write GOAL_REVISED audit log for agent %s", agentId);
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn install -pl api -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=GoalRevisionEvaluatorTest -q`
Expected: All tests PASS (existing + 3 new)

- [ ] **Step 5: Commit**

```
git add runtime/src/main/java/io/casehub/engine/internal/routing/GoalRevisionEvaluator.java \
       runtime/src/test/java/io/casehub/engine/internal/routing/GoalRevisionEvaluatorTest.java
git commit -m "feat(#903): handle ABANDON/COMPLETE in GoalRevisionEvaluator"
```

---

### Task 3: LlmGoalRevisionStrategy — prompt + parsing (runtime)

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/routing/LlmGoalRevisionStrategy.java`
- Test: `runtime/src/test/java/io/casehub/engine/internal/routing/LlmGoalRevisionStrategyTest.java`

**Interfaces:**
- Consumes: `GoalRevisionAction` enum, `RevisedGoal(goalName, action, revisedDescription, revisionReason)` from Task 1
- Produces: Updated `revise(GoalRevisionContext) → GoalRevisionProposal` that parses `action` field from LLM JSON. System prompt includes abandonment/completion vocabulary.

- [ ] **Step 1: Write failing tests for action parsing**

Add these tests to `LlmGoalRevisionStrategyTest`:

```java
import io.casehub.api.spi.routing.GoalRevisionAction;

@Test
void parsesReviseAction() {
  String jsonResponse =
      "{\"revisions\": [{\"goalName\": \"g1\", \"action\": \"REVISE\", "
          + "\"revisedDescription\": \"better desc\", \"revisionReason\": \"aligned\"}], "
          + "\"rationale\": \"test\"}";
  LlmGoalRevisionStrategy strategy = strategyWithResponse(jsonResponse);

  GoalRevisionProposal proposal = strategy.revise(buildContext());
  assertThat(proposal.revisions().get(0).action()).isEqualTo(GoalRevisionAction.REVISE);
  assertThat(proposal.revisions().get(0).revisedDescription()).isEqualTo("better desc");
}

@Test
void parsesAbandonAction() {
  String jsonResponse =
      "{\"revisions\": [{\"goalName\": \"g1\", \"action\": \"ABANDON\", "
          + "\"revisedDescription\": null, \"revisionReason\": \"unachievable\"}], "
          + "\"rationale\": \"test\"}";
  LlmGoalRevisionStrategy strategy = strategyWithResponse(jsonResponse);

  GoalRevisionProposal proposal = strategy.revise(buildContext());
  assertThat(proposal.revisions().get(0).action()).isEqualTo(GoalRevisionAction.ABANDON);
  assertThat(proposal.revisions().get(0).revisedDescription()).isNull();
}

@Test
void parsesCompleteAction() {
  String jsonResponse =
      "{\"revisions\": [{\"goalName\": \"g1\", \"action\": \"COMPLETE\", "
          + "\"revisedDescription\": null, \"revisionReason\": \"achieved\"}], "
          + "\"rationale\": \"test\"}";
  LlmGoalRevisionStrategy strategy = strategyWithResponse(jsonResponse);

  GoalRevisionProposal proposal = strategy.revise(buildContext());
  assertThat(proposal.revisions().get(0).action()).isEqualTo(GoalRevisionAction.COMPLETE);
}

@Test
void missingActionDefaultsToRevise() {
  String jsonResponse =
      "{\"revisions\": [{\"goalName\": \"g1\", \"revisedDescription\": \"new\", "
          + "\"revisionReason\": \"updated\"}], \"rationale\": \"test\"}";
  LlmGoalRevisionStrategy strategy = strategyWithResponse(jsonResponse);

  GoalRevisionProposal proposal = strategy.revise(buildContext());
  assertThat(proposal.revisions().get(0).action()).isEqualTo(GoalRevisionAction.REVISE);
}

@Test
void invalidActionDefaultsToRevise() {
  String jsonResponse =
      "{\"revisions\": [{\"goalName\": \"g1\", \"action\": \"REMOVE\", "
          + "\"revisedDescription\": \"new\", \"revisionReason\": \"updated\"}], "
          + "\"rationale\": \"test\"}";
  LlmGoalRevisionStrategy strategy = strategyWithResponse(jsonResponse);

  GoalRevisionProposal proposal = strategy.revise(buildContext());
  assertThat(proposal.revisions().get(0).action()).isEqualTo(GoalRevisionAction.REVISE);
}

@Test
void systemPromptIncludesActionVocabulary() {
  // Verify the system prompt mentions ABANDON and COMPLETE
  String jsonResponse = "{\"revisions\": [], \"rationale\": \"ok\"}";
  ChatModel chatModel = mock(ChatModel.class);

  java.util.concurrent.atomic.AtomicReference<ChatRequest> capturedRequest =
      new java.util.concurrent.atomic.AtomicReference<>();
  when(chatModel.chat(any(ChatRequest.class))).thenAnswer(inv -> {
    capturedRequest.set(inv.getArgument(0));
    return ChatResponse.builder().aiMessage(AiMessage.from(jsonResponse)).build();
  });

  ChatModelProvider provider = new ChatModelProvider() {
    @Override public ModelType type() { return ModelType.OPENAI; }
    @Override public ChatModel get() { return chatModel; }
  };
  @SuppressWarnings("unchecked")
  Instance<ChatModelProvider> instance = mock(Instance.class);
  when(instance.isUnsatisfied()).thenReturn(false);
  when(instance.get()).thenReturn(provider);
  LlmGoalRevisionStrategy strategy = new LlmGoalRevisionStrategy(instance);

  strategy.revise(buildContext());

  String systemPrompt = capturedRequest.get().messages().get(0).text();
  assertThat(systemPrompt).contains("ABANDON");
  assertThat(systemPrompt).contains("COMPLETE");
  assertThat(systemPrompt).contains("REVISE");
}
```

Also update the existing `producesProposalFromLlmResponse` test to use the new 4-arg `RevisedGoal` constructor (it creates `RevisedGoal` indirectly via the parser, but the assertion will need to check the action field):

```java
@Test
void producesProposalFromLlmResponse() {
  String jsonResponse =
      "{\"revisions\": [{\"goalName\": \"g1\", \"revisedDescription\": \"better desc\", "
          + "\"revisionReason\": \"aligned with outcomes\"}], \"rationale\": \"test\"}";
  LlmGoalRevisionStrategy strategy = strategyWithResponse(jsonResponse);

  GoalRevisionContext context = buildContext();
  GoalRevisionProposal proposal = strategy.revise(context);

  assertThat(proposal.revisions()).hasSize(1);
  assertThat(proposal.revisions().get(0).goalName()).isEqualTo("g1");
  assertThat(proposal.revisions().get(0).action()).isEqualTo(GoalRevisionAction.REVISE);
  assertThat(proposal.revisions().get(0).revisedDescription()).isEqualTo("better desc");
  assertThat(proposal.rationale()).isEqualTo("test");
}
```

And update `nullDescriptionAllowed` — this test will fail because REVISE now requires a description. Change it to test ABANDON with null description instead:

```java
@Test
void abandonWithNullDescriptionAllowed() {
  String jsonResponse =
      "{\"revisions\": [{\"goalName\": \"g1\", \"action\": \"ABANDON\", "
          + "\"revisedDescription\": null, \"revisionReason\": \"no longer relevant\"}], "
          + "\"rationale\": \"ok\"}";
  LlmGoalRevisionStrategy strategy = strategyWithResponse(jsonResponse);

  GoalRevisionProposal proposal = strategy.revise(buildContext());
  assertThat(proposal.revisions().get(0).revisedDescription()).isNull();
  assertThat(proposal.revisions().get(0).action()).isEqualTo(GoalRevisionAction.ABANDON);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn install -pl api -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=LlmGoalRevisionStrategyTest -q`
Expected: Test failures (parser doesn't read `action` field, system prompt doesn't mention actions, null-description REVISE now throws)

- [ ] **Step 3: Update system prompt**

Replace the system prompt string in `LlmGoalRevisionStrategy.revise()`:

```java
.systemPrompt(
    "You are a goal effectiveness analyst. Given an agent's goals and their "
        + "performance metrics, evaluate each goal and recommend one of three actions:\n"
        + "- REVISE: refine the goal description to better capture what the agent "
        + "accomplishes. Only when meaningfully misaligned with observed outcomes.\n"
        + "- ABANDON: drop the goal entirely. Only when the goal is unachievable or "
        + "no longer relevant based on persistent failure patterns.\n"
        + "- COMPLETE: mark the goal as achieved. Only when the goal has been "
        + "consistently met and keeping it adds no further value.\n"
        + "Respond with JSON only.")
```

- [ ] **Step 4: Update buildPrompt to include action field in schema**

Replace the JSON schema instruction at the end of `buildPrompt()`:

```java
sb.append("\nRespond with JSON: {\"revisions\": [{\"goalName\": \"...\", ")
    .append("\"action\": \"REVISE|ABANDON|COMPLETE\", ")
    .append("\"revisedDescription\": \"...\"|null, \"revisionReason\": \"...\"}], ")
    .append("\"rationale\": \"...\"}");
```

- [ ] **Step 5: Update parseResponse to read action field with tolerance**

Replace the `parseResponse()` method:

```java
private GoalRevisionProposal parseResponse(String response) {
  try {
    JsonNode root = MAPPER.readTree(response);
    JsonNode revisionsNode = root.get("revisions");
    String rationale = root.has("rationale") ? root.get("rationale").asText() : "";

    List<GoalRevisionProposal.RevisedGoal> revisions = new ArrayList<>();
    if (revisionsNode != null && revisionsNode.isArray()) {
      for (JsonNode node : revisionsNode) {
        String goalName = node.get("goalName").asText();
        GoalRevisionAction action = parseAction(node);
        String desc =
            node.has("revisedDescription") && !node.get("revisedDescription").isNull()
                ? node.get("revisedDescription").asText()
                : null;
        String reason = node.get("revisionReason").asText();
        revisions.add(new GoalRevisionProposal.RevisedGoal(goalName, action, desc, reason));
      }
    }
    return new GoalRevisionProposal(revisions, rationale);
  } catch (Exception e) {
    throw new AgentException("Failed to parse LLM goal revision response", e);
  }
}

private GoalRevisionAction parseAction(JsonNode node) {
  if (!node.has("action") || node.get("action").isNull()) {
    return GoalRevisionAction.REVISE;
  }
  try {
    return GoalRevisionAction.valueOf(node.get("action").asText().toUpperCase());
  } catch (IllegalArgumentException e) {
    return GoalRevisionAction.REVISE;
  }
}
```

Add the import: `import io.casehub.api.spi.routing.GoalRevisionAction;`

- [ ] **Step 6: Run tests to verify they pass**

Run: `mvn install -pl api -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=LlmGoalRevisionStrategyTest -q`
Expected: All tests PASS

- [ ] **Step 7: Run full test suite for both modules**

Run: `mvn install -pl api -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -q`
Expected: All runtime tests PASS (no regressions)

- [ ] **Step 8: Commit**

```
git add runtime/src/main/java/io/casehub/engine/internal/routing/LlmGoalRevisionStrategy.java \
       runtime/src/test/java/io/casehub/engine/internal/routing/LlmGoalRevisionStrategyTest.java
git commit -m "feat(#903): update LlmGoalRevisionStrategy prompt and parsing for action enum"
```
